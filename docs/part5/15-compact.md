# 第 15 章：上下文压缩（Auto-Compact）

> 遗忘是智能的一部分。知道该记住什么、该忘记什么，是上下文管理的核心。

---

## 15.1 上下文窗口的有限性

Claude 的上下文窗口是有限的。随着对话进行，消息列表不断增长：

```
对话开始：
[系统提示 5K] + [用户消息 0.1K] = 5.1K tokens

10 轮对话后：
[系统提示 5K] + [10 轮对话 × 平均 5K] = 55K tokens

50 轮对话后（含大量工具调用）：
[系统提示 5K] + [50 轮对话 × 平均 5K] = 255K tokens → 超出限制！
```

当上下文接近限制时，有两个选择：
1. **截断**：删除早期消息（简单但会丢失重要信息）
2. **压缩**：用摘要替换详细历史（复杂但保留关键信息）

Claude Code 选择了压缩。

---

## 15.2 Auto-Compact 的触发机制

`src/services/compact/autoCompact.ts` 实现了自动压缩的触发逻辑：

```typescript
export function calculateTokenWarningState(
  messages: Message[],
  maxTokens: number
): TokenWarningState {
  const currentTokens = estimateTokenCount(messages)
  const ratio = currentTokens / maxTokens

  if (ratio > 0.95) {
    return 'critical'   // 必须立即压缩
  } else if (ratio > 0.85) {
    return 'warning'    // 建议压缩
  } else {
    return 'normal'
  }
}

export function isAutoCompactEnabled(): boolean {
  // 检查用户配置和环境变量
  return !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_COMPACT)
}
```

当 token 使用量超过 85% 时显示警告，超过 95% 时强制压缩。

---

## 15.3 压缩的核心算法

压缩的核心思想：**用 Claude 自己来总结对话历史**。

![Auto-Compact 压缩流程](../diagrams/auto-compact-flow.svg)

代码实现：

```typescript
// src/services/compact/compact.ts（简化）
async function compactConversation(
  messages: Message[],
  systemPrompt: SystemPrompt
): Promise<Message[]> {

  // 1. 找到压缩边界（保留最近的 N 条消息不压缩）
  const { toCompress, toKeep } = splitAtCompactBoundary(messages)

  // 2. 用 Claude 生成摘要
  const summary = await generateSummary(toCompress, systemPrompt)

  // 3. 构建压缩后的消息列表
  return [
    // 摘要作为第一条用户消息
    createUserMessage({
      content: `[对话历史摘要]\n${summary}`
    }),
    // 保留最近的消息（完整）
    ...toKeep
  ]
}
```

---

## 15.4 摘要生成的策略

生成摘要时，Claude Code 会指示 Claude 保留哪些信息：

```
请总结以下对话历史，保留：
1. 已完成的任务和结果
2. 重要的决策和原因
3. 当前正在进行的任务状态
4. 关键的代码变更（文件名和变更摘要）
5. 用户的重要偏好和约束

不需要保留：
- 工具调用的详细输出（只保留结果）
- 中间步骤的详细过程
- 已解决的错误的详细信息
```

这个策略确保摘要包含"继续工作所需的最少信息"。

---

## 15.5 压缩边界的选择

不是所有消息都可以压缩。有些消息必须保留完整：

```typescript
function splitAtCompactBoundary(messages: Message[]) {
  // 规则 1：保留最近的 N 条消息（默认 10 条）
  // 规则 2：不在工具调用中间压缩（工具调用和结果必须成对）
  // 规则 3：不压缩包含思考块的消息（思考块有严格的位置规则）
  // 规则 4：保留用户最近的明确指令

  const KEEP_RECENT = 10
  const boundary = findSafeBoundary(messages, KEEP_RECENT)

  return {
    toCompress: messages.slice(0, boundary),
    toKeep: messages.slice(boundary)
  }
}
```

---

## 15.6 Reactive Compact：响应式压缩

除了基于 token 数量的自动压缩，Claude Code 还有响应式压缩（`REACTIVE_COMPACT` feature flag）：

```typescript
// 当 API 返回 prompt_too_long 错误时，自动触发压缩
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null

// 在 query.ts 中
if (error.type === 'prompt_too_long') {
  if (reactiveCompact) {
    // 压缩后重试
    messages = await reactiveCompact.compact(messages)
    continue  // 重新发起请求
  }
}
```

这是一个"防御性"机制：即使自动压缩没有及时触发，也能在 API 报错时自动恢复。

---

## 15.7 Snip Compact：精细化压缩

`HISTORY_SNIP` feature flag 启用了更精细的压缩策略：

```typescript
// 不是压缩整个历史，而是"剪切"特定的片段
// 例如：一个已完成的子任务的详细过程可以被剪切，只保留结果
const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null
```

Snip Compact 的优势：
- 保留更多有用的上下文
- 只压缩真正不需要的部分
- 压缩后的历史更连贯

---

## 15.8 Context Collapse：极端压缩

当上下文极度紧张时，`CONTEXT_COLLAPSE` feature flag 启用极端压缩：

```typescript
const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? require('./services/contextCollapse/index.js')
  : null
```

Context Collapse 会更激进地压缩历史，牺牲一些上下文完整性来换取更多的工作空间。

---

## 15.9 压缩的 UI 反馈

压缩是一个重要的操作，Claude Code 会给用户明确的反馈：

```
⠸ 正在压缩对话历史...

✓ 对话已压缩
  原始：127,432 tokens
  压缩后：23,891 tokens
  节省：81%

  摘要包含：
  - 已完成：重构了 UserService（3 个文件）
  - 已完成：修复了登录 bug
  - 进行中：添加用户权限系统
```

这让用户知道发生了什么，以及压缩保留了哪些关键信息。

---

## 15.10 手动压缩：/compact 命令

用户也可以手动触发压缩：

```bash
> /compact
```

手动压缩的使用场景：
- 完成一个大任务后，清理历史，开始新任务
- 上下文中有很多不再需要的工具调用结果
- 想要"重置"上下文，但保留关键信息

---

## 15.11 压缩的局限性

压缩不是万能的，有几个局限：

**信息丢失**：摘要总是会丢失一些细节。如果后续任务需要这些细节，Claude 可能需要重新读取文件。

**摘要质量**：摘要的质量取决于 Claude 的判断。有时 Claude 可能压缩了重要信息，或保留了不重要的信息。

**压缩成本**：生成摘要本身需要 API 调用，有时间和费用成本。

**不可逆**：压缩后的历史无法恢复（除非有备份）。

---

## 15.12 设计启示：遗忘的艺术

Auto-Compact 揭示了一个深刻的设计原则：**智能系统需要有选择地遗忘**。

人类的记忆也是这样工作的：我们不记得每一个细节，但我们记得重要的事情。这种选择性遗忘让我们能够处理新信息，而不被旧信息淹没。

对于 AI Agent 系统，上下文管理是一个核心工程问题。Auto-Compact 是 Claude Code 对这个问题的工程解答：**用 AI 来管理 AI 的记忆**。

---

## 15.13 小结

Auto-Compact 是 Claude Code Context Engineering 的关键组件：

- **触发机制**：基于 token 使用率（85% 警告，95% 强制）
- **压缩算法**：用 Claude 生成摘要，替换详细历史
- **边界保护**：不在工具调用中间压缩，保留最近消息
- **多种策略**：Auto-Compact、Reactive Compact、Snip Compact、Context Collapse
- **用户可见**：压缩过程和结果对用户透明

---

*下一章：[任务系统设计](../part6/16-task-system.md)*
