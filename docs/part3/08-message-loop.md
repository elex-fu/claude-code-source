# 第 8 章：消息循环与流式处理

> 流式处理不只是用户体验的优化，它是整个系统的基础设计模式。

---

## 8.1 为什么流式处理如此重要

在 AI 系统中，流式处理（Streaming）有两层含义：

**用户体验层面**：用户不需要等待完整响应，可以实时看到 Claude 的输出。对于长任务，这意味着用户能立刻知道 Claude 在做什么，而不是盯着空白屏幕等待。

**系统架构层面**：流式处理让整个系统变成了一个**数据流管道**。消息从 API 流出，经过解析、工具执行、结果回填，再流回 API，形成一个连续的数据流。

Claude Code 从底层到顶层都是流式设计的。

---

## 8.2 消息的类型系统

在深入流式处理之前，先理解 Claude Code 的消息类型系统：

```typescript
// src/types/message.ts（简化）
type Message =
  | UserMessage          // 用户输入
  | AssistantMessage     // Claude 的响应
  | SystemMessage        // 系统消息（工具结果、错误等）
  | AttachmentMessage    // 附件（图片、文件）
  | ToolUseSummaryMessage // 工具使用摘要
  | TombstoneMessage     // 已删除消息的占位符

type AssistantMessage = {
  type: 'assistant'
  uuid: string
  message: {
    content: ContentBlock[]  // 可以包含多种块
  }
  apiError?: string
}

type ContentBlock =
  | TextBlock           // 普通文本
  | ThinkingBlock       // 思考块（扩展思考模式）
  | RedactedThinkingBlock // 被编辑的思考块
  | ToolUseBlock        // 工具调用请求
```

这个类型系统的设计很重要：**一条 AssistantMessage 可以包含多种类型的内容块**。Claude 可以在一次响应中同时输出文本、思考和多个工具调用。

---

## 8.3 流式响应的解析

Claude API 返回的是 Server-Sent Events（SSE）流。流式解析架构：

![流式响应解析流程 — 文本块实时输出，工具调用块累积后执行](../diagrams/query-engine-flow.svg)

`query.ts` 中的流式解析大致如下：

```typescript
// 简化的流式解析逻辑
async function* parseStream(apiStream) {
  let currentTextBlock = ''
  let currentThinkingBlock = ''
  const toolUseBlocks = new Map()

  for await (const event of apiStream) {
    switch (event.type) {
      case 'content_block_start':
        if (event.content_block.type === 'text') {
          // 开始文本块
        } else if (event.content_block.type === 'thinking') {
          // 开始思考块
        } else if (event.content_block.type === 'tool_use') {
          // 开始工具调用块
          toolUseBlocks.set(event.index, {
            id: event.content_block.id,
            name: event.content_block.name,
            input: ''
          })
        }
        break

      case 'content_block_delta':
        if (event.delta.type === 'text_delta') {
          currentTextBlock += event.delta.text
          // 立即 yield 给 UI 显示
          yield { type: 'text_delta', text: event.delta.text }
        } else if (event.delta.type === 'thinking_delta') {
          currentThinkingBlock += event.delta.thinking
          // 思考块不 yield 给 UI
        } else if (event.delta.type === 'input_json_delta') {
          // 工具调用参数是流式 JSON
          toolUseBlocks.get(event.index).input += event.delta.partial_json
        }
        break

      case 'content_block_stop':
        // 块结束，处理完整的块
        break

      case 'message_stop':
        // 消息结束，yield 完整的工具调用列表
        yield { type: 'tool_calls', calls: [...toolUseBlocks.values()] }
        break
    }
  }
}
```

关键点：**文本块实时 yield，工具调用块等完整后再处理**。这是因为工具调用需要完整的 JSON 参数才能执行。

---

## 8.4 StreamingToolExecutor：流式工具执行

`src/services/tools/StreamingToolExecutor.ts` 是工具执行的核心：

```typescript
class StreamingToolExecutor {
  async* execute(toolCalls: ToolUseBlock[], context: ToolUseContext) {
    // 并行执行所有工具调用
    const executions = toolCalls.map(call =>
      this.executeSingle(call, context)
    )

    // 流式 yield 每个工具的执行过程
    for (const execution of executions) {
      for await (const event of execution) {
        yield event
      }
    }
  }

  async* executeSingle(call: ToolUseBlock, context: ToolUseContext) {
    // yield 工具开始事件
    yield { type: 'tool_start', toolName: call.name, toolUseId: call.id }

    try {
      const tool = findToolByName(call.name, context.options.tools)
      const result = await tool.execute(call.input, context)

      // yield 工具结果
      yield { type: 'tool_result', toolUseId: call.id, result }
    } catch (error) {
      // yield 工具错误
      yield { type: 'tool_error', toolUseId: call.id, error }
    }
  }
}
```

---

## 8.5 消息规范化：normalizeMessagesForAPI

在每次 API 调用前，消息列表需要规范化。`normalizeMessagesForAPI()` 处理多种情况：

```typescript
function normalizeMessagesForAPI(messages: Message[]): APIMessage[] {
  return messages
    // 过滤掉不需要发送给 API 的消息类型
    .filter(msg => !isSyntheticMessage(msg))
    // 合并相邻的同类型消息（API 要求 user/assistant 交替）
    .reduce(mergeAdjacentMessages, [])
    // 处理思考块的特殊规则
    .map(handleThinkingBlocks)
    // 截断过长的工具结果
    .map(truncateLargeToolResults)
}
```

这个函数处理了很多边界情况：
- API 要求消息必须 user/assistant 交替，不能连续两条 user 消息
- 思考块有严格的位置规则
- 工具结果可能很大（比如读取了一个大文件），需要截断

---

## 8.6 消息队列管理

Claude Code 有一个消息队列系统（`src/utils/messageQueueManager.ts`），处理并发输入：

```typescript
// 用户可能在 Claude 还在执行时就输入了新消息
// 消息队列确保这些消息按优先级处理

const queue = {
  // 高优先级：斜杠命令（如 /stop）
  // 普通优先级：用户消息
  // 低优先级：后台任务消息
}
```

这解决了一个实际问题：用户在 Claude 执行长任务时可能想中断或修改指令，消息队列确保这些操作能及时响应。

---

## 8.7 流式输出的 UI 渲染

Claude Code 使用 Ink（React for CLI）渲染 UI。流式文本的渲染是这样工作的：

```tsx
// 简化的消息渲染组件
function StreamingMessage({ message }) {
  const [displayText, setDisplayText] = useState('')

  useEffect(() => {
    // 订阅流式事件
    const unsubscribe = subscribeToStream(message.id, (delta) => {
      setDisplayText(prev => prev + delta)
    })
    return unsubscribe
  }, [message.id])

  return <Text>{displayText}</Text>
}
```

Ink 的渲染是增量的：每次 `setDisplayText` 只更新变化的部分，不重新渲染整个界面。这保证了流式输出的流畅性。

---

## 8.8 工具调用的 UI 反馈

工具执行时，UI 会显示实时进度：

```
> 帮我找出所有未使用的变量

Claude: 我来分析项目中的未使用变量。

⠸ GlobTool: 搜索 **/*.ts...
✓ GlobTool: 找到 47 个文件

⠸ FileReadTool: 读取 src/main.ts...
✓ FileReadTool: 完成

⠸ FileReadTool: 读取 src/utils.ts...
✓ FileReadTool: 完成

分析完成，发现以下未使用的变量：
...
```

这种实时反馈让用户知道 Claude 在做什么，可以随时决定是否中断。

---

## 8.9 背压（Backpressure）处理

当工具执行速度快于 UI 渲染速度时，需要背压机制防止内存溢出：

```typescript
// Claude Code 使用异步生成器的自然背压
async function* query(params) {
  // 生成器只在消费方 await 时才继续执行
  // 这天然提供了背压
  for await (const event of apiStream) {
    yield event  // 等待消费方处理完才继续
  }
}
```

异步生成器的背压是"免费"的：消费方不 `await` 下一个值，生产方就不会继续执行。

---

## 8.10 流式处理的错误边界

流式处理中的错误处理比同步代码更复杂：

```typescript
async function* safeQuery(params) {
  try {
    yield* query(params)
  } catch (error) {
    if (error instanceof AbortError) {
      // 用户主动中断，正常退出
      yield { type: 'interrupted' }
    } else if (isRetryableError(error)) {
      // 可重试的错误，自动重试
      yield* safeQuery(params)
    } else {
      // 不可恢复的错误，yield 错误消息
      yield { type: 'error', error: error.message }
    }
  }
}
```

---

## 8.11 小结

Claude Code 的消息循环和流式处理设计：

- **端到端流式**：从 API 响应到 UI 渲染，全程流式
- **类型安全的消息系统**：清晰的消息类型层次
- **并行工具执行**：多个工具调用并行处理
- **消息规范化**：处理 API 的各种约束
- **消息队列**：处理并发输入
- **自然背压**：异步生成器天然提供背压
- **分层错误处理**：中断、重试、错误各有对应处理

这套流式架构是 Claude Code 能够流畅处理长任务的基础。

---

*下一章：[工具系统的设计哲学](../part4/09-tool-design.md)*
