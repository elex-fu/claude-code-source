# 第 7 章：状态管理设计

> 状态是复杂系统的根源，也是复杂系统的必需品。

---

## 7.1 为什么状态管理很难

在 AI Agent 系统中，状态管理面临独特的挑战：

- **并发**：多个工具可能同时执行，都需要读写状态
- **异步**：工具执行是异步的，状态更新需要线程安全
- **持久化**：会话中断后需要恢复状态
- **可观察性**：UI 需要实时反映状态变化
- **一致性**：状态变更需要原子性，防止部分更新

Claude Code 用一套精心设计的状态系统解决了这些问题。

---

## 7.2 两层状态架构

![两层状态架构](../diagrams/state-management.svg)

**Bootstrap State** 是进程级别的全局单例，存储跨会话的持久信息。

**AppState** 是会话级别的 React 状态树，存储当前会话的所有动态信息。

---

## 7.3 Bootstrap State：全局单例

`src/bootstrap/state.ts` 是整个系统的"地基"，注释明确写道：

```typescript
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
```

这个警告很重要。全局状态是复杂性的来源，应该尽量少用。

Bootstrap State 存储的内容：

```typescript
type State = {
  // 路径信息
  originalCwd: string          // 启动时的工作目录
  projectRoot: string          // 项目根目录（稳定，不随 worktree 变化）
  cwd: string                  // 当前工作目录（可变）

  // 费用追踪
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number

  // 每轮统计
  turnHookDurationMs: number
  turnToolDurationMs: number
  turnClassifierDurationMs: number
  turnToolCount: number
  turnHookCount: number

  // 模型信息
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting

  // 会话信息
  isInteractive: boolean
  clientType: string
  sessionId: SessionId

  // OpenTelemetry（可观测性）
  tracerProvider: BasicTracerProvider | null
  meterProvider: MeterProvider | null
  loggerProvider: LoggerProvider | null

  // Hooks 注册表
  registeredHooks: RegisteredHookMatcher[]
}
```

注意 `projectRoot` 的注释：
```typescript
// Stable project root - set once at startup (including by --worktree flag),
// never updated by mid-session EnterWorktreeTool.
// Use for project identity (history, skills, sessions) not file operations.
```

这个设计决策很微妙：`projectRoot` 在启动时设置，即使用户在会话中切换 worktree，`projectRoot` 也不变。这保证了项目身份（历史记录、Skills、会话）的稳定性。

---

## 7.4 AppState：React 状态树

AppState 是一个大型的 React 状态对象，通过 `AppStateStore` 管理：

```typescript
// src/state/AppStateStore.ts（简化）
export type AppState = {
  // 对话状态
  messages: Message[]
  isLoading: boolean
  currentStreamingMessage: string | null

  // 工具执行状态
  inProgressToolUseIDs: Set<string>
  hasInterruptibleToolInProgress: boolean

  // 任务系统
  tasks: TaskStateBase[]

  // 权限系统
  toolPermissionContext: ToolPermissionContext
  pendingPermissionRequests: PermissionRequest[]

  // UI 状态
  showCostThresholdDialog: boolean
  showBypassPermissionsDialog: boolean
  notifications: Notification[]

  // 文件历史
  fileHistoryState: FileHistoryState
  attributionState: AttributionState

  // 模型状态
  mainLoopModel: ModelSetting
  thinkingConfig: ThinkingConfig

  // 投机执行状态
  speculationState: SpeculationState
}
```

---

## 7.5 Store 模式：函数式更新

AppState 使用函数式更新模式，类似 Redux：

```typescript
// src/state/store.ts
export function createStore(initialState: AppState, onChange?) {
  let state = initialState

  return {
    getState(): AppState {
      return state
    },

    setState(updater: (prev: AppState) => AppState): void {
      const newState = updater(state)
      const oldState = state
      state = newState
      onChange?.({ newState, oldState })
      // 通知所有订阅者
      subscribers.forEach(sub => sub())
    },

    subscribe(listener: () => void): () => void {
      subscribers.add(listener)
      return () => subscribers.delete(listener)
    }
  }
}
```

函数式更新的好处：
- **不可变性**：每次更新返回新对象，旧状态不被修改
- **可预测性**：状态变更是纯函数，易于测试
- **时间旅行**：可以保存历史状态，支持撤销

---

## 7.6 React 集成：useSyncExternalStore

AppState 通过 React 的 `useSyncExternalStore` 与 UI 集成：

```typescript
// src/state/AppState.tsx
export function AppStateProvider({ children, initialState, onChangeAppState }) {
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )

  return (
    <AppStoreContext.Provider value={store}>
      <VoiceProvider>
        <MailboxProvider>
          {children}
        </MailboxProvider>
      </VoiceProvider>
    </AppStoreContext.Provider>
  )
}

// 在组件中使用
function MyComponent() {
  const store = useContext(AppStoreContext)
  const messages = useSyncExternalStore(
    store.subscribe,
    () => store.getState().messages
  )
  // messages 变化时自动重渲染
}
```

`useSyncExternalStore` 是 React 18 引入的 API，专门用于订阅外部状态源，保证并发模式下的状态一致性。

---

## 7.7 状态更新的并发安全

工具并行执行时，多个工具可能同时更新状态。Claude Code 通过函数式更新保证安全：

```typescript
// 不安全的写法（竞态条件）
const current = getAppState()
setAppState({ ...current, tasks: [...current.tasks, newTask] })

// 安全的写法（函数式更新）
setAppState(prev => ({
  ...prev,
  tasks: [...prev.tasks, newTask]
}))
```

函数式更新确保每次更新都基于最新的状态，即使多个更新并发执行也不会丢失数据。

---

## 7.8 选择器：精细化订阅

`src/state/selectors.ts` 提供了状态选择器，让组件只订阅自己关心的状态片段：

```typescript
// 只有 tasks 变化时才重渲染
const tasks = useSelector(state => state.tasks)

// 只有特定任务变化时才重渲染
const task = useSelector(state =>
  state.tasks.find(t => t.id === taskId)
)
```

这是性能优化的关键：避免不必要的重渲染。

---

## 7.9 状态变更的副作用：onChangeAppState

`src/state/onChangeAppState.ts` 处理状态变更的副作用：

```typescript
export function onChangeAppState({ newState, oldState }) {
  // 任务完成时发送 OS 通知
  if (newState.tasks !== oldState.tasks) {
    const completedTasks = newState.tasks.filter(
      t => isTerminalTaskStatus(t.status) &&
           !oldState.tasks.find(ot => ot.id === t.id && isTerminalTaskStatus(ot.status))
    )
    completedTasks.forEach(task => sendOSNotification(task))
  }

  // 费用超出阈值时显示警告
  if (newState.totalCostUSD > COST_THRESHOLD && !oldState.showCostThresholdDialog) {
    // 触发弹窗
  }
}
```

这种模式把副作用集中管理，避免散落在各处。

---

## 7.10 状态设计的权衡

Claude Code 的状态设计做了几个有意思的权衡：

**全局 vs 局部**：Bootstrap State 是全局的，AppState 是会话局部的。这个边界划分很清晰：跨会话的信息放全局，会话内的信息放局部。

**React vs 自定义**：AppState 使用 React 状态，但通过 `useSyncExternalStore` 而不是 `useState`/`useReducer`。这让状态可以在 React 组件树之外访问（工具执行时需要读写状态，但工具不是 React 组件）。

**不可变 vs 可变**：消息历史（`mutableMessages`）在 QueryEngine 内部是可变的，但通过 `setAppState` 更新到 AppState 时是不可变的。这个设计在性能（避免频繁复制大数组）和安全性（外部不能直接修改）之间取得平衡。

---

## 7.11 小结

Claude Code 的状态管理设计：

- **两层架构**：Bootstrap State（全局）+ AppState（会话）
- **函数式更新**：保证并发安全和可预测性
- **React 集成**：通过 `useSyncExternalStore` 连接 UI
- **选择器**：精细化订阅，避免不必要的重渲染
- **副作用集中**：`onChangeAppState` 统一处理状态变更的副作用

这套设计在复杂的并发场景下保持了状态的一致性和可观察性。

---

*下一章：[消息循环与流式处理](./08-message-loop.md)*
