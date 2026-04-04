# Claude Code Prompt System 技术深度分析

## 1. 模块总结介绍

Prompt System（提示系统）是 Claude Code 的核心基础设施模块，负责将内部状态化的对话历史、工具定义、环境上下文和系统指令，组装成符合 Anthropic API 规范的请求负载。该模块贯穿于从用户输入到 API 调用的完整链路，涵盖了系统提示词构建、消息规范化、上下文压缩、Token 管理和缓存策略等关键能力。

从架构定位上看，Prompt System 是连接"应用层对话状态"与"LLM API 接口"的 Adapter 层。它需要处理高度复杂的问题：消息类型多样性（user/assistant/system/attachment/progress 等）、多轮工具调用的配对约束、动态上下文（git 状态、MCP 服务端指令、Claude.md）的注入、Prompt Caching 的命中优化，以及当对话超出上下文窗口时的自动压缩（Auto-Compact）。

本分析基于 `/Users/lex/play/claude-code-sourcemap/restored-src/src` 下的核心源码，重点围绕 `context.ts`、`utils/messages.ts`、`services/compact/`、`utils/api.ts`、`constants/prompts.ts` 以及 `utils/systemPrompt.ts` 等文件展开。

---

## 2. 系统架构

Prompt System 的架构可以抽象为四个核心层次：

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Layer                                │
│   (services/api/claude.ts — BetaMessage API 调用构建)            │
├─────────────────────────────────────────────────────────────────┤
│                     Compaction Layer                            │
│   (services/compact/ — 自动/手动上下文压缩与摘要生成)               │
├─────────────────────────────────────────────────────────────────┤
│                      Message Layer                              │
│   (utils/messages.ts — 消息规范化、配对修复、合并、序列化)          │
├─────────────────────────────────────────────────────────────────┤
│                      Context Layer                              │
│   (context.ts / constants/prompts.ts / utils/systemPrompt.ts)   │
└─────────────────────────────────────────────────────────────────┘
```

**Context Layer** 负责生成系统提示词（System Prompt）和上下文注入（Context Injection）；**Message Layer** 负责原始对话历史的类型归一化和 API 适配；**Compaction Layer** 在 Token 用量接近阈值时触发，用摘要替换旧消息；**API Layer** 将前三层产物组装为最终的 API 请求体（messages + system + tools + betas）。

---

## 3. 分层说明

### 3.1 Context Layer（上下文层）

上下文层的核心是 `src/context.ts` 和 `src/constants/prompts.ts`，它们分别提供了**动态上下文**和**静态系统提示词**的生成能力。

**系统上下文（System Context）**由 `getSystemContext()` 提供，是一个 memoize 缓存的异步函数，会话期间不会改变。它包含：
- `gitStatus`：当前分支、默认分支、最近提交、工作区状态（截断至 2000 字符）。
- `cacheBreaker`：仅内部使用，用于主动打破 Prompt Cache（`BREAK_CACHE_COMMAND` feature 下生效）。

**用户上下文（User Context）**由 `getUserContext()` 提供，同样 memoize 缓存，包含：
- `claudeMd`：从工作目录及 `--add-dir` 指定的目录中收集的 `CLAUDE.md` 文件内容。
- `currentDate`：当前本地 ISO 日期。

这两个上下文对象最终通过 `prependUserContext()`（位于 `src/utils/api.ts`）以 `<system-reminder>` 包裹的形式注入到第一条用户消息中：

```typescript
// src/utils/api.ts:461-473
return [
  createUserMessage({
    content: `<system-reminder>\nAs you answer the user's questions, you can use the following context:\n${Object.entries(context)
      .map(([key, value]) => `# ${key}\n${value}`)
      .join('\n')}\n\n      IMPORTANT: this context may or may not be relevant to your tasks. You should not respond to this context unless it is highly relevant to your task.\n</system-reminder>\n`,
    isMeta: true,
  }),
  ...messages,
]
```

**系统提示词（System Prompt）**的生成逻辑位于 `src/constants/prompts.ts` 的 `getSystemPrompt()` 函数中。它支持多条路径：
- **简单模式**（`CLAUDE_CODE_SIMPLE`）：极为精简的系统提示。
- **主动模式**（`PROACTIVE` / `KAIROS`）：面向自主代理的精简提示。
- **默认模式**：包含 `# System`、`# Doing tasks`、`# Executing actions with care`、`# Using your tools`、`# Tone and style`、`# Output efficiency` 等多个章节。

`src/utils/systemPrompt.ts` 中的 `buildEffectiveSystemPrompt()` 则负责根据优先级组合最终提示：
1. `overrideSystemPrompt`（最高优先级，完全替换）
2. `coordinator` 模式提示
3. `agentSystemPrompt`（主线程代理定义）
4. `customSystemPrompt`（用户通过 `--system-prompt` 传入）
5. `defaultSystemPrompt`（默认提示词）

此外，`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）作为硬编码边界标记，将系统提示词分为**静态前缀**（可全局缓存）和**动态后缀**（会话相关、不可全局缓存），这是 Prompt Caching 策略的关键基础（`src/utils/api.ts: splitSysPromptPrefix()`）。

### 3.2 Message Layer（消息层）

Message Layer 的核心文件是 `src/utils/messages.ts`（超过 5500 行）。它负责将内部丰富的消息类型（`Message` Union Type）转换为 API 可用的标准格式，同时处理大量边缘情况。

**消息规范化（Normalization）**分为两个阶段：

#### (1) UI 层规范化：`normalizeMessages()`
将包含多内容块的消息拆分为多条单内容块的消息。例如，一个 `assistant` 消息包含 `[text, tool_use, text]`，会被拆成三条 `NormalizedAssistantMessage`。

```typescript
// src/utils/messages.ts:741-829
export function normalizeMessages(messages: Message[]): NormalizedMessage[] {
  let isNewChain = false
  return messages.flatMap(message => {
    switch (message.type) {
      case 'assistant': {
        isNewChain = isNewChain || message.message.content.length > 1
        return message.message.content.map((_, index) => {
          const uuid = isNewChain ? deriveUUID(message.uuid, index) : message.uuid
          return { type: 'assistant' as const, ..., message: { ...message.message, content: [_] }, ...}
        })
      }
      // ... user / attachment / progress / system
    }
  })
}
```

这里的关键设计是 `deriveUUID()`：当消息被拆分后，子消息通过确定性算法（基于父 UUID 和索引）生成新 UUID，保证 UI 渲染稳定性和幂等性。

#### (2) API 层规范化：`normalizeMessagesForAPI()`
这是真正发送给 API 前的最后一次清洗，逻辑极其复杂：
- **重排附件**：`reorderAttachmentsForAPI()` 将 `attachment` 消息上溯合并到最近的 `user` 或 `tool_result` 消息中。
- **移除虚拟消息**：`isVirtual` 标记的消息仅用于 REPL 等内部展示，不发给 API。
- **修复工具调用配对**：`ensureToolResultPairing()` 确保每个 `tool_use` 都有对应的 `tool_result`，否则会插入合成占位符或剥离孤立的 `tool_result`。
- **合并连续用户消息**：因为 Bedrock 不支持连续多条 `user` 消息，必须通过 `mergeUserMessages()` 合并。
- **合并连续 Assistant 消息**：通过 `message.id` 匹配，合并同一次 API 响应中的多个内容块。
- **工具输入清洗**：`normalizeToolInputForAPI()` 会剥离仅在内部使用的字段（如 `ExitPlanModeV2` 的 `plan` 和 `planFilePath`）。
- **图片/文档大小超限容错**：通过 `stripTargets` 映射，当某条用户消息中的图片/PDF 因为过大被 API 拒绝后，后续请求会自动从该 `isMeta` 消息中剥离这些元素，防止反复 400 错误。
- **工具引用边界注入**：当消息包含 `tool_reference` 块但其后没有文本块时，会追加 `TOOL_REFERENCE_TURN_BOUNDARY` 文本，防止模型误将其当作停止序列。

### 3.3 Compaction Layer（压缩层）

Compaction Layer 由 `src/services/compact/` 目录下的多个模块组成，核心目标是**在上下文窗口耗尽前，用 LLM 生成的摘要替换掉早期的对话历史**。

**自动压缩（Auto-Compact）**的触发逻辑在 `src/services/compact/autoCompact.ts` 中：

```typescript
// src/services/compact/autoCompact.ts
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS // 默认缓冲 13_000 tokens
}
```

当 `tokenCountWithEstimation(messages) - snipTokensFreed` 超过该阈值时，`shouldAutoCompact()` 返回 `true`，随后主流程调用 `compactConversation()`（`src/services/compact/compact.ts`）。

**压缩摘要生成**使用了一个专门的 forked agent（`runForkedAgent`），它不带工具集，仅接收一段提示词和完整对话历史，输出 `<analysis>` + `<summary>` 结构化的文本。提示词定义在 `src/services/compact/prompt.ts`：

```typescript
// src/services/compact/prompt.ts:19-25
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`
```

提示词模板分为 `BASE_COMPACT_PROMPT`（总结全部对话）和 `PARTIAL_COMPACT_PROMPT`（仅总结未被保留的近期消息）。摘要输出后续经过 `formatCompactSummary()` 处理，剥离 `<analysis>` 草稿区，替换 `<summary>` 标签为可读标题。

**压缩后的消息注入**通过 `getCompactUserSummaryMessage()` 生成一条新的 `user` 消息，告知模型"这是从之前对话继续的摘要"，并可以选择性地保留最近 N 条消息不变（`recentMessagesPreserved`）。

**Post-Compact 清理** (`src/services/compact/postCompactCleanup.ts`) 会尝试恢复摘要中引用的关键文件内容（最多 5 个文件，每个 5000 tokens），以弥补摘要可能丢失的精确代码细节。

### 3.4 API Layer（API 层）

API Layer 位于 `src/services/api/claude.ts`，负责将整理好的 Messages、System Prompt、Tools 组装为 `anthropic.beta.messages.create()` 的参数。

**System Prompt 分块与缓存**：`buildSystemPromptBlocks()` 将系统提示词拆分为多个 `text_block`，并附加 `cache_control: { type: 'ephemeral', scope, ttl }`。分块策略由 `splitSysPromptPrefix()`（`src/utils/api.ts`）决定：
- `attributionHeader` → `cacheScope: null`
- `CLI_SYSPROMPT_PREFIXES` → `cacheScope: null` 或 `org`
- 边界前的静态内容 → `cacheScope: 'global'`（仅 1P 且启用全局缓存时）
- 边界后的动态内容 → `cacheScope: null`

**消息参数转换**：`userMessageToMessageParam()` 和 `assistantMessageToMessageParam()` 负责将内部消息结构映射为 SDK 的 `MessageParam`。当启用 Prompt Caching 时，会在消息内容的最后一个块上附加 `cache_control`。

**工具模式构建**：`toolToAPISchema()`（`src/utils/api.ts`）将内部 `Tool` 对象转换为 API 的 `BetaToolUnion`。此处包含多个 feature gate：
- `tengu_tool_pear`：启用 `strict` 模式工具（结构化输出）。
- `tengu_fgts`：启用 `eager_input_streaming`（细粒度工具流，1P only）。
- `defer_loading`：针对 Tool Search 功能的延迟加载标记。

**反蒸馏（Anti-Distillation）保护**：在 `getExtraBodyParams()` 中，当满足内部条件时，会注入 `anti_distillation: ['fake_tools']` 参数。

---

## 4. 交互流程

从用户输入到最终 API Payload 的完整交互流程如下：

```
1. 原始输入（用户消息 / 工具结果 / 系统通知 / 附件）
          ↓
2. Context Layer 生成
   - getSystemContext() → gitStatus, cacheBreaker
   - getUserContext() → claudeMd, currentDate
   - getSystemPrompt() → 默认/主动/代理/自定义系统提示词
          ↓
3. prependUserContext() 注入到首条用户消息
          ↓
4. Message Layer 规范化
   - normalizeMessages()        (UI 层拆分多内容块)
   - normalizeMessagesForAPI()  (API 层清洗、合并、配对修复)
          ↓
5. Compaction Layer 检查
   - shouldAutoCompact() ?
   - 若触发：forkedAgent 生成摘要 → 替换早期消息 → postCompactCleanup 恢复关键文件
          ↓
6. API Layer 组装
   - splitSysPromptPrefix() → 系统提示词分块 + cache_control
   - toolToAPISchema() → 工具定义 + defer_loading/strict/FGTS
   - 构建 messages 数组（UserParam + AssistantParam）
   - 附加 betas、metadata、extraBodyParams、task_budget、effort 等
          ↓
7. 发送至 Anthropic Beta Messages API
```

关键数据流示例：
- `src/context.ts` 的缓存上下文 → `src/utils/api.ts: prependUserContext()` → 首条 `isMeta` user message。
- `src/constants/prompts.ts: getSystemPrompt()` → `src/utils/systemPrompt.ts: buildEffectiveSystemPrompt()` → `src/services/api/claude.ts: query()` → `splitSysPromptPrefix()` → `buildSystemPromptBlocks()`。
- `src/utils/messages.ts: normalizeMessagesForAPI()` → `src/services/api/claude.ts: messagesForAPI` → `userMessageToMessageParam()` / `assistantMessageToMessageParam()`。

---

## 5. 技术原理

### 5.1 Prompt Caching 与全局缓存策略

Claude Code 深度利用了 Anthropic 的 Prompt Caching（提示缓存）机制。为了最大化缓存命中率，系统采用了**前缀哈希匹配**策略：API 服务端会对 system prompt 和 messages 的前缀计算哈希，如果与之前的请求匹配，则复用缓存。

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 的设计正是为了服务这一策略。所有 Stable 的内容（核心行为指令、工具使用规范等）放在边界之前，形成**跨会话、跨组织可共享**的 `global` 缓存块；而动态内容（MCP 指令、当前日期、 Claude.md 内容）放在边界之后，不进入全局缓存，避免边界后的轻微变动导致整个系统提示词缓存失效。

```typescript
// src/utils/api.ts:362-410
if (useGlobalCacheFeature) {
  const boundaryIndex = systemPrompt.findIndex(s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
  if (boundaryIndex !== -1) {
    // staticBlocks → cacheScope: 'global'
    // dynamicBlocks → cacheScope: null
  }
}
```

### 5.2 消息拆分与 UUID 稳定性

`normalizeMessages()` 在拆分多内容块消息时，使用 `deriveUUID(parentUUID, index)` 而不是 `randomUUID()`。其原理是：取 UUID 的前 10 位十六进制字符，转为 base36 后取前 6 位，从而得到确定性短 ID。这保证了：
- 同一段对话历史，每次规范化后子消息 ID 不变。
- React UI 不会因为 ID 抖动产生不必要的重渲染。
- 附件、进度条等 UI 组件能稳定绑定到对应的内容块。

### 5.3 工具调用的配对约束修复

LLM API（尤其是 Claude）要求 `tool_use` 和 `tool_result` 必须严格一一对应。在复杂场景下（如远程会话恢复、并发代理、流式中断），可能出现配对断裂。`ensureToolResultPairing()` 通过双向扫描解决：
- 对于孤立的 `tool_use`（没有 `tool_result`），插入一个合成的 `tool_result`，内容为 `SYNTHETIC_TOOL_RESULT_PLACEHOLDER`。
- 对于孤立的 `tool_result`（引用了一个不存在的 `tool_use`），直接剥离该消息。

这个设计非常稳健，它让上层逻辑无需担心流式异常或状态恢复时的配对问题。

### 5.4 Auto-Compact 的 Token 估算与缓冲策略

自动压缩不是等到上下文完全耗尽才触发，而是预留了安全缓冲（`AUTOCOMPACT_BUFFER_TOKENS = 13_000`）。Token 估算函数 `tokenCountWithEstimation()` 结合 API 返回的精确 usage 和本地启发式估算（`roughTokenCountEstimation`），对新加入的消息进行增量估算。

当触发压缩时，系统会 fork 一个轻量级无工具代理，仅在 20_000 output tokens 的预算内生成摘要。这避免了主线程被阻塞，也防止了压缩任务本身占用过大的上下文窗口。

---

## 6. 创新点

### 6.1 系统提示词的"边界分块缓存"设计

通过 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 将系统提示词物理切分为静态和动态两部分，并分别标注 `global` / `org` / `null` 缓存作用域，是 Prompt Caching 工程化落地的一个精妙设计。它使得几十万 tokens 的核心指令能够在数百万次调用间被全局复用，大幅降低延迟和成本。

### 6.2 高度工程化的消息规范化流水线

`normalizeMessagesForAPI()` 不是一个简单的类型转换函数，而是一个完整的消息"编译器"：它集成了附件重排、错误剥离、图片过大容错、工具引用边界修复、虚拟消息过滤、连续消息合并、orphan thinking 过滤等十几个 Pass。这种多 Pass 清洗架构虽然复杂，但有效地将上游各种不确定性隔离在 API 之外。

### 6.3 Prompt Cache Breaker（缓存刷新）

内部调试功能 `BREAK_CACHE_COMMAND` 允许通过 `setSystemPromptInjection()` 在系统上下文中注入一个 ephemeral 的 `[CACHE_BREAKER: xxx]` 标记，强制使当前请求的前缀哈希与缓存不同，从而验证或绕过服务端缓存命中问题。这种可控的缓存失效机制对于排查模型响应 stale 的问题非常有价值。

### 6.4 压缩后的"上下文恢复"（Post-Compact Cleanup）

自动压缩虽然解决了窗口超限问题，但代价是丢失了精确的代码细节。Claude Code 在压缩后不满足于仅仅保留摘要，而是通过 `runPostCompactCleanup()` 智能恢复摘要中引用的最多 5 个关键文件内容（基于 Token 预算），在"保留长期记忆"和"维持短期精确性"之间取得了平衡。

### 6.5 针对工具调用结果的长上下文优化

`getFunctionResultClearingSection()` 会在系统提示词中告知模型："旧工具结果将被自动清除以释放空间"。同时配合 `CACHED_MICROCOMPACT` 功能，系统可以在不 fork 代理的情况下，定期清除旧的 function results，减少长会话中工具结果的无意义堆叠。

---

## 7. 关键技术

| 技术点 | 文件位置 | 作用 |
|--------|----------|------|
| `memoize` 缓存上下文 | `src/context.ts` | 避免每轮重复计算 git status 和 Claude.md |
| `deriveUUID` 确定性 UUID | `src/utils/messages.ts:725` | 拆分多内容块消息时保持 ID 稳定 |
| `normalizeMessagesForAPI` | `src/utils/messages.ts:1989` | API 前最终消息清洗与适配 |
| `ensureToolResultPairing` | `src/utils/messages.ts` | 修复 tool_use/tool_result 配对断裂 |
| `splitSysPromptPrefix` | `src/utils/api.ts:321` | 系统提示词分块与缓存策略 |
| `buildSystemPromptBlocks` | `src/services/api/claude.ts` | 将系统提示词数组转为带 cache_control 的 API blocks |
| `toolToAPISchema` | `src/utils/api.ts:119` | 工具定义转 API Schema，带 session-stable 缓存 |
| `compactConversation` | `src/services/compact/compact.ts` | forkedAgent 摘要生成与消息替换 |
| `getAutoCompactThreshold` | `src/services/compact/autoCompact.ts` | Token 阈值计算与缓冲策略 |
| `NO_TOOLS_PREAMBLE` | `src/services/compact/prompt.ts` | 压缩 agent 的严格无工具前置指令 |
| `formatCompactSummary` | `src/services/compact/prompt.ts:311` | 剥离 analysis 草稿，规范化 summary |
| `getExtraBodyParams` | `src/services/api/claude.ts` | 注入 betas、anti_distillation、额外参数 |

---

## 8. 思考总结

Claude Code 的 Prompt System 不是简单的"字符串拼接"模块，而是一个高度成熟、经过大量 A/B 测试和工程打磨的企业级 LLM 交互中间件。它的设计哲学体现在以下几点：

1. **防御性设计（Defense in Depth）**：从消息拆分、合并、配对修复、到图片超限的容错处理，每一层都在假设"上游可能出错"，并通过多 Pass 清洗确保发给 API 的数据一定合法。

2. **缓存即架构（Caching as Architecture）**：Prompt Caching 不再是一个简单的性能优化，而是深刻影响了系统提示词的结构设计（边界标记、分块策略）、工具 schema 的缓存稳定性（`toolSchemaCache`），甚至是消息的正常化顺序。

3. **可观测性与内部调试**：从 `logEvent('tengu_sysprompt_block')` 到 `logContextMetrics()`，再到 `CACHE_BREAKER` 和详尽的 checkpoint 埋点，系统为内部工程师提供了丰富的诊断能力。

4. **长会话生命力的保障**：Auto-Compact 及其配套的 Post-Compact Cleanup，让理论上"无限长"的对话成为可能。这种设计不是简单截断，而是通过语义摘要 + 关键文件恢复的方式，尽量保留对话的连贯性和可操作性。

5. **Feature Flag 驱动的渐进 rollout**：几乎每一个新特性（tool search、advisor、anti-distillation、FGTS、1h cache TTL）都通过 GrowthBook feature gate 控制，并在代码中留有清晰的条件分支，这让产品可以在小流量验证后再全量放开。

如果要指出潜在的改进空间，那么 `normalizeMessagesForAPI()` 中多 Pass 顺序的脆弱性（代码注释中已明确提到：`These multi-pass normalizations are inherently fragile`）是一个长期的维护挑战。未来可以考虑将其重构为一个统一的单 Pass 数据流，即先清洗内容，再一次性验证，以降低 Pass 间状态耦合带来的 bug 风险。

总体而言，Claude Code 的 Prompt System 是当前 LLM 应用工程化领域的一个优秀范本，它充分展示了如何将提示工程、缓存策略、错误恢复和长上下文管理融为一体。
