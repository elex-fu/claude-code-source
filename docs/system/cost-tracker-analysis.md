# Claude Code Cost Tracker 技术分析

## 1. 模块总结介绍

Cost Tracker 是 Claude Code 的「成本计量中心」，负责实时追踪 API 调用成本、Token 使用量和模型费用。该模块为用户提供了完全透明的成本可见性，帮助用户理解和控制使用成本。

核心功能：
- **实时成本计算**：每次 API 调用后立即更新成本
- **多维度统计**：按模型、按功能、按会话的成本归因
- **预算预警**：接近成本阈值时发出警告
- **历史对比**：与历史会话成本对比分析

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                       Cost Tracker                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Token Counter │  │   Model Usage   │  │   Budget    │ │
│  │                 │  │                 │  │   Control   │ │
│  │  Input Tokens   │  │  Per-Model      │  │             │ │
│  │  Output Tokens  │  │  Cost Tracking  │  │  Thresholds │ │
│  │  Cache Tokens   │  │                 │  │  Alerts     │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬──────┘ │
│           │                    │                  │        │
│           └────────────────────┼──────────────────┘        │
│                                ▼                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Cost Aggregation Engine                 │  │
│  │                                                      │  │
│  │  • Session Total Cost                               │  │
│  │  • Model Breakdown                                  │  │
│  │  • API Duration                                     │  │
│  │  • Cost Per Query                                   │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心数据结构

### 3.1 模型使用量

```typescript
// entrypoints/agentSdkTypes.ts
type ModelUsage = {
  // Token 使用量
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number

  // 成本（美元）
  costUSD: number

  // API 调用统计
  apiCallCount: number
  totalApiDuration: number

  // 上下文窗口
  contextWindow: number
}
```

### 3.2 成本状态

```typescript
// cost-tracker.ts
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number

  // 代码变更统计
  totalLinesAdded: number
  totalLinesRemoved: number

  // 上次操作耗时
  lastDuration: number | undefined

  // 各模型详细统计
  modelUsage: { [modelName: string]: ModelUsage } | undefined
}
```

## 4. 成本计算引擎

### 4.1 Token 成本计算

```typescript
// utils/modelCost.ts
interface ModelPricing {
  inputCostPerToken: number      // 每输入 token 成本
  outputCostPerToken: number     // 每输出 token 成本
  cacheReadCostPerToken: number  // 缓存读取成本
  cacheWriteCostPerToken: number // 缓存写入成本
}

const MODEL_PRICING: Record<string, ModelPricing> = {
  'claude-opus-4-6': {
    inputCostPerToken: 0.000_015,
    outputCostPerToken: 0.000_075,
    cacheReadCostPerToken: 0.000_001_875,
    cacheWriteCostPerToken: 0.000_018_75,
  },
  'claude-sonnet-4-6': {
    inputCostPerToken: 0.000_003,
    outputCostPerToken: 0.000_015,
    cacheReadCostPerToken: 0.000_000_375,
    cacheWriteCostPerToken: 0.000_003_75,
  },
  // ... 其他模型
}
```

### 4.2 成本计算函数

```typescript
export function calculateUSDCost(
  model: string,
  usage: {
    inputTokens: number
    outputTokens: number
    cacheReadInputTokens?: number
    cacheCreationInputTokens?: number
  }
): number {
  const pricing = MODEL_PRICING[model]
  if (!pricing) {
    setHasUnknownModelCost(true)
    return 0
  }

  const inputCost = usage.inputTokens * pricing.inputCostPerToken
  const outputCost = usage.outputTokens * pricing.outputCostPerToken

  // 缓存成本
  const cacheReadCost =
    (usage.cacheReadInputTokens || 0) * pricing.cacheReadCostPerToken
  const cacheWriteCost =
    (usage.cacheCreationInputTokens || 0) * pricing.cacheWriteCostPerToken

  return inputCost + outputCost + cacheReadCost + cacheWriteCost
}
```

## 5. 实时成本追踪

### 5.1 API 调用成本记录

```typescript
// services/api/claude.ts
export async function queryModelWithStreaming(
  params: QueryParams
): Promise<StreamResponse> {
  const startTime = Date.now()

  const response = await anthropic.messages.create({
    model: params.model,
    messages: params.messages,
    stream: true,
  })

  let inputTokens = 0
  let outputTokens = 0

  for await (const chunk of response) {
    // 处理流式响应...

    // 更新 token 计数
    if (chunk.type === 'message_start') {
      inputTokens = chunk.message.usage.input_tokens
    }
    if (chunk.type === 'content_block_delta') {
      outputTokens += estimateTokens(chunk.delta.text)
    }
  }

  // 记录成本
  const duration = Date.now() - startTime
  updateUsage({
    model: params.model,
    inputTokens,
    outputTokens,
    duration,
  })

  return response
}
```

### 5.2 成本状态更新

```typescript
// bootstrap/state.ts - 全局状态存储
export function updateUsage(usage: {
  model: string
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens?: number
  cacheCreationInputTokens?: number
  duration: number
}): void {
  const costUSD = calculateUSDCost(usage.model, usage)

  // 更新总成本
  addToTotalCostState(costUSD)

  // 更新模型特定统计
  const modelUsage = getModelUsage(usage.model)
  modelUsage.inputTokens += usage.inputTokens
  modelUsage.outputTokens += usage.outputTokens
  modelUsage.costUSD += costUSD
  modelUsage.apiCallCount += 1
  modelUsage.totalApiDuration += usage.duration

  // 触发成本更新事件
  notifyCostUpdate()
}
```

## 6. 多维度成本归因

### 6.1 按模型统计

```typescript
export function getUsageForModel(model: string): ModelUsage {
  return (
    getModelUsage(model) || {
      inputTokens: 0,
      outputTokens: 0,
      cacheReadInputTokens: 0,
      cacheCreationInputTokens: 0,
      costUSD: 0,
      apiCallCount: 0,
      totalApiDuration: 0,
      contextWindow: getContextWindowForModel(model),
    }
  )
}

export function getAllModelUsage(): Record<string, ModelUsage> {
  return getCostCounter().modelUsage || {}
}
```

### 6.2 按功能统计

```typescript
// 在 QueryEngine.ts 中按功能追踪
type QuerySource =
  | 'main_query'        // 主查询
  | 'compact'           // 上下文压缩
  | 'tool_use_summary'  // 工具结果摘要
  | 'agent_subtask'     // 子代理任务
  | 'hook_agent'        // Hook 子代理

function trackCostBySource(source: QuerySource, usage: Usage): void {
  const sourceKey = `cost_by_source.${source}`
  const current = getSessionStorage(sourceKey) || 0
  setSessionStorage(sourceKey, current + usage.costUSD)
}
```

## 7. 成本展示与报告

### 7.1 /cost 命令实现

```typescript
// commands/cost/cost.ts
export async function showCostReport(): Promise<string> {
  const totalCost = getTotalCostUSD()
  const modelUsage = getAllModelUsage()

  let report = `Session Cost Report\n`
  report += `===================\n\n`
  report += `Total Cost: ${formatCost(totalCost)}\n`
  report += `Total API Duration: ${formatDuration(getTotalAPIDuration())}\n\n`

  report += `By Model:\n`
  for (const [model, usage] of Object.entries(modelUsage)) {
    report += `  ${model}:\n`
    report += `    Cost: ${formatCost(usage.costUSD)}\n`
    report += `    Input: ${formatNumber(usage.inputTokens)} tokens\n`
    report += `    Output: ${formatNumber(usage.outputTokens)} tokens\n`
    report += `    Calls: ${usage.apiCallCount}\n\n`
  }

  return report
}
```

### 7.2 格式化函数

```typescript
export function formatCost(costUSD: number): string {
  if (costUSD < 0.01) {
    return `${(costUSD * 100).toFixed(2)}¢`
  }
  return `$${costUSD.toFixed(2)}`
}

export function formatNumber(num: number): string {
  return num.toLocaleString()
}

export function formatDuration(ms: number): string {
  const seconds = Math.floor(ms / 1000)
  const minutes = Math.floor(seconds / 60)

  if (minutes > 0) {
    return `${minutes}m ${seconds % 60}s`
  }
  return `${seconds}s`
}
```

## 8. 成本预算与预警

### 8.1 预算配置

```typescript
// utils/config.ts
interface CostConfig {
  costThreshold?: number        // 成本阈值（美元）
  costAlertEnabled: boolean     // 是否启用预警
  costAlertThreshold: number    // 预警阈值（占预算百分比）
}
```

### 8.2 预算检查

```typescript
// cost-tracker.ts
export function checkCostThreshold(): CostAlert | null {
  const config = getCostConfig()
  if (!config.costThreshold) return null

  const totalCost = getTotalCostUSD()
  const ratio = totalCost / config.costThreshold

  if (ratio >= 1) {
    return {
      type: 'exceeded',
      message: `Cost threshold exceeded: ${formatCost(totalCost)} / ${formatCost(config.costThreshold)}`,
    }
  }

  if (ratio >= config.costAlertThreshold && config.costAlertEnabled) {
    return {
      type: 'warning',
      message: `Cost warning: ${(ratio * 100).toFixed(0)}% of budget used`,
    }
  }

  return null
}
```

### 8.3 预警对话框

```typescript
// CostThresholdDialog.tsx
export function CostThresholdDialog({ alert }: { alert: CostAlert }) {
  return (
    <Dialog>
      <Text color={alert.type === 'exceeded' ? 'red' : 'yellow'}>
        {alert.type === 'exceeded' ? '⚠️ ' : 'ℹ️ '}
        {alert.message}
      </Text>
      <Text>Continue anyway?</Text>
      <Button onClick={() => dismissAlert()}>Continue</Button>
      <Button onClick={() => exit()}>Exit</Button>
    </Dialog>
  )
}
```

## 9. 会话间成本持久化

### 9.1 保存会话成本

```typescript
export function saveCurrentSessionCosts(): void {
  const sessionId = getSessionId()
  const costState: StoredCostState = {
    totalCostUSD: getTotalCostUSD(),
    totalAPIDuration: getTotalAPIDuration(),
    totalAPIDurationWithoutRetries: getTotalAPIDurationWithoutRetries(),
    totalToolDuration: getTotalToolDuration(),
    totalLinesAdded: getTotalLinesAdded(),
    totalLinesRemoved: getTotalLinesRemoved(),
    lastDuration: getLastDuration(),
    modelUsage: getAllModelUsage(),
  }

  saveCurrentProjectConfig({
    lastSessionId: sessionId,
    lastCostState: costState,
  })
}
```

### 9.2 加载历史成本

```typescript
export function getStoredSessionCosts(
  sessionId: string
): StoredCostState | undefined {
  const projectConfig = getCurrentProjectConfig()

  // 仅当是同一会话时返回
  if (projectConfig.lastSessionId !== sessionId) {
    return undefined
  }

  return projectConfig.lastCostState
}
```

## 10. 创新点与反思

### 10.1 设计创新

1. **实时成本流**：每次 API 调用后立即更新成本，而非事后计算
2. **多维度归因**：支持按模型、按功能、按会话的成本分析
3. **缓存成本追踪**：单独的缓存读取/写入成本计算

### 10.2 架构权衡

**实时 vs 精确**

- 流式响应中，output tokens 是估计值（最终统计由 API 返回）
- 实时显示的成本可能有误差，最终成本以 API 返回为准

### 10.3 未来演进

1. **成本预测**：基于对话趋势预测最终成本
2. **智能模型选择**：根据任务复杂度自动选择最经济的模型
3. **团队成本看板**：多用户成本聚合与报告

---

Claude Code 的 Cost Tracker 是一个功能完整的成本计量系统。通过实时追踪、多维度归因、预算预警等设计，为用户提供了完全透明的成本可见性。理解其设计，对于构建任何需要成本追踪的 LLM 应用都具有参考价值。
