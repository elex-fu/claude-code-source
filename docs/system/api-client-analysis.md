# Claude Code API 客户端技术分析

## 1. 模块总结介绍
API Client 是 Claude Code 与 Anthropic API 通信的核心模块。它实现了流式响应处理、智能重试、成本追踪等高级功能。

## 2. 核心架构
```typescript
// services/api/claude.ts
export async function* queryModelWithStreaming(
  params: QueryParams
): AsyncGenerator<StreamEvent, QueryResult, unknown> {
  const client = getAnthropicClient()
  
  const stream = await client.messages.create({
    model: params.model,
    messages: params.messages,
    system: params.system,
    tools: params.tools,
    stream: true,
    max_tokens: params.maxTokens,
  })

  for await (const event of stream) {
    yield* processEvent(event)
  }
}
```

## 3. 流式事件处理
```typescript
async function* processEvent(
  event: MessageStreamEvent
): AsyncGenerator<StreamEvent> {
  switch (event.type) {
    case 'message_start':
      yield { type: 'start', messageId: event.message.id }
      break
      
    case 'content_block_delta':
      if (event.delta.type === 'text_delta') {
        yield { 
          type: 'text', 
          content: event.delta.text 
        }
      }
      break
      
    case 'message_stop':
      yield { type: 'stop' }
      break
  }
}
```

## 4. 重试机制
```typescript
// services/api/withRetry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  for (let i = 0; i < options.maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      const delay = calculateBackoff(i, options.baseDelay)
      await sleep(delay)
    }
  }
  throw new Error('Max retries exceeded')
}
```

## 5. 成本追踪集成
```typescript
export async function trackAPICost(
  usage: BetaUsage,
  model: string
): Promise<void> {
  const cost = calculateUSDCost(model, usage)
  
  updateUsage({
    model,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    costUSD: cost,
  })
}
```

---

API Client 通过流式处理、智能重试、成本追踪等设计，实现了高效可靠的 LLM 通信。
