# Claude Code Query Engine 技术分析

## 1. 模块总结介绍

Query Engine 是 Claude Code 的「查询执行中枢」，负责协调 LLM 查询的整个生命周期，包括消息准备、API 调用、流式响应处理和结果后处理。该模块是 Agent Loop 的核心依赖，实现了复杂的查询编排逻辑。

核心功能：
- **消息规范化**：将内部消息格式转换为 API 格式
- **系统提示词组装**：动态构建 System Prompt
- **工具配置**：根据权限和上下文配置可用工具
- **流式处理**：实时处理和展示 LLM 输出
- **错误处理**：重试、降级、优雅失败

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                       Query Engine                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  Message Prep   │  │  System Prompt  │  │   Tools     │ │
│  │                 │  │   Assembly      │  │  Config     │ │
│  │ • Normalization │  │                 │  │             │ │
│  │ • Attachment    │──▶ • CLAUDE.md     │──▶ • Filter    │ │
│  │   Injection     │    • Skills       │    • Convert    │ │
│  │ • Compact       │    • Dynamic      │    • Schema     │ │
│  │   Recovery      │      Boundary     │                  │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬──────┘ │
│           │                    │                  │        │
│           └────────────────────┼──────────────────┘        │
│                                ▼                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Anthropic API Client                    │  │
│  │                                                      │  │
│  │  • Streaming Response  • Error Handling             │  │
│  │  • Retry Logic         • Usage Tracking             │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. 查询执行流程

### 3.1 主查询流程

```typescript
// QueryEngine.ts
export async function* query(
  params: QueryParams
): AsyncGenerator<QueryEvent, QueryResult, unknown> {
  // 1. 准备消息
  const messages = prepareMessages(params)
  yield { type: 'messages_prepared', messageCount: messages.length }

  // 2. 构建系统提示词
  const systemPrompt = await buildSystemPrompt(params)
  yield { type: 'system_prompt_ready', tokenCount: systemPrompt.length }

  // 3. 配置工具
  const tools = configureTools(params.toolUseContext)
  yield { type: 'tools_configured', toolCount: tools.length }

  // 4. 调用 API
  const response = await callAnthropicAPI({
    model: params.model,
    messages,
    system: systemPrompt,
    tools,
    stream: true,
  })

  // 5. 流式处理响应
  for await (const chunk of response) {
    yield* processStreamChunk(chunk)
  }

  // 6. 返回最终结果
  return { type: 'complete', usage: response.usage }
}
```

### 3.2 消息准备

```typescript
function prepareMessages(params: QueryParams): Message[] {
  let messages = [...params.messages]

  // 1. 规范化消息格式
  messages = normalizeMessagesForAPI(messages)

  // 2. 注入附件
  messages = injectAttachments(messages, params.attachments)

  // 3. 应用紧凑边界（如果存在）
  if (params.compactBoundary) {
    messages = applyCompactBoundary(messages, params.compactBoundary)
  }

  // 4. 确保工具调用配对
  messages = ensureToolResultPairing(messages)

  // 5. 限制图片大小
  messages = limitImageSizes(messages)

  return messages
}
```

## 4. 系统提示词构建

### 4.1 多层组装

```typescript
async function buildSystemPrompt(params: QueryParams): Promise<string> {
  const parts: string[] = []

  // 1. 基础系统提示词
  parts.push(getBaseSystemPrompt())

  // 2. CLAUDE.md 内容
  const claudeMd = await loadClaudeMdContext()
  if (claudeMd) {
    parts.push(claudeMd)
  }

  // 3. 动态边界标记（用于缓存）
  parts.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)

  // 4. 运行时上下文
  parts.push(buildRuntimeContext(params))

  // 5. 工具描述
  parts.push(formatToolDescriptions(params.tools))

  // 6. 技能提示词
  const skillsPrompt = await loadSkillsPrompt()
  if (skillsPrompt) {
    parts.push(skillsPrompt)
  }

  return parts.join('\n\n')
}
```

### 4.2 CLAUDE.md 加载

```typescript
async function loadClaudeMdContext(): Promise<string | null> {
  const cwd = getCwd()

  // 从当前目录向上查找所有 CLAUDE.md
  const claudeMdFiles = await findClaudeMdFiles(cwd)

  const contexts: string[] = []
  for (const file of claudeMdFiles) {
    const content = await readFile(file, 'utf-8')
    contexts.push(`<!-- ${file} -->\n${content}`)
  }

  return contexts.join('\n\n')
}
```

### 4.3 动态边界与缓存

```typescript
// 用于分割静态和动态内容的特殊标记
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '<!-- DYNAMIC_CONTEXT_BOUNDARY -->'

// 构建带缓存控制标记的提示词
function buildCacheAwarePrompt(parts: SystemPromptPart[]): ContentBlock[] {
  return parts.map((part, index) => ({
    type: 'text',
    text: part.content,
    cache_control:
      index === parts.length - 1
        ? { type: 'ephemeral' } // 最后一部分启用缓存
        : undefined,
  }))
}
```

## 5. 工具配置

### 5.1 工具过滤

```typescript
function configureTools(context: ToolUseContext): Tool[] {
  let tools = getAllAvailableTools()

  // 1. 应用权限过滤
  tools = tools.filter((tool) =>
    hasPermissionToUseTool(tool.name, context.permissionContext)
  )

  // 2. 应用显式允许列表
  if (context.allowedTools) {
    tools = tools.filter((tool) =>
      context.allowedTools!.some((allowed) => toolMatchesName(tool, allowed))
    )
  }

  // 3. 应用禁止列表
  if (context.disallowedTools) {
    tools = tools.filter(
      (tool) =>
        !context.disallowedTools!.some((disallowed) =>
          toolMatchesName(tool, disallowed)
        )
    )
  }

  // 4. MCP 工具注入
  const mcpTools = getMCPTools()
  tools = [...tools, ...mcpTools]

  return tools
}
```

### 5.2 工具描述格式化

```typescript
function formatToolDescriptions(tools: Tool[]): string {
  return tools
    .map((tool) => {
      const params = Object.entries(tool.parameters.properties)
        .map(([name, schema]) => {
          const required = tool.parameters.required?.includes(name)
          return `  ${name}: ${schema.type}${required ? ' (required)' : ''}`
        })
        .join('\n')

      return `### ${tool.name}
${tool.description}

Parameters:
${params}`
    })
    .join('\n\n')
}
```

## 6. 流式响应处理

### 6.1 事件解析

```typescript
async function* processStreamChunk(
  chunk: MessageStreamEvent
): AsyncGenerator<QueryEvent> {
  switch (chunk.type) {
    case 'message_start':
      yield {
        type: 'message_start',
        messageId: chunk.message.id,
        usage: chunk.message.usage,
      }
      break

    case 'content_block_start':
      yield {
        type: 'content_block_start',
        blockType: chunk.content_block.type,
      }
      break

    case 'content_block_delta':
      if (chunk.delta.type === 'text_delta') {
        yield {
          type: 'text_delta',
          text: chunk.delta.text,
        }
      } else if (chunk.delta.type === 'thinking_delta') {
        yield {
          type: 'thinking_delta',
          thinking: chunk.delta.thinking,
        }
      }
      break

    case 'content_block_stop':
      yield { type: 'content_block_stop' }
      break

    case 'message_delta':
      yield {
        type: 'usage_update',
        usage: chunk.usage,
      }
      break

    case 'message_stop':
      yield { type: 'message_stop' }
      break
  }
}
```

### 6.2 Tool Use 检测

```typescript
function detectToolUse(content: ContentBlock[]): ToolUseBlock[] {
  return content
    .filter((block): block is ToolUseBlock => block.type === 'tool_use')
    .map((block) => ({
      id: block.id,
      name: block.name,
      input: block.input,
    }))
}
```

## 7. 错误处理与重试

### 7.1 错误分类

```typescript
// services/api/errors.ts
export function categorizeRetryableAPIError(error: unknown): ErrorCategory {
  if (!isAnthropicError(error)) return 'fatal'

  switch (error.status) {
    case 429: // Rate limit
      return 'rate_limited'

    case 500: // Server error
    case 502: // Bad gateway
    case 503: // Service unavailable
      return 'retryable'

    case 400: // Bad request
      if (error.message.includes('context limit')) {
        return 'context_too_long'
      }
      return 'fatal'

    case 401: // Unauthorized
    case 403: // Forbidden
      return 'auth_error'

    default:
      return 'unknown'
  }
}
```

### 7.2 重试逻辑

```typescript
// services/api/withRetry.ts
export async function withRetry<T>(
  operation: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const { maxRetries = 3, baseDelay = 1000 } = options

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      const category = categorizeRetryableAPIError(error)

      if (category === 'fatal' || attempt === maxRetries) {
        throw error
      }

      if (category === 'rate_limited') {
        const retryAfter = extractRetryAfter(error) || 60
        await sleep(retryAfter * 1000)
      } else if (category === 'retryable') {
        const delay = baseDelay * Math.pow(2, attempt) // 指数退避
        await sleep(delay)
      }
    }
  }

  throw new Error('Retry attempts exhausted')
}
```

## 8. 上下文压缩集成

### 8.1 Token 估算

```typescript
// services/tokenEstimation.ts
export function roughTokenCountEstimation(text: string): number {
  // 粗略估算：1 token ≈ 4 字符（英文）
  return Math.ceil(text.length / 4)
}

export function roughTokenCountEstimationForMessages(
  messages: Message[]
): number {
  return messages.reduce((total, msg) => {
    const contentLength =
      typeof msg.content === 'string'
        ? msg.content.length
        : JSON.stringify(msg.content).length
    return total + roughTokenCountEstimation(contentLength.toString())
  }, 0)
}
```

### 8.2 压缩触发

```typescript
async function maybeCompactContext(
  messages: Message[],
  model: string
): Promise<CompactResult | null> {
  const estimatedTokens = roughTokenCountEstimationForMessages(messages)
  const contextWindow = getContextWindowForModel(model)
  const safetyThreshold = contextWindow - 13000 // 安全缓冲

  if (estimatedTokens < safetyThreshold) {
    return null // 无需压缩
  }

  // 触发压缩
  const compactResult = await executeCompact(messages, {
    targetTokens: safetyThreshold,
  })

  return compactResult
}
```

## 9. 模型选择

### 9.1 模型解析

```typescript
// utils/model/model.ts
export function parseUserSpecifiedModel(input: string): ModelSetting | null {
  const modelMap: Record<string, string> = {
    opus: 'claude-opus-4-6',
    sonnet: 'claude-sonnet-4-6',
    haiku: 'claude-haiku-4-5',
  }

  const modelId = modelMap[input.toLowerCase()] || input

  if (!isValidModel(modelId)) {
    return null
  }

  return {
    id: modelId,
    name: getModelName(modelId),
    contextWindow: getContextWindowForModel(modelId),
  }
}
```

### 9.2 动态模型切换

```typescript
export function getMainLoopModel(): ModelSetting {
  // 1. 用户显式指定
  const userModel = getUserSpecifiedModel()
  if (userModel) return userModel

  // 2. 设置中的默认模型
  const settingsModel = getSettings().defaultModel
  if (settingsModel) return parseUserSpecifiedModel(settingsModel)!

  // 3. 回退到默认
  return parseUserSpecifiedModel('sonnet')!
}
```

## 10. 创新点与反思

### 10.1 设计创新

1. **Generator-based Streaming**：使用 AsyncGenerator 实现真正的流式处理
2. **模块化组装**：System Prompt 分层组装，支持缓存优化
3. **动态工具配置**：基于权限上下文的运行时工具过滤

### 10.2 性能优化

1. **消息规范化缓存**：避免重复规范化相同消息
2. **并发预加载**：API 预连接、提示词并行加载
3. **增量更新**：只更新变化的工具配置

### 10.3 未来演进

1. **智能模型路由**：根据查询复杂度自动选择模型
2. **请求批处理**：合并短时间内的多个请求
3. **预测性压缩**：基于趋势预测提前压缩上下文

---

Claude Code 的 Query Engine 是一个功能丰富的查询编排系统。通过分层架构、流式处理、智能重试等设计，实现了高效可靠的 LLM 查询执行。理解其设计，对于构建任何需要与 LLM 深度集成的应用都具有参考价值。
