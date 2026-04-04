# Claude Code CLI 与命令系统技术解析

## 1. 模块总结介绍

CLI & Commands System 是 Claude Code 的**入口门户与交互控制中心**，负责将用户的键盘输入、CLI 参数、远程消息统一转换为可执行动作。该模块横跨四个核心子系统：

- **CLI 入口层 (`entrypoints/cli.tsx`, `main.tsx`)**：处理进程启动参数、初始化全局状态、解析 Commander.js 命令行。
- **命令注册中心 (`commands.ts`, `types/command.ts`)**：统一管理内置命令、插件命令、Skill 命令、MCP 命令与工作流命令的发现、加载与权限控制。
- **命令实现层 (`commands/`)**：每个子目录对应一个斜杠命令（slash command），支持 `prompt`、`local`、`local-jsx` 三种执行形态。
- **键盘与对话框层 (`keybindings/`, `dialogLaunchers.tsx`)**：提供终端级的快捷键体系、和弦键（chord bindings）支持，以及 JSX 对话框的动态渲染入口。

该模块的设计哲学是**"渐进加载、按需执行、严格分层"**：启动路径被裁剪到极致（`--version` 零模块加载），而复杂的交互命令通过动态 import 延后加载，确保 TUI 启动速度不受功能膨胀影响。

---

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         User Input Layer                            │
│  (Terminal keystrokes / CLI args / Remote control messages)         │
└──────────────────┬────────────────────────────────┬─────────────────┘
                   │                                │
     ┌─────────────▼─────────────┐    ┌────────────▼────────────┐
     │   Commander.js (main.tsx) │    │   Keybinding System     │
     │   - parse argv            │    │   (keybindings/)        │
     │   - subcommands (mcp,     │    │   - chord support       │
     │     plugin, auth, etc.)   │    │   - context resolution    │
     │   - default action → REPL │    │   - hot-reload            │
     └─────────────┬─────────────┘    └────────────┬────────────┘
                   │                               │
     ┌─────────────▼───────────────────────────────▼─────────────┐
     │              Command Registry (commands.ts)               │
     │   • Built-ins  • Skills  • Plugins  • MCPs  • Workflows   │
     │   • Availability gating  • Remote filtering  • Dedup      │
     └──────────────────────┬────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
   │   prompt    │   │   local     │   │  local-jsx  │
   │  (model)    │   │  (headless) │   │   (TUI)     │
   └─────────────┘   └─────────────┘   └─────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
               ┌────────────▼────────────┐
               │   Dialog Launchers      │
               │   (dialogLaunchers.tsx) │
               │   - SnapshotUpdateDlg   │
               │   - ResumeChooser       │
               │   - Teleport dialogs    │
               └─────────────────────────┘
```

---

## 3. 分层说明

### 3.1 CLI Layer (`entrypoints/cli.tsx` & `main.tsx`)

`entrypoints/cli.tsx` 是**最轻量的启动闸门**。它通过纯 `process.argv` 检查实现多条快速路径（fast-path），在加载任何重型模块之前即返回：

```ts
// Fast-path for --version/-v: zero module loading needed
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v' || args[0] === '-V')) {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;
}
```

其他快速路径包括 `--dump-system-prompt`、`--daemon-worker`、`remote-control` (`bridge`)、`daemon`、`ps/logs/attach/kill`、模板任务、环境运行器等。这种设计让子命令的启动成本与主 TUI 解耦。

`main.tsx` 是真正的主程序。它使用 **Commander.js** (`@commander-js/extra-typings`) 构建命令树，并通过 `program.hook('preAction', ...)` 统一执行初始化（`init()`、迁移、遥测、远程策略限制加载等）。所有参数定义集中在 `main.tsx` 中，例如 `-p/--print`（非交互模式）、`--bare`（最小化模式）、`--settings`（外部配置注入）等。

### 3.2 Command Registry Layer (`commands.ts`, `types/command.ts`)

`commands.ts` 是所有命令的**单一事实来源**。它通过显式 import 构建内置命令数组，并结合动态 `require()` 实现特性标志（feature flag）死码消除（dead code elimination）：

```ts
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default
  : null
```

核心数据结构定义在 `types/command.ts`，命令分为三类：

| 类型 | 说明 | 典型用例 |
|------|------|----------|
| `prompt` | 扩展为文本提示词发送给模型 | `/commit`, `/diff`, `/review` |
| `local` | 本地同步/异步执行，返回文本结果 | `/cost`, `/clear` |
| `local-jsx` | 渲染 Ink/React 组件的交互式命令 | `/help`, `/skills`, `/context` |

注册中心暴露以下关键 API：

- `getCommands(cwd)`：聚合内置命令 + Skill 命令 + 插件命令 + 工作流命令 + MCP 命令。
- `meetsAvailabilityRequirement(cmd)`：基于 `claude-ai` / `console` 认证类型过滤命令可见性。
- `filterCommandsForRemoteMode(commands)`：仅保留 `REMOTE_SAFE_COMMANDS`（如 `exit`, `help`, `theme`）。
- `getSkillToolCommands(cwd)` / `getSlashCommandToolSkills(cwd)`：为模型提供可调用的 Skill 清单。
- `isBridgeSafeCommand(cmd)`：控制移动端/Bridge 场景下哪些 slash 命令可安全执行。

### 3.3 Command Implementation Layer (`commands/`)

每个命令通常是一个极简的元数据文件 `index.ts`（或 `index.js`），将重型实现延迟加载：

```ts
// commands/clear/index.ts
const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history and free up context',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false,
  load: () => import('./clear.js'),  // lazy load
} satisfies Command
```

`prompt` 类型命令（如 `/commit`）则利用 `getPromptForCommand` 动态生成带嵌入式 shell 指令的提示词，并通过 `executeShellCommandsInPrompt` 在本地执行 git 状态收集后再提交给模型。这实现了**"本地感知 + 模型推理"**的混合工作流。

部分命令存在**双形态实现**：例如 `context` 同时提供 `context`（TUI 的 `local-jsx` 版本）与 `contextNonInteractive`（headless 的 `local` 版本），根据 `getIsNonInteractiveSession()` 条件启隐。

### 3.4 UI Layer (`keybindings/` & `dialogLaunchers.tsx`)

`dialogLaunchers.tsx` 是一组**薄封装函数**，将原本内联在 `main.tsx` 中的 JSX 弹窗调用抽离为可复用的异步 launcher，例如 `launchResumeChooser`、`launchTeleportResumeWrapper`、`launchSnapshotUpdateDialog`。它们统一通过 `showSetupDialog` 或 `renderAndRun` 挂载到 Ink 根节点上，避免 `main.tsx` 直接耦合 React 组件。

`keybindings/` 子系统提供了完整的键盘交互框架：

- **`defaultBindings.ts`**：定义默认键位（平台相关的 `IMAGE_PASTE_KEY`、`MODE_CYCLE_KEY`）。
- **`parser.ts`**：解析 `"ctrl+k ctrl+s"` 风格字符串为结构化 `ParsedKeystroke[]`。
- **`resolver.ts`**：纯函数解析器，支持单键与和弦键（chord）匹配。
- **`loadUserBindings.ts`**：读取 `~/.claude/keybindings.json`，支持热重载（chokidar + 防抖）。
- **`KeybindingProviderSetup.tsx`**：组合 Context Provider + `ChordInterceptor`，管理与弦状态机与超时（1000ms）。
- **`useKeybinding.ts`**：React hook，供组件注册 action handler。

---

## 4. 交互流程

### 4.1 斜杠命令（Slash Command）执行流程

1. **输入检测**：REPL 中的用户输入以 `/` 开头时，由 `PromptInput` 或外层 parser 识别为 slash command。
2. **命令查找**：调用 `findCommand(name, await getCommands(cwd))`，支持 `name` 与 `aliases` 匹配。
3. **类型分发**：
   - **prompt**：执行 `cmd.getPromptForCommand(args, toolUseContext)`，生成的 `ContentBlockParam[]` 作为新的用户消息追加到对话中，随后触发模型查询。
   - **local**：执行 `await (await cmd.load()).call(args, context)`，结果可能是文本、compact 结果或 skip。
   - **local-jsx**：执行 `await (await cmd.load()).call(onDone, context, args)`，返回 React 节点挂载到 TUI。
4. **生命周期回调**：`local`/`local-jsx` 通过 `onDone` 回调控制后续行为（是否继续查询模型、如何展示结果）。

### 4.2 CLI 参数到执行动作的转换流程

1. **入口分流**：`cli.tsx` 判断 `--version`、子命令（`daemon`、`remote-control`）或进入 `main.tsx`。
2. **Commander 解析**：`main.tsx` 中 `program.action(async (prompt, options) => { ... })` 捕获所有参数。
3. **preAction 钩子**：触发 `init()` 初始化全局状态（auth、telemetry、config）、应用迁移、加载远程策略。
4. **模式分支**：
   - **交互模式**：调用 `setup()` 构建 `AppState`，最终 `launchRepl(...)` 启动 TUI。
   - **非交互模式 (`-p`)**：调用 `runHeadless(...)`，通过 `print.ts` 将模型输出发送到 stdout 后退出。
   - **SDK 模式 (`--sdk-url`)**：构建 `RemoteIO` → `StructuredIO` 循环，与远程会话 ingress 保持双向通信。
5. **命令原语映射**：CLI 选项（如 `--allowedTools`、`--model`、`--permission-mode`）被转换为 `AppState` 或 `ToolUseContext` 中的字段，供后续命令查询使用。

---

## 5. 技术原理

### 5.1 Commander.js 与参数隔离策略

Claude Code 使用单一 Commander 实例承载所有参数。关键技巧：

- **`.enablePositionalOptions()`**：允许位置参数选项独立解析。
- **`preAction` 钩子**：只在真正执行动作时初始化，避免 `--help` 等场景触发重型 `init()`。
- **eager settings parsing**：`--settings` 与 `--setting-sources` 在 `preAction` 之前被手动提取（`eagerParseCliFlag`），以便在 `init()` 运行前即加载配置。
- **参数重写**：对于 `claude ssh <host>` 或 `claude assistant [sessionId]`，预处理逻辑会从 `process.argv` 中剥离子命令，将其改写为普通参数，确保默认 action 接管。

### 5.2 命令加载的 Memoization 与缓存失效

`commands.ts` 大量使用 `lodash-es/memoize` 缓存命令列表，因为 Skill/Plugin/Workflow 的磁盘扫描成本高昂：

```ts
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

但可用性检查（`meetsAvailabilityRequirement`、`isCommandEnabled`）不缓存，因为用户可能在会话中途登录（`/login`），状态变化必须立即生效。缓存失效 API（`clearCommandMemoizationCaches`、`clearCommandsCache`）在动态 Skill 添加或插件变更时被调用。

### 5.3 Remote Mode 命令过滤

远程模式（`--remote` 或 CCR bridge）下本地文件系统访问受限。`REMOTE_SAFE_COMMANDS` 使用 `Set<Command>` 做白名单精确到模块引用级别：

```ts
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage, ...
])
```

`filterCommandsForRemoteMode` 在 `main.tsx` 的 REPL 渲染前执行，防止本地命令在 CCR 初始化完成前短暂可见（race condition）。Bridge 场景下另有 `BRIDGE_SAFE_COMMANDS` 控制从移动端传入的 slash 命令。

### 5.4 和弦键（Chord）解析与状态机

`resolver.ts` 中的 `resolveKeyWithChordState` 实现了一个纯函数状态机：

- **Phase 1**：判断当前按键是否为某个更长 chord 的前缀，若是则进入 `chord_started` 状态。
- **Phase 2**：若当前 chord 序列与某个 binding 精确匹配，返回 `match`。
- **Phase 3**：若 chord 长度不匹配且无更长前缀，则 `chord_cancelled`。

`KeybindingProviderSetup.tsx` 维护 `pendingChordRef`（同步读取）与 `pendingChord` state（触发重渲染），并通过 `setTimeout(CHORD_TIMEOUT_MS)` 自动取消超时未完成的 chord。

### 5.5 Dialog Launchers 的异步模型

所有 dialog launcher 返回 `Promise<T>`，内部通过 `showSetupDialog(root, done => <Component ... onComplete={done} />)` 将 React 渲染与异步等待统一起来。该模式让 `main.tsx` 能用顺序代码表达交互流程：

```ts
const choice = await launchSnapshotUpdateDialog(root, { agentType, scope, snapshotTimestamp })
```

---

## 6. 创新点

1. **Fast-path 零加载启动**：`cli.tsx` 在 `init()` 之前即拦截 `--version`、子命令、daemon worker 等路径，最大限度降低冷启动开销。
2. **三态命令模型**：`prompt` / `local` / `local-jsx` 的划分让同一命令接口既能驱动模型、又能做纯本地计算、也能渲染富交互 TUI，扩展性极强。
3. **引用级安全白名单**：`REMOTE_SAFE_COMMANDS` 不是按字符串过滤，而是直接 `Set<Command>` 按模块对象比较，避免名称重名或别名绕过。
4. **和弦键热重载**：终端应用通常将键位硬编码，Claude Code 却提供了类 VS Code 的 `keybindings.json` 配置 + 文件监控热重载，且通过 `ChordInterceptor` 保证和弦前缀不会泄露到输入框。
5. **延迟加载的 Skill 与工作流生态**：命令注册中心将内置命令、文件系统 Skill、插件、MCP、工作流平滑地归并为同一 `Command` 类型，用户无感知地扩展功能。

---

## 7. 关键技术

- **Commander.js** (`@commander-js/extra-typings`)：类型安全的命令行解析与 help 生成。
- **Ink / React**：TUI 渲染基础，`useInput` 与自定义 keybinding hook 构建交互层。
- **lodash-es/memoize**：命令列表与名称集合的启动级缓存。
- **chokidar**：`keybindings.json` 的本地文件热重载监听。
- **Bun bundle DCE**：通过 `feature()` 编译期标志配合 `require()` 条件导入，将内测功能完全从外部构建中剔除。
- **Zod**：`schema.ts` 中使用 Zod 定义 `keybindings.json` 的验证模式与 JSON Schema 导出。

---

## 8. 思考总结

Claude Code 的 CLI & Commands System 是一个**高度工程化**的入口控制层，其设计处处体现对产品形态（CLI TUI + headless SDK + remote bridge）的深刻适配：

- **启动速度优先**：从 `cli.tsx` 的快速路径到命令的 `load()` 懒加载，每一层都在避免"功能越多启动越慢"的陷阱。
- **安全与隔离并重**：remote mode 和 bridge mode 的双层白名单、命令级别的 `availability` 与 `isEnabled` 分离，让权限控制既有粗粒度也有细粒度。
- **扩展性内建**：Skill、Plugin、Workflow、MCP 四种外部扩展以同一 `Command` 接口接入，未来新增命令来源无需改动核心解析逻辑。
- **终端 UX 的精致打磨**：和弦键、平台相关快捷键回退（Windows VT mode 检测）、热重载配置，这些在纯终端工具中极为罕见，体现了对开发者体验的长期投入。

一个值得注意的工程权衡是：代码中大量使用**顶层 side-effect import**（如 `import '../bootstrap/state.js'`）和**显式条件 require** 来管理初始化顺序与 bundle size。这种模式在 Bun 的单文件编译环境下工作良好，但在迁移到其他 bundler 或 Node 运行时时需要格外注意 tree-shaking 与执行顺序的兼容性。
