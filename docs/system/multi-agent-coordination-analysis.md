# Claude Code 多智能体协调模块技术分析

## 1. 模块总结介绍

Claude Code 的多智能体协调模块（Multi-Agent Coordination）是一个支持并发智能体协作的复杂子系统，它允许用户同时启动多个 AI 智能体来并行处理不同任务。该模块的核心设计目标包括：

- **并行执行**：支持多个智能体同时运行，提高任务处理效率
- **身份隔离**：通过多种机制确保智能体间的上下文不互相干扰
- **灵活通信**：提供消息传递、广播、权限委托等多种协作方式
- **生命周期管理**：完整的智能体创建、运行、监控、终止流程

该模块主要应用于 Coordinator Mode（协调者模式），在此模式下，主智能体可以作为协调者，将复杂任务分解并分派给多个工作智能体并行执行，最后汇总结果。

## 2. 系统架构

多智能体协调模块采用分层架构设计，主要包含以下核心组件：

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层 (Application)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ AgentTool    │  │SendMessage   │  │ TaskStopTool     │  │
│  │ (智能体创建)  │  │ (消息传递)    │  │ (任务终止)        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    任务层 (Task Framework)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │LocalAgentTask│  │InProcess     │  │ RemoteAgentTask  │  │
│  │ (本地异步)    │  │TeammateTask  │  │ (远程会话)        │  │
│  │              │  │ (进程内同伴)  │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    运行时层 (Runtime)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │TeammateContext│  │MailboxSystem │  │TeamFileManager   │  │
│  │(AsyncLocalStorage)│ (文件邮箱)  │  │ (团队配置)        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    定义层 (Definitions)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │loadAgentsDir │  │AgentColor    │  │CoordinatorMode   │  │
│  │(智能体定义)   │  │Manager       │  │ (协调者模式)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件说明

1. **AgentTool**：智能体创建的核心工具，负责根据配置选择适当的执行模式
2. **Task Framework**：统一的任务状态管理系统，支持多种任务类型
3. **TeammateContext**：基于 AsyncLocalStorage 的进程内上下文隔离机制
4. **MailboxSystem**：基于文件的智能体间通信系统
5. **TeamFileManager**：团队配置的持久化管理

## 3. 分层说明

### 3.1 定义层 (Definitions Layer)

智能体定义通过 Markdown 文件的前置元数据（frontmatter）进行配置：

```typescript
// restored-src/src/tools/AgentTool/loadAgentsDir.ts
export type AgentDefinition = {
  name: string                    // 智能体名称
  description: string             // 功能描述
  tools?: string[]               // 允许使用的工具
  disallowedTools?: string[]     // 禁止使用的工具
  model?: string                 // 模型覆盖
  effort?: 'low' | 'medium' | 'high'  // 努力程度
  permissionMode?: PermissionMode     // 权限模式
  memory?: boolean               // 是否启用记忆
  background?: boolean           // 是否后台运行
  isolation?: 'none' | 'worktree' | 'fork' | 'remote'  // 隔离模式
  color?: string                 // UI 颜色
  // ... 其他配置
}
```

智能体定义来源包括：
- **内置智能体**：`agents/` 目录下的 Markdown 文件
- **自定义智能体**：`~/.claude/agents/` 目录下的用户定义
- **插件智能体**：通过插件系统加载的第三方定义

### 3.2 运行时层 (Runtime Layer)

#### 3.2.1 TeammateContext（进程内上下文）

```typescript
// restored-src/src/utils/teammateContext.ts
export type TeammateContext = {
  agentId: string              // 完整智能体ID，如 "researcher@my-team"
  agentName: string            // 显示名称
  teamName: string             // 所属团队
  color?: string               // UI 颜色
  planModeRequired: boolean    // 是否必须计划模式
  parentSessionId: string      // 领导者会话ID
  isInProcess: true            // 标识为进程内智能体
  abortController: AbortController  // 生命周期控制器
}

const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

export function runWithTeammateContext<T>(
  context: TeammateContext,
  fn: () => T,
): T {
  return teammateContextStorage.run(context, fn)
}
```

`AsyncLocalStorage` 是 Node.js 提供的异步上下文存储机制，它确保在异步调用链中能够正确访问到对应的智能体上下文，而不会与其他并发智能体混淆。

#### 3.2.2 MailboxSystem（邮箱系统）

基于文件的邮箱系统实现智能体间通信：

```typescript
// restored-src/src/utils/teammateMailbox.ts
export type MailboxMessage = {
  id: string
  from: string
  to: string
  content: string
  timestamp: number
  type?: 'permission_request' | 'permission_response' | 
         'shutdown_request' | 'shutdown_approved' |
         'plan_approval_request' | 'plan_approval_response'
}
```

邮箱文件路径：`~/.claude/teams/{team}/inboxes/{agent}.json`

使用 lockfile 机制确保并发安全，支持以下消息类型：
- **权限请求/响应**：智能体间委托权限决策
- **关闭请求/确认**：优雅终止智能体
- **计划审批请求/响应**：计划模式的协作

### 3.3 任务层 (Task Framework Layer)

任务框架统一定义了多种任务类型的状态结构：

```typescript
// restored-src/src/tasks/types.ts
export type TaskState =
  | BashTaskState
  | InProcessTeammateTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | DreamTaskState
  // ... 其他任务类型

export type BackgroundTaskState =
  | InProcessTeammateTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | DreamTaskState
```

#### InProcessTeammateTaskState（进程内同伴任务）

```typescript
// restored-src/src/tasks/InProcessTeammateTask/types.ts
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  status: 'running' | 'completed' | 'failed' | 'killed'
  identity: TeammateIdentity
  prompt: string
  model?: string
  abortController: AbortController
  awaitingPlanApproval: boolean
  messages: TeammateMessage[]        // 对话历史（上限50条）
  pendingUserMessages: string[]      // 待处理用户消息
  // ... 其他字段
}
```

### 3.4 应用层 (Application Layer)

#### AgentTool 执行模式选择逻辑

```typescript
// restored-src/src/tools/AgentTool/AgentTool.tsx
const shouldRunAsync = (
  run_in_background === true ||
  selectedAgent.background === true ||
  isCoordinator ||
  forceAsync ||
  assistantForceAsync ||
  (proactiveModule?.isProactiveActive() ?? false)
) && !isBackgroundTasksDisabled

// 执行路径分支
if (teamName && name) {
  // 同伴智能体路径（协调者模式）
  return spawnTeammate(...)
} else if (isolation === 'fork') {
  // Fork 子智能体（实验性功能）
  return spawnForkSubagent(...)
} else if (shouldRunAsync) {
  // 本地异步执行
  return backgroundAgentTask(...)
} else {
  // 同步执行
  return executeAgentSync(...)
}
```

## 4. 交互流程

### 4.1 协调者启动同伴智能体流程

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ Coordinator│     │ AgentTool│     │spawnInProcess│     │inProcessRunner│
│  (领导者)  │────▶│         │────▶│  (创建任务)   │────▶│  (执行循环)   │
└──────────┘     └──────────┘     └──────────────┘     └──────────────┘
                                                          │
                                                          ▼
                              ┌─────────────────────────────────────────┐
                              │  1. runWithTeammateContext() 设置上下文  │
                              │  2. runAgent() 启动智能体循环            │
                              │  3. 定期轮询 mailbox 检查消息            │
                              │  4. 处理权限请求（委托给领导者）          │
                              │  5. 更新任务状态到 AppState              │
                              └─────────────────────────────────────────┘
```

### 4.2 智能体间消息传递流程

```
发送方智能体                    SendMessageTool                    接收方智能体
    │                              │                                  │
    │── send_message(to, content)──▶│                                  │
    │                              │── 1. 解析接收者                   │
    │                              │── 2. 如果是同伴：                  │
    │                              │   - 查找 taskId                   │
    │                              │   - 加入 pendingUserMessages      │
    │                              │   - 或写入 mailbox 文件           │
    │                              │── 3. 如果是本地异步：              │
    │                              │   - resumeAgentBackground()       │
    │                              │   - 加入消息队列                   │
    │                              ▼                                  │
    │                    ┌─────────────────────┐                      │
    │                    │  Mailbox 文件系统    │                      │
    │                    │ ~/.claude/teams/... │                      │
    │                    └─────────────────────┘                      │
    │                              │                                  │
    │                              │◀── 轮询读取消息 ─────────────────│
    │                              │                                  │
```

### 4.3 团队初始化流程

```typescript
// restored-src/src/utils/swarm/reconnection.ts
export function computeInitialTeamContext(): AppState['teamContext'] | undefined {
  // 1. 从环境变量或动态上下文获取团队信息
  const teamName = getTeamName()
  const agentId = getAgentId()
  
  // 2. 读取团队配置文件
  const teamFile = readTeamFile(teamName)
  
  // 3. 判断当前角色（领导者/同伴）
  const isLeader = agentId === teamFile.leadAgentId
  
  // 4. 构建团队上下文
  return {
    teamName,
    teamFilePath,
    leadAgentId: teamFile.leadAgentId,
    selfAgentId: agentId,
    selfAgentName: agentName,
    isLeader,
    teammates: {}  // 运行时动态填充
  }
}
```

## 5. 技术原理

### 5.1 AsyncLocalStorage 上下文隔离

Node.js 的 `AsyncLocalStorage` 是一个基于异步资源追踪的上下文存储 API。其核心原理是：

1. **异步资源追踪**：Node.js 在创建异步操作（Promise、定时器、I/O 回调等）时会建立异步资源树
2. **上下文继承**：子异步操作自动继承父操作的上下文
3. **隔离保证**：不同异步调用链之间的存储完全隔离

```typescript
// 使用示例
const context = createTeammateContext({
  agentId: 'researcher@team-alpha',
  agentName: 'researcher',
  teamName: 'team-alpha',
  // ...
})

// 在此回调中运行的所有代码都能通过 getTeammateContext() 
// 获取到正确的上下文，即使多个智能体并发执行
runWithTeammateContext(context, () => {
  return runAgent()  // 智能体主循环
})
```

### 5.2 文件锁机制（Mailbox 并发控制）

邮箱系统使用 `proper-lockfile` 库实现进程间互斥：

```typescript
// 写入邮箱时获取锁
await lock(mailboxPath, {
  stale: 5000,      // 5秒后认为锁已过期
  retries: 3,       // 重试3次
  retryWait: 100    // 每次重试间隔100ms
})

try {
  // 读取并更新邮箱内容
  const messages = await readMailbox(teamName, agentName)
  messages.push(newMessage)
  await writeMailbox(teamName, agentName, messages)
} finally {
  // 确保释放锁
  await unlock(mailboxPath)
}
```

### 5.3 权限桥接机制

进程内智能体的权限请求需要桥接到领导者的 UI：

```typescript
// restored-src/src/utils/swarm/leaderPermissionBridge.ts
export function bridgeTeammatePermissionToLeader(
  teammateTaskId: string,
  request: PermissionRequest
): Promise<PermissionResponse> {
  return new Promise((resolve) => {
    // 1. 将请求加入领导者的权限队列
    setAppState(prev => ({
      ...prev,
      teammatePermissionQueue: [
        ...prev.teammatePermissionQueue,
        { taskId: teammateTaskId, request, resolve }
      ]
    }))
    
    // 2. 领导者 UI 显示权限请求
    // 3. 用户决策后，通过 SendMessageTool 发送响应
    // 4. 解析 Promise 继续智能体执行
  })
}
```

## 6. 创新点

### 6.1 三种身份隔离机制的统一抽象

Claude Code 实现了三种不同层次的身份隔离，并通过统一的 API 对外提供：

| 机制 | 实现方式 | 适用场景 |
|------|----------|----------|
| 环境变量 | `CLAUDE_CODE_AGENT_ID` | 基于 tmux/iTerm2 的进程级隔离 |
| 动态团队上下文 | `dynamicTeamContext` Map | 运行时加入团队的进程级智能体 |
| AsyncLocalStorage | `TeammateContext` | 进程内并发智能体（最高效） |

```typescript
// restored-src/src/utils/teammate.ts
export function getAgentId(): string | undefined {
  // 优先级：AsyncLocalStorage > 动态上下文 > 环境变量
  return (
    getTeammateContext()?.agentId ??
    dynamicTeamContext.getStore()?.agentId ??
    process.env.CLAUDE_CODE_AGENT_ID
  )
}
```

### 6.2 智能体颜色管理

为不同智能体分配固定颜色，便于在 UI 中区分：

```typescript
// restored-src/src/tools/AgentTool/agentColorManager.ts
const AGENT_COLORS = [
  { key: 'red', terminal: 'ansiRGB(255, 100, 100)', theme: 'red' },
  { key: 'blue', terminal: 'ansiRGB(100, 150, 255)', theme: 'blue' },
  { key: 'green', terminal: 'ansiRGB(100, 200, 100)', theme: 'green' },
  // ... 共8种颜色
] as const
```

颜色分配是确定性的，基于智能体名称哈希，确保同一智能体始终获得相同颜色。

### 6.3 计划模式与权限模式的组合

支持 `planModeRequired` 配置，要求智能体在实施前必须先制定计划：

```typescript
// restored-src/src/utils/swarm/inProcessRunner.ts
if (taskState.awaitingPlanApproval) {
  // 智能体已提交计划，等待用户审批
  return { status: 'waiting_for_plan_approval' }
}

if (taskState.permissionMode === 'plan' && !planApproved) {
  // 需要制定计划
  return { status: 'requires_plan' }
}
```

## 7. 关键技术

### 7.1 任务状态机

```
┌─────────┐    启动     ┌─────────┐   完成   ┌──────────┐
│  initial │ ─────────▶ │ running │ ───────▶ │ completed│
└─────────┘            └─────────┘          └──────────┘
                            │
                            │ 错误
                            ▼
                       ┌─────────┐
                       │  failed │
                       └─────────┘
                            │
                            │ 杀死
                            ▼
                       ┌─────────┐
                       │  killed │
                       └─────────┘
```

### 7.2 消息历史管理

为防止内存无限增长，同伴智能体的消息历史有上限：

```typescript
// restored-src/src/tasks/InProcessTeammateTask/types.ts
const TEAMMATE_MESSAGES_UI_CAP = 50  // 最多保留50条消息

// 当超过上限时，移除最早的消息（保留系统消息）
if (messages.length > TEAMMATE_MESSAGES_UI_CAP) {
  const systemMessages = messages.filter(m => m.role === 'system')
  const nonSystemMessages = messages.filter(m => m.role !== 'system')
  const toRemove = nonSystemMessages.length - TEAMMATE_MESSAGES_UI_CAP
  messages = [...systemMessages, ...nonSystemMessages.slice(toRemove)]
}
```

### 7.3 优雅关闭机制

```typescript
// 1. 发送关闭请求
export function requestTeammateShutdown(
  teamName: string, 
  agentName: string
): void {
  sendMailboxMessage(teamName, agentName, {
    type: 'shutdown_request',
    from: 'leader',
    content: 'Shutdown requested by leader'
  })
}

// 2. 智能体检测到关闭请求后，完成当前工作再退出
// 3. 超时后强制终止（通过 AbortController）
```

### 7.4 Perfetto 追踪集成

```typescript
// restored-src/src/utils/swarm/spawnInProcess.ts
if (isPerfettoTracingEnabled()) {
  registerPerfettoAgent(agentId, name, parentSessionId)
}

// 在智能体生命周期事件中记录追踪数据
// 支持可视化智能体层次结构和执行时间线
```

## 8. 思考总结

### 8.1 架构设计亮点

1. **渐进式复杂度**：从简单的环境变量隔离，到动态上下文，再到 AsyncLocalStorage，系统根据场景选择适当的隔离级别

2. **统一抽象**：不同类型的智能体（本地异步、进程内、远程）通过统一的任务框架管理，对外提供一致的接口

3. **容错设计**：文件锁超时、AbortController 取消、优雅关闭等机制确保系统在面对错误时能正确处理

4. **可观测性**：Perfetto 追踪、详细的日志记录、任务状态可视化，便于调试和监控

### 8.2 潜在改进方向

1. **Mailbox 性能优化**：当前基于文件的通信在极高频场景下可能成为瓶颈，可考虑可选的内存通道或 Unix Domain Socket

2. **智能体发现机制**：目前智能体需要预定义，未来可考虑动态发现和能力注册

3. **分布式扩展**：当前的团队机制主要面向单机，可扩展为真正的分布式多机协作

4. **更细粒度的权限控制**：当前权限模式较为粗粒度，可支持更细的工具级、资源级权限

### 8.3 应用场景

多智能体协调模块特别适用于以下场景：

- **大规模代码重构**：多个智能体并行处理不同模块
- **多维度分析**：代码分析、安全审计、性能优化同时进行
- **复杂项目管理**：需求分析、架构设计、测试用例生成并行执行
- **实时协作编程**：多个专业智能体（前端、后端、算法）协同完成功能

### 8.4 总结

Claude Code 的多智能体协调模块是一个设计精良、实现完善的并发协作系统。它通过 AsyncLocalStorage 实现了高效的进程内隔离，通过 Mailbox 系统实现了可靠的跨智能体通信，通过统一的任务框架简化了多类型智能体的管理。该模块为 AI 辅助编程提供了强大的并行处理能力，是 Claude Code 区别于其他 AI 编程工具的重要特性之一。

---

*文档生成时间：2026-04-01*
*基于 Claude Code 源代码分析*
