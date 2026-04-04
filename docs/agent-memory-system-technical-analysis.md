# Claude Code Agent 记忆系统深度技术分析

> 基于源码的架构、分层、关键技术、交互流程与创新点分析

---

## 一、系统概述

Claude Code 的 Agent 记忆系统是一个**分层、多模态、分布式**的记忆架构，旨在为 AI Agent 提供跨会话的持久化记忆能力。该系统不仅支持单个用户的个人记忆，还支持团队协作的共享记忆，是一个企业级的记忆解决方案。

### 1.1 核心设计目标

- **跨会话持久化**: 记忆不随会话结束而丢失
- **分层存储**: 用户级、项目级、团队级多层级记忆
- **隐私与共享平衡**: 私人记忆与团队共享记忆的灵活切换
- **自动整理**: 背景压缩、摘要提取等自动化维护
- **安全合规**: 秘密扫描、权限控制、审计日志

---

## 二、架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Agent Memory System                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  User Memory │  │Project Memory│  │  Team Memory │  │ Session Mem  │   │
│  │   (用户级)    │  │   (项目级)    │  │   (团队级)    │  │   (会话级)    │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                 │                 │           │
│         └─────────────────┴─────────────────┴─────────────────┘           │
│                                    │                                       │
│                    ┌───────────────┴───────────────┐                       │
│                    │      Memory Directory          │                       │
│                    │      (memdir/)                 │                       │
│                    └───────────────┬───────────────┘                       │
│                                    │                                       │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         │                          │                          │            │
│  ┌──────▼──────┐          ┌────────▼────────┐       ┌────────▼──────┐    │
│  │  Local FS   │          │  Sync Service   │       │  Remote API   │    │
│  │  本地文件    │◄────────►│  同步服务      │◄─────►│  云服务       │    │
│  └─────────────┘          └─────────────────┘       └───────────────┘    │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 记忆类型分层

系统定义了四种核心记忆类型（`MemoryType`）:

| 类型 | 作用域 | 描述 | 保存时机 |
|------|--------|------|----------|
| **user** | 始终私有 | 用户角色、目标、知识背景 | 了解用户细节时 |
| **feedback** | 默认私有，项目约定可共享 | 工作方式指导（该避免/保持什么） | 用户纠正或确认时 |
| **project** | 私有或团队，偏向团队 | 项目中的工作、目标、bug、事件 | 了解项目状态时 |
| **reference** | 通常是团队 | 外部系统信息指针（Linear、Slack等） | 了解外部资源时 |

### 2.3 三层存储架构

```typescript
// 核心记忆作用域定义
export type AgentMemoryScope = 'user' | 'project' | 'local'

// 1. User Scope: ~/.claude/agent-memory/<agentType>/
// 2. Project Scope: <cwd>/.claude/agent-memory/<agentType>/
// 3. Local Scope: <cwd>/.claude/agent-memory-local/<agentType>/ (不纳入版本控制)
```

---

## 三、核心组件详解

### 3.1 Memory Directory (memdir)

**文件位置**: `src/memdir/`

记忆目录是系统的核心存储抽象，负责：

#### 3.1.1 目录结构

```
~/.claude/                          # 用户级记忆根目录
├── agent-memory/                   # Agent 专用记忆
│   └── <agent-type>/
│       ├── MEMORY.md               # 记忆索引入口
│       ├── user_role.md
│       ├── feedback_testing.md
│       └── project_deadlines.md
├── projects/
│   └── <project-slug>/
│       ├── memory/                 # 自动记忆
│       │   ├── MEMORY.md
│       │   └── topic-files.md
│       └── agent-memory/           # 项目级Agent记忆
└── session-memory/                 # 会话记忆
    └── <session-id>.md
```

#### 3.1.2 核心功能

```typescript
// src/memdir/memdir.ts

/**
 * 构建记忆提示词（包含MEMORY.md内容）
 */
export function buildMemoryPrompt(params: {
  displayName: string
  memoryDir: string
  extraGuidelines?: string[]
}): string

/**
 * 加载记忆提示词（用于系统提示）
 */
export async function loadMemoryPrompt(): Promise<string | null>

/**
 * 确保记忆目录存在
 */
export async function ensureMemoryDirExists(memoryDir: string): Promise<void>

/**
 * 内容截断（防止上下文溢出）
 * 限制：200行 或 25KB
 */
export function truncateEntrypointContent(raw: string): EntrypointTruncation
```

### 3.2 Session Memory Service

**文件位置**: `src/services/SessionMemory/`

会话记忆是系统最具创新性的特性之一，它在后台自动维护当前会话的笔记。

#### 3.2.1 工作原理

```typescript
// src/services/SessionMemory/sessionMemory.ts

/**
 * 会话记忆自动提取触发条件：
 * 1. 初始化阈值：达到最小token数后首次提取
 * 2. 更新阈值：token增长 + 工具调用数达标
 * 3. 安全时机：上一轮助手消息无工具调用（自然断点）
 */
export function shouldExtractMemory(messages: Message[]): boolean {
  // 检查初始化阈值
  if (!isSessionMemoryInitialized()) {
    return hasMetInitializationThreshold(currentTokenCount)
  }

  // 检查更新阈值
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)
  const hasMetToolCallThreshold = toolCallsSinceLastUpdate >= getToolCallsBetweenUpdates()
  const hasToolCallsInLastTurn = hasToolCallsInLastAssistantTurn(messages)

  // 触发条件：token+工具调用 都达标，或者 token达标且上轮无工具调用
  return (hasMetTokenThreshold && hasMetToolCallThreshold) ||
         (hasMetTokenThreshold && !hasToolCallsInLastTurn)
}
```

#### 3.2.2 后台提取流程

```
用户对话 ──► 达到阈值 ──► fork子Agent ──► 分析对话内容 ──► 更新 session-memory.md
                │
                └──► 主流程继续，无感知
```

**关键技术**：使用 `runForkedAgent` 创建隔离上下文，避免污染父状态。

### 3.3 Team Memory Sync

**文件位置**: `src/services/teamMemorySync/`

团队记忆同步服务实现跨设备、跨用户的记忆共享。

#### 3.3.1 同步语义

```typescript
// src/services/teamMemorySync/index.ts

/**
 * 同步状态管理
 */
export type SyncState = {
  lastKnownChecksum: string | null     // ETag用于条件请求
  serverChecksums: Map<string, string> // 服务器内容哈希
  serverMaxEntries: number | null      // 服务器限制（动态学习）
}

/**
 * 同步规则：
 * - Pull：服务器内容覆盖本地（服务器优先）
 * - Push：仅上传哈希变化的条目（增量上传）
 * - 删除不传播：本地删除不会同步到服务器
 */
```

#### 3.3.2 API 契约

```
GET  /api/claude_code/team_memory?repo={owner/repo}           → 完整数据
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → 仅哈希
PUT  /api/claude_code/team_memory?repo={owner/repo}           → 上传条目
```

### 3.4 Agent Memory 系统

**文件位置**: `src/tools/AgentTool/agentMemory.ts`

为特定 Agent 类型提供持久化记忆。

```typescript
/**
 * 加载Agent记忆提示词
 *
 * @param agentType Agent类型名称（作为目录名）
 * @param scope 'user' | 'project' | 'local'
 */
export function loadAgentMemoryPrompt(
  agentType: string,
  scope: AgentMemoryScope
): string {
  const memoryDir = getAgentMemoryDir(agentType, scope)

  // 根据作用域添加指导说明
  const scopeNote = scope === 'user'
    ? 'user-scope记忆通用，适用于所有项目'
    : scope === 'project'
    ? 'project-scope记忆与团队共享，针对特定项目'
    : 'local-scope记忆仅本地使用'

  return buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines: [scopeNote]
  })
}
```

---

## 四、关键技术创新

### 4.1 记忆内容双重截断机制

```typescript
// src/memdir/memdir.ts

/**
 * 智能内容截断：
 * 1. 先按行截断（200行）- 保持语义完整
 * 2. 再按字节截断（25KB）- 处理超长行
 *
 * 截断点在最后一个换行符处，避免切断单词
 */
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES  // 200
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES  // 25_000

  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }

  // 添加警告提示
  return {
    content: truncated + `\n\n> WARNING: ${ENTRYPOINT_NAME} is ${reason}...`,
    // ...
  }
}
```

### 4.2 增量同步与冲突解决

```typescript
// src/services/teamMemorySync/index.ts

/**
 * 增量上传算法：
 * 1. 计算本地每个文件的 sha256 哈希
 * 2. 与 serverChecksums 比较
 * 3. 仅上传哈希不同的文件
 * 4. 学习服务器限制（从 413 错误中提取 max_entries）
 */
async function pushTeamMemory(
  state: SyncState,
  entries: TeamMemoryEntry[]
): Promise<TeamMemorySyncPushResult> {
  // 过滤出需要上传的条目
  const entriesToUpload = entries.filter(entry => {
    const localHash = hashContent(entry.content)
    return localHash !== state.serverChecksums.get(entry.key)
  })

  // 分批上传（避免网关限制 200KB）
  const batches = createBatches(entriesToUpload, MAX_PUT_BODY_BYTES)
  for (const batch of batches) {
    await uploadBatch(batch)
  }
}
```

### 4.3 会话记忆自动压缩

```typescript
// src/services/compact/compact.ts

/**
 * 自动压缩策略：
 * 1. 监控token使用量
 * 2. 达到阈值时触发压缩
 * 3. fork子Agent提取关键信息
 * 4. 重写会话历史，保留关键上下文
 */
export async function compactConversation(
  messages: Message[],
  threshold: number
): Promise<CompactResult> {
  // 提取不可丢失的消息（工具调用序列）
  const toolSequences = extractToolSequences(messages)

  // 提取会话摘要
  const summary = await extractSummary(messages)

  // 构建压缩后的消息序列
  return buildCompactedMessages(summary, toolSequences)
}
```

### 4.4 In-Process Teammate 的 AsyncLocalStorage 隔离

```typescript
// src/utils/swarm/inProcessRunner.ts

/**
 * In-process teammate 使用 AsyncLocalStorage 实现上下文隔离：
 * - 每个 teammate 有自己的 AsyncLocalStorage 上下文
 * - 共享同一线程，但逻辑完全隔离
 * - 支持mailbox通信、权限代理、进度上报
 */
export async function runInProcessTeammate(
  identity: TeammateIdentity,
  prompt: string,
  toolUseContext: ToolUseContext
): Promise<void> {
  return runWithTeammateContext(identity, async () => {
    // 在此上下文中运行的代码可以访问 teammate 身份
    // 通过 AsyncLocalStorage 自动隔离
    return runAgent({
      prompt,
      // ...
    })
  })
}
```

---

## 五、交互流程分析

### 5.1 记忆保存流程

```
用户请求保存 ──► AgentTool.call() ──► 确定 MemoryType ──► 选择存储位置
                                                          │
                    ┌─────────────────────────────────────┼─────────────────────┐
                    │                                     │                     │
                    ▼                                     ▼                     ▼
              User Scope                            Project Scope          Team Scope
         ~/.claude/agent-memory/                 .claude/agent-memory/    (远程同步)
                    │                                     │                     │
                    ▼                                     ▼                     ▼
             写入 topic 文件                        写入 topic 文件       PUT /api/team_memory
                    │                                     │                     │
                    └─────────────────────────────────────┼─────────────────────┘
                                                          ▼
                                                  更新 MEMORY.md 索引
                                                          │
                                                          ▼
                                                  触发 Team Memory Sync
```

### 5.2 记忆读取流程

```
系统提示构建 ──► loadMemoryPrompt() ──► 检查各作用域
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
              读取 User                  读取 Project               读取 Team
               Memory                      Memory                    Memory
                    │                         │                         │
                    └─────────────────────────┼─────────────────────────┘
                                              ▼
                                      构建 Memory Lines
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
              TYPES_SECTION           WHAT_NOT_TO_SAVE          WHEN_TO_ACCESS
              (记忆类型说明)              (排除项说明)              (访问时机)
                    │                         │                         │
                    └─────────────────────────┴─────────────────────────┘
                                              │
                                              ▼
                                      截断并组装提示词
                                              │
                                              ▼
                                      注入系统提示上下文
```

### 5.3 Agent 生命周期中的记忆交互

```
Agent 启动
    │
    ├─► loadAgentMemoryPrompt() ──► 读取持久化记忆 ──► 注入系统提示
    │
    ▼
Agent 运行
    │
    ├─► 使用记忆指导行为
    │
    ├─► 可能产生新记忆 ──► Write/Edit tool ──► 保存到记忆目录
    │                              │
    │                              ├─► 写入 topic 文件
    │                              └─► 更新 MEMORY.md 索引
    │
    ▼
Agent 完成
    │
    └─► 清理运行时状态（保留持久化记忆）
```

---

## 六、安全与隐私设计

### 6.1 秘密扫描

```typescript
// src/services/teamMemorySync/secretScanner.ts

/**
 * 预上传扫描：防止敏感信息泄露
 * 使用 entropy 检测 + 模式匹配
 */
export function scanForSecrets(content: string): SecretScanResult {
  const findings: SecretFinding[] = []

  // 高熵检测（API keys, tokens）
  const highEntropyStrings = extractHighEntropyStrings(content)

  // 已知模式匹配
  for (const pattern of KNOWN_SECRET_PATTERNS) {
    const matches = content.match(pattern.regex)
    if (matches) {
      findings.push({
        type: pattern.type,
        line: getLineNumber(content, matches.index),
        snippet: maskSecret(matches[0])
      })
    }
  }

  return { safe: findings.length === 0, findings }
}
```

### 6.2 权限控制

```typescript
// src/utils/permissions/permissions.ts

/**
 * Agent 工具权限检查
 */
export function filterDeniedAgents(
  agents: AgentDefinition[],
  permissionContext: ToolPermissionContext,
  toolName: string
): AgentDefinition[] {
  return agents.filter(agent => {
    const denyRule = getDenyRuleForAgent(permissionContext, toolName, agent.agentType)
    return !denyRule  // 没有被拒绝规则的才保留
  })
}
```

### 6.3 记忆漂移检测

```typescript
// src/memdir/memoryTypes.ts

/**
 * 记忆漂移警告：记忆可能过时，使用前应验证
 */
export const MEMORY_DRIFT_CAVEAT =
  '- Memory records can become stale over time. Use memory as context for what was true at a given point in time. Before answering the user or building assumptions based solely on information in memory records, verify that the memory is still correct and up-to-date by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory rather than acting on it.'
```

---

## 七、性能优化策略

### 7.1 记忆提示词截断

```typescript
// 限制 MEMORY.md 加载量，防止上下文膨胀
const MAX_ENTRYPOINT_LINES = 200      // 最多200行
const MAX_ENTRYPOINT_BYTES = 25_000   // 最多25KB
```

### 7.2 异步初始化

```typescript
// 记忆目录创建是 fire-and-forget
void ensureMemoryDirExists(memoryDir)

// 同步回调中返回提示词，mkdir 在后台执行
```

### 7.3 增量同步

- 使用 SHA256 哈希比较，仅上传变化内容
- 支持 ETag 条件请求，避免重复下载
- 动态学习服务器限制，自适应分批次上传

### 7.4 后台任务优化

```typescript
// Agent 任务消息上限（UI层面）
export const TEAMMATE_MESSAGES_UI_CAP = 50

// 使用双数组结构，避免大数据拷贝
export function appendCappedMessage<T>(
  prev: readonly T[] | undefined,
  item: T
): T[] {
  if (prev && prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1))
    next.push(item)
    return next
  }
  return [...(prev ?? []), item]
}
```

---

## 八、创新点总结

### 8.1 架构层面

1. **四层记忆模型**: User/Project/Reference/Session 的分层设计，清晰划分记忆作用域
2. **双轨存储**: 本地文件系统 + 远程云服务的无缝同步
3. **Agent 记忆隔离**: 不同类型 Agent 拥有独立的记忆空间

### 8.2 技术层面

1. **自动会话摘要**: 后台 fork 子 Agent 自动提取关键信息，零感知维护
2. **智能内容截断**: 行级+字节级双重截断，语义完整性与上下文限制兼顾
3. **增量同步算法**: 哈希比对 + 动态学习服务器限制，高效安全
4. **AsyncLocalStorage 隔离**: In-process teammate 的轻量级上下文隔离

### 8.3 产品层面

1. **隐私与协作平衡**: 私人记忆与团队记忆的灵活切换
2. **零配置使用**: 自动创建目录、自动同步、自动压缩
3. **安全内置**: 秘密扫描、权限控制、漂移检测等安全机制

---

## 九、源码文件索引

| 文件路径 | 功能描述 |
|----------|----------|
| `src/memdir/memdir.ts` | 记忆目录核心操作 |
| `src/memdir/memoryTypes.ts` | 记忆类型定义与提示词模板 |
| `src/memdir/paths.ts` | 记忆路径管理 |
| `src/memdir/teamMemPaths.ts` | 团队记忆路径 |
| `src/services/SessionMemory/sessionMemory.ts` | 会话记忆服务 |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | 会话记忆工具函数 |
| `src/services/teamMemorySync/index.ts` | 团队记忆同步服务 |
| `src/services/teamMemorySync/secretScanner.ts` | 秘密扫描 |
| `src/tools/AgentTool/agentMemory.ts` | Agent 记忆加载 |
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具实现 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 本地 Agent 任务 |
| `src/tasks/InProcessTeammateTask/` | In-process teammate 实现 |
| `src/utils/swarm/inProcessRunner.ts` | In-process runner |
| `src/utils/memory/types.ts` | 记忆类型定义 |

---

## 十、总结

Claude Code 的 Agent 记忆系统是一个**工程化程度极高**的实现，它不仅仅是简单的文件存储，而是融合了：

- **分布式系统思维**: 本地/远程同步、增量更新、冲突解决
- **安全工程实践**: 秘密扫描、权限控制、审计日志
- **性能优化技巧**: 截断策略、异步初始化、缓存机制
- **AI 原生设计**: 自动摘要、上下文感知、提示词工程

这套系统为多 Agent 协作、跨会话学习、团队知识共享提供了坚实的基础设施，是 Claude Code 区别于其他 AI 编程工具的核心竞争力之一。
