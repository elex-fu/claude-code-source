# Claude Code Analytics 系统技术分析

## 1. 模块总结介绍

Analytics 系统是 Claude Code 的「数据洞察中枢」，负责收集、处理和上报产品使用数据。该系统在保护用户隐私的前提下，为产品决策提供数据支持。

核心特性：
- **无依赖设计**：核心模块零依赖，避免循环依赖
- **隐私优先**：PII 数据特殊标记和隔离
- **异步队列**：事件队列保证不阻塞主流程
- **多 Sink 路由**：支持 Datadog、1P 日志等多个后端

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Analytics System                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Event Queue   │  │   Sink Router   │  │   Backends  │ │
│  │                 │  │                 │  │             │ │
│  │ • Pre-init      │──▶ • Datadog       │──▶ • Datadog   │ │
│  │   buffering     │    • 1P Logging    │    • BigQuery  │ │
│  │ • Async drain   │    • Strip PII     │    • Internal  │ │
│  └─────────────────┘  └─────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心 API 设计

### 3.1 类型安全的事件标记

```typescript
// index.ts
/**
 * Marker type for verifying analytics metadata doesn't contain sensitive data
 */
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

/**
 * Marker type for PII-tagged proto columns
 */
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```

这些标记类型利用 TypeScript 的类型系统强制开发者显式声明数据安全：

```typescript
// 正确：显式声明无敏感信息
logEvent('tool_used', {
  tool_name: 'FileReadTool' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
})

// 错误：尝试记录文件路径需要显式标记
logEvent('file_read', {
  path: '/secret/path',  // Type Error!
})
```

### 3.2 事件队列机制

```typescript
type QueuedEvent = {
  eventName: string
  metadata: LogEventMetadata
  async: boolean
}

const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

/**
 * 核心日志函数 - 无依赖设计
 */
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata = {}
): void {
  if (sink) {
    // Sink 已连接，直接发送
    sink.logEvent(eventName, metadata)
  } else {
    // 缓冲到队列，等待 Sink 连接
    eventQueue.push({ eventName, metadata, async: false })
  }
}

export async function logEventAsync(
  eventName: string,
  metadata: LogEventMetadata = {}
): Promise<void> {
  if (sink) {
    await sink.logEventAsync(eventName, metadata)
  } else {
    eventQueue.push({ eventName, metadata, async: true })
  }
}
```

### 3.3 Sink 连接

```typescript
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (
    eventName: string,
    metadata: LogEventMetadata
  ) => Promise<void>
}

export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return // 幂等

  sink = newSink

  // 异步排空队列，避免阻塞启动
  queueMicrotask(() => {
    drainEventQueue()
  })
}

async function drainEventQueue(): Promise<void> {
  while (eventQueue.length > 0) {
    const event = eventQueue.shift()!

    if (event.async) {
      await sink!.logEventAsync(event.eventName, event.metadata)
    } else {
      sink!.logEvent(event.eventName, event.metadata)
    }
  }
}
```

## 4. Sink 实现

### 4.1 复合 Sink

```typescript
// sink.ts
export function createCompositeSink(sinks: AnalyticsSink[]): AnalyticsSink {
  return {
    logEvent(eventName, metadata) {
      // 移除 PII 字段
      const safeMetadata = stripProtoFields(metadata)

      for (const sink of sinks) {
        try {
          sink.logEvent(eventName, safeMetadata)
        } catch (error) {
          // 单个 Sink 失败不影响其他
          console.error('Sink error:', error)
        }
      }
    },

    async logEventAsync(eventName, metadata) {
      const safeMetadata = stripProtoFields(metadata)

      await Promise.all(
        sinks.map(async (sink) => {
          try {
            await sink.logEventAsync(eventName, safeMetadata)
          } catch (error) {
            console.error('Sink error:', error)
          }
        })
      )
    },
  }
}
```

### 4.2 Datadog Sink

```typescript
// datadog.ts
import { StatsD } from 'node-statsd'

export function createDatadogSink(config: DatadogConfig): AnalyticsSink {
  const client = new StatsD({
    host: config.host,
    port: config.port,
    prefix: 'claude_code.',
  })

  return {
    logEvent(eventName, metadata) {
      // 计数器
      client.increment(`event.${eventName}`, 1)

      // 分布值
      if (metadata.duration_ms) {
        client.timing(`duration.${eventName}`, metadata.duration_ms)
      }

      // 标签
      const tags = Object.entries(metadata)
        .filter(([, v]) => typeof v === 'string' || typeof v === 'boolean')
        .map(([k, v]) => `${k}:${v}`)

      client.increment(`event.${eventName}.tagged`, 1, 1, tags)
    },

    async logEventAsync(eventName, metadata) {
      // Datadog 是 UDP，同步即可
      this.logEvent(eventName, metadata)
    },
  }
}
```

### 4.3 First-Party Event Logging

```typescript
// firstPartyEventLoggingExporter.ts
export function createFirstPartySink(
  config: FirstPartyConfig
): AnalyticsSink {
  const batch: Event[] = []
  let flushTimer: NodeJS.Timeout | null = null

  async function flush(): Promise<void> {
    if (batch.length === 0) return

    const events = batch.splice(0, batch.length)

    await fetch(config.endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': config.apiKey,
      },
      body: JSON.stringify({ events }),
    })
  }

  function scheduleFlush(): void {
    if (flushTimer) return

    flushTimer = setTimeout(() => {
      flushTimer = null
      flush().catch(console.error)
    }, 5000) // 5秒批量刷新
  }

  return {
    logEvent(eventName, metadata) {
      // 提取 _PROTO_* 字段到顶层
      const protoFields: Record<string, unknown> = {}
      const normalFields: Record<string, unknown> = {}

      for (const [key, value] of Object.entries(metadata)) {
        if (key.startsWith('_PROTO_')) {
          protoFields[key.slice(7)] = value // 去掉 _PROTO_ 前缀
        } else {
          normalFields[key] = value
        }
      }

      batch.push({
        event_name: eventName,
        timestamp: Date.now(),
        metadata: normalFields,
        ...protoFields, // PII 字段提升到顶层
      })

      if (batch.length >= 100) {
        flush().catch(console.error)
      } else {
        scheduleFlush()
      }
    },

    async logEventAsync(eventName, metadata) {
      this.logEvent(eventName, metadata)
      await flush()
    },
  }
}
```

## 5. GrowthBook 集成

### 5.1 特性标志

```typescript
// growthbook.ts
import { GrowthBook } from '@growthbook/growthbook'

let growthBook: GrowthBook | null = null

export function initGrowthBook(config: GrowthBookConfig): void {
  growthBook = new GrowthBook({
    apiHost: config.apiHost,
    clientKey: config.clientKey,
    enableDevMode: config.enableDevMode,

    // 属性
    attributes: {
      id: getUserId(),
      version: getVersion(),
      platform: process.platform,
    },
  })

  growthBook.init({ streaming: true })
}

export function getFeatureValue<T>(key: string, defaultValue: T): T {
  return growthBook?.getFeatureValue(key, defaultValue) ?? defaultValue
}

export function isFeatureEnabled(key: string): boolean {
  return growthBook?.isOn(key) ?? false
}
```

### 5.2 实验追踪

```typescript
export function trackExperiment(
  experimentId: string,
  variationId: string
): void {
  logEvent('experiment_viewed', {
    experiment_id: experimentId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
    variation_id: variationId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  })
}
```

## 6. 关键事件追踪

### 6.1 会话事件

```typescript
// 会话开始
logEvent('session_started', {
  session_id: sessionId,
  model: modelName,
  permission_mode: permissionMode,
  cwd_hash: hashPath(cwd), // 哈希化路径保护隐私
})

// 会话结束
logEvent('session_ended', {
  session_id: sessionId,
  duration_ms: Date.now() - startTime,
  total_cost_usd: totalCost,
  message_count: messageCount,
  tool_use_count: toolUseCount,
})
```

### 6.2 工具使用事件

```typescript
// 工具调用开始
logEvent('tool_use_started', {
  tool_name: toolName,
  query_id: queryId,
})

// 工具调用完成
logEvent('tool_use_completed', {
  tool_name: toolName,
  query_id: queryId,
  duration_ms: duration,
  success: !error,
  error_type: error?.type,
})
```

### 6.3 API 事件

```typescript
// API 调用
logEvent('api_called', {
  model: model,
  input_tokens: usage.inputTokens,
  output_tokens: usage.outputTokens,
  cost_usd: cost,
  duration_ms: duration,
  cached: wasCached,
})

// API 错误
logEvent('api_error', {
  model: model,
  error_type: errorType,
  retryable: isRetryable,
  status_code: statusCode,
})
```

## 7. 隐私保护

### 7.1 数据脱敏

```typescript
// metadata.ts
export function sanitizeMetadata(
  metadata: Record<string, unknown>
): Record<string, unknown> {
  const sanitized: Record<string, unknown> = {}

  for (const [key, value] of Object.entries(metadata)) {
    // 检测可能的路径
    if (typeof value === 'string' && looksLikePath(value)) {
      sanitized[key] = hashPath(value) // 哈希化
    }
    // 检测可能的代码
    else if (typeof value === 'string' && value.length > 1000) {
      sanitized[key] = '[content_hash:' + hashContent(value) + ']'
    }
    // 检测可能的密钥
    else if (isKeyLikeField(key)) {
      sanitized[key] = '[redacted]'
    } else {
      sanitized[key] = value
    }
  }

  return sanitized
}

function looksLikePath(str: string): boolean {
  return /^[~/]|[a-zA-Z]:\\|^\.+\//.test(str)
}

function isKeyLikeField(key: string): boolean {
  const keyPatterns = [/key/i, /token/i, /secret/i, /password/i, /auth/i]
  return keyPatterns.some((p) => p.test(key))
}
```

### 7.2 PII 字段处理

```typescript
export function stripProtoFields<V>(
  metadata: Record<string, V>
): Record<string, V> {
  let result: Record<string, V> | undefined

  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) {
        result = { ...metadata }
      }
      delete result[key]
    }
  }

  return result ?? metadata // 无 _PROTO_ 字段时返回原对象
}
```

## 8. 性能优化

### 8.1 批量发送

```typescript
const BATCH_SIZE = 100
const FLUSH_INTERVAL = 5000

export function createBatchedSink(
  innerSink: AnalyticsSink
): AnalyticsSink {
  const batch: QueuedEvent[] = []
  let flushTimer: NodeJS.Timeout | null = null

  function scheduleFlush(): void {
    if (flushTimer) return
    flushTimer = setTimeout(() => {
      flushTimer = null
      flush()
    }, FLUSH_INTERVAL)
  }

  async function flush(): Promise<void> {
    if (batch.length === 0) return

    const events = batch.splice(0, BATCH_SIZE)

    // 合并相同事件
    const merged = mergeSimilarEvents(events)

    await Promise.all(
      merged.map((e) => innerSink.logEventAsync(e.eventName, e.metadata))
    )
  }

  return {
    logEvent(eventName, metadata) {
      batch.push({ eventName, metadata })

      if (batch.length >= BATCH_SIZE) {
        flush()
      } else {
        scheduleFlush()
      }
    },

    async logEventAsync(eventName, metadata) {
      this.logEvent(eventName, metadata)
      await flush()
    },
  }
}
```

### 8.2 采样

```typescript
export function createSampledSink(
  innerSink: AnalyticsSink,
  sampleRate: number
): AnalyticsSink {
  return {
    logEvent(eventName, metadata) {
      if (Math.random() > sampleRate) return
      innerSink.logEvent(eventName, metadata)
    },

    async logEventAsync(eventName, metadata) {
      if (Math.random() > sampleRate) return
      await innerSink.logEventAsync(eventName, metadata)
    },
  }
}
```

## 9. 调试与监控

### 9.1 调试模式

```typescript
export function enableAnalyticsDebug(): void {
  const originalLogEvent = logEvent

  ;(global as any).logEvent = function (
    eventName: string,
    metadata: LogEventMetadata
  ) {
    console.log('[Analytics]', eventName, metadata)
    return originalLogEvent(eventName, metadata)
  }
}
```

### 9.2 事件计数器

```typescript
const eventCounter: Record<string, number> = {}

export function getEventStats(): Record<string, number> {
  return { ...eventCounter }
}

// 在 logEvent 中计数
export function logEvent(...): void {
  eventCounter[eventName] = (eventCounter[eventName] || 0) + 1
  // ...
}
```

## 10. 创新点与反思

### 10.1 设计创新

1. **类型安全标记**：利用 TypeScript 强制隐私审查
2. **无依赖核心**：避免循环依赖，确保早期初始化
3. **Sink 可组合**：Datadog + 1P + Debug 并行发送

### 10.2 隐私优先

1. **路径哈希化**：保护目录结构隐私
2. **代码内容哈希**：不发送实际代码
3. **PII 隔离**：`_PROTO_` 前缀特殊处理

### 10.3 生产经验

1. **批量发送减少网络请求**
2. **异步队列不阻塞主流程**
3. **Sink 失败隔离，不影响其他**

### 10.4 未来演进

1. **客户端聚合**：复杂指标本地计算
2. **离线支持**：无网络时本地缓冲
3. **用户可控**：可选的详细遥测

---

Claude Code 的 Analytics 系统是一个隐私优先的数据收集方案。通过类型安全标记、无依赖设计、多 Sink 路由等创新，在保护用户隐私的同时提供了丰富的产品洞察。理解其设计，对于构建任何需要遥测的 CLI 工具都具有参考价值。
