# Claude Code Compact Service 技术分析

## 1. 模块总结介绍

Compact Service 是 Claude Code 的「上下文压缩引擎」，负责在对话历史接近上下文窗口限制时，自动压缩旧消息以释放 Token 空间。该服务是 Claude Code 能够处理超长对话的核心机制。

核心功能：
- **多级压缩策略**：Snip、Micro、Auto、Context Collapse 四级压缩
- **Forked Agent 摘要**：使用子智能体生成高质量摘要
- **后压缩恢复**：自动恢复关键文件到上下文
- **渐进降级**：从最小损失开始，逐步增加压缩强度

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Compact Service                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Trigger       │  │  Compression    │  │   Post      │ │
│  │   Detection     │  │   Engine        │  │  Compact    │ │
│  │                 │  │                 │  │  Recovery   │ │
│  │ • Token Count   │──▶ • Snip         │──▶             │ │
│  │ • Safety Margin │    • Micro         │    • File     │ │
│  │ • API Error     │    • Auto          │      Restore  │ │
│  │                 │    • Collapse      │    • Skill    │ │
│  └─────────────────┘  └─────────────────┘    Injection  │ │
│                                                     └─────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 压缩触发机制

### 3.1 Token 估算触发

```typescript
// services/compact/compact.ts
const SAFETY_MARGIN_TOKENS = 13000 // 安全缓冲

export async function maybeCompact(
  messages: Message[],
  model: string
): Promise<CompactResult | null> {
  const estimatedTokens = await tokenCountWithEstimation(messages)
  const contextWindow = getContextWindowForModel(model)
  const threshold = contextWindow - SAFETY_MARGIN_TOKENS

  if (estimatedTokens < threshold) {
    return null // 无需压缩
  }

  return executeCompact(messages, {
    targetTokens: threshold,
    level: determineCompressionLevel(estimatedTokens, contextWindow),
  })
}
```

### 3.2 API 错误触发

```typescript
// API 返回 413 Payload Too Large 时触发
export async function handleContextTooLong(
  error: APIError,
  messages: Message[]
): Promise<CompactResult> {
  const tokenGap = getPromptTooLongTokenGap(error)

  return executeCompact(messages, {
    targetTokens: messages.length - tokenGap - SAFETY_MARGIN_TOKENS,
    level: 'aggressive', // API 错误时更激进
  })
}
```

### 3.3 压缩级别决策

```typescript
function determineCompressionLevel(
  currentTokens: number,
  contextWindow: number
): CompactLevel {
  const ratio = currentTokens / contextWindow

  if (ratio > 0.95) {
    return 'collapse' // 极端情况
  } else if (ratio > 0.9) {
    return 'auto'     // 标准压缩
  } else if (ratio > 0.85) {
    return 'micro'    // 轻量压缩
  } else {
    return 'snip'     // 最小压缩
  }
}
```

## 4. 四级压缩策略

### 4.1 Level 1: Snip Compact

```typescript
function executeSnipCompact(messages: Message[]): CompactResult {
  // 1. 删除已处理的工具输出
  const snipped = messages.filter((msg) => {
    if (msg.type === 'tool_result') {
      // 保留最近的 tool_result，删除旧的
      return isRecentToolResult(msg)
    }
    return true
  })

  // 2. 简化过长的思考过程
  const simplified = snipped.map((msg) => {
    if (msg.type === 'thinking' && msg.content.length > 1000) {
      return {
        ...msg,
        content: msg.content.slice(0, 500) + '... [truncated]',
      }
    }
    return msg
  })

  return {
    type: 'snip',
    messages: simplified,
    tokensSaved: estimateTokenDiff(messages, simplified),
  }
}
```

### 4.2 Level 2: Micro Compact

```typescript
function executeMicroCompact(
  messages: Message[],
  targetTokens: number
): CompactResult {
  // 只压缩当前轮次的消息，保留历史
  const currentRound = extractCurrentRound(messages)
  const history = messages.slice(0, messages.length - currentRound.length)

  // 为当前轮次生成摘要
  const summary = generateLocalSummary(currentRound)

  const compacted = [
    ...history,
    createCompactBoundaryMessage(summary),
  ]

  return {
    type: 'micro',
    messages: compacted,
    summary,
    tokensSaved: estimateTokenDiff(messages, compacted),
  }
}
```

### 4.3 Level 3: Auto Compact（标准压缩）

```typescript
async function executeAutoCompact(
  messages: Message[],
  targetTokens: number
): Promise<CompactResult> {
  // 1. 分组消息（按 API 调用轮次）
  const rounds = groupMessagesByApiRound(messages)

  // 2. 选择要压缩的轮次（保留最近 2 轮）
  const toCompress = rounds.slice(0, -2)
  const toPreserve = rounds.slice(-2)

  // 3. 使用 Forked Agent 生成摘要
  const summaries: Summary[] = []
  for (const round of toCompress) {
    const summary = await summarizeWithForkedAgent(round)
    summaries.push(summary)
  }

  // 4. 构建压缩后的消息列表
  const compacted = [
    createSystemCompactBoundary(summaries),
    ...toPreserve.flat(),
  ]

  return {
    type: 'auto',
    messages: compacted,
    summaries,
    tokensSaved: estimateTokenDiff(messages, compacted),
  }
}
```

### 4.4 Level 4: Context Collapse（极端压缩）

```typescript
function executeContextCollapse(
  messages: Message[],
  targetTokens: number
): CompactResult {
  // 极端情况：只保留系统消息和最近用户消息
  const systemMessages = messages.filter((m) => m.type === 'system')
  const lastUserMessage = messages
    .filter((m) => m.type === 'user')
    .pop()

  const collapsed = [
    ...systemMessages,
    createEmergencyBoundaryMessage(),
    lastUserMessage,
  ]

  return {
    type: 'collapse',
    messages: collapsed,
    isEmergency: true,
    tokensSaved: estimateTokenDiff(messages, collapsed),
  }
}
```

## 5. Forked Agent 摘要

### 5.1 摘要 Agent 配置

```typescript
// utils/forkedAgent.ts
const SUMMARY_AGENT_CONFIG: AgentConfig = {
  model: 'claude-haiku-4-5', // 使用轻量模型降低成本
  maxTokens: COMPACT_MAX_OUTPUT_TOKENS,
  temperature: 0.3, // 低温度，更确定的摘要
  systemPrompt: `
You are a conversation summarizer. Create a concise summary of the conversation
that preserves key facts, decisions, and file changes. Focus on actionable
information that would be needed to continue the task.

Format:
- Key actions taken
- Files modified
- Current state/decisions
- Next steps (if apparent)
`,
}
```

### 5.2 异步摘要执行

```typescript
export async function runForkedAgent(
  params: CacheSafeParams & {
    messages: Message[]
    userContent: string
  }
): Promise<AgentResult> {
  // 1. 创建隔离的 Agent 上下文
  const agentContext = createAgentContext({
    parentSessionId: getSessionId(),
    isForked: true,
    noToolPermissions: true, // 摘要 Agent 无工具权限
  })

  // 2. 在隔离上下文中执行
  return runWithAgentContext(agentContext, async () => {
    const response = await queryModelWithStreaming({
      ...SUMMARY_AGENT_CONFIG,
      messages: params.messages,
    })

    // 3. 收集完整响应
    let summary = ''
    for await (const chunk of response) {
      if (chunk.type === 'text_delta') {
        summary += chunk.text
      }
    }

    return { summary, usage: response.usage }
  })
}
```

## 6. 消息分组

### 6.1 API 轮次分组

```typescript
// services/compact/grouping.ts
export function groupMessagesByApiRound(
  messages: Message[]
): MessageRound[] {
  const rounds: MessageRound[] = []
  let currentRound: Message[] = []

  for (const msg of messages) {
    // 新的用户消息开始新一轮
    if (msg.type === 'user') {
      if (currentRound.length > 0) {
        rounds.push({ messages: currentRound })
      }
      currentRound = [msg]
    } else {
      currentRound.push(msg)
    }
  }

  // 添加最后一轮
  if (currentRound.length > 0) {
    rounds.push({ messages: currentRound })
  }

  return rounds
}
```

### 6.2 轮次重要性评分

```typescript
function scoreRoundImportance(round: MessageRound): number {
  let score = 0

  for (const msg of round.messages) {
    // 包含工具调用：重要
    if (msg.type === 'assistant' && msg.tool_calls?.length) {
      score += 10 * msg.tool_calls.length
    }

    // 包含文件编辑：重要
    if (msg.type === 'tool_result') {
      const toolName = msg.tool_use_id?.split('_')[0]
      if (toolName === 'FileEditTool') {
        score += 20
      }
    }

    // 用户明确标记重要
    if (msg.type === 'user' && msg.content.includes('!important')) {
      score += 50
    }
  }

  // 时间衰减（越近的轮次越重要）
  const age = Date.now() - round.timestamp
  score += Math.max(0, 100 - age / 60000) // 每分钟衰减 1 分

  return score
}
```

## 7. 后压缩恢复

### 7.1 关键文件识别

```typescript
// 从摘要中提取引用的文件
function extractReferencedFiles(summary: string): string[] {
  const filePattern = /`([^`]+\.(ts|tsx|js|jsx|py|md))`/g
  const matches = summary.matchAll(filePattern)
  return [...matches].map((m) => m[1])
}

// 识别修改过的文件
function identifyModifiedFiles(rounds: MessageRound[]): string[] {
  const modified = new Set<string>()

  for (const round of rounds) {
    for (const msg of round.messages) {
      if (msg.type === 'tool_result' && msg.tool_name === 'FileEditTool') {
        const filePath = msg.tool_input?.file_path
        if (filePath) {
          modified.add(filePath)
        }
      }
    }
  }

  return [...modified]
}
```

### 7.2 文件恢复策略

```typescript
const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
const POST_COMPACT_TOKEN_BUDGET = 50000
const POST_COMPACT_MAX_TOKENS_PER_FILE = 5000

async function executePostCompactRecovery(
  compactedMessages: Message[],
  originalMessages: Message[]
): Promise<Message[]> {
  // 1. 识别需要恢复的文件
  const filesToRestore = identifyModifiedFiles(originalMessages)
    .slice(0, POST_COMPACT_MAX_FILES_TO_RESTORE)

  // 2. 读取文件内容
  const fileAttachments: AttachmentMessage[] = []
  let tokensUsed = 0

  for (const filePath of filesToRestore) {
    const content = await readFile(filePath, 'utf-8')
    const tokens = roughTokenCountEstimation(content)

    if (tokens > POST_COMPACT_MAX_TOKENS_PER_FILE) {
      // 大文件只读取前部分
      const truncated = content.slice(0, POST_COMPACT_MAX_TOKENS_PER_FILE * 4)
      fileAttachments.push(createAttachment(filePath, truncated, true))
      tokensUsed += POST_COMPACT_MAX_TOKENS_PER_FILE
    } else {
      fileAttachments.push(createAttachment(filePath, content))
      tokensUsed += tokens
    }

    if (tokensUsed >= POST_COMPACT_TOKEN_BUDGET) {
      break
    }
  }

  // 3. 注入到压缩边界之后
  const boundaryIndex = compactedMessages.findIndex(isCompactBoundaryMessage)
  if (boundaryIndex >= 0) {
    compactedMessages.splice(boundaryIndex + 1, 0, ...fileAttachments)
  }

  return compactedMessages
}
```

### 7.3 Skill 重新注入

```typescript
async function reinjectSkillsAfterCompact(
  messages: Message[]
): Promise<Message[]> {
  const invokedSkills = getInvokedSkillsForAgent()

  const skillContents: string[] = []
  let tokensUsed = 0

  for (const skill of invokedSkills) {
    const content = await loadSkillContent(skill.name)
    const tokens = roughTokenCountEstimation(content)

    // 每个 skill 有独立的 token 限制
    if (tokens > POST_COMPACT_MAX_TOKENS_PER_SKILL) {
      skillContents.push(content.slice(0, POST_COMPACT_MAX_TOKENS_PER_SKILL * 4))
      tokensUsed += POST_COMPACT_MAX_TOKENS_PER_SKILL
    } else {
      skillContents.push(content)
      tokensUsed += tokens
    }

    if (tokensUsed >= POST_COMPACT_SKILLS_TOKEN_BUDGET) {
      break
    }
  }

  // 将 skill 内容作为系统消息注入
  const skillMessage = createSystemMessage(
    skillContents.map((c) => `<skill>\n${c}\n</skill>`).join('\n')
  )

  return [skillMessage, ...messages]
}
```

## 8. Compact 边界消息

### 8.1 边界消息格式

```typescript
function createCompactBoundaryMessage(summaries: Summary[]): SystemMessage {
  const content = `
## Previous Conversation Summary

The following is a summary of earlier parts of this conversation:

${summaries
  .map(
    (s, i) => `
### Part ${i + 1}
${s.content}
`
  )
  .join('\n')}

Key files referenced in the summary are attached below.
`

  return {
    type: 'system',
    content,
    uuid: generateUUID(),
  }
}
```

### 8.2 部分压缩消息

```typescript
function createPartialCompactMessage(
  partialSummary: PartialSummary
): UserMessage {
  return {
    type: 'user',
    content: `
[Earlier conversation summarized]
${partialSummary.content}

[Continue from where we left off]
`,
    uuid: generateUUID(),
  }
}
```

## 9. 压缩分析与监控

### 9.1 压缩事件日志

```typescript
export function logCompactEvent(event: {
  level: CompactLevel
  tokensBefore: number
  tokensAfter: number
  roundsCompressed: number
  duration: number
  modelUsed: string
}): void {
  logEvent('conversation_compacted', {
    level: event.level,
    tokens_saved: event.tokensBefore - event.tokensAfter,
    compression_ratio: event.tokensAfter / event.tokensBefore,
    rounds_compressed: event.roundsCompressed,
    duration_ms: event.duration,
    model: event.modelUsed,
  })
}
```

### 9.2 压缩质量反馈

```typescript
// 检测压缩后模型是否需要重新读取文件
function detectReReadPattern(messages: Message[]): boolean {
  // 如果模型在压缩后立即读取了摘要中提到的文件
  // 说明压缩可能过于激进
  const recentTools = getRecentToolCalls(messages, 5)

  return recentTools.some(
    (tool) =>
      tool.name === 'FileReadTool' &&
      isFileMentionedInCompactSummary(tool.input.file_path)
  )
}
```

## 10. 创新点与反思

### 10.1 设计创新

1. **Forked Agent 摘要**：使用 LLM 生成高质量摘要，而非简单的文本截断
2. **后压缩恢复**：智能识别并恢复关键文件到上下文
3. **渐进降级**：四级压缩策略确保最小信息损失

### 10.2 架构权衡

**摘要质量 vs 成本**

- 使用 Haiku 而非 Sonnet 进行摘要，平衡质量与成本
- 摘要本身消耗 Token，但这是「生存必需」的成本

**压缩粒度**

- 轮次级别压缩保持对话结构
- 消息级别压缩会更精细但破坏结构

### 10.3 生产经验

1. **Token 估算误差**：实际 Token 数可能高于估算，需要安全缓冲
2. **摘要召回率**：用户可能需要通过 @file 显式引用被压缩的内容
3. **视觉反馈**：压缩后显示 "[Earlier conversation summarized]" 提示用户

### 10.4 未来演进

1. **增量摘要**：维护运行中的摘要，而非事后生成
2. **语义压缩**：基于 embedding 的语义相似度合并消息
3. **用户可控压缩**：允许用户标记哪些内容不可压缩

---

Claude Code 的 Compact Service 是一个智能的上下文压缩系统。通过 Forked Agent 摘要、后压缩恢复、渐进降级等设计，在有限上下文窗口内最大化保留有用信息。理解其设计，对于构建任何需要处理长对话的 LLM 应用都具有重要参考价值。
