# Claude Code Agent Loop & Query Engine 技术解析

## 一、模块概述

**Agent Loop & Query Engine** 是 Claude Code 的核心运行时引擎，负责将用户的自然语言请求转化为大模型（Claude）的流式 API 调用，并协调工具执行、上下文管理、状态机转换与外部系统集成。它既服务于交互式 REPL，也支撑着 headless/SDK 模式，是整个应用从"接收输入"到"输出结果"的中枢神经。

该模块横跨多个子系统：从 `main.tsx` 的 CLI 入口，到 `query.ts` 的主循环状态机；从 `StreamingToolExecutor.ts` 的并发工具执行，到 `bridge/bridgeMain.ts` 的远端会话桥接；还包括 stop hooks、任务后台化、上下文压缩等进阶能力。其设计目标是：**高吞吐量**（并发执行只读工具）、**强容错性**（max-output-tokens 恢复、流式回退）、**可扩展性**（技能与 MCP 的动态挂载）以及**多模式兼容**（本地 REPL、headless 脚本、远程 CCR 会话）。

## 二、系统架构

从架构上看，Agent Loop & Query Engine 呈现典型的 **"入口 - 引擎 - 执行层 - 桥接/任务层"** 四层结构：

1. **CLI 入口层** (`main.tsx`)  
   负责参数解析、模式分支（`launchRepl` vs `runHeadless`）、权限模式初始化、工具的加载与注册（含 MCP 与自定义 Agent）。这是整个生命周期的启动器。

2. **查询引擎层** (`query.ts` + `QueryEngine.ts`)  
   `query.ts` 提供底层的 `query()` 与 `queryLoop()` 异步生成器，维护一个跨多轮对话的 `State` 状态机；`QueryEngine.ts` 则将其封装为面向 SDK/headless 的 `QueryEngine` 类，统一管理 `canUseTool`、权限拒绝追踪和工具使用摘要（`generateToolUseSummary`）。

3. **工具执行层** (`StreamingToolExecutor.ts`, `toolOrchestration.ts`)  
   处理模型返回的 `tool_use` 调用。通过并发安全检测（`isConcurrencySafe`）将只读工具批量并行执行，涉及写操作或非安全工具则串行执行；并通过 `siblingAbortController` 实现错误级联取消（尤其针对 Bash 工具）。

4. **桥接与任务层** (`bridge/bridgeMain.ts`, `tasks/LocalMainSessionTask.ts`)  
   `bridgeMain.ts` 实现远程会话的长轮询桥接，管理 CCR v1/v2、心跳、Token 刷新与容量控制；`LocalMainSessionTask.ts` 支持主会话的"后台化"（Ctrl+B 两次），让用户在后台继续运行查询的同时获得一个全新的前台交互界面。

## 三、核心组件逐层拆解

### 3.1 CLI 入口 (`main.tsx`)

`main.tsx` 使用 Commander.js 构建 CLI。核心 `run()` 函数通过 `preAction` hook 完成初始化（`init()`），随后根据命令行参数分流：
- **交互模式**：`launchRepl` 启动基于 Ink/React 的 TUI；
- **Headless 模式**：`runHeadless` 从 `src/cli/print.js` 导入，直接输出结果并退出；
- **Bridge 模式**：在远程场景下进入 `bridgeMain.ts` 的循环。

`run()` 还处理权限模式（`--permission-*` 系列参数）、resume/continue/teleport 等持久化状态恢复逻辑，以及 MCP 配置的加载。

### 3.2 查询状态机 (`query.ts`)

`query.ts` 是引擎的心脏。它定义了如下 `State` 结构：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

`queryLoop()` 本质上是一个 `while (true)` 循环，每轮执行：
1. **技能预取**（skill prefetch）：在进入 API 流之前，异步准备可能用到的自定义技能；
2. **API 流式请求**：通过 Anthropic SDK 发送消息，处理 `content_block_delta`、`message_stop` 等事件；
3. **流式回退（streaming fallback）**：若流因异常中断，会尝试重新请求，并在失败时回退到非流式调用；
4. **`max_output_tokens` 恢复**：当模型输出因长度限制被截断时，通过 reactive-compact、micro-compact、snip-compact 四级恢复策略压缩上下文，以腾出空间让模型继续生成；
5. **Stop Hooks 执行**：每轮结束后进入 `handleStopHooks`，运行用户自定义的 Stop hook、任务完成 hook 与队友空闲 hook；
6. **工具执行**：将模型产生的 `tool_use` 块交给工具执行层，执行结果重新拼入 `messages`；
7. **Attachment 注入**：在消息正式发向模型之前，通过 attachment 机制注入文件引用的结构化内容；
8. **状态转换**：每轮通过 `transition`（`Continue`）决定是否进入下一轮。`turnCount` 持续递增，作为无限轮次的计数器。

### 3.3 QueryEngine 封装 (`QueryEngine.ts`)

`QueryEngine` 将 `query()` 包装为面向消费者的类接口，主要增强点：
- 拦截并统计 SDK 用户对工具的拒绝（`permissionContext.denials`）；
- 在工具批次完成后，调用 `generateToolUseSummary` 利用轻量模型（Haiku，约 1 秒延迟）生成工具使用的自然语言摘要，便于上下文理解与调试；
- 提供 `submitMessage()` 方法，统一处理输入消息的提交与结果迭代，适配 SDK 的回调与事件体系。

### 3.4 流式工具执行器 (`StreamingToolExecutor.ts`)

这是并发控制复杂度最高的组件之一。核心设计：
- `addTool()`：在模型流式返回 tool_use 时立即入队，不等全部 tool_use 到达就开始执行；
- `processQueue()`：动态检查并发条件，若队首工具是 concurrency-safe 且当前无 exclusive 工具在执行，则立即启动；
- `siblingAbortController`：专属子控制器。当某个 Bash 工具报错时，触发 `sibling_error`，自动取消其他正在并行的 Bash 进程（避免 `mkdir` 失败后后续命令继续执行的无意义消耗），但不会取消读文件、WebFetch 等独立工具；
- `pendingProgress`：进度消息立刻 yield，不等待工具完成，保证 UI 的实时性；
- `getRemainingResults()`：在轮询中等待后续结果，同时响应 progress 事件。

### 3.5 工具编排 (`toolOrchestration.ts`)

这是非流式（或批量）场景下的工具执行器，提供 `runTools()` 函数：
1. `partitionToolCalls()`：将模型返回的一连串 `tool_use` 按 `isConcurrencySafe` 切分为 batch。连续的 safe tool 合并为一个并发 batch，单个 unsafe tool 独占一个串行 batch；
2. `runToolsConcurrently()`：使用 `all()` 并发器按 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`（默认 10）限制并行度；
3. `runToolsSerially()`：逐个执行，并在过程中实时更新 `toolUseContext`。

### 3.6 远程桥接 (`bridge/bridgeMain.ts`)

`bridgeMain.ts` 实现了一个长生命周期的 polling loop，用于将本地 Claude Code 进程作为远程会话的 worker：
- 维护 `activeSessions` Map，记录每个 session 的 worktree 与运行状态；
- 定时向控制平面发送 heartbeats 与状态汇报；
- 处理 OAuth token refresh；
- 支持 CCR v2（`useCcrV2`），将自身注册为 worker，接收并执行远端分派的 query；
- 容量管理：在负载过高时拒绝新 session，保证本地资源不被耗尽。

### 3.7 主会话后台化 (`tasks/LocalMainSessionTask.ts`)

Claude Code 支持将当前主会话查询"后台化"（快捷键 `Ctrl+B` 按两次）：
- `registerMainSessionTask()`：用 `s` 前缀生成唯一 task ID，复用 `LocalAgentTaskState` 结构，但标记 `agentType: 'main-session'`；
- `startBackgroundSession()`：启动一个完全独立的 `query()` 调用，在独立的 AgentContext 下运行，其 transcript 通过 symlink 隔离，避免主会话执行 `/clear` 后污染后台任务的输出；
- 任务完成后，若仍处于后台状态，则通过 XML 标签向消息队列推送通知；若用户已将其切回前台，则直接 merge 结果到当前会话视图。

## 四、交互流程：从用户输入到结果输出

以一次典型的 REPL 用户提问为例，交互流程如下：

1. **输入捕获**  
   Ink TUI 捕获用户消息，将其传入 `QueryEngine.submitMessage()` 或 `query()`。

2. **Attachment 注入与上下文准备**  
   `query.ts` 在发送前调用附件解析器，将用户通过 `@file` 或自动检测的文件引用转化为结构化的 `Message` 内容。

3. **API 请求与流式消费**  
   进入 `queryLoop()` 循环，向 Anthropic API 发起流式请求。循环消费 SSE 事件，逐步构建 `assistant` 消息。

4. **工具调用识别**  
   若 assistant 消息包含 `tool_use` 块，循环退出 API 消费阶段，进入工具执行阶段。

5. **并发/串行工具执行**  
   - 流式场景：`StreamingToolExecutor` 边接收边执行，通过 `siblingAbortController` 管理错误级联；
   - 批量场景：`toolOrchestration.ts` 对 tool_use 进行分区后并发或串行执行。

6. **结果回注与状态推进**  
   工具执行结果（`tool_result`）被拼入 `messages`。`State` 的 `transition` 设为 `Continue`，循环进入下一轮。

7. **Stop Hooks 与生命周期事件**  
   每轮结束时，`handleStopHooks` 可能执行：
   - 用户自定义 Stop hook（如 lint、测试、代码审查脚本）；
   - 队友模式的 `TaskCompleted` 和 `TeammateIdle` hook；
   - 后台 `executePromptSuggestion`、`executeAutoDream`、`extractMemories`（仅在非 bare 模式下）。

8. **轮次收敛与输出**  
   当模型不再请求工具，且 stop hooks 未阻止 continuation 时，assistant 的最终文本响应通过 TUI 或 SDK 返回给用户，当前查询结束。

## 五、关键技术原理详解

### 5.1 流式回退与容错

Claude Code 的流式消费并非"全有或全无"。当 SSE 连接异常断开或模型输出被截断时，`queryLoop` 能够：
- 保留已接收的消息块；
- 再次发起请求（必要时关闭 streaming）；
- 利用 `maxOutputTokensRecoveryCount` 和上下文压缩算法，尝试在 token 超限后恢复输出。

这种设计保证了在复杂长对话与网络抖动场景下，查询能够尽量自我恢复，而非直接向用户报错。

### 5.2 并发安全检测

工具并发不是盲目并行，而是通过 `isConcurrencySafe` 函数逐工具判断安全级别：
- 只读操作（如 `Grep`、`Read`、`WebFetch`）通常返回 `true`；
- 写操作（如 `Edit`、`Write`）或涉及外部状态变更的工具返回 `false`；
- 若解析失败或判断函数抛出异常，保守地视为不安全。

`StreamingToolExecutor` 进一步细化：即使某个安全工具在执行中，非安全工具仍会被阻塞，直到所有执行中的工具完成。

### 5.3 上下文压缩的四级恢复

当模型输出被 `max_output_tokens` 截断时，`query.ts` 依次尝试：
1. **Reactive Compact**：快速移除冗余上下文（如旧系统提示、过时的附件）；
2. **Micro Compact**：更激进地丢弃低优先级的消息；
3. **Snip Compact**：对超长消息进行分段截断（snip）；
4. **Context Collapse**：在极端情况下，将历史消息折叠为摘要。

这种分层策略避免了一上来就激进压缩导致信息丢失，同时确保在极端长对话中查询仍能继续。

### 5.4 Stop Hooks 的串并联执行

`stopHooks.ts` 中的 hooks 本质上是外部命令或脚本。系统通过生成器（`AsyncGenerator`）消费 hook 进度，支持：
- 非阻塞错误（`hook_non_blocking_error`）仅记录不影响 continuation；
- 阻塞错误（`blockingError`）会被当成 `tool_result` 注入对话，阻止模型继续；
- `preventContinuation` 机制允许 hook 显式中断当前查询（例如检测到严重 lint 错误时要求用户确认）。

## 六、设计亮点与创新

1. **State-as-a-Generator 模式**  
   `query.ts` 没有使用传统的回调地狱或 Promise 链，而是通过 `AsyncGenerator<StreamEvent | Message, State>` 将流式事件与状态转换统一在一个可恢复的时间线中。这种设计天然适配 React/Suspense 式的增量 UI 渲染，也便于测试和调试。

2. **即时并发与延迟串行的混合调度**  
   `StreamingToolExecutor` 不等所有 tool_use 收到就开始执行，最大限度压缩"模型思考结束"到"第一条命令开始运行"的空白时间；同时通过 batch 化与 sibling abort 保证并发正确性。

3. **后台化主会话**  
   将主会话查询当作一个可后台化的 agent task 来处理，突破了传统 CLI"一次一个前台命令"的交互范式，让用户可以在等待长任务的同时发起新的查询。

4. **桥接层的 worker 化**  
   本地 Claude Code 进程可以作为远端控制平面的 worker，支持 OAuth、Token 刷新、容量控制、CCR v2 注册等一系列企业级远程协作能力。

## 七、关键技术与依赖

- **Anthropic SDK**：`messageStream` 提供 SSE 流式消费基础。
- **Commander.js**：构建 CLI 参数体系与命令层次。
- **Ink + React**：TUI 渲染框架，`interactiveHelpers.tsx` 提供对话框、错误页等原子组件。
- **Async Hooks / AsyncLocalStorage**：`runWithAgentContext` 用于隔离后台任务与主会话的 agent 上下文。
- **Zod**：工具的 `inputSchema` 与 `safeParse` 支撑输入校验和 `isConcurrencySafe` 判断。
- **Bash 错误级联（siblingAbortController）**：手动实现的子控制器树，模拟了类似 Promise 取消语义的效果。

## 八、总结与反思

Agent Loop & Query Engine 的设计展示了如何在单进程 Node 应用中构建一个**高并发、可容错、多模式**的生产级 Agent 运行时。其核心哲学可以概括为三点：

- **流优先（Streaming-first）**：从 API 消费到工具执行再到 UI 渲染，都以流式事件为最小单元，最小化用户感知延迟；
- **防御性并发（Defensive Concurrency）**：并发不是默认开启，而是基于每个工具调用动态判断，配合 sibling abort 实现"快但安全"；
- **状态机显式化（Explicit State Machine）**：将原本隐式的对话轮次、上下文压缩次数、恢复尝试次数全部显式放入 `State`，让系统行为可预测、可追踪、可测试。

当然，这一模块也面临持续挑战：
- 随着工具数量和 hook 复杂度的增长，`query.ts` 的状态机逻辑有膨胀风险，未来可能需要进一步拆分为子 reducer；
- 远程桥接与本地后台任务引入了进程/上下文隔离的复杂度，`AsyncLocalStorage` 的调试与错误追踪不够直观；
- 上下文压缩策略虽然分层精细，但本质上仍是一种"权衡艺术"，在极端长对话中摘要与截断的信息损失难以完全消除。

总体而言，Agent Loop & Query Engine 是 Claude Code 能够从"一个对话式 CLI"进化为"可后台化、可远程化、可企业级部署的 Agent 平台"的关键基石。
