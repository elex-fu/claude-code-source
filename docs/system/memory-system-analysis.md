# Claude Code 记忆系统（Memory System）技术架构分析

## 1. 模块总结介绍

Claude Code 的 Memory System 是一套**多层、多形态的持久化记忆架构**，目标是在单次会话内以及跨会话之间维持上下文连续性。它解决了大模型因为上下文窗口限制而“遗忘”的问题，分为两大类：

- **长期记忆（Persistent Memory）**：以 `CLAUDE.md` 族文件（User/Project/Local/Managed）和自动记忆目录（`~/.claude/projects/<project>/memory/`，即 AutoMem）为载体，跨会话持久化。
- **会话记忆（Session Memory）**：以 `session-memory.md` 为载体，在单会话内随着对话推进自动更新，用于上下文压缩（autocompact）后的快速回溯。

系统在用户对话结束后，通过**后台 forked subagent** 自动提取关键信息写入记忆文件；在下次对话开始时，通过系统提示词（system prompt）或 `<system-reminder>` attachment 将相关记忆注入上下文。

---

## 2. 系统架构

记忆系统的整体架构可以用“**提取—存储—召回—注入**”四阶段模型概括：

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Extraction     │ ──► │    Storage       │ ──► │    Recall       │ ──► │   Injection     │
│  (后台子代理)    │     │  (文件系统层)     │     │  (相关度选择器)  │     │  (Attachments)  │
└─────────────────┘     └──────────────────┘     └─────────────────┘     └─────────────────┘
```

### 核心模块分布

| 目录/文件 | 职责 |
|----------|------|
| `services/SessionMemory/` | 会话记忆的触发、提取、模板渲染与阈值控制 |
| `services/extractMemories/` | 长期记忆的自动提取（forked agent）与工具权限限制 |
| `memdir/` | 自动记忆目录的扫描、相关度排序、路径解析 |
| `utils/attachments.ts` | 记忆的 attachment 注入、相关记忆预取（prefetch）、去重过滤 |
| `utils/claudemd.ts` | `CLAUDE.md` 族文件的扫描、层级加载、条件规则匹配 |
| `components/memory/` | `/memory` 命令的 UI（文件选择器、更新通知） |
| `commands/memory/` | `/memory` 命令的交互实现 |
| `history.ts` | 用户输入历史（prompt history）的持久化与回显 |

---

## 3. 分层说明

### 3.1 持久化层（Persistence Layer）

所有记忆最终落地于**本地文件系统**，按信任层级和作用域分为多个目录/文件：

- **`~/.claude/CLAUDE.md`**：User 记忆，全局生效，仅当前用户可见。
- **`./CLAUDE.md`** / **`./CLAUDE.local.md`**：Project / Local 记忆，按项目作用域生效；`.local` 版本不纳入版本控制。
- **`~/.claude/rules/*.md`** / **`.claude/rules/*.md`**：Managed / User / Project 的条件规则（conditional rules），可按 glob 匹配目标文件。
- **`~/.claude/projects/<sanitized-root>/memory/`**：AutoMem 自动记忆目录，包含 `MEMORY.md` 索引和按主题拆分的 `.md` 文件。
- **`<session-memory-path>`**：会话级 `session-memory.md`，位于 Claude 配置目录下。

文件访问权限通过 `filesystem.ts` 中的 `isAutoMemPath()` 做沙箱校验，确保子代理只能写入获批目录。

### 3.2 服务层（Service Layer）

#### SessionMemory 服务

`sessionMemory.ts` 注册了 `postSamplingHook`，在每次模型采样完成后异步评估是否触发提取：

- **初始化阈值**：`minimumMessageTokensToInit = 10000`（默认），会话消息 token 数达到该值后才首次启动。
- **更新阈值**：`minimumTokensBetweenUpdate = 5000`（默认），自上次提取以来新增 token 数。
- **工具调用阈值**：`toolCallsBetweenUpdates = 3`（默认）， or 最后一轮 Assistant 没有工具调用时也可以触发。

`sessionMemoryUtils.ts` 管理这些阈值的状态机，并提供 `waitForSessionMemoryExtraction()` 等待提取完成（用于上下文压缩前的同步）。

#### extractMemories 服务

`extractMemories.ts` 在**完整 query loop 结束**（模型给出最终无 tool-call 的回复）时，通过 `handleStopHooks` 触发。关键设计：

- 使用 `runForkedAgent` 创建一个与主对话** prompt-cache 共享**的完美 fork；隔离父级状态，避免污染主线程。
- 通过 `canUseTool` 限制子代理只能对 memory 目录执行 `FILE_READ` / `GREP` / `GLOB` / 只读 `BASH` / `FILE_EDIT|WRITE`。
- 若主代理自己已经在当前区间内写过记忆文件（`hasMemoryWritesSince`），则跳过后台提取，避免重复劳动。
- 支持 **coalescing + trailing run**：如果新请求在提取进行中到达，stash 最新上下文，当前提取结束后再跑一轮增量提取。
- `turnsSinceLastExtraction` 实现提取频率节流（Feature Gate `tengu_bramble_lintel`）。

### 3.3 UI 层（Presentation Layer）

- **`MemoryFileSelector.tsx`**：`/memory` 命令弹出的交互式文件选择器（Ink/React），列出 User/Project/Local/AutoMem/Agent memory 等入口，支持打开文件夹、开关 Auto-memory / Auto-dream。
- **`MemoryUpdateNotification.tsx`**：当记忆被更新时，在终端底部显示 `Memory updated in <path> · /memory to edit`。

---

## 4. 交互流程

### 4.1 会话记忆（Session Memory）生命周期

```text
用户对话持续进行中
          │
          ▼
    每一轮采样结束
          │
    executePostSamplingHooks()
          │
    extractSessionMemory()
          │
    ┌─────┴─────┐
    ▼           ▼
 未达阈值    阈值满足
  (跳过)      │
              ▼
      setupSessionMemoryFile()
              │
      ┌───────┴───────┐
      ▼               ▼
   文件不存在       读取现有内容
   加载模板         (FileReadTool)
      │               │
      └───────┬───────┘
              ▼
      buildSessionMemoryUpdatePrompt()
              │
              ▼
      runForkedAgent({
        canUseTool: createMemoryFileCanUseTool(memoryPath)
      })
              │
              ▼
      子代理用 Edit 工具更新 session-memory.md
              │
              ▼
      markExtractionCompleted()
```

### 4.2 长期记忆（AutoMem）生命周期

```text
完整 query loop 结束（最终回复，无 tool calls）
          │
    handleStopHooks
          │
    executeExtractMemories()
          │
    ┌─────┴─────┐
    ▼           ▼
   Gate 关闭   Gate 开启
   (跳过)      │
               ▼
      hasMemoryWritesSince?
               │
         是 ──► 跳过并推进 cursor
               │
               否
               ▼
      scanMemoryFiles() ──► formatMemoryManifest()
               │
               ▼
      buildExtractAutoOnlyPrompt() / buildExtractCombinedPrompt()
               │
               ▼
      runForkedAgent({ maxTurns: 5, skipTranscript: true })
               │
               ▼
      子代理读取需更新的记忆文件
               │
               ▼
      子代理并行 Edit / Write 更新
               │
               ▼
      推送系统消息 "memory_saved"（可选 appendSystemMessage）
               │
               ▼
      更新 lastMemoryMessageUuid（cursor 推进）
```

### 4.3 记忆召回与注入生命周期

```text
新一轮用户输入到达
          │
    startRelevantMemoryPrefetch()
          │
    ┌─────┴─────┐
    ▼           ▼
   单字输入    正常输入
   (跳过)      │
               ▼
      collectSurfacedMemories()  // 去重 + 会话总字节节流
               │
               ▼
      getRelevantMemoryAttachments()
               │
      ┌────────┴────────┐
      ▼                 ▼
  提取 agent mentions  否则
       │               默认 getAutoMemPath()
       ▼               ▼
  getAgentMemoryDir() findRelevantMemories()
       │               │
       └───────┬───────┘
               ▼
      selectRelevantMemories()  // sideQuery (Sonnet JSON schema)
               │
               ▼
      readMemoriesForSurfacing()  // MAX_MEMORY_LINES=200, MAX_MEMORY_BYTES=4096
               │
               ▼
      生成 { type: 'relevant_memories', memories: [...] } attachment
               │
               ▼
      后续被 filterDuplicateMemoryAttachments() 去重
               │  （排除 readFileState 已包含的文件）
               ▼
      以 <system-reminder> 形式注入下一轮系统提示
```

---

## 5. 技术原理

### 5.1 Forked Agent 与 Prompt Cache 共享

后台提取子代理并非重新构造完整 prompt，而是使用 `createCacheSafeParams(context)` 复用主线程的 system prompt、user context、tool context。`runForkedAgent` 的 `forkLabel` 被设置为 `session_memory` 或 `extract_memories`，保证：

- 子代理能享受父级已预热好的 prompt cache（`cache_read_input_tokens` 命中率高）。
- 子代理通过 `overrides: { readFileState: setupContext.readFileState }` 拥有独立的文件读取缓存，不会把对 memory 文件的 `FileReadTool` 调用写入父缓存，从而避免污染主线程的变更检测。

### 5.2 记忆提取的互斥与去重

`extractMemories.ts` 中的 `hasMemoryWritesSince()` 扫描自 `lastMemoryMessageUuid` 以来的所有 Assistant 消息，若发现 `FILE_EDIT_TOOL_NAME` / `FILE_WRITE_TOOL_NAME` 的 target path 落在 `isAutoMemPath()` 内，则判定主代理已经手动维护过记忆，后台代理**直接跳过**该区间。这使得“主代理主动写记忆”与“后台自动补漏”互为补充、互不冲突。

### 5.3 会话记忆的阈值与 Token 计数

`sessionMemoryUtils.ts` 中所有 token 计数统一使用 `tokenCountWithEstimation(messages)`，该方法与 autocompact 保持一致：

```ts
export function hasMetInitializationThreshold(currentTokenCount: number): boolean {
  return currentTokenCount >= sessionMemoryConfig.minimumMessageTokensToInit
}

export function hasMetUpdateThreshold(currentTokenCount: number): boolean {
  const tokensSinceLastExtraction = currentTokenCount - tokensAtLastExtraction
  return tokensSinceLastExtraction >= sessionMemoryConfig.minimumTokensBetweenUpdate
}
```

这保证了 session memory 的提取频率与上下文压缩节奏对齐，避免过度 I/O。

### 5.4 相关性选择器（Relevance Selector）

`findRelevantMemories.ts` 使用一个轻量的 **Sonnet sideQuery** 做二阶筛选：

- 输入：用户的最新 query + 可用记忆文件的 manifest（文件名 + frontmatter description + 修改时间）。
- 输出：JSON schema 约束的 `selected_memories: string[]`，最多 5 个文件。
- 系统提示中明确要求“如果最近成功使用的工具在列表中有文档类记忆，则抑制该文档，但保留其警告/Gotcha 类记忆”。

sideQuery 的 latencies 被 `startRelevantMemoryPrefetch()` **预取（prefetch）**隐藏：在用户输入处理、工具执行的同时，后台并行跑相关性选择；等到真正需要 attachments 时，若已 settle 则直接 consume，否则 skip 等下一轮。

### 5.5 记忆的陈旧度（Staleness）处理

`memoryAge.ts` 为每份被召回的记忆计算 `memoryAgeDays(mtimeMs)`，并在 `memoryFreshnessText()` 中生成如下提示：

> "This memory is N days old. Memories are point-in-time observations, not live state — claims about code behavior or file:line citations may be outdated."

这段文本在 `attachments.ts` 的 `readMemoriesForSurfacing()` 里以固定字符串形式**在 attachment 创建时就写入 header**，而不是在渲染时动态计算。这样做是为了保证 `<system-reminder>` 字节流在不同回合间完全一致，从而不破坏 prompt cache。

### 5.6 历史系统（history.ts）

`history.ts` 管理用户**prompt history**（与 memory system 互补），特色设计：

- **双队列异步刷盘**：`pendingEntries`（内存缓冲）+ `history.jsonl`（持久化），通过 `flushPromptHistory()` 异步写入。
- **锁文件并发控制**：使用 `lockfile` 对 `history.jsonl` 加锁，防止多进程并发写冲突。
- **去重与 lazy resolve**：`getTimestampedHistory()` 按 display text 去重，`getHistory()` 当前会话优先于其他会话。
- **撤销机制**：`removeLastFromHistory()` 在 Esc 回滚时移除最后一条历史，防止用户输入框回显时出现重复。

### 5.7 `CLAUDE.md` 的层级与条件加载

`claudemd.ts` 中的 `getMemoryFiles()` 是一个 `memoize` 包装的大型加载器：

1. **Managed**（政策级）→ **User** → 从根目录向下到 CWD 的 **Project** / **Local**。
2. 支持**嵌套 include**：`CLAUDE.md` 可通过 `@include path/to/file.md` 引用外部文件。
3. **条件规则**：`.claude/rules/*.md` 文件可配置 `globs: [...]`，仅当匹配当前操作的目标路径时才被注入上下文。
4. **Git worktree 去重**：若检测到嵌套 worktree，会跳过主 repo 根目录以上已加载的 Project 文件，避免内容重复。

---

## 6. 创新点

### 6.1 主代理与后台代理的“写互斥”

系统提示词始终教导主代理“你可以写记忆”，但后台提取代理只在主代理**没有写**的区间运行。这种设计巧妙地把“即时响应”和“后台补漏”结合起来，避免了双重写入，也保证了用户让模型“记住某事”时能立刻生效。

### 6.2 Prompt Cache 友好的稳定字节流

`memoryHeader()` 把 `staleness` 提示在 attachment 创建时就固化进内容，避免了渲染阶段 `Date.now()` 差异导致的 cache miss。类似的，`collectSurfacedMemories()` 扫描 messages 来计算 already surfaced，而不是维护一个随时间变化的全局状态。

### 6.3 Relevant Memory Prefetch + Disposable

`MemoryPrefetch` 实现了 `[Symbol.dispose]`，并与 query loop 的 `using` 绑定。这使得：
- 预取任务的取消和埋点自动化（~13 个 return path 无需逐个 instrument）。
- telemetry 能精确报告“首轮被隐藏” vs “第 N 轮被消费”的 latency。

### 6.4 会话记忆与会话压缩的协同

Session Memory 的产物 `session-memory.md` 本身在 autocompact 时会被**注入到 compact 后的第一条系统消息**中，作为压缩后上下文的“摘要锚点”。这意味着：即使 10 万 token 的对话被压缩到 1 万 token，模型仍能通过 session memory 快速恢复对当前任务状态、已犯错误、待办事项的理解。

### 6.5 AutoMem 的 frontmatter 类型学

`memoryTypes.ts` 定义了四类记忆（user / feedback / project / reference），并为每类规定了 `<scope>`、`<when_to_save>`、`<body_structure>`。这不仅是提示工程，更是**数据结构约束**，使得 sideQuery 在选择时只需要看 description，后续蒸馏（/dream）也能按类型做结构化合并。

---

## 7. 关键技术

- **Forked Subagent (`runForkedAgent`)**：在隔离上下文中运行轻量级任务，共享 prompt cache。
- **PostSamplingHook / StopHook**：分别用于会话记忆（每轮采样后）和长期记忆（完整 query 结束后）的异步触发。
- **SideQuery with JSON Schema**：用小模型快速做相关度排序，输出结构化文件名列表。
- **`memoize` + `lockfile`**：`getMemoryFiles()` 避免重复 I/O；`history.jsonl` 避免并发写冲突。
- **`readFileInRange` + `truncateOnByteLimit`**：对召回的记忆强制执行行/字节双上限，保证单份记忆不超过 4KB，防止其吞掉整个上下文预算。
- **`AbortController` 链式传递**：子代理、prefetch、工具调用共享同一条 abort 链，用户按 `Esc` 可级联取消所有后台任务。
- **`canUseTool` 回调**：动态权限系统，在运行时根据路径和工具类型决定 allow/deny。

---

## 8. 思考总结

Claude Code 的记忆系统展示了一种**工程化程度极高的“软记忆”架构**：它不依赖向量数据库或外部检索服务，而是充分利用了**文件系统 + LLM 子代理 + 提示工程**的组合，达成了以下平衡：

1. **准确性 vs 开销**：通过 `hasMemoryWritesSince` 和 threshold gating，避免不必要的子代理调用；通过 prefetch 把相关性选择的 latency 隐藏在主线程工作之下。
2. **即时性 vs 持久性**：Session Memory 保证单会话内的状态不丢；AutoMem 和 CLAUDE.md 保证跨会话的上下文可继承。
3. **可控性 vs 自动化**：用户可以通过 `/memory` 命令显式编辑记忆文件，也可以在自然对话中让模型“记住/忘记”某事——系统自动在显式操作和后台补漏之间切换。

值得注意的工程细节包括：**staleness 提示的 cache-safe 固化**、**git worktree 的去重处理**、**条件规则（conditional rules）的按需注入**。这些设计表明，memory system 已经不是一个简单的“往文件里写摘要”的功能，而是一个与上下文管理、权限控制、提示缓存、版本控制深度耦合的子系统。

未来可以进一步观察的方向：
- `TEAMMEM` 特性开启后，team 记忆与 private 记忆的隔离和合并策略。
- `autoDream`（`/dream`）对日志文件的 nightly 蒸馏，如何从日纪要生成结构化主题记忆。
- 记忆系统的 telemetry（`MEMORY_SHAPE_TELEMETRY`）如何反馈于 prompt 迭代。
