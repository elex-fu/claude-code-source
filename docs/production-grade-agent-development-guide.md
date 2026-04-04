# 生产级Agent开发实战 —— 基于 Claude Code 架构

> 基于 Claude Code 2.1.88 源码深度分析，总结从零构建生产级 AI Agent 的方法论

---

## 目录

1. [核心架构模式：PDAOL 循环](#一核心架构模式pdaol-循环)
2. [统一工具系统](#二统一工具系统toolts)
3. [流式工具执行器](#三流式工具执行器streamingtoolexecutor)
4. [分层权限系统](#四分层权限系统5层模型)
5. [状态管理架构](#五状态管理架构)
6. [多代理系统](#六多代理系统agenttool)
7. [MCP 协议集成](#七mcp-协议集成)
8. [从零构建生产级Agent的10个步骤](#八从零构建生产级agent的10个步骤)
9. [关键技术要点总结](#九关键技术要点总结)
10. [推荐阅读源码路径](#十推荐阅读源码路径)

---

## 一、核心架构模式：PDAOL 循环

Claude Code 采用 **Perception-Decision-Action-Observation-Loop** 架构：

```
用户输入 ──▶ [感知]构建系统提示 ──▶ [决策]调用Claude API
                   │                        │
                   ▼                        ▼
最终结果 ◀── [循环]继续? ◀── [观察]工具结果 ◀── [行动]执行工具
```

### 核心代码

文件：`src/query.ts:219-250`

```typescript
export async function* query(params: QueryParams): AsyncGenerator<...> {
  // 1. [感知] 构建系统提示
  const systemPrompt = buildSystemPrompt(...)

  // 2. [决策] 流式调用 Claude API
  const stream = await anthropic.messages.stream({
    model, messages, systemPrompt, tools, stream: true
  })

  // 3. [行动] 并行执行工具调用
  for await (const chunk of stream) {
    if (chunk.type === 'tool_use') {
      const executor = new StreamingToolExecutor(tools, canUseTool, context)
      executor.addTool(chunk, assistantMessage)

      // 4. [观察] 实时产出工具结果
      for await (const result of executor.getRemainingResults()) {
        yield result.message
      }
    }
    yield chunk
  }

  // 5. [循环] 检查是否继续
  if (hasMoreToolCalls(messages)) {
    yield* query({ ...params, messages })
  }
}
```

### 关键设计要点

- **使用 `AsyncGenerator`** 实现自然的背压控制
- **流式处理**从 API 到 UI 的全链路
- **支持并行**工具执行和取消信号

---

## 二、统一工具系统（Tool.ts）

所有工具遵循统一接口设计，文件：`src/Tool.ts:362-409`

```typescript
export type Tool<Input, Output, Progress> = {
  // 执行
  call(args, context, canUseTool, onProgress): Promise<ToolResult>

  // 权限检查
  checkPermissions(input, context): Promise<PermissionResult>

  // UI 渲染
  renderToolUseMessage(input, options): ReactNode
  renderToolResultMessage(result, options): ReactNode

  // 元信息
  isConcurrencySafe(input): boolean    // 是否可并行
  isReadOnly(input): boolean           // 是否只读
  isDestructive(input): boolean        // 是否破坏性
}
```

### 从零构建工具系统的步骤

#### 1. 定义工具基类/接口

所有工具必须实现相同的契约

#### 2. 实现权限检查

每个工具自行决定权限策略

#### 3. 分离渲染逻辑

工具结果渲染与执行分离

#### 4. 支持进度回调

长任务需要 `onProgress` 反馈

### 简化示例

```typescript
// 1. 定义工具接口
interface ITool {
  name: string
  inputSchema: z.ZodType
  async call(input, context): Promise<Result>
  checkPermissions(input): PermissionResult
}

// 2. 实现具体工具
class FileReadTool implements ITool {
  name = 'read'
  inputSchema = z.object({ path: z.string() })

  async call(input, context) {
    const content = await fs.readFile(input.path, 'utf-8')
    return { data: content }
  }

  checkPermissions(input) {
    // L1: 只读自动允许
    return { behavior: 'allow' }
  }

  isReadOnly() { return true }
  isConcurrencySafe() { return true }
}
```

---

## 三、流式工具执行器（StreamingToolExecutor）

这是 Claude Code 的核心创新之一，文件：`src/services/tools/StreamingToolExecutor.ts:40-130`

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private siblingAbortController: AbortController

  /**
   * 并发控制逻辑：
   * - 并发安全工具可并行执行
   * - 非并发工具独占执行
   */
  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executingTools = this.tools.filter(t => t.status === 'executing')
    return (
      executingTools.length === 0 ||
      (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
    )
  }

  /**
   * 添加工具到执行队列
   */
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const isConcurrencySafe = toolDefinition.isConcurrencySafe(parsedInput.data)
    this.tools.push({
      id: block.id,
      block,
      assistantMessage,
      status: 'queued',
      isConcurrencySafe,
      pendingProgress: [],
    })
    void this.processQueue()  // 开始处理
  }
}
```

### 并发控制策略

| 工具类型 | 示例 | 执行方式 |
|---------|------|---------|
| 安全只读 | Read, Grep, Glob | 并行执行 |
| 文件修改 | Edit, Write | 独占执行（串行）|
| 命令执行 | Bash | 独占执行（串行）|

### 错误级联处理

```typescript
// Bash 错误会取消兄弟进程
if (tool.block.name === BASH_TOOL_NAME && isErrorResult) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

---

## 四、分层权限系统（5层模型）

```
┌─────────────────────────────────────────────────────┐
│                    权限决策流程                       │
├─────────────────────────────────────────────────────┤
│  工具调用请求                                        │
│      ↓                                              │
│  ┌──────────────┐                                  │
│  │ validateInput│ ──▶ Zod 校验                      │
│  └────────┬─────┘                                  │
│           ↓                                         │
│  ┌──────────────────┐                              │
│  │ checkPermissions │                              │
│  └────────┬─────────┘                              │
│     ┌─────┼─────┬─────────┐                        │
│     ▼     ▼     ▼         ▼                        │
│  ┌────┐ ┌────┐ ┌────┐  ┌────┐                    │
│  │L1  │ │L2  │ │L3  │  │L4  │                    │
│  │自动│ │目录│ │模式│  │命令│                    │
│  │允许│ │隔离│ │区分│  │分析│                    │
│  └────┘ └────┘ └────┘  └────┘                    │
└─────────────────────────────────────────────────────┘
```

### 实现示例

文件：`src/tools/BashTool/bashPermissions.ts`

```typescript
export async function bashToolHasPermission(
  input: BashToolInput,
  context: ToolPermissionContext,
): Promise<PermissionResult> {
  // L1: 只读命令自动允许
  if (isReadOnlyCommand) {
    return { behavior: 'allow' }
  }

  // L2: 目录隔离检查
  if (!isWithinWorkingDirectory(input.command)) {
    return { behavior: 'ask', reason: 'outside_project' }
  }

  // L3: 模式区分（ask/notify/plan）
  if (context.mode === 'ask') {
    return { behavior: 'ask' }
  }

  // L4: 命令语义分析
  const classification = await classifyBashCommand(input.command)
  if (classification.behavior === 'deny') {
    return { behavior: 'deny', reason: 'dangerous_command' }
  }

  return { behavior: 'allow' }
}
```

---

## 五、状态管理架构

```
┌─────────────────────────────────────────────────────────────┐
│                      状态管理架构                            │
├─────────────────────────────────────────────────────────────┤
│  UI 层 (React)          State 层 (React)      Global 层    │
│  ┌─────────────┐       ┌──────────────┐      ┌──────────┐  │
│  │ 组件        │◀─────▶│ AppState.tsx │◀────▶│bootstrap │  │
│  │ useAppState │       │  状态树       │      │ /state.ts│  │
│  └─────────────┘       └──────────────┘      └──────────┘  │
│         │                     │                   │        │
│         │ 订阅状态切片         │ 函数式更新         │ Signal │
│         ▼                     ▼                   ▼        │
│  选择性重渲染              会话级状态          跨会话持久    │
│  (React Compiler)          messages           totalCost    │
│                            tasks              projectRoot  │
└─────────────────────────────────────────────────────────────┘
```

### 关键实现

文件：`src/bootstrap/state.ts`

```typescript
// Signal 信号系统，跨会话持久
type GlobalState = {
  totalCost: number           // 累计费用
  sessionId: string          // 会话ID
  projectRoot: string        // 项目根目录
  // ... 严格控制的全局状态
}

// 警示注释：DO NOT ADD MORE STATE HERE
```

---

## 六、多代理系统（AgentTool）

Claude Code 内置 6 种专业子代理，文件：`src/tools/AgentTool/builtInAgents.ts`

| 代理类型 | 用途 | 限制 |
|---------|------|------|
| `Explore` | 代码库探索 | 只读，禁用编辑工具 |
| `Plan` | 架构规划 | 不执行，只输出方案 |
| `general-purpose` | 通用任务 | 全套工具 |
| `claude-code-guide` | 帮助咨询 | 无文件操作 |
| `verification` | 验证检查 | 只读验证 |

### Fork 子代理机制

文件：`src/tools/AgentTool/forkSubagent.ts`

```typescript
/**
 * Fork 子代理继承父代理的完整上下文
 * 通过 prompt cache 共享优化性能
 */
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): Message[] {
  // 1. 保留完整父助手消息
  const fullAssistantMessage = { ...assistantMessage }

  // 2. 为每个 tool_use 创建占位结果
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 3. 添加子代理指令
  return [fullAssistantMessage, toolResultMessage]
}
```

### Explore Agent 示例

文件：`src/tools/AgentTool/built-in/exploreAgent.ts`

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases...',
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  source: 'built-in',
  baseDir: 'built-in',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

---

## 七、MCP 协议集成

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 用户配置    │────▶│ MCPClient   │────▶│MCPTransport │────▶│ MCP Server  │
│settings.json│     │  (初始化)   │     │stdio/http   │     │ (外部进程)  │
└─────────────┘     └──────┬──────┘     └─────────────┘     └──────┬──────┘
                           │                                        │
                           │ ① 读取配置                            │
                           │ ② 启动进程/连接                       │
                           │ ③ capability协商                      │
                           │◀─────────────────────────────────────▶│
                           │ ④ listTools()                        │
┌─────────────┐     ┌──────┴──────┐     ┌─────────────┐            │
│ Claude使用  │◀────│wrapMCPTool  │◀────│ Tool注册中心 │◀───────────┘
│ MCP工具     │     │(动态包装)   │     │  tools数组   │   调用工具
└─────────────┘     └─────────────┘     └─────────────┘
```

### 动态工具注册

文件：`src/services/mcp/client.ts`

```typescript
export async function getMcpToolsCommandsAndResources(
  connections: MCPServerConnection[],
): Promise<McpToolsCommandsAndResourcesResult> {
  const tools: Tool[] = []

  for (const connection of connections) {
    // 连接到 MCP 服务器
    const client = await ensureConnectedClient(connection)

    // 获取工具列表
    const { tools: mcpTools } = await client.listTools()

    // 包装为标准 Tool 接口
    for (const mcpTool of mcpTools) {
      tools.push(wrapMCPTool(connection.serverName, mcpTool, client))
    }
  }

  return { tools }
}
```

---

## 八、从零构建生产级Agent的10个步骤

基于 Claude Code 源码分析，总结从零构建的方法：

### 1. 选择基础架构

推荐：AsyncGenerator 实现流式处理

```typescript
async function* agentLoop(userInput: string): AsyncGenerator<StreamEvent> {
  const messages = [{ role: 'user', content: userInput }]

  while (true) {
    // 调用 LLM API
    const stream = await llm.chat.completions.create({
      messages, tools, stream: true
    })

    // 处理流式响应
    for await (const chunk of stream) {
      yield chunk

      if (chunk.type === 'tool_use') {
        // 执行工具
        const result = await executeTool(chunk)
        messages.push(result)
      }
    }

    // 检查是否继续
    if (!hasToolCalls(messages)) break
  }
}
```

### 2. 设计工具接口

```typescript
interface Tool {
  name: string
  description: string
  parameters: JSONSchema
  execute(args: unknown): Promise<unknown>
  checkPermissions(args: unknown): PermissionResult
}
```

### 3. 实现权限系统

- L1：只读操作自动允许
- L2：工作目录隔离
- L3：用户模式（ask/notify/auto）
- L4：命令语义分析
- L5：破坏性操作确认

### 4. 构建并发控制

```typescript
class ToolExecutor {
  private queue: ToolCall[] = []

  async executeParallel(tools: ToolCall[]) {
    const safeTools = tools.filter(t => t.isConcurrencySafe)
    const unsafeTools = tools.filter(t => !t.isConcurrencySafe)

    // 安全工具并行执行
    await Promise.all(safeTools.map(t => this.runTool(t)))

    // 非安全工具串行执行
    for (const tool of unsafeTools) {
      await this.runTool(tool)
    }
  }
}
```

### 5. 实现状态管理

```typescript
// 会话级状态
interface SessionState {
  messages: Message[]
  tools: Tool[]
  cost: number
}

// 全局状态
interface GlobalState {
  totalCost: number
  settings: Settings
}
```

### 6. 添加流式UI

```typescript
// 使用 React + Ink（CLI）或 React（Web）
function AgentUI() {
  const { messages, isLoading } = useAgentState()

  return (
    <div>
      {messages.map(m => <Message key={m.id} data={m} />)}
      {isLoading && <Spinner />}
    </div>
  )
}
```

### 7. 集成 MCP 协议

```typescript
// 支持外部工具扩展
const mcpClient = new MCPClient({
  transport: new StdioTransport('mcp-server')
})

const tools = await mcpClient.listTools()
// 动态添加到工具注册表
```

### 8. 实现自动压缩

```typescript
// 上下文窗口管理
if (tokenCount > threshold) {
  const summary = await generateSummary(messages)
  messages = [summary, ...recentMessages]
}
```

### 9. 添加可观测性

```typescript
// OpenTelemetry 集成
const tracer = opentelemetry.trace.getTracer('agent')

async function executeWithTracing(tool, args) {
  return tracer.startActiveSpan(tool.name, async span => {
    try {
      const result = await tool.execute(args)
      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (error) {
      span.recordException(error)
      throw error
    }
  })
}
```

### 10. 多代理协调

```typescript
// 子代理系统
class AgentTool implements Tool {
  async execute(args) {
    const subagent = createSubagent({
      type: args.agentType,
      systemPrompt: getAgentPrompt(args.agentType),
      tools: getAgentTools(args.agentType)
    })

    // 后台执行
    const task = await this.taskManager.spawn(subagent, args.prompt)

    return { taskId: task.id }
  }
}
```

---

## 九、关键技术要点总结

| 技术领域 | Claude Code 做法 | 小白理解 |
|---------|-----------------|---------|
| **Agent循环** | AsyncGenerator + 流式 | 用生成器让AI"边想边说" |
| **工具系统** | 统一接口 + 元信息标注 | 每个工具要报告"我是否安全" |
| **权限模型** | 5层分级 | 像洋葱一样层层保护 |
| **并发控制** | 安全工具并行，危险工具串行 | 读文件可以一起，写文件要排队 |
| **状态管理** | 双层（React + Signal） | UI状态和全局数据分开管 |
| **多代理** | Fork机制 + Prompt缓存共享 | 克隆一个AI助手来帮忙 |
| **MCP集成** | 动态工具包装 | 外挂技能系统 |
| **压缩系统** | Token阈值触发摘要 | 聊天记录太多就自动总结 |
| **可观测性** | OpenTelemetry全链路 | 随时能看到AI在干什么 |

---

## 十、推荐阅读源码路径

| 模块 | 文件路径 | 学习重点 |
|------|---------|---------|
| 工具基类 | `src/Tool.ts` | 统一接口设计 |
| Agent循环 | `src/query.ts` | 流式处理架构 |
| 工具执行器 | `src/services/tools/StreamingToolExecutor.ts` | 并发控制 |
| 权限系统 | `src/tools/BashTool/bashPermissions.ts` | 安全策略 |
| 子代理 | `src/tools/AgentTool/forkSubagent.ts` | 多代理实现 |
| MCP客户端 | `src/services/mcp/client.ts` | 外部工具集成 |
| 状态管理 | `src/bootstrap/state.ts` | 全局状态设计 |
| 任务系统 | `src/Task.ts` | 后台任务管理 |
| 压缩系统 | `src/services/compact/autoCompact.ts` | 上下文管理 |
| 消息处理 | `src/utils/messages.ts` | 消息规范化 |

---

## 总结

这份分析基于 Claude Code 约 50 万行 TypeScript 源码，展示了当前最先进的 Agent 工程化实践。

### 核心思想

1. **分层架构** - 清晰的职责分离
2. **统一接口** - 所有工具遵循相同契约
3. **安全第一** - 5层权限模型确保边界
4. **流式体验** - 从 API 到 UI 全链路流式处理

### 工程化要点

- **类型安全** - TypeScript 严格模式 + Zod 运行时校验
- **测试覆盖** - 单元测试 + 集成测试 + E2E 测试
- **可观测性** - OpenTelemetry 追踪/指标/日志
- **性能优化** - 提示缓存（减少 90%+ Token 费用）
- **错误处理** - 分类重试、熔断机制、优雅降级

---

> 声明：本分析文档基于 Claude Code npm 包内的 source map 还原的源码（版本 2.1.88），仅供技术研究和学习使用。
