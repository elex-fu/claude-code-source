# Claude Code Task Framework 技术分析

## 1. 模块总结介绍

Task Framework 是 Claude Code 的「异步任务调度中心」，统一管理后台任务（Agent、Bash、Teammate 等）的生命周期。该框架提供了统一的状态管理、进度跟踪和任务间协调机制。

核心特性：
- **统一状态机**：所有任务遵循统一的状态模型
- **并发控制**：支持任务并行执行和依赖管理
- **进度追踪**：实时任务进度和输出捕获
- **优雅取消**：AbortController 支持的任务取消

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Task Framework                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Task State    │  │  Task Executor  │  │   Output    │ │
│  │   Management    │  │                 │  │   Capture   │ │
│  │                 │  │ • LocalAgent    │  │             │ │
│  │ • State Machine │──▶ • Bash          │──▶ • stdout    │ │
│  │ • Transitions   │    • RemoteAgent    │    • stderr    │ │
│  │ • Persistence   │    • Teammate       │    • Progress  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心类型定义

### 3.1 任务状态类型

```typescript
// tasks/types.ts
export type TaskStatus =
  | 'pending'      // 等待执行
  | 'running'      // 执行中
  | 'completed'    // 完成成功
  | 'failed'       // 执行失败
  | 'killed'       // 被终止
  | 'cancelled'    // 被取消

export type TaskStateBase = {
  id: string
  type: string
  status: TaskStatus
  createdAt: number
  startedAt?: number
  completedAt?: number
  error?: string
}

// 具体任务类型
export type BashTaskState = TaskStateBase & {
  type: 'bash'
  command: string
  cwd: string
  exitCode?: number
  stdout: string[]
  stderr: string[]
}

export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  prompt: string
  model?: string
  messages: Message[]
  output?: string
}

export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity
  prompt: string
  model?: string
  messages: TeammateMessage[]
  pendingUserMessages: string[]
  abortController: AbortController
  awaitingPlanApproval: boolean
}

export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  sessionId: string
  prompt: string
  status: 'running' | 'completed' | 'failed' | 'killed'
}

export type TaskState =
  | BashTaskState
  | LocalAgentTaskState
  | InProcessTeammateTaskState
  | RemoteAgentTaskState
  | DreamTaskState
```

### 3.2 后台任务联合类型

```typescript
export type BackgroundTaskState =
  | InProcessTeammateTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | DreamTaskState
```

## 4. 任务状态机

### 4.1 状态转换图

```
                    ┌─────────┐
              ┌────▶│ pending │
              │     └────┬────┘
              │          │ start()
              │          ▼
     retry()  │     ┌─────────┐     complete()
              └────┤ running │────────▶┌──────────┐
                   └────┬────┘         │completed │
        ┌───────────────┼──────────────┘          │
        │               │                         │
        │         error │                         │
        │               ▼                         │
   kill()        ┌─────────┐                      │
        │        │ failed  │                      │
        │        └────┬────┘                      │
        │             │                           │
        └─────────────┴───────────────────────────┘
                      │ cancel()
                      ▼
                 ┌─────────┐
                 │ killed  │
                 └─────────┘
```

### 4.2 状态转换函数

```typescript
export function transitionTaskState(
  task: TaskState,
  newStatus: TaskStatus
): TaskState {
  const validTransitions: Record<TaskStatus, TaskStatus[]> = {
    pending: ['running', 'cancelled'],
    running: ['completed', 'failed', 'killed'],
    completed: [],
    failed: ['running'], // 允许重试
    killed: [],
    cancelled: [],
  }

  if (!validTransitions[task.status].includes(newStatus)) {
    throw new Error(
      `Invalid state transition: ${task.status} -> ${newStatus}`
    )
  }

  return {
    ...task,
    status: newStatus,
    completedAt: ['completed', 'failed', 'killed'].includes(newStatus)
      ? Date.now()
      : undefined,
  }
}
```

## 5. 任务执行器

### 5.1 基础执行器接口

```typescript
export interface TaskExecutor<T extends TaskState> {
  // 执行任务
  execute(task: T, signal: AbortSignal): AsyncGenerator<TaskEvent, T, unknown>

  // 取消任务
  cancel(taskId: string): void

  // 获取任务输出
  getOutput(taskId: string): string | undefined
}
```

### 5.2 Bash 任务执行器

```typescript
// tasks/LocalShellTask/LocalShellTask.ts
export const bashTaskExecutor: TaskExecutor<BashTaskState> = {
  async function* execute(
    task: BashTaskState,
    signal: AbortSignal
  ): AsyncGenerator<TaskEvent, BashTaskState, unknown> {
    yield { type: 'started', taskId: task.id }

    const process = spawn(task.command, {
      cwd: task.cwd,
      shell: true,
    })

    const stdout: string[] = []
    const stderr: string[] = []

    // 捕获输出
    process.stdout?.on('data', (data) => {
      const line = data.toString()
      stdout.push(line)
      yield { type: 'stdout', taskId: task.id, data: line }
    })

    process.stderr?.on('data', (data) => {
      const line = data.toString()
      stderr.push(line)
      yield { type: 'stderr', taskId: task.id, data: line }
    })

    // 处理取消
    signal.addEventListener('abort', () => {
      process.kill('SIGTERM')
    })

    // 等待完成
    const exitCode = await new Promise<number>((resolve) => {
      process.on('close', resolve)
    })

    if (signal.aborted) {
      return { ...task, status: 'killed', stdout, stderr }
    }

    if (exitCode === 0) {
      return { ...task, status: 'completed', exitCode, stdout, stderr }
    } else {
      return {
        ...task,
        status: 'failed',
        exitCode,
        stdout,
        stderr,
        error: `Exit code ${exitCode}`,
      }
    }
  },

  cancel(taskId: string): void {
    // 通过 AbortController 取消
  },

  getOutput(taskId: string): string | undefined {
    const task = getTask(taskId) as BashTaskState
    return task?.stdout.join('')
  },
}
```

### 5.3 Agent 任务执行器

```typescript
// tasks/LocalAgentTask/LocalAgentTask.ts
export const localAgentExecutor: TaskExecutor<LocalAgentTaskState> = {
  async function* execute(
    task: LocalAgentTaskState,
    signal: AbortSignal
  ): AsyncGenerator<TaskEvent, LocalAgentTaskState, unknown> {
    yield { type: 'started', taskId: task.id }

    const messages: Message[] = [
      { role: 'user', content: task.prompt },
    ]

    // 运行 Agent 循环
    const result = await runAgentLoop({
      messages,
      model: task.model,
      signal,
      onMessage: (msg) => {
        messages.push(msg)
        yield { type: 'message', taskId: task.id, message: msg }
      },
    })

    if (signal.aborted) {
      return { ...task, status: 'killed', messages }
    }

    if (result.success) {
      return {
        ...task,
        status: 'completed',
        messages,
        output: result.output,
      }
    } else {
      return {
        ...task,
        status: 'failed',
        messages,
        error: result.error,
      }
    }
  },

  cancel(taskId: string): void {
    // 触发 AbortController
  },

  getOutput(taskId: string): string | undefined {
    const task = getTask(taskId) as LocalAgentTaskState
    return task?.output
  },
}
```

## 6. 任务管理器

### 6.1 任务注册表

```typescript
// tasks.ts
const tasks: Map<string, TaskState> = new Map()
const abortControllers: Map<string, AbortController> = new Map()
const executors: Map<string, TaskExecutor<TaskState>> = new Map()

export function registerTask<T extends TaskState>(
  task: T,
  executor: TaskExecutor<T>
): T {
  tasks.set(task.id, task)
  executors.set(task.id, executor as TaskExecutor<TaskState>)

  // 通知状态变更
  notifyTaskChange(task)

  return task
}

export function getTask<T extends TaskState>(id: string): T | undefined {
  return tasks.get(id) as T | undefined
}

export function getAllTasks(): TaskState[] {
  return Array.from(tasks.values())
}

export function getBackgroundTasks(): BackgroundTaskState[] {
  return getAllTasks().filter(
    (t): t is BackgroundTaskState =>
      t.type === 'in_process_teammate' ||
      t.type === 'local_agent' ||
      t.type === 'remote_agent' ||
      t.type === 'dream'
  )
}
```

### 6.2 任务启动

```typescript
export async function startTask<T extends TaskState>(
  task: T,
  executor: TaskExecutor<T>
): Promise<void> {
  // 创建 AbortController
  const abortController = new AbortController()
  abortControllers.set(task.id, abortController)

  // 更新状态为 running
  const runningTask = transitionTaskState(task, 'running')
  tasks.set(task.id, { ...runningTask, startedAt: Date.now() })

  // 启动执行器
  const generator = executor.execute(runningTask, abortController.signal)

  // 处理事件流
  try {
    let result = await generator.next()

    while (!result.done) {
      const event = result.value

      // 广播事件
      broadcastTaskEvent(event)

      // 继续执行
      result = await generator.next()
    }

    // 任务完成
    tasks.set(result.value.id, result.value)
    notifyTaskChange(result.value)
  } catch (error) {
    // 任务异常
    const failedTask: T = {
      ...task,
      status: 'failed',
      error: String(error),
      completedAt: Date.now(),
    }
    tasks.set(task.id, failedTask)
    notifyTaskChange(failedTask)
  } finally {
    abortControllers.delete(task.id)
    executors.delete(task.id)
  }
}
```

### 6.3 任务取消

```typescript
export function cancelTask(taskId: string): void {
  const controller = abortControllers.get(taskId)
  if (controller) {
    controller.abort()
  }

  const task = tasks.get(taskId)
  if (task && task.status === 'running') {
    const killedTask = transitionTaskState(task, 'killed')
    tasks.set(taskId, killedTask)
    notifyTaskChange(killedTask)
  }
}

export function killAllTasks(): void {
  for (const [id, controller] of abortControllers) {
    controller.abort()

    const task = tasks.get(id)
    if (task) {
      const killedTask = transitionTaskState(task, 'killed')
      tasks.set(id, killedTask)
    }
  }

  // 批量通知
  notifyAllTasksChanged()
}
```

## 7. 任务事件系统

### 7.1 事件类型

```typescript
export type TaskEvent =
  | { type: 'started'; taskId: string }
  | { type: 'completed'; taskId: string }
  | { type: 'failed'; taskId: string; error: string }
  | { type: 'killed'; taskId: string }
  | { type: 'stdout'; taskId: string; data: string }
  | { type: 'stderr'; taskId: string; data: string }
  | { type: 'progress'; taskId: string; percent: number }
  | { type: 'message'; taskId: string; message: Message }
```

### 7.2 事件订阅

```typescript
const listeners: Set<(event: TaskEvent) => void> = new Set()

export function subscribeToTaskEvents(
  listener: (event: TaskEvent) => void
): () => void {
  listeners.add(listener)
  return () => listeners.delete(listener)
}

function broadcastTaskEvent(event: TaskEvent): void {
  for (const listener of listeners) {
    try {
      listener(event)
    } catch (error) {
      console.error('Task event listener error:', error)
    }
  }
}
```

## 8. 任务持久化

### 8.1 状态保存

```typescript
export function saveTaskState(taskId: string): void {
  const task = tasks.get(taskId)
  if (!task) return

  const statePath = getTaskStatePath(taskId)
  writeFileSync(statePath, JSON.stringify(task, null, 2))
}

export function loadTaskState(taskId: string): TaskState | null {
  try {
    const statePath = getTaskStatePath(taskId)
    const data = readFileSync(statePath, 'utf-8')
    return JSON.parse(data) as TaskState
  } catch {
    return null
  }
}
```

### 8.2 会话恢复

```typescript
export async function restoreTasksFromSession(
  sessionId: string
): Promise<void> {
  const taskIds = getSessionTaskIds(sessionId)

  for (const taskId of taskIds) {
    const state = loadTaskState(taskId)
    if (!state) continue

    // 恢复任务到内存
    tasks.set(taskId, state)

    // 如果任务在恢复时仍在运行，标记为失败
    if (state.status === 'running') {
      const failedState = transitionTaskState(state, 'failed')
      failedState.error = 'Session restored'
      tasks.set(taskId, failedState)
    }
  }
}
```

## 9. 与 AppState 集成

### 9.1 任务状态同步

```typescript
// onChangeAppState.ts
export function onTaskChange(task: TaskState): void {
  setAppState((prev) => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [task.id]: task,
    },
  }))
}

// 订阅任务事件
subscribeToTaskEvents((event) => {
  const task = getTask(event.taskId)
  if (task) {
    onTaskChange(task)
  }
})
```

### 9.2 前台任务管理

```typescript
export function setForegroundTask(taskId: string): void {
  setAppState((prev) => ({
    ...prev,
    foregroundedTaskId: taskId,
  }))
}

export function getForegroundTask(): TaskState | undefined {
  const state = getAppState()
  return state.foregroundedTaskId
    ? getTask(state.foregroundedTaskId)
    : undefined
}
```

## 10. 创新点与反思

### 10.1 设计创新

1. **统一状态机**：所有任务类型遵循相同的状态模型
2. **Generator-based 执行**：支持流式事件和取消
3. **AbortController 集成**：标准的取消机制

### 10.2 架构权衡

**集中式 vs 分布式**

当前采用集中式任务管理：
- ✅ 状态一致性
- ✅ 易于监控
- ❌ 单点瓶颈

对于大量并发任务，可能需要分布式设计。

### 10.3 生产经验

1. **任务超时**：长时间运行的任务需要超时机制
2. **资源限制**：并发任务数限制防止资源耗尽
3. **优雅关闭**：应用退出前等待任务完成

### 10.4 未来演进

1. **任务依赖图**：DAG 执行模型
2. **优先级调度**：高优先级任务优先执行
3. **资源配额**：每个用户/任务组的资源限制

---

Claude Code 的 Task Framework 是一个统一的异步任务管理系统。通过状态机、Generator、AbortController 等设计，实现了可靠的任务调度和生命周期管理。理解其设计，对于构建任何需要后台任务管理的应用都具有参考价值。
