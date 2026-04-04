# CLI Commands 深度技术思考

## 一、CLI 的架构哲学

### 1.1 启动速度作为核心指标

Claude Code 将「启动速度」视为 CLI 工具的核心 UX 指标：

```
用户感知时间线：

0ms ─────▶ 进程启动
   │
   ├── 1ms: cli.tsx 快速路径判断
   │         ├── --version? → 直接输出，零模块加载
   │         ├── daemon? → 轻量分支
   │         └── 否则 → 进入 main.tsx
   │
   ├── 50ms: Commander.js 参数解析
   │
   ├── 100ms: init() 初始化
   │         ├── 认证状态
   │         ├── 配置加载
   │         └── 遥测设置
   │
   └── 200ms: TUI 首次渲染
```

这种「分层启动」确保了用户在最短时间内看到反馈。

### 1.2 Fast-path 的设计模式

```typescript
// entrypoints/cli.tsx
// 在加载任何重型模块之前，先检查快速路径

if (args.length === 1 && 
    (args[0] === '--version' || args[0] === '-v')) {
  console.log(`${MACRO.VERSION} (Claude Code)`)
  return  // 直接退出，零依赖加载
}
```

这种模式的关键是：**在 `import` 重型模块之前，先用纯 `process.argv` 做判断**。这让 `--version` 等简单命令的启动成本接近于零。

## 二、命令系统的类型美学

### 2.1 三态命令模型的语义区分

```
┌─────────────┬──────────────────┬──────────────────┬──────────────┐
│    类型     │      语义        │     执行环境      │    典型用例   │
├─────────────┼──────────────────┼──────────────────┼──────────────┤
│   prompt    │ 「问模型」        │ LLM 推理         │ /commit, /diff│
├─────────────┼──────────────────┼──────────────────┼──────────────┤
│   local     │ 「本地计算」      │ 同步/异步 JS     │ /cost, /clear │
├─────────────┼──────────────────┼──────────────────┼──────────────┤
│  local-jsx  │ 「交互界面」      │ React/Ink 渲染   │ /help, /skills│
└─────────────┴──────────────────┴──────────────────┴──────────────┘
```

这种区分不是随意的，而是对应了三种根本不同的「计算范式」：
- **prompt**：需要模型推理，结果不确定
- **local**：确定性计算，结果可预测
- **local-jsx**：需要用户交互，状态复杂

### 2.2 命令作为纯元数据

```typescript
// commands/clear/index.ts
const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false,
  load: () => import('./clear.js'),  // 延迟加载
} satisfies Command
```

命令定义是「纯数据」，实际实现通过 `load()` 延迟加载。这种「声明-实现分离」使得：
- 命令注册表可以快速扫描所有命令
- 实际代码只在执行时加载
- 支持热更新（重新加载模块）

## 三、渐进加载的工程智慧

### 3.1 Memoization 的精确控制

```typescript
// 命令列表缓存（昂贵操作）
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  // 扫描 Skill/Plugin/Workflow/MCP
  // 磁盘 I/O + 配置解析
})

// 可用性检查不缓存（状态变化敏感）
function meetsAvailabilityRequirement(cmd: Command): boolean {
  // 检查用户登录状态
  // 必须在运行时实时判断
}
```

缓存策略的区分体现了「数据稳定性」的分析：
- 命令列表变化频率低 → 缓存
- 用户登录状态变化频率高 → 不缓存

### 3.2 特性标志的死码消除

```typescript
// 通过 feature() 编译期标志
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default
  : null

// Bun bundle 时，如果 feature 为 false
// 整个 proactive.js 模块会被 DCE (Dead Code Elimination)
```

这种「条件 require」配合 Bun 的 bundler，实现了真正的零成本抽象——未启用特性的代码不会出现在最终 bundle 中。

## 四、安全模型的分层防御

### 4.1 Remote Mode 的白名单哲学

```typescript
// 不是「黑名单禁止危险命令」
// 而是「白名单仅允许安全命令」

export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session,  // 引用级精确匹配
  exit,
  clear,
  help,
  // ... 明确允许
])
```

这种「fail-closed」设计确保了：
- 新增命令默认不安全
- 必须通过显式添加才能远程执行
- 避免了「遗漏黑名单」的安全漏洞

### 4.2 引用级 vs 字符串级匹配

```typescript
// 不安全的字符串匹配：
if (command.name !== 'dangerous') {
  // 允许执行
}
// 绕过方式：改名、别名

// 安全的引用级匹配：
if (!REMOTE_SAFE_COMMANDS.has(command)) {
  throw new Error('Command not allowed in remote mode')
}
// 绕过方式：无（必须是同一个模块对象）
```

使用 `Set<Command>` 而非字符串比较，防止了「名字伪装」攻击。

## 五、和弦键的交互创新

### 5.1 和弦解析的状态机

```typescript
// resolver.ts 中的纯函数状态机

resolveKeyWithChordState(pressedKey, currentChord, bindings):
  
  // Phase 1: 是否有以 currentChord + pressedKey 为前缀的绑定？
  if (hasPrefix(bindings, currentChord + pressedKey)) {
    return { status: 'chord_started', pendingChord: [...currentChord, pressedKey] }
  }
  
  // Phase 2: 是否有精确匹配？
  if (hasExactMatch(bindings, currentChord + pressedKey)) {
    return { status: 'match', action: getAction(...) }
  }
  
  // Phase 3: 和弦取消
  return { status: 'chord_cancelled' }
```

这种纯函数设计使得：
- 状态转换可预测、可测试
- 无副作用，便于并发执行
- 易于实现撤销/重做

### 5.2 ChordInterceptor 的输入保护

```
用户按下 Ctrl+K
      │
      ▼
┌─────────────┐
│ChordInterceptor
└──────┬──────┘
       │
       ├── 等待下一个按键（1000ms 超时）
       │
       ├── 如果 Ctrl+S ──▶ 保存快捷键，拦截输入
       │
       └── 如果超时或其他键 ──▶ 释放到输入框
```

`ChordInterceptor` 确保了和弦前缀不会「泄露」到输入框中——这是终端应用中常见的 UX 问题。

### 5.3 热重载的工程实现

```
用户修改 ~/.claude/keybindings.json
      │
      ▼
┌─────────────┐
│  chokidar   │  文件监听
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  debounce   │  防抖 100ms
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 重新加载配置 │  验证 + 生效
└─────────────┘
```

无需重启应用，键位配置即时生效——这在传统终端工具中极为罕见。

## 六、对话框的异步封装

### 6.1 Dialog Launcher 模式

```typescript
// 将 React 渲染转换为 Promise

export function launchResumeChooser(root: InkRoot, options: Options): Promise<Choice> {
  return new Promise((resolve) => {
    renderAndRun(root, (onDone) => (
      <ResumeChooser
        {...options}
        onComplete={(choice) => {
          onDone()  // 卸载组件
          resolve(choice)  // 解析 Promise
        }}
      />
    ))
  })
}

// 使用方可以用同步风格写异步逻辑
const choice = await launchResumeChooser(root, options)
// 用户选择后才会继续执行
```

这种「异步 launcher」模式让复杂的交互流程可以用线性的代码表达。

### 6.2 对话框栈的管理

```
┌─────────────────┐
│   Dialog Stack  │
├─────────────────┤
│  SnapshotDlg    │  ← 顶层，有焦点
├─────────────────┤
│  PermissionDlg  │  ← 被遮挡，等待
├─────────────────┤
│  Main TUI       │  ← 暂停处理
└─────────────────┘
```

对话框栈确保了「模态」语义——后面的交互必须等待前面完成后才能继续。

## 七、CLI 到 TUI 的渐进增强

### 7.1 双模命令的实现

```typescript
// context 命令的双态实现

// TUI 模式
export const context: Command = {
  type: 'local-jsx',
  name: 'context',
  load: () => import('./context-ui.js'),
}

// Headless 模式
export const contextNonInteractive: Command = {
  type: 'local',
  name: 'context',
  load: () => import('./context-cli.js'),
}

// 运行时选择
const cmd = getIsNonInteractiveSession() 
  ? contextNonInteractive 
  : context
```

同一功能提供两种交互形态，根据运行环境自动选择。

### 7.2 Headless 模式的输出契约

```
交互模式 (-p):
  输出: 格式化文本 + ANSI 颜色
  交互: 可能触发二次确认
  退出: 会话结束后

非交互模式:
  输出: 纯文本，stdout
  交互: 无，所有决策使用默认值
  退出: 单轮查询后
```

Headless 模式的设计遵循「Unix 哲学」——输出可被管道处理，适合脚本集成。

## 八、反思与演进

### 8.1 当前架构的权衡

**顶层 Side-effect Import 的风险**

```typescript
// 这种模式在代码中常见
import '../bootstrap/state.js'  // 执行初始化

// 优点：简洁，确保执行顺序
// 缺点：隐式依赖，难以 tree-shake
```

在 Bun 环境下工作良好，但迁移到其他 bundler 时需要特别注意。

**命令发现的性能瓶颈**

```
getCommands(cwd) 需要扫描：
  ├── 内置命令
  ├── ~/.claude/skills/ 目录
  ├── 插件目录
  ├── MCP 配置
  └── 工作流定义
```

虽然使用了 memoization，但在大型工作区首次加载时仍有延迟。

### 8.2 未来演进方向

**命令预加载**

预测用户可能使用的命令，后台预加载，减少执行延迟。

**图形化配置界面**

为 `keybindings.json` 和命令配置提供 Web UI，降低配置门槛。

**命令组合语法**

支持 `claude /commit && /push` 这样的链式调用，或 `claude /diff file1 file2 | grep function` 的管道集成。

**Natural Language 命令**

允许用户输入自然语言描述，系统自动匹配最合适的命令组合。

---

Claude Code 的 CLI Commands 模块是一个将「传统 CLI 工具」与「现代 TUI 应用」融合的典范。它通过 fast-path 优化确保了启动速度，通过三态命令模型提供了灵活的扩展框架，通过分层安全模型实现了 fail-closed 的安全保障。理解其设计，对于构建任何需要同时支持交互式和脚本化使用的 CLI 工具都具有重要参考价值。
