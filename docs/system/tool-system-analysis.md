# Claude Code 工具系统技术深度解析

## 1. 模块总结介绍

Claude Code 的工具系统（Tool System）是整个 AI 代理架构的核心执行层，负责将模型的意图转化为实际的操作。它提供了一套完整的工具定义、注册、执行、权限控制和结果处理机制。工具系统的设计理念是"fail-closed"（默认拒绝），确保在没有明确授权的情况下不会执行潜在危险的操作。

该模块涵盖了从底层工具抽象（`Tool` 接口）到高层工具注册表（`tools.ts`），从同步执行到流式并发，从本地工具到 MCP（Model Context Protocol）远程工具的全方位支持。它是 Claude Code 能够安全、高效地与文件系统、网络、子代理交互的关键基础设施。

## 2. 系统架构

工具系统采用分层架构设计，各层职责清晰：

```
┌─────────────────────────────────────────────────────────────────┐
│                     应用层 (Application)                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│  │   AgentTool  │ │  BashTool    │ │    FileEditTool          │ │
│  └──────────────┘ └──────────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                     注册层 (Registry)                            │
│                    tools.ts - getAllBaseTools()                  │
│         工具发现、条件加载、MCP 工具整合                          │
├─────────────────────────────────────────────────────────────────┤
│                     执行层 (Execution)                           │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐ │
│  │ toolExecution.ts │  │    StreamingToolExecutor.ts          │ │
│  │   主执行流程      │  │    并发流式执行器                     │ │
│  └──────────────────┘  └──────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                     权限层 (Permission)                          │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐ │
│  │ useCanUseTool.ts │  │    permissions.ts                    │ │
│  │   权限检查 Hook   │  │    规则引擎 + 自动分类器              │ │
│  └──────────────────┘  └──────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                     抽象层 (Abstraction)                         │
│                     Tool.ts - 核心接口定义                        │
│         Tool<Input, Output, P> 泛型接口 + buildTool 工厂          │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 分层说明

### 3.1 抽象层（Tool.ts）

核心接口 `Tool<Input, Output, P>` 定义了工具的完整契约：

```typescript
export type Tool<Input extends AnyObject, Output, P extends ToolProgressData> = {
  name: string
  call(args: z.infer<Input>, context: ToolUseContext, canUseTool: CanUseToolFn, ...): Promise<ToolResult<Output>>
  inputSchema: Input  // Zod schema
  checkPermissions(input: z.infer<Input>, context: ToolUseContext): Promise<PermissionResult>
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam
  renderToolResultMessage?(content: Output, ...): React.ReactNode
  renderToolUseMessage(input: Partial<z.infer<Input>>, ...): React.ReactNode
  // ... 其他渲染和元数据方法
}
```

`buildTool` 工厂函数提供 fail-closed 默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 默认不安全，需显式声明
  isReadOnly: () => false,         // 默认非只读，需显式声明
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',  // 默认跳过自动分类器
  userFacingName: () => '',
}
```

### 3.2 注册层（tools.ts）

`getAllBaseTools()` 是所有可用工具的单一事实来源：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    WebSearchTool,
    // ... 条件加载的工具
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, ...] : []),
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
    // MCP 工具通过 assembleToolPool 动态整合
  ]
}
```

`assembleToolPool()` 合并内置工具和 MCP 工具：

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // 按名称排序确保提示缓存稳定性，内置工具优先
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

### 3.3 执行层

**toolExecution.ts** - 主执行流程 `runToolUse`：

```typescript
async function* runToolUse(...): AsyncGenerator<MessageUpdateLazy> {
  // 1. 输入验证（Zod schema）
  const parsedInput = tool.inputSchema.safeParse(toolUse.input)
  
  // 2. 执行 PreToolUse Hooks
  for await (const hookResult of runPreToolUseHooks(...)) { ... }
  
  // 3. 权限检查（checkPermissionsAndCallTool）
  const { decision, input: permissionInput } = await resolveHookPermissionDecision(...)
  
  // 4. 执行工具调用
  const toolResult = await tool.call(...)
  
  // 5. 执行 PostToolUse Hooks
  for await (const hookResult of runPostToolUseHooks(...)) { ... }
  
  // 6. 结果映射和持久化
  const resultBlock = tool.mapToolResultToToolResultBlockParam(toolResult.data, toolUseID)
  
  // 7. 分析日志和遥测
  logEvent('tengu_tool_executed', { ... })
}
```

**StreamingToolExecutor.ts** - 并发流式执行器：

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

执行器将工具调用分区为：
- **并发安全批次**：多个只读工具（如 Glob、Grep、Read）可并行执行
- **串行批次**：写入工具或 Bash 命令需顺序执行，Bash 错误会中止同批次兄弟任务

### 3.4 权限层

**permissions.ts** - 规则引擎核心 `hasPermissionsToUseTool`：

```typescript
export async function hasPermissionsToUseTool(...): Promise<PermissionDecision> {
  // 1. 检查显式规则（allow/deny/ask）
  const ruleCheck = checkRuleBasedPermissions(tool, input, context)
  if (ruleCheck?.behavior === 'deny') return ruleCheck
  if (ruleCheck?.behavior === 'ask') return ruleCheck
  
  // 2. 特殊模式处理（acceptEdits、bypassPermissions、dontAsk）
  if (mode === 'acceptEdits' && isEditTool(tool)) {
    return { behavior: 'allow', decisionReason: { type: 'mode', mode: 'acceptEdits' } }
  }
  
  // 3. 自动模式分类器（TRANSCRIPT_CLASSIFIER feature）
  if (mode === 'auto') {
    const classification = await classifyYoloAction(...)
    // 根据分类结果决定 allow/ask
  }
  
  // 4. 交互式权限提示
  return canUseTool(tool, input, context, assistantMessage, toolUseID)
}
```

权限决策类型（types/permissions.ts）：

```typescript
export type PermissionDecision<Input> =
  | PermissionAllowDecision<Input>   // { behavior: 'allow', ... }
  | PermissionAskDecision<Input>     // { behavior: 'ask', message, ... }
  | PermissionDenyDecision           // { behavior: 'deny', message, ... }
```

## 4. 交互流程

### 4.1 单次工具调用完整流程

```
模型输出 tool_use
       ↓
┌──────────────────┐
│ 1. 查找工具      │ ← findToolByName(tools, toolUse.name)
└──────────────────┘
       ↓
┌──────────────────┐
│ 2. 输入验证      │ ← tool.inputSchema.safeParse(input)
└──────────────────┘
       ↓
┌──────────────────┐
│ 3. PreTool Hooks │ ← executePreToolHooks(...) 
│    执行前置钩子   │    (可修改输入、阻止执行、设置权限行为)
└──────────────────┘
       ↓
┌──────────────────┐
│ 4. 权限检查      │ ← checkPermissions() + 规则引擎
│    决策流程      │    (allow → 继续, deny → 拒绝, ask → 提示用户)
└──────────────────┘
       ↓
┌──────────────────┐
│ 5. 执行工具      │ ← tool.call(input, context, canUseTool, ...)
│    核心逻辑      │    (可能产生进度更新 yield progress)
└──────────────────┘
       ↓
┌──────────────────┐
│ 6. PostTool Hooks│ ← executePostToolHooks(...)
│    执行后置钩子   │    (可修改输出、阻止继续、添加上下文)
└──────────────────┘
       ↓
┌──────────────────┐
│ 7. 结果处理      │ ← mapToolResultToToolResultBlockParam()
│    映射与持久化   │    大结果 → 持久化到磁盘，发送预览
└──────────────────┘
       ↓
┌──────────────────┐
│ 8. 遥测记录      │ ← logEvent('tengu_tool_executed', ...)
└──────────────────┘
       ↓
返回 tool_result 给模型
```

### 4.2 并发执行流程

```
tool_use_1 (Read) ──┐
tool_use_2 (Glob) ──┼──→ 并发批次 1 [全部并发安全] ──→ 并行执行
                    │                              └── 进度流式输出
tool_use_3 (Bash) ──┼──→ 串行批次 2 [非并发安全] ───→ 顺序执行
                    │                              └── Bash 错误中止批次
tool_use_4 (Edit) ──┘
```

## 5. 技术原理

### 5.1 大结果持久化机制

当工具输出超过 `maxResultSizeChars`（默认 30,000 字符）时，系统会将结果持久化到磁盘：

```typescript
// utils/toolResultStorage.ts
export async function persistToolResult(
  content: NonNullable<ToolResultBlockParam['content']>,
  toolUseId: string,
): Promise<PersistedToolResult | PersistToolResultError> {
  const filepath = getToolResultPath(toolUseId, isJson)
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
  
  return {
    filepath,
    originalSize: contentStr.length,
    preview: generatePreview(contentStr, PREVIEW_SIZE_BYTES),
    hasMore: contentStr.length > PREVIEW_SIZE_BYTES,
  }
}

// 构建包含预览的消息
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  return `<persisted-output>
Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}

Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):
${result.preview}
...
</persisted-output>`
}
```

### 5.2 子代理上下文隔离

AgentTool 通过 `createSubagentContext` 创建隔离的执行上下文：

```typescript
// utils/forkedAgent.ts
export function createSubagentContext(parentContext: ToolUseContext): ToolUseContext {
  return {
    ...parentContext,
    agentId: createAgentId(),
    // 隔离状态更新，防止子代理影响父状态
    setAppState: () => {}, // no-op for subagents
    // 本地拒绝追踪，用于异步代理
    localDenialTracking: createLocalDenialTracking(),
    // 保留工具结果用于用户查看
    preserveToolUseResults: parentContext.preserveToolUseResults,
  }
}
```

### 5.3 MCP 工具整合

MCP 工具通过前缀命名空间与内置工具隔离：

```typescript
// 默认模式：mcp__serverName__toolName
// 无前缀模式（CLAUDE_AGENT_SDK_MCP_NO_PREFIX）：toolName

// 工具查找时考虑 MCP 信息
export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}

// MCP 工具包含元数据
mcpInfo?: { serverName: string; toolName: string }
```

## 6. 创新点

### 6.1 Fail-Closed 默认策略

`buildTool` 的默认设置确保安全：
- `isConcurrencySafe: false` - 必须显式声明并发安全
- `isReadOnly: false` - 必须显式声明只读
- `checkPermissions: allow` - 但需通过通用权限系统复核

### 6.2 流式并发执行

`StreamingToolExecutor` 实现了真正的流式并发：
- 并发安全的只读工具（Read、Glob、Grep）并行执行
- 写入工具和 Bash 命令串行化
- Bash 错误自动中止同批次兄弟任务，防止无效操作

### 6.3 智能权限模式

- **acceptEdits 模式**：自动批准文件编辑，适合代码重构工作流
- **auto 模式**：基于 AI 分类器自动决策低风险投资
- **deny 规则优先**：即使 hooks 批准，deny 规则仍可覆盖

### 6.4 工具结果预算管理

通过 `contentReplacementState` 实现跨会话的工具结果预算，防止上下文被大结果填满。

## 7. 关键技术

### 7.1 类型安全的工具定义

```typescript
// 使用 Zod 进行运行时输入验证
const inputSchema = z.object({
  command: z.string(),
  timeout: z.number().optional(),
})

// 完整的泛型类型推导
type Input = z.infer<typeof inputSchema>
type Output = { output: string; exitCode: number }

const tool: Tool<typeof inputSchema, Output, BashProgress> = buildTool({
  name: 'Bash',
  inputSchema,
  async call(input, context, canUseTool) {
    // 类型安全的输入访问
    const { command, timeout } = input
    // ... 执行逻辑
    return { data: { output, exitCode } }
  },
})
```

### 7.2 权限规则匹配

支持通配符和工具特定的匹配逻辑：

```typescript
// Bash 工具支持命令模式匹配
preparePermissionMatcher(input: BashInput): Promise<(pattern: string) => boolean> {
  return (pattern: string) => {
    // 支持 "Bash(git *)" 模式匹配 git 相关命令
    return matchesPattern(input.command, pattern)
  }
}
```

### 7.3 进度流式传输

```typescript
export type ToolCallProgress<P extends ToolProgressData> = (
  progress: ToolProgress<P>,
) => void

// Bash 工具实时输出
async call(input, context, canUseTool, parentMessage, onProgress) {
  const process = spawn(command)
  process.stdout.on('data', (data) => {
    onProgress?.({
      toolUseID,
      data: { type: 'bash_progress', output: data.toString() },
    })
  })
}
```

## 8. 思考总结

Claude Code 的工具系统展现了生产级 AI 代理系统的设计智慧：

**安全优先**：通过 fail-closed 默认值、多层权限检查、deny 规则覆盖机制，确保系统在复杂场景下仍能保持安全边界。

**性能优化**：流式并发执行、工具结果持久化、提示缓存稳定性（通过工具排序），在保证安全的同时最大化执行效率。

**可扩展性**：MCP 协议支持、hooks 系统、条件工具加载，使系统能够适应从简单脚本到复杂多代理协作的各种场景。

**类型安全**：完整的 TypeScript 泛型支持，从输入验证到结果映射都保持类型安全，减少运行时错误。

**运维友好**：详细的遥测日志、Perfetto 追踪、工具结果持久化，为问题排查和性能优化提供充足的数据支持。

该系统的核心设计哲学是**显式优于隐式**：工具必须显式声明其安全属性（并发安全、只读、破坏性），权限必须显式配置，结果必须显式处理。这种设计虽然增加了工具开发者的负担，但大大降低了系统整体的安全风险。
