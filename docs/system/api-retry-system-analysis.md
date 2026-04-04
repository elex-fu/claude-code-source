# Claude Code API 重试系统技术分析

## 1. 模块总结介绍

API Retry 系统是 Claude Code 的「弹性连接层」，负责处理与 Anthropic API 通信时的各种错误情况。该系统实现了智能重试策略、指数退避、模型降级、快速模式回退等机制，确保在网络不稳定或服务过载时仍能提供可靠服务。

核心特性：
- **智能重试策略**：区分可重试/不可重试错误
- **指数退避**：带抖动的指数退避算法
- **模型降级**：529 过载时自动降级到备用模型
- **快速模式回退**：429/529 时自动切换标准速度
- **持久重试模式**：无人值守会话的持续重试

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                   API Retry System                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Error Classification                                       │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│   │   529 Overload  │  │   429 RateLimit │  │  401/403    │ │
│   │   Retry 3 times │  │   Retry/Backoff │  │  Refresh    │ │
│   │   Then fallback │  │   Fast→Standard │  │  Token      │ │
│   └─────────────────┘  └─────────────────┘  └─────────────┘ │
│                                                              │
│   Retry Strategies                                           │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  • Exponential backoff (base 500ms, max 32s)         │  │
│   │  • Persistent mode (max 5min backoff, 6hr cap)       │  │
│   │  • Foreground vs Background retry policies           │  │
│   │  • Fast mode cooldown (30min hold)                   │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                              │
│   Special Handlers                                           │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│   │  Model Fallback │  │  Token Refresh  │  │  Max Tokens │ │
│   │  Opus→Sonnet    │  │  OAuth/Cloud    │  │  Adjustment │ │
│   └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 重试配置

### 3.1 默认配置

```typescript
const DEFAULT_MAX_RETRIES = 10           // 默认最大重试次数
const BASE_DELAY_MS = 500                // 基础延迟 500ms
const FLOOR_OUTPUT_TOKENS = 3000         // 最小输出 token 数
const MAX_529_RETRIES = 3                // 529 错误最大连续重试

// 快速模式回退配置
const DEFAULT_FAST_MODE_FALLBACK_HOLD_MS = 30 * 60 * 1000  // 30 分钟
const SHORT_RETRY_THRESHOLD_MS = 20 * 1000                 // 20 秒
const MIN_COOLDOWN_MS = 10 * 60 * 1000                     // 10 分钟

// 持久重试配置
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000   // 5 分钟最大退避
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000 // 6 小时上限
const HEARTBEAT_INTERVAL_MS = 30_000              // 30 秒心跳
```

### 3.2 前景/后台重试策略

```typescript
// 用户正在等待结果的场景 - 需要重试
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'repl_main_thread:outputStyle:custom',
  'sdk',
  'agent:custom',
  'agent:default',
  'compact',
  'hook_agent',
  'verification_agent',
  'auto_mode',
  'bash_classifier',  // 特性门控
])

function shouldRetry529(querySource: QuerySource | undefined): boolean {
  return querySource === undefined || 
         FOREGROUND_529_RETRY_SOURCES.has(querySource)
}
```

## 4. 指数退避算法

### 4.1 延迟计算

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000
): number {
  // 优先使用服务器指定的 Retry-After
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }
  
  // 指数退避：base * 2^(attempt-1)
  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs
  )
  
  // 添加 25% 抖动避免惊群
  const jitter = Math.random() * 0.25 * baseDelay
  
  return baseDelay + jitter
}
```

### 4.2 持久重试退避

```typescript
// 无人值守模式使用更大的退避
if (persistent) {
  persistentAttempt++
  
  // 检查 rate limit reset 时间戳
  const resetDelay = getRateLimitResetDelayMs(error)
  
  const delayMs = Math.min(
    resetDelay ?? getRetryDelay(
      persistentAttempt,
      retryAfter,
      PERSISTENT_MAX_BACKOFF_MS
    ),
    PERSISTENT_RESET_CAP_MS  // 6 小时上限
  )
  
  // 长时间等待需要发送心跳
  if (delayMs > 60_000) {
    yield createSystemAPIErrorMessage(error, delayMs, attempt, maxRetries)
  }
  
  // 分块睡眠，定期发送心跳
  let remaining = delayMs
  while (remaining > 0) {
    yield createSystemAPIErrorMessage(error, remaining, attempt, maxRetries)
    const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)
    await sleep(chunk, options.signal, { abortError })
    remaining -= chunk
  }
}
```

## 5. 错误分类与处理

### 5.1 重试决策矩阵

```typescript
function shouldRetry(error: APIError): boolean {
  // 模拟错误不重试（来自 /mock-limits 命令）
  if (isMockRateLimitError(error)) return false
  
  // 持久模式：429/529 总是重试
  if (isPersistentRetryEnabled() && isTransientCapacityError(error)) {
    return true
  }
  
  // CCR 模式：401/403 视为瞬态错误
  if (isCCRMode() && (error.status === 401 || error.status === 403)) {
    return true
  }
  
  // 服务器指示重试
  const shouldRetryHeader = error.headers?.get('x-should-retry')
  if (shouldRetryHeader === 'true' && isEnterpriseSubscriber()) {
    return true
  }
  if (shouldRetryHeader === 'false' && !isAntEmployee()) {
    return false
  }
  
  // 按状态码处理
  switch (error.status) {
    case 408: return true   // Request Timeout
    case 409: return true   // Lock Timeout
    case 429: return !isClaudeAISubscriber()  // Rate Limit
    case 401: 
      clearApiKeyHelperCache()
      return true  // Auth failure, retry after clearing cache
    case 403:
      if (isOAuthTokenRevokedError(error)) return true
      return false
    default:
      return error.status >= 500  // Server errors
  }
}
```

### 5.2 认证错误处理

```typescript
// OAuth Token 刷新
if (error.status === 401 || isOAuthTokenRevokedError(error)) {
  const failedAccessToken = getClaudeAIOAuthTokens()?.accessToken
  if (failedAccessToken) {
    await handleOAuth401Error(failedAccessToken)
  }
  client = await getClient()  // 获取新客户端
}

// AWS 认证错误
if (isBedrockAuthError(error)) {
  clearAwsCredentialsCache()
  return true  // 允许重试
}

// GCP 认证错误
if (isVertexAuthError(error)) {
  clearGcpCredentialsCache()
  return true
}
```

### 5.3 连接错误处理

```typescript
function isStaleConnectionError(error: unknown): boolean {
  if (!(error instanceof APIConnectionError)) return false
  const details = extractConnectionErrorDetails(error)
  return details?.code === 'ECONNRESET' || details?.code === 'EPIPE'
}

// 处理过时连接
if (isStaleConnectionError(lastError)) {
  logForDebugging('Stale connection — disabling keep-alive for retry')
  disableKeepAlive()  // 禁用 keep-alive 重新连接
}
```

## 6. 模型降级机制

### 6.1 529 降级触发

```typescript
if (is529Error(error)) {
  consecutive529Errors++
  
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
      })
      
      // 触发降级错误
      throw new FallbackTriggeredError(
        options.model,
        options.fallbackModel
      )
    }
  }
}
```

### 6.2 降级执行

```typescript
// 在调用处捕获降级错误
try {
  const result = await withRetry(getClient, operation, {
    model: 'claude-opus-4',
    fallbackModel: 'claude-sonnet-4',
  })
} catch (error) {
  if (error instanceof FallbackTriggeredError) {
    // 使用降级模型重试
    return await withRetry(getClient, operation, {
      model: error.fallbackModel,
      maxRetries: 3,
    })
  }
  throw error
}
```

## 7. 快速模式回退

### 7.1 快速模式检测

```typescript
if (wasFastModeActive && error instanceof APIError) {
  // 超额使用被拒绝
  const overageReason = error.headers?.get(
    'anthropic-ratelimit-unified-overage-disabled-reason'
  )
  if (overageReason !== null && overageReason !== undefined) {
    handleFastModeOverageRejection(overageReason)
    retryContext.fastMode = false
    continue  // 重试，但使用标准速度
  }
  
  // API 明确拒绝快速模式
  if (isFastModeNotEnabledError(error)) {
    handleFastModeRejectedByAPI()
    retryContext.fastMode = false
    continue
  }
  
  // 429/529 根据 retry-after 决定
  if (error.status === 429 || is529Error(error)) {
    const retryAfterMs = getRetryAfterMs(error)
    
    if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
      // 短等待：保持快速模式重试
      await sleep(retryAfterMs, options.signal)
      continue
    }
    
    // 长等待：进入冷却期
    const cooldownMs = Math.max(
      retryAfterMs ?? DEFAULT_FAST_MODE_FALLBACK_HOLD_MS,
      MIN_COOLDOWN_MS
    )
    triggerFastModeCooldown(Date.now() + cooldownMs, 'overloaded')
    retryContext.fastMode = false
    continue
  }
}
```

## 8. 上下文溢出处理

### 8.1 溢出检测

```typescript
function parseMaxTokensContextOverflowError(error: APIError) {
  if (error.status !== 400) return undefined
  
  // 格式："input length and `max_tokens` exceed context limit: 188059 + 20000 > 200000"
  const regex = /input length and `max_tokens` exceed context limit: (\d+) \+ (\d+) > (\d+)/
  const match = error.message.match(regex)
  
  if (!match) return undefined
  
  return {
    inputTokens: parseInt(match[1], 10),
    maxTokens: parseInt(match[2], 10),
    contextLimit: parseInt(match[3], 10),
  }
}
```

### 8.2 Token 调整

```typescript
if (error instanceof APIError) {
  const overflowData = parseMaxTokensContextOverflowError(error)
  if (overflowData) {
    const { inputTokens, contextLimit } = overflowData
    
    const safetyBuffer = 1000
    const availableContext = Math.max(
      0,
      contextLimit - inputTokens - safetyBuffer
    )
    
    if (availableContext < FLOOR_OUTPUT_TOKENS) {
      throw error  // 空间不足，无法恢复
    }
    
    // 计算新的 max_tokens
    const minRequired = thinkingEnabled 
      ? thinkingBudgetTokens + 1 
      : 1
    
    retryContext.maxTokensOverride = Math.max(
      FLOOR_OUTPUT_TOKENS,
      availableContext,
      minRequired
    )
    
    logEvent('tengu_max_tokens_context_overflow_adjustment', {
      inputTokens,
      contextLimit,
      adjustedMaxTokens: retryContext.maxTokensOverride,
    })
    
    continue  // 使用调整后的 token 数重试
  }
}
```

## 9. 生成器实现

### 9.1 流式错误通知

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions
): AsyncGenerator<SystemAPIErrorMessage, T> {
  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      return await operation(client, attempt, retryContext)
    } catch (error) {
      // ... 错误处理
      
      // 向用户流式报告重试状态
      if (error instanceof APIError) {
        yield createSystemAPIErrorMessage(error, delayMs, attempt, maxRetries)
      }
      
      await sleep(delayMs, options.signal, { abortError })
    }
  }
}
```

### 9.2 使用示例

```typescript
const retryGen = withRetry(getAnthropicClient, async (client) => {
  return await client.messages.create({
    model: retryContext.model,
    max_tokens: retryContext.maxTokensOverride || 4096,
    messages: inputMessages,
  })
}, {
  model: 'claude-opus-4',
  fallbackModel: 'claude-sonnet-4',
  querySource: 'repl_main_thread',
})

// 消费生成器，处理中间错误消息
for await (const message of retryGen) {
  if (message.type === 'system' && message.subtype === 'api_retry') {
    // 显示重试状态给用户
    renderRetryStatus(message)
  }
}
```

## 10. 技术创新点

1. **智能错误分类**：根据错误类型、用户类型、查询来源等多维度决定重试策略

2. **双模式退避**：普通模式（32s 上限）和持久模式（5min 退避，6h 上限）

3. **心跳机制**：持久模式下分块睡眠并定期发送心跳，避免会话被标记为空闲

4. **模型降级链**：连续 529 后自动降级到备用模型，保证服务可用性

5. **快速模式智能回退**：根据 retry-after 长短决定保持快速模式还是切换标准速度

6. **上下文溢出恢复**：自动计算可用 token 数并调整请求参数

7. **云认证集成**：统一处理 OAuth、AWS、GCP 的认证刷新逻辑

---

API Retry 系统通过智能重试、指数退避、模型降级、快速模式回退等设计，实现了高可用、弹性的 API 通信。理解其设计，对于构建可靠的 API 客户端具有参考价值。
