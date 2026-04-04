# Token Management & Cost Tracking 模块技术解析

## 1. 模块总结介绍

**Token Management & Cost Tracking** 是 Claude Code 核心基础设施之一，负责在对话全生命周期内：
- **度量（Measure）**：精确或近似地统计上下文窗口中的 token 数量；
- **计费（Charge）**：记录每次 API 调用的 token 消耗并折算为 USD；
- **压缩（Compact）**：在上下文接近模型上限时，通过多层次压缩策略主动释放空间；
- **预警（Warn）**：在 token 使用触及不同阈值时向用户或系统发出状态信号；
- **优化（Optimize）**：利用工具摘要生成、微压缩、缓存编辑等手段降低无效 token 开销。

该模块贯穿 `bootstrap/state`（全局状态）、`utils/tokens.ts`（token 计算）、`cost-tracker.ts`（成本追踪）、`services/compact/`（压缩服务）以及 `toolUseSummaryGenerator.ts`（工具摘要生成）。

---

## 2. 系统架构

模块整体呈**分层 + 管道**架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    Policy Layer（策略层）                    │
│   autoCompact.ts / context.ts      阈值判定、窗口配置、策略开关   │
├─────────────────────────────────────────────────────────────┤
│                  Compaction Layer（压缩层）                  │
│   compact.ts / autoCompact.ts / sessionMemoryCompact.ts      │
│   microCompact.ts / apiMicrocompact.ts                       │
├─────────────────────────────────────────────────────────────┤
│                  Tracking Layer（追踪层）                    │
│   cost-tracker.ts / costHook.ts / bootstrap/state.ts         │
├─────────────────────────────────────────────────────────────┤
│                  Counting Layer（计数层）                    │
│   utils/tokens.ts / services/tokenEstimation.ts              │
└─────────────────────────────────────────────────────────────┘
```

- **Counting Layer**：提供**精确 API 回执统计**与**基于字符数的启发式估算**两种能力；
- **Tracking Layer**：维护会话粒度的累计 USD、token 数、按模型 breakdown，并通过 OpenTelemetry Counter 上报；
- **Compaction Layer**：实现从“微压缩”到“对话级压缩”再到“会话记忆压缩”的多级减压策略；
- **Policy Layer**：定义上下文窗口大小、自动压缩阈值、警告/错误/阻塞水位线，并与 GrowthBook 远程配置联动。

---

## 3. 分层说明

### 3.1 Counting Layer — token 计数

#### 3.1.1 API 回执统计（精确值）
`utils/tokens.ts` 中的 `getTokenUsage(message)` 从 assistant message 的 `usage` 字段提取 Anthropic API 返回的真实 token 数。关键函数：
- `tokenCountFromLastAPIResponse(messages)`：返回最近一次 API 调用的总 token（input + output + cache_creation + cache_read）。
- `finalContextTokensFromLastResponse(messages)`：使用 `usage.iterations`（若存在）计算服务器侧上下文窗口，用于 `task_budget.remaining` 对齐。
- `getCurrentUsage(messages)`：返回最近一次调用的分项 token 数。

> 代码引用（`restored-src/src/utils/tokens.ts:46-52`）：
> ```typescript
> export function getTokenCountFromUsage(usage: Usage): number {
>   return (
>     usage.input_tokens +
>     (usage.cache_creation_input_tokens ?? 0) +
>     (usage.cache_read_input_tokens ?? 0) +
>     usage.output_tokens
>   )
> }
> ```

#### 3.1.2 启发式估算（fallback）
当新增消息尚未获得 API 回执时，`tokenCountWithEstimation(messages)` 以**最后一次有 usage 的 message 为锚点**，对其后新增的消息调用 `roughTokenCountEstimationForMessages` 进行估算。估算规则：
- 普通文本：`content.length / 4`（默认 bytesPerToken=4）；
- JSON/JSONL：`content.length / 2`（单字符 token 更密集）；
- 图片/文档：固定 `2000` tokens；
- thinking/redacted_thinking：按文本长度估算；
- tool_use：按 `name + JSON.stringify(input)` 长度估算；
- tool_result：递归估算其 content。

> 代码引用（`restored-src/src/utils/tokens.ts:226-261`）：
> `tokenCountWithEstimation` 通过向前回溯同 `message.id` 的拆分记录，避免漏算并行工具调用之间插着的 `tool_result`。

#### 3.1.3 第三方 API token 计数
`services/tokenEstimation.ts` 提供：
- `countMessagesTokensWithAPI`：直接调用 Anthropic `countTokens` beta API；
- `countTokensViaHaikuFallback`：在 Vertex/Bedrock 等环境使用轻量模型（Haiku 或 Sonnet）做 create 请求并以 usage 反推；
- `countTokensWithBedrock`：通过 AWS `CountTokensCommand` 实现 Bedrock 原生计数。

### 3.2 Tracking Layer — 成本追踪

#### 3.2.1 全局状态
`bootstrap/state.ts` 维护：
- `totalCostUSD`：会话累计美元成本；
- `modelUsage`：按模型的 input/output/cache/read/web_search 细分；
- `costCounter`、`tokenCounter`：OpenTelemetry AttributedCounter，用于遥测上报。

> 代码引用（`restored-src/src/bootstrap/state.ts:557-567`）：
> ```typescript
> export function addToTotalCostState(
>   cost: number,
>   modelUsage: ModelUsage,
>   model: string,
> ): void {
>   STATE.modelUsage[model] = modelUsage
>   STATE.totalCostUSD += cost
> }
> ```

#### 3.2.2 成本计算
`utils/modelCost.ts` 定义了各模型的 per-million-token 价格表 `MODEL_COSTS`，并支持：
- 按模型 canonical name 查表；
- 未知模型 fallback 到默认模型价格并标记 `hasUnknownModelCost`；
- Opus 4.6 fast mode 动态 tier（`COST_TIER_30_150`）。

函数 `calculateUSDCost(resolvedModel, usage)` 将 `input_tokens`、`output_tokens`、`cache_read_input_tokens`、`cache_creation_input_tokens`、`web_search_requests` 统一折现。

#### 3.2.3 会话级持久化
`cost-tracker.ts` 实现：
- `saveCurrentSessionCosts(fpsMetrics?)`：在进程退出前把成本、时长、行数变更写入项目级配置（`projectConfig`）；
- `restoreCostStateForSession(sessionId)`：恢复同一会话的成本状态，支持 `--resume` 续聊时成本不重置。

> 代码引用（`restored-src/src/cost-tracker.ts:143-175`）：
> ```typescript
> export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
>   saveCurrentProjectConfig(current => ({
>     ...current,
>     lastCost: getTotalCostUSD(),
>     lastAPIDuration: getTotalAPIDuration(),
>     ...
>   }))
> }
> ```

#### 3.2.4 React Hook 绑定
`costHook.ts` 的 `useCostSummary` 在组件 mount 时监听 `process.on('exit')`，确保终端在退出时打印本次会话的总成本与 token 汇总（仅在 `hasConsoleBillingAccess()` 为 true 时输出）。

### 3.3 Compaction Layer — 压缩策略

压缩层是 Token Management 的核心执行器，按粒度从粗到细分为四级：

#### 3.3.1 对话级压缩（compact.ts）
`compactConversation()` 通过 forked agent 或 streaming 调用模型本身，对历史消息生成自然语言摘要，然后将旧消息替换为：
- 一条 `SystemCompactBoundaryMessage`（标记压缩边界）
- 一条/多条 `UserMessage` 摘要文本
- 可选的附件（最近阅读的文件、计划、skill、异步 agent 状态等）
- SessionStart hooks 结果

关键特性：
- **Prompt Cache Sharing**：默认开启，forked agent 复用主会话的 system/tools 前缀缓存，降低压缩成本。
- **PTL Retry**：若压缩请求本身触发 `prompt_too_long`，会自动 `truncateHeadForPTLRetry` 丢弃最旧 API round 后重试。
- **Partial Compact**：支持以某条消息为 pivot，保留前面或保留后面。

> 代码引用（`restored-src/src/services/compact/compact.ts:470-480`）：
> 使用 `runForkedAgent` 共享 prompt cache，且**不设置 `maxOutputTokens`**，避免 thinking config 不一致导致缓存失效。

#### 3.3.2 自动压缩（autoCompact.ts）
`autoCompactIfNeeded()` 是主循环中的守门人：
1. 先检查 `shouldAutoCompact()`：
   - 排除 `querySource === 'session_memory' | 'compact' | 'marble_origami'`（防止递归死锁）；
   - 检查 `DISABLE_COMPACT` / `DISABLE_AUTO_COMPACT` / 用户配置开关；
   - 检查 `REACTIVE_COMPACT` 与 `CONTEXT_COLLAPSE` 实验是否接管。
2. 若阈值触发，先尝试 `trySessionMemoryCompaction()`（轻量、无 API 调用）；失败再回退到 `compactConversation()`（重量级、有 API 调用）。
3. **Circuit Breaker**：连续失败 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`（3 次）后永久跳过自动压缩，避免对不可恢复超限上下文反复浪费 API 调用。

阈值计算（`restored-src/src/services/compact/autoCompact.ts:72-91`）：
```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 13_000
}
```

`getEffectiveContextWindowSize` 会在模型 context window 基础上再预留 `MAX_OUTPUT_TOKENS_FOR_SUMMARY`（20_000）作为输出空间。

#### 3.3.3 会话记忆压缩（sessionMemoryCompact.ts）
**无 API 调用的轻量压缩**。前提：
- GrowthBook 开启 `tengu_session_memory` + `tengu_sm_compact`；
- 磁盘上存在已提取的 session memory；
- 通过 `lastSummarizedMessageId` 知道哪些消息已被纳入记忆。

逻辑：
1. 以 `lastSummarizedMessageId` 为切分点，保留其后消息；
2. 向后扩展以满足 `minTokens`（10_000）与 `minTextBlockMessages`（5）的保底；
3. 通过 `adjustIndexToPreserveAPIInvariants()` 确保不切断 `tool_use/tool_result` 对、不丢失同 `message.id` 的 thinking block；
4. 用现有 session memory 文本作为摘要，生成 `CompactionResult`。

#### 3.3.4 微压缩（microCompact.ts）
在每次 API 请求前执行，属于**请求级**压缩，不改变对话语义结构，只减少工具结果内容：
- **Time-based MC**：若距上次 assistant 消息超过阈值（默认由 GrowthBook 配置），将旧 tool result 内容替换为 `'[Old tool result content cleared]'`。
- **Cached MC（cache editing）**：对 `main thread` 对话，通过 API 层的 `cache_edits` 机制删除指定 tool result 的缓存条目，而**不修改本地消息内容**。下轮请求时由 API 层注入 `cache_reference` 与 `cache_edits`。
- 被清除的工具限于 `COMPACTABLE_TOOLS`：FileRead、Shell、Glob、Grep、WebFetch、WebSearch、FileEdit、FileWrite。

> 代码引用（`restored-src/src/services/compact/microCompact.ts:27-40`）：
> ```typescript
> const COMPACTABLE_TOOLS = new Set<string>([
>   FILE_READ_TOOL_NAME,
>   ...SHELL_TOOL_NAMES,
>   GREP_TOOL_NAME,
>   GLOB_TOOL_NAME,
>   WEB_SEARCH_TOOL_NAME,
>   WEB_FETCH_TOOL_NAME,
>   FILE_EDIT_TOOL_NAME,
>   FILE_WRITE_TOOL_NAME,
> ])
> ```

#### 3.3.5 API 原生上下文管理（apiMicrocompact.ts）
面向 Anthropic API 的 **server-side context management**：在请求体中注入 `context_management_config`，声明：
- `clear_tool_uses_20250919`：在 input_tokens 触发阈值后清除旧 tool use/result；
- `clear_thinking_20251015`：保留或清除历史 assistant thinking block。

该策略仅对 ant 内部用户生效，由环境变量 `USE_API_CLEAR_TOOL_RESULTS` / `USE_API_CLEAR_TOOL_USES` 控制。

### 3.4 Policy Layer — 策略与阈值

`utils/context.ts` 统一负责：
- **Context Window 解析**：默认 200K，支持 `[1m]` 后缀或 beta header 升至 1M；支持 `CLAUDE_CODE_DISABLE_1M_CONTEXT` 关闭；支持 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 覆盖。
- **Max Output Tokens**：按模型返回 default / upperLimit（如 Opus 4.6 为 64K/128K，Sonnet 4.6 为 32K/128K）。
- **Token 使用百分比**：`calculateContextPercentages(currentUsage, contextWindowSize)` 计算已用百分比。

`autoCompact.ts` 在此基础上叠加策略：
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`
- `ERROR_THRESHOLD_BUFFER_TOKENS = 20_000`
- `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`
- `isAtBlockingLimit`：当自动压缩关闭时，若达到 `effectiveWindow - 3_000` 则阻塞后续请求并返回 `PROMPT_TOO_LONG_ERROR_MESSAGE`。

---

## 4. 交互流程

以一次典型主循环查询为例，Token / Cost 的完整生命周期如下：

```
1. 用户输入 / 工具返回
        ↓
2. applyToolResultBudget()    ← 对超大 tool result 做内容替换
        ↓
3. snipCompactIfNeeded()      ← History snip：丢弃极旧消息
        ↓
4. microcompactMessages()     ← Time-based / Cached 微压缩
        ↓
5. shouldAutoCompact()        ← tokenCountWithEstimation() 估算当前上下文
        ↓
   ├─ 触发阈值 ─→ trySessionMemoryCompaction() ─→ 成功则替换消息
   │                           └─ 失败 ─→ compactConversation() ─→ 模型生成摘要
   └─ 未触发 ─→ 继续
        ↓
6. calculateTokenWarningState() ← 检查阻塞限制（若 auto-compact 关闭）
        ↓
7. 发送 API 请求
        ↓
8. 收到 streaming response + usage
        ↓
9. addToTotalSessionCost()    ← cost-tracker.ts 更新全局 cost/token 计数
        ↓
10. OpenTelemetry Counter 上报 + 本地持久化（exit 时）
```

**关键衔接点**：
- `query.ts` 在调用模型前依次执行 `snip` → `microcompact` → `autoCompact` → `blockingLimit` 检查（`restored-src/src/query.ts:367-660`）。
- 若 Reactive Compact 开启且接收到 API 的 413 / prompt-too-long，则进入 `tryReactiveCompact()` 进行**事后压缩**（fallback 路径）。
- 每次成功压缩后调用 `notifyCompaction()` 与 `markPostCompaction()`，防止 Prompt Cache Break Detection 将合法的 cache read 下降误判为缓存断裂。

---

## 5. 技术原理

### 5.1 精确 + 估算的混合计数

由于对话消息在不断增长，**不能简单叠加所有 message 的 token 数**（会导致重复计算 growing context）。Claude Code 的 canonical 做法是：
1. 找到**最近一次 API 回执**（assistant message 含 `usage`）；
2. 以此为基线，加上之后新增消息的启发式估算；
3. 若存在并行工具调用导致同一 API response 被拆成多条 assistant record，向前回溯到第一条同 `message.id` 的记录，确保中间插着的 `tool_result` 都被纳入估算。

### 5.2 多级压缩的降级漏斗

压缩策略按“开销从低到高”排列成漏斗：
- **Level 0**：`apiMicrocompact`（server-side，零客户端开销）；
- **Level 1**：`microcompact`（本地内容替换或 cache editing，无模型调用）；
- **Level 2**：`sessionMemoryCompact`（磁盘记忆复用，无模型调用）；
- **Level 3**：`compactConversation`（模型调用生成摘要，有成本）；
- **Level 4**：`reactiveCompact`（API 返回 413 后紧急截断，有损）。

只有当轻量手段无法将上下文压到阈值以下时，才会触发更重的手段。

### 5.3 Prompt Cache 感知

压缩层大量围绕 **prompt caching** 做优化：
- Forked compact agent 复用主线程缓存前缀（`promptCacheSharingEnabled`），使压缩请求的 `cache_read` 比例极高；
- Cached microcompact 使用 `cache_edits` 删除缓存中的 tool result，而不是重新发送整段对话，避免 cache bust；
- 每次压缩后重置 cache read baseline（`notifyCompaction`），让后续的 cache break 监测只检测真正的意外失效。

### 5.4 成本追踪的模型归一化

`cost-tracker.ts` 在 `formatModelUsage()` 中将不同模型名（如 `claude-sonnet-4-6-20251022` 与带 `[1m]` 后缀的版本）按 `getCanonicalName()` 聚合成短名，避免同一模型多个变体导致统计分散。

---

## 6. 创新点

1. **tokenCountWithEstimation 的并行工具调用回溯机制**：解决 streaming 拆分 assistant message 后估算漏算 tool_result 的问题，保证阈值判断的准确性。
2. **Session Memory 零调用压缩**：利用持久化的 session memory 文件替代模型摘要生成，在命中条件时实现亚秒级压缩且零 API 成本。
3. **Circuit Breaker 防止自动压缩死循环**：连续 3 次失败后彻底关闭自动压缩，避免因不可恢复的超限上下文每天浪费数十万 API 调用。
4. **Prompt Cache Sharing 的 Forked Agent 压缩**：通过不设置 `maxOutputTokens`、保持 thinking config 一致，让压缩请求几乎完全命中主会话的 prompt cache，极大降低压缩本身的成本。
5. **Cached Microcompact 的 cache_edits 架构**：本地消息不变，仅通过 API 层注入 `cache_edits` 删除缓存条目，实现“逻辑压缩”与“物理消息保留”的分离。
6. **多重缓冲阈值体系**：同一 effective context window 上叠加 `autocompact(−13K)`、`warning(−20K)`、`error(−20K)`、`blocking(−3K)` 四级水位，形成渐进式干预而非一刀切。

---

## 7. 关键技术

| 技术点 | 作用 | 对应文件 |
|--------|------|----------|
| `tokenCountWithEstimation` | 混合精确+估算的上下文大小 canonical 函数 | `utils/tokens.ts` |
| `getEffectiveContextWindowSize` | 动态计算有效窗口（模型上限 − 摘要输出预留 − 环境覆盖） | `services/compact/autoCompact.ts` |
| `calculateTokenWarningState` | 四级阈值状态机 | `services/compact/autoCompact.ts` |
| `autoCompactIfNeeded` | 自动压缩守门人 + Circuit Breaker | `services/compact/autoCompact.ts` |
| `compactConversation` | 对话级模型摘要压缩，支持 PTL retry 与 partial compact | `services/compact/compact.ts` |
| `trySessionMemoryCompaction` | 会话记忆轻量压缩 | `services/compact/sessionMemoryCompact.ts` |
| `microcompactMessages` | 请求级微压缩（time-based / cached） | `services/compact/microCompact.ts` |
| `getAPIContextManagement` | Server-side context management 配置生成 | `services/compact/apiMicrocompact.ts` |
| `calculateUSDCost` / `MODEL_COSTS` | 模型级成本计算与查表 | `utils/modelCost.ts` |
| `addToTotalSessionCost` | 成本与 token 的聚合、OTel 上报 | `cost-tracker.ts` |
| `generateToolUseSummary` | Haiku 生成工具批次摘要（~30 字符），降低后续压缩压力 | `services/toolUseSummary/toolUseSummaryGenerator.ts` |

---

## 8. 思考总结

Token Management & Cost Tracking 模块体现了 Claude Code 在**大规模生产环境**中运行 LLM CLI 的系统性工程思维：

- **准确性优先**：不盲目累加 token，而是通过 API usage 回执锚定 + 向后估算的方式，确保阈值触发的时机准确；
- **成本敏感**：从免费的 apiMicrocompact / microcompact，到零 API 调用的 session memory，再到共享 prompt cache 的 forked compact，层层降级，绝不轻易调用高成本模型做摘要；
- **容错与边界**：Circuit Breaker、PTL retry、blocking limit 等多重保护，让模块在极端超长对话下仍能优雅降级而不是崩溃或无限循环；
- **可观测性**：每一处压缩、每一次阈值跨越、每一笔 cost 都通过 `logEvent` 与 OpenTelemetry Counter 输出，支撑后续的数据驱动优化；
- **生态兼容**：同时适配 Anthropic 1P API、Vertex、Bedrock 三种后端，token 计数与压缩策略都做了相应 fallback，体现了跨平台工程意识。

该模块的复杂度恰恰来自于 LLM 应用的核心矛盾：**模型需要尽可能多的上下文以保持一致性，而上下文越长成本越高、延迟越大、最终还会触碰硬上限**。Claude Code 通过“精确度量 + 多级压缩 + 缓存优化 + 成本追踪”四位一体的方案，在这一矛盾中取得了工程上的平衡。
