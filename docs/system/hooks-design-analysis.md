# Claude Code Hooks 设计模块深度解析

## 1. 模块总结介绍

Claude Code 的 `hooks/` 相关模块是整个系统中连接**用户界面（React/TUI）**、**Agent 执行循环**与**终端运行时**的核心黏合层。该模块不仅包含传统意义上的 React Hooks（UI 状态与副作用），还涵盖了一套完整的**生命周期钩子系统**（Hook Event System），允许在 Agent 会话的多个关键节点（如工具调用前后、会话启停、权限请求、Stop/StopFailure 等）注入自定义逻辑。

从代码分布来看，相关逻辑横跨：
- `src/hooks/`：UI 层 React Hooks（输入处理、通知、权限、模型选择等）
- `src/utils/hooks/`：底层 Agent/会话钩子引擎（注册、执行、异步管理、配置）
- `src/ink/hooks/`：终端专属的 Ink/React 基础 Hooks（stdin、focus、viewport、动画帧等）
- `src/components/hooks/`：钩子配置的 TUI 组件（菜单、对话框、模式选择）

本模块的设计目标是：在保持 UI 实时响应的前提下，让 Agent 行为具备可扩展、可拦截、可观测的特性，并确保钩子不会阻塞主循环或污染全局状态。

---

## 2. 系统架构

整体架构可以抽象为四层，由外到内分别是：

```
┌─────────────────────────────────────────────────────────────┐
│  UI Hooks Layer (src/hooks/)                                │
│  useCanUseTool, useTextInput, useVimInput, notifications    │
├─────────────────────────────────────────────────────────────┤
│  Terminal Hooks Layer (src/ink/hooks/)                      │
│  useInput, useStdin, useTerminalFocus, useAnimationFrame    │
├─────────────────────────────────────────────────────────────┤
│  Agent Hooks Layer (src/utils/hooks/)                       │
│  SessionHooks, PostSamplingHooks, AsyncHookRegistry         │
│  execAgentHook, execPromptHook, execHttpHook, hookEvents    │
├─────────────────────────────────────────────────────────────┤
│  Utility Hooks Layer (src/utils/hooks/)                     │
│  HooksConfigManager, HooksConfigSnapshot, fileChangedWatcher│
└─────────────────────────────────────────────────────────────┘
```

- **UI Hooks Layer** 直接服务于 React 组件，承担着键盘输入、权限确认队列、通知弹窗、命令键绑定等高频交互。
- **Terminal Hooks Layer** 是 Claude Code 基于 `ink` 的 TUI 基础能力，负责原始终端输入捕获、焦点报告、动画同步、选择复制等。
- **Agent Hooks Layer** 是 Agent 循环的扩展点，支持 `command`、`prompt`、`agent`、`http`、`function` 五类钩子，以及最近引入的 `postSamplingHooks` 和 `sessionHooks`。
- **Utility Hooks Layer** 提供配置治理（快照、策略、只允许 managed hooks 等）、文件变化监听、SSRFGuard、SandBox Proxy 路由等横切关注点。

---

## 3. 分层说明

### 3.1 UI Hooks Layer

该层包含大量以 `use` 开头的 React Hooks，重点关注几个核心文件：

#### `useCanUseTool.tsx`
这是整个系统中**工具权限决策**的入口 Hook。它返回一个异步函数 `canUseTool`，其内部流程如下：
1. 优先调用 `hasPermissionsToUseTool` 做配置层面的 allow/deny/ask 判定。
2. 如果行为为 `allow`，直接允许；若 `deny`，直接拒绝。
3. 若需要询问（`ask`），进入分层处理器：
   - `handleCoordinatorPermission`（Coordinator Worker）
   - `handleSwarmWorkerPermission`（Swarm Worker，通过 Mailbox 向 Leader 请求）
   - 弹窗交互的 `handleInteractivePermission`（主 Agent）

`handleInteractivePermission` 中使用了 **resolve-once** 竞态 guarding（`createResolveOnce`），确保用户点击、Bridge 远程响应、Classifier 自动批准、Channel 远程批复中只有一个能胜出，避免重复 resolve 或竞态条件。

代码片段（`PermissionContext.ts`）展示了这种原子 claim 机制：
```ts
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  return {
    claim() {
      if (claimed) return false
      claimed = true
      return true
    },
    resolve(value: T) { if (!claimed) return; /* ... */ },
  }
}
```

#### `useTextInput.ts` / `useVimInput.ts` / `useInputBuffer.ts`
这三者共同构成了命令行输入的编辑引擎：
- `useTextInput` 实现了类 Emacs 键绑定（Ctrl+A/E/K/Y、Meta+F/B、Kill Ring、Yank Pop），并处理了 SSH coalesced Enter、DEL 字符过滤、滚轮事件过滤、Paste 模式等复杂场景。
- `useVimInput` 在其之上封装了 Vim 模式状态机，引用 `src/vim/transitions.js` 和 `src/vim/operators.js`，实现 NORMAL/INSERT 切换、操作符待决模式（operator-pending）、`.` 重复录制。
- `useInputBuffer` 提供带防抖（debounce）的输入缓冲区与撤销（undo）能力。

#### Notifications (`hooks/notifs/`)
通知层采用统一的 `useNotifications` 上下文。`useStartupNotification` 提供了一个高阶抽象，确保 compute 函数仅在组件 mount 时运行一次，并且自动过滤远程模式（`getIsRemoteMode()`）。其他如 `useMcpConnectivityStatus`、`useAutoModeUnavailableNotification` 等都基于它构建，负责 MCP 服务器失败告警、速率限制提示、LSP 初始化状态等。

### 3.2 Terminal Hooks Layer (`src/ink/hooks/`)

这一层是 Ink（React for Terminal）基础设施的延伸。核心文件包括：

- **`use-input.ts`**：封装终端 raw mode，使用 `useLayoutEffect` 在 commit 阶段同步启用/禁用原始输入，避免 useEffect 延迟导致的字符回显泄露。它通过 `EventEmitter` 注册输入监听，并借助 `useEventCallback` 保持监听器引用稳定，确保 `stopImmediatePropagation()` 的优先级不被破坏。

- **`use-stdin.ts`** / **`use-app.ts`**：分别暴露 `StdinContext` 和 `AppContext`，供子组件获取 stdin 流或手动退出应用。

- **`use-terminal-focus.ts`**：读取终端焦点状态，依赖 DECSET 1004 focus reporting，失焦时返回 false，用于权限弹窗的 checkmark 停留时间计算（聚焦 3s / 失焦 1s）。

- **`use-animation-frame.ts`** / **`use-interval.ts`**：基于全局 `ClockContext` 实现统一动画时钟。所有动画（如 spinner）共享一个时钟，通过 `keepAlive` 订阅机制决定时钟是否滴答；当终端失焦或没有可见动画时，时钟自动 pause，减少 CPU 占用。

- **`use-selection.ts`**：提供全屏模式下的文本选择操作（复制、清除、shift anchor、滚屏捕获等），通过 `instances.get(process.stdout)` 获取唯一的 Ink 实例。

### 3.3 Agent Hooks Layer (`src/utils/hooks/`)

这是 Claude Code 扩展性最强的子系统。核心设计原则包括：

#### Session Hooks (`sessionHooks.ts`)
所有通过 frontmatter（skill/agent）或运行时 API 注册的钩子都被存储在 `AppState.sessionHooks`——一个 **`Map<string, SessionStore>`** 中。

关键设计决策：
> "Map (not Record) so .set/.delete don't change the container's identity... With Map: .set() is O(1), return prev means zero listener fires."

在高并发 workflow（如 `parallel()` 启动 N 个 schema-mode agent）时，这一设计将复杂度从 O(N²) 降至 O(1)，并且不会触发 store 的监听器通知。

`addFunctionHook` 支持内存中的 TypeScript callback 钩子，返回 `boolean | Promise<boolean>`，常用于验证（如 `registerStructuredOutputEnforcement`）。这种 `function` 类型的钩子**不能持久化到 settings.json**，仅存在于 session 作用域。

#### Post-Sampling Hooks (`postSamplingHooks.ts`)
与 settings.json 暴露的 hook event 不同，`postSamplingHooks` 是一组**纯内部 API**，供程序化注册。例如 `skillImprovement.ts` 利用它注册了一个在每次主采样完成后执行的回调，用以分析最近的用户消息，检测是否需要优化 skill 定义。

```ts
export type PostSamplingHook = (context: REPLHookContext) => Promise<void> | void
export function registerPostSamplingHook(hook: PostSamplingHook): void
```

执行时，它会依次调用所有已注册的 hook，捕获并记录异常，但**不因单个 hook 失败而中断主循环**。

#### Async Hook Registry (`AsyncHookRegistry.ts`)
当 hook 类型为 `command` 且声明了 `async: true` 时，命令会在后台异步执行，其输出通过 `AsyncHookRegistry` 进行管理。

核心流程：
1. `registerPendingAsyncHook`：启动 shell 命令，注册进度轮询（默认 15s 超时）。
2. `startHookProgressInterval`：每秒读取 stdout/stderr，通过 `hookEvents` 广播 progress 事件。
3. `checkForAsyncHookResponses`：在主循环的适当时机轮询已完成任务，解析 stdout 中的 JSON 行作为同步响应，返回给 Agent。
4. `finalizePendingAsyncHooks`：会话清理时统一回收（kill 未完成的、finalize 已完成的）。

#### Hook 执行器 (`exec*Hook.ts`)
Claude Code 为不同类型的钩子提供了专用执行器：
- `execPromptHook.ts`：调用轻量级模型（默认 Haiku），通过 prompt 让 LLM 判断条件是否满足，返回结构化 JSON `{"ok": true}` 或 `{"ok": false, "reason": "..."}`。
- `execAgentHook.ts`：启动一个**子 Agent**（`querySource: 'hook_agent'`），赋予其 `SyntheticOutputTool`，最多允许 50 轮对话，用于执行复杂的 Stop/Verify 条件检查。
- `execHttpHook.ts`：通过 axios POST 发送 JSON，支持 sandbox proxy 路由、URL allowlist、env var 插值（仅限 `allowedEnvVars` 中的变量）、SSRFGuard 等安全措施。

#### Hook Events (`hookEvents.ts`)
一个独立的事件总线，用于向 SDK 或外部消费者广播 hook 生命周期（started / progress / response）。它支持 pending 队列（最多 100 个）， handler 未注册时先缓存，注册后一次性 flush。默认始终发射 `SessionStart` 和 `Setup`，其余事件需要开启 `includeHookEvents`。

### 3.4 Utility Hooks Layer

#### `hooksConfigManager.ts` / `hooksConfigSnapshot.ts`
这两套模块治理 hook 配置的加载与策略约束：
- `hooksConfigSnapshot` 在启动时捕获一次配置快照，并支持 `allowManagedHooksOnly`、`disableAllHooks` 等策略设置。
- `hooksConfigManager` 则负责将配置按 Event 和 Matcher 分组、排序（按 source 优先级：User > Project > Local > Plugin），并提供 UI 展示用的 display string。

#### `fileChangedWatcher.ts`
通过 `chokidar` 监听文件变化，触发 `FileChanged` 和 `CwdChanged` hooks：
- 支持 matcher 中静态路径（如 `.envrc|.env`）与 hook 输出中动态返回的 `watchPaths` 混合监听。
- 当 CWD 改变时，清理旧环境文件、执行 `CwdChanged` hooks、重新解析 watch list。
- hook 可以通过写入 `CLAUDE_ENV_FILE` 来影响后续 BashTool 的环境变量。

#### `ssrfGuard.ts`
`execHttpHook` 在无 proxy 时使用 `ssrfGuardedLookup` 作为 axios 的 custom lookup，阻止对私有/链路本地地址的直接请求。

---

## 4. 交互流程

以一个典型的**工具权限请求**为例，说明 hooks 如何协调各层系统：

```
1. Agent 循环调用 tool → 触发 toolUseContext
        ↓
2. useCanUseTool 生成的 canUseTool 被调用
        ↓
3. hasPermissionsToUseTool 返回 ask
        ↓
4. 检查是否 coordinator / swarm worker
        ├─ coordinator: 顺序执行 PermissionRequest hooks → Classifier → 若仍 ask 则 fallthrough
        ├─ swarm worker: 尝试 classifier → 否则 mailbox 请求 leader
        └─ main agent: handleInteractivePermission
                ↓
5. 将 ToolUseConfirm 推入 React 状态队列 (UI 层)
        ↓
6. 同时启动多个异步竞态:
   a) PermissionRequest hooks (异步执行，可能提前 resolve)
   b) Bash Classifier (若适用)
   c) REPL Bridge (CCR 远程批复, feature BRIDGE_MODE)
   d) Channel Permission Relay (Telegram/iMessage 等, feature KAIROS)
        ↓
7. 任一来源通过 createResolveOnce.claim() 获胜
        ↓
8. 取消其余来源，更新 queue/permissions，继续 tool 执行
```

再以 **Skill Improvement** 为例：
1. `initSkillImprovement()` 注册 `PostSamplingHook`。
2. 主线程每次 sampling 完成后，该 hook 检查是否积累了 5 条新 user 消息。
3. 若满足条件，调用 LLM（Haiku）分析对话，提取 `<updates>` 标签中的 skill 修改建议。
4. 将建议写入 `AppState.skillImprovement`，供 UI 展示“是否更新 skill”的提示。
5. 若用户确认，调用 `applySkillImprovement`，再次使用 LLM 重写 `.claude/skills/<name>/SKILL.md`。

---

## 5. 技术原理

### 5.1 React Compiler Memoization
观察 `useCanUseTool.tsx` 和 `useCommandKeybindings.tsx` 的头行为，代码中大量使用了：
```ts
import { c as _c } from "react/compiler-runtime";
const $ = _c(N);
```
这是 React Compiler（原 React Forget）自动编译的产物，通过编译时缓存数组减少运行时依赖比较。TUI 场景中输入事件极频繁，memoization 对保持 60fps 级别的响应至关重要。

### 5.2 AbortSignal 与资源清理
几乎每一个涉及异步 I/O 的 hook 都接受 `AbortSignal`，并通过 `createCombinedAbortSignal` 与超时信号合并。这种“信号合成”模式确保了：
- 会话结束 → 所有pending hooks 被优雅取消。
- 超时到达 → hook 返回 `cancelled` outcome，不会抛未处理异常。
- Agent 被 abort → 子 Agent hook 同步收到 abort 事件。

### 5.3 Map 身份稳定优化
`sessionHooks.ts` 使用 `Map` 而非 `Record`/`Object` 并且 `return prev` 的显式约定，利用了状态管理库（如 Zustand）的 `Object.is` 短路机制。避免了高并发场景下的不必要重渲染。

---

## 6. 创新点

1. **多层权限竞态模型**：在主 Agent 中，UI 弹窗、本地 hooks、远程 bridge、channels、classifier 五种来源通过统一的 `ResolveOnce` 安全竞争，既保证用户体验不被阻塞，又防止重复 resolve。

2. **Agent 作为 Hook**：`execAgentHook` 开创性地用“子 Agent”执行 Stop/Verify 条件， allowing 50 轮工具调用与结构化输出强制执行，让 hook 从简单的脚本/提示升级为具备完整思考-行动循环的 mini-agent。

3. **Post-Sampling Hook 与 Skill 自进化**：将采样后的对话上下文暴露给内部 hook，让 Claude Code 能基于近期交互自动提出 Skill 改进建议，形成了“执行-反馈-迭代”的闭环。

4. **统一时钟驱动的 TUI 动画**：`ClockContext` 让所有动画和计时器共享一个调度源，终端失焦时自动休眠，大幅降低后台功耗。

5. **Function Hook 的零序列化验证**：在 session 内存中直接注册 TypeScript callback，绕过所有 I/O 与序列化开销，为实时校验（如结构化输出强制）提供了极低延迟的扩展点。

---

## 7. 关键技术

| 技术点 | 实现位置 | 作用 |
|--------|---------|------|
| `createResolveOnce` | `PermissionContext.ts` | 竞态决议的原子 guarding |
| `PostSamplingHook` | `postSamplingHooks.ts` | 程序化的主循环扩展点 |
| `SessionHooksState = Map<string, SessionStore>` | `sessionHooks.ts` | 高并发下的 O(1) 状态更新 |
| `useEventCallback` + `useLayoutEffect` | `use-input.ts` | 稳定监听器引用与同步 raw mode |
| `ClockContext.subscribe(keepAlive)` | `use-animation-frame.ts` | 统一动画时钟与休眠策略 |
| `ssrfGuardedLookup` | `ssrfGuard.ts` | HTTP hook 的内网防护 |
| `registerStructuredOutputEnforcement` | `hookHelpers.ts` | 子 Agent 的结构化输出强制 |
| `fileChangedWatcher` (chokidar) | `fileChangedWatcher.ts` | 动态 env 与文件变更触发 hooks |
| `AsyncHookRegistry` | `AsyncHookRegistry.ts` | 异步命令的生命周期与进度管理 |
| `React Compiler memoization` | 多个 UI hooks 文件 | 高频输入下的渲染性能保障 |

---

## 8. 思考总结

Claude Code 的 Hooks 设计模块体现了**“UI 与 Agent 循环深度耦合但边界清晰”**的工程哲学：
- **边界清晰**：通过 `ToolUseContext` 和 `AppState` 传递上下文，hooks 不直接调用 UI 渲染，而是通过状态队列（`toolUseConfirmQueue`）和事件总线（`hookEvents`）间接驱动。
- **深度耦合**：权限系统、输入编辑、模型选择、通知体系等业务逻辑都被抽象为 hooks，使组件树保持扁平，业务逻辑可测且可复用。
- **扩展优先**：从 `command` 到 `prompt` 再到 `agent` 乃至 `http` 和 `function`，hooks 的执行能力不断演进，为第三方插件、企业策略、自动化流水线预留了充足空间。
- **安全与治理并重**：`hooksConfigSnapshot` 和 policy settings 的引入，以及 `ssrfGuard`、`sandbox proxy`、`allowedEnvVars` 等机制，说明该模块已从“功能扩展”走向了“生产级治理”。

未来值得关注的发展方向：
- `postSamplingHooks` 是否会向用户暴露为 settings.json 配置项。
- `function` hooks 与 TypeScript Sandbox 的结合，可能成为轻量级插件的新形态。
- `ClockContext` 和 `useAnimationFrame` 的模式是否会进一步推广到非 TUI 场景（如 Web UI 的共享 RAF 调度器）。
