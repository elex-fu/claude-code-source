# 第 6 章：查询引擎 —— 对话的心脏

> 如果说 Claude Code 是一台机器，QueryEngine 就是它的发动机。

---

## 6.1 QueryEngine 的职责

`QueryEngine`（`src/QueryEngine.ts`，1295 行）是 Claude Code 中最核心的类。它管理一次完整对话的全生命周期：

- 维护消息历史
- 调用 Claude API
- 执行工具调用
- 管理 token 预算
- 处理错误和重试
- 触发上下文压缩

一句话：**用户的每一次输入，到最终输出，都经过 QueryEngine 的编排**。

---

## 6.2 QueryEngine 的配置

```typescript
// src/QueryEngine.ts
export type QueryEngineConfig = {
  cwd: string                          // 工作目录
  tools: Tools                         // 可用工具集
  commands: Command[]                  // 可用斜杠命令
  mcpClients: MCPServerConnection[]    // MCP 服务器连接
  agents: AgentDefinition[]            // 代理定义
  canUseTool: CanUseToolFn             // 权限检查函数
  getAppState: () => AppState          // 读取全局状态
  setAppState: (f) => void             // 更新全局状态
  initialMessages?: Message[]          // 初始消息（用于恢复会话）
  readFileCache: FileStateCache        // 文件读取缓存
  customSystemPrompt?: string          // 自定义系统提示
  appendSystemPrompt?: string          // 追加系统提示
  userSpecifiedModel?: string          // 用户指定模型
  maxTurns?: number                    // 最大轮次限制
  maxBudgetUsd?: number                // 最大费用限制（美元）
  taskBudget?: { total: number }       // token 预算
  jsonSchema?: Record<string, unknown> // 结构化输出 schema
  handleElicitation?: ...              // MCP 权限请求处理
}
```

这个配置揭示了 QueryEngine 的设计思路：**它是无状态的配置驱动**。所有行为都由配置决定，QueryEngine 本身不持有业务逻辑，只负责编排。

---

## 6.3 QueryEngine 的内部状态

```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]           // 消息历史（可变）
  private abortController: AbortController     // 中断控制器
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage         // 累计 token 使用量
  private discoveredSkillNames = new Set<string>()  // 已发现的 Skills
  private loadedNestedMemoryPaths = new Set<string>() // 已加载的 Memory 路径
}
```

注意 `mutableMessages`：这是整个对话的消息列表，每次工具调用的结果都会追加到这里。这个列表就是 Claude 的"记忆"——它能看到的所有历史。

---

## 6.4 submitMessage：一次对话轮次的完整流程

![Query Engine 执行流程](../diagrams/query-engine-flow.svg)

`submitMessage` 是 QueryEngine 的核心方法，处理一次用户输入：

---

## 6.5 query()：真正的执行循环

`query()` 函数（`src/query.ts`，1729 行）是实际的 Agent 循环：

![query() 执行循环](../diagrams/query-loop.svg)

简化的伪代码：

```typescript
// 简化的伪代码，展示核心逻辑
async function* query(params: QueryParams) {
  let messages = params.messages
  let turnCount = 0

  while (true) {
    turnCount++

    // 检查轮次限制
    if (turnCount > maxTurns) break

    // 调用 Claude API（流式）
    const stream = await callClaudeAPI({
      messages,
      systemPrompt,
      tools,
      model,
    })

    // 解析流式响应
    const toolCalls = []
    for await (const chunk of stream) {
      if (chunk.type === 'text') {
        yield { type: 'text', content: chunk.text }  // 流式输出文本
      } else if (chunk.type === 'thinking') {
        // 思考块，内部处理，不输出给用户
      } else if (chunk.type === 'tool_use') {
        toolCalls.push(chunk)
      }
    }

    // 如果没有工具调用，对话结束
    if (toolCalls.length === 0) break

    // 执行工具调用（可并行）
    const toolResults = await runTools(toolCalls, context)

    // 工具结果追加到消息列表
    messages = [...messages, assistantMessage, ...toolResults]

    // 检查 token 预算
    if (tokenBudgetExceeded(messages)) {
      yield { type: 'budget_exceeded' }
      break
    }
  }
}
```

这个循环有几个关键设计决策：

**生成器函数（`async function*`）**：`query()` 是一个异步生成器，支持流式输出。调用方可以逐步接收结果，不需要等待整个任务完成。

**工具并行执行**：Claude 可以在一次响应中请求多个工具调用，`runTools()` 会并行执行它们，显著提升效率。

**消息不可变追加**：每轮的消息都追加到列表末尾，不修改历史。这保证了对话历史的完整性和可审计性。

---

## 6.6 思考块（Thinking Blocks）的处理

Claude 支持"扩展思考"模式，在生成回答前先进行内部推理。`query.ts` 中有专门的注释解释思考块的规则：

```typescript
/**
 * The rules of thinking are lengthy and fortuitous...
 *
 * 1. 包含 thinking 块的消息必须在 max_thinking_length > 0 的请求中
 * 2. thinking 块不能是消息的最后一个块
 * 3. thinking 块必须在整个助手轨迹中保留
 *    （单轮，或包含工具调用时延伸到工具结果和下一个助手消息）
 */
```

这些规则很严格，违反会导致 API 错误。Claude Code 在消息规范化时会仔细处理这些约束。

---

## 6.7 错误恢复机制

`query.ts` 实现了多层错误恢复：

**max_output_tokens 恢复**：
```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

// 当响应被截断时，自动继续生成
// 最多重试 3 次
```

**API 错误重试**：通过 `categorizeRetryableAPIError()` 判断错误是否可重试（网络超时、速率限制等），自动重试。

**工具执行错误**：工具执行失败时，错误信息作为工具结果返回给 Claude，Claude 可以根据错误调整策略。

**中断处理**：用户按 `Ctrl+C` 时，`AbortController` 触发中断，`yieldMissingToolResultBlocks()` 为未完成的工具调用生成错误结果，保证消息列表的完整性。

---

## 6.8 token 预算系统

Claude Code 有精细的 token 预算管理：

```typescript
// src/query/tokenBudget.ts
export function createBudgetTracker(config) {
  return {
    checkBudget(messages): BudgetStatus {
      const currentTokens = estimateTokens(messages)
      if (currentTokens > config.maxTokens) {
        return { exceeded: true, reason: 'token_limit' }
      }
      if (config.maxBudgetUsd && estimateCost(currentTokens) > config.maxBudgetUsd) {
        return { exceeded: true, reason: 'cost_limit' }
      }
      return { exceeded: false }
    }
  }
}
```

预算系统有两个维度：
- **token 数量**：防止上下文超出模型限制
- **费用（USD）**：防止意外的高额账单

当预算超出时，QueryEngine 会停止执行并通知用户，而不是静默失败。

---

## 6.9 会话持久化

每次对话结束后，QueryEngine 会保存会话状态：

```typescript
// 保存到本地文件系统
await flushSessionStorage()
await recordTranscript(messages)
```

这让用户可以：
- 恢复之前的对话（`/resume`）
- 查看历史记录
- 分析 token 使用情况

---

## 6.10 QueryEngine vs query()：职责分离

| 职责 | QueryEngine | query() |
|------|-------------|---------|
| 消息历史管理 | ✓ | ✗ |
| 会话状态 | ✓ | ✗ |
| token 累计统计 | ✓ | ✗ |
| 会话持久化 | ✓ | ✗ |
| API 调用循环 | ✗ | ✓ |
| 工具执行 | ✗ | ✓ |
| 流式输出 | ✗ | ✓ |
| 思考块处理 | ✗ | ✓ |

`QueryEngine` 是**会话管理者**，`query()` 是**执行引擎**。这种分离让两者都可以独立测试和演化。

---

## 6.11 小结

QueryEngine 的设计体现了几个重要原则：

1. **配置驱动**：行为由配置决定，不硬编码业务逻辑
2. **流式优先**：使用生成器函数，支持实时输出
3. **不可变历史**：消息只追加，不修改，保证可审计性
4. **多层错误恢复**：从 API 错误到工具失败，都有对应的恢复策略
5. **预算意识**：token 和费用双维度预算，防止失控

这些原则共同构成了一个可靠、可观察、可控的 Agent 执行引擎。

---

*下一章：[状态管理设计](./07-state-management.md)*
