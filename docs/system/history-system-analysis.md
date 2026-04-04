# Claude Code History 系统技术分析

## 1. 模块总结介绍

History 系统是 Claude Code 的「记忆回溯中枢」，负责管理用户输入历史、粘贴内容追踪和会话恢复。该系统让用户能够快速重用之前的输入，并追踪大型粘贴内容的来源。

核心功能：
- **输入历史**：保存和检索用户历史输入（最多100条）
- **粘贴追踪**：追踪粘贴内容，支持引用和展开
- **图片历史**：管理粘贴的图片内容
- **会话恢复**：支持从断点恢复对话

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      History System                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  Input History  │  │ Pasted Content  │  │  Sessions   │ │
│  │   (用户输入)     │  │   (粘贴内容)     │  │  (会话存档)  │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬──────┘ │
│           │                    │                  │        │
│           ▼                    ▼                  ▼        │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Persistent Storage                      │  │
│  │  ~/.claude/history.jsonl                            │  │
│  │  ~/.claude/paste-store/                             │  │
│  │  ~/.claude/sessions/                                │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心组件

### 3.1 输入历史

```typescript
// history.ts
const MAX_HISTORY_ITEMS = 100

type HistoryEntry = {
  id: number
  type: 'text' | 'image'
  content?: string           // 文本内容（小内容内联）
  contentHash?: string       // 大内容哈希引用
  mediaType?: string         // 图片类型
  filename?: string          // 图片文件名
  timestamp: number
}
```

### 3.2 粘贴引用格式

```typescript
/**
 * 粘贴引用格式：
 *   Text: [Pasted text #1 +10 lines]
 *   Image: [Image #2]
 *
 * ID 在单次提示中唯一，使用数字便于用户理解
 */

export function formatPastedTextRef(id: number, numLines: number): string {
  if (numLines === 0) {
    return `[Pasted text #${id}]`
  }
  return `[Pasted text #${id} +${numLines} lines]`
}

export function formatImageRef(id: number): string {
  return `[Image #${id}]`
}
```

## 4. 引用解析与展开

### 4.1 引用解析

```typescript
// 解析引用模式
const REFERENCE_PATTERN =
  /\[(Pasted text|Image|\.\.\.Truncated text) #(\d+)(?: \+\d+ lines)?(\.)*\]/g

export function parseReferences(
  input: string
): Array<{ id: number; match: string; index: number }> {
  const matches = [...input.matchAll(REFERENCE_PATTERN)]
  return matches
    .map((match) => ({
      id: parseInt(match[2] || '0'),
      match: match[0],
      index: match.index!,
    }))
    .filter((match) => match.id > 0)
}
```

### 4.2 引用展开

```typescript
/**
 * 将引用替换为实际内容
 */
export function expandPastedTextRefs(
  input: string,
  pastedContents: Record<number, PastedContent>
): string {
  const refs = parseReferences(input)
  let expanded = input

  // 从后向前替换，保持索引正确
  for (let i = refs.length - 1; i >= 0; i--) {
    const ref = refs[i]!
    const content = pastedContents[ref.id]

    if (content?.type !== 'text') continue

    expanded =
      expanded.slice(0, ref.index) +
      content.content +
      expanded.slice(ref.index + ref.match.length)
  }

  return expanded
}
```

## 5. 粘贴内容存储

### 5.1 存储策略

```typescript
const MAX_PASTED_CONTENT_LENGTH = 1024

type StoredPastedContent = {
  id: number
  type: 'text' | 'image'
  content?: string        // 小内容内联存储
  contentHash?: string    // 大内容哈希引用
  mediaType?: string
  filename?: string
}

// 存储决策
function storePastedContent(content: PastedContent): StoredPastedContent {
  if (content.content.length <= MAX_PASTED_CONTENT_LENGTH) {
    // 小内容：内联存储
    return {
      id: generateId(),
      type: content.type,
      content: content.content,
    }
  } else {
    // 大内容：哈希引用，存储到 paste-store
    const hash = hashPastedText(content.content)
    storePasteToDisk(hash, content.content)

    return {
      id: generateId(),
      type: content.type,
      contentHash: hash,
    }
  }
}
```

### 5.2 Paste Store 磁盘存储

```typescript
// pasteStore.ts
const PASTE_STORE_DIR = '~/.claude/paste-store/'

export async function storePasteToDisk(hash: string, content: string): Promise<void> {
  const filePath = join(PASTE_STORE_DIR, `${hash}.txt`)
  await writeFile(filePath, content, 'utf-8')
}

export async function retrievePastedText(hash: string): Promise<string | null> {
  const filePath = join(PASTE_STORE_DIR, `${hash}.txt`)
  try {
    return await readFile(filePath, 'utf-8')
  } catch (error) {
    return null
  }
}
```

## 6. 历史文件管理

### 6.1 文件锁机制

```typescript
// 并发安全的历史写入
export async function appendHistoryEntry(entry: HistoryEntry): Promise<void> {
  const historyPath = join(getClaudeConfigHomeDir(), 'history.jsonl')

  // 获取文件锁
  const release = await lock(historyPath, {
    stale: 5000,
    retries: 3,
    retryWait: 100,
  })

  try {
    const line = JSON.stringify(entry) + '\n'
    await appendFile(historyPath, line, 'utf-8')

    // 裁剪历史（保持最多100条）
    await trimHistoryIfNeeded(historyPath)
  } finally {
    await release()
  }
}
```

### 6.2 历史裁剪

```typescript
async function trimHistoryIfNeeded(historyPath: string): Promise<void> {
  // 读取所有条目
  const lines = await readFile(historyPath, 'utf-8')
  const entries = lines
    .split('\n')
    .filter(Boolean)
    .map((line) => JSON.parse(line))

  if (entries.length <= MAX_HISTORY_ITEMS) return

  // 保留最近的条目
  const trimmed = entries.slice(-MAX_HISTORY_ITEMS)

  // 原子写入
  const tempPath = historyPath + '.tmp'
  await writeFile(
    tempPath,
    trimmed.map((e) => JSON.stringify(e)).join('\n') + '\n'
  )
  await rename(tempPath, historyPath)
}
```

## 7. 历史检索

### 7.1 反向读取

```typescript
// 从历史末尾读取（最近的优先）
export async function* readHistoryReverse(
  limit?: number
): AsyncGenerator<HistoryEntry> {
  const historyPath = join(getClaudeConfigHomeDir(), 'history.jsonl')

  // 使用反向行读取优化
  for await (const line of readLinesReverse(historyPath)) {
    if (!line.trim()) continue

    try {
      yield JSON.parse(line)
    } catch {
      continue // 跳过损坏的条目
    }

    if (limit !== undefined) {
      limit--
      if (limit <= 0) break
    }
  }
}
```

### 7.2 模糊搜索

```typescript
export async function searchHistory(query: string): Promise<HistoryEntry[]> {
  const results: HistoryEntry[] = []

  for await (const entry of readHistoryReverse()) {
    if (entry.type !== 'text' || !entry.content) continue

    // 简单子串匹配（可扩展为模糊匹配）
    if (entry.content.toLowerCase().includes(query.toLowerCase())) {
      results.push(entry)
    }

    if (results.length >= 10) break // 最多10条结果
  }

  return results
}
```

## 8. 会话存档与恢复

### 8.1 会话元数据

```typescript
// sessionStorage.ts
type SessionMetadata = {
  sessionId: string
  createdAt: number
  lastActivityAt: number
  cwd: string
  gitBranch?: string
  gitRepoUrl?: string
  model: string
  totalCostUSD: number
}

export function recordTranscript(metadata: SessionMetadata): void {
  const transcriptPath = getTranscriptPath(metadata.sessionId)

  // 追加会话元数据
  const line = JSON.stringify({
    type: 'metadata',
    ...metadata,
  }) + '\n'

  appendFileSync(transcriptPath, line)
}
```

### 8.2 消息记录

```typescript
export function recordMessage(
  sessionId: string,
  message: Message
): void {
  const transcriptPath = getTranscriptPath(sessionId)

  const line = JSON.stringify({
    type: 'message',
    timestamp: Date.now(),
    message,
  }) + '\n'

  appendFileSync(transcriptPath, line)
}
```

### 8.3 会话恢复

```typescript
export async function loadSession(sessionId: string): Promise<{
  metadata: SessionMetadata
  messages: Message[]
} | null> {
  const transcriptPath = getTranscriptPath(sessionId)

  try {
    const content = await readFile(transcriptPath, 'utf-8')
    const lines = content.split('\n').filter(Boolean)

    let metadata: SessionMetadata | null = null
    const messages: Message[] = []

    for (const line of lines) {
      const record = JSON.parse(line)

      if (record.type === 'metadata') {
        metadata = record
      } else if (record.type === 'message') {
        messages.push(record.message)
      }
    }

    if (!metadata) return null

    return { metadata, messages }
  } catch (error) {
    return null
  }
}
```

## 9. 垃圾回收

### 9.1 过期清理

```typescript
const SESSION_MAX_AGE_MS = 30 * 24 * 60 * 60 * 1000 // 30天

export async function cleanupOldSessions(): Promise<void> {
  const sessionsDir = join(getClaudeConfigHomeDir(), 'sessions')
  const entries = await readdir(sessionsDir, { withFileTypes: true })

  const now = Date.now()

  for (const entry of entries) {
    if (!entry.isFile() || !entry.name.endsWith('.jsonl')) continue

    const filePath = join(sessionsDir, entry.name)
    const stats = await stat(filePath)

    if (now - stats.mtimeMs > SESSION_MAX_AGE_MS) {
      await unlink(filePath)
    }
  }
}
```

### 9.2 Orphaned Paste 清理

```typescript
export async function cleanupOrphanedPastes(): Promise<void> {
  // 收集所有被引用的哈希
  const referencedHashes = new Set<string>()

  for await (const entry of readHistoryReverse()) {
    if (entry.contentHash) {
      referencedHashes.add(entry.contentHash)
    }
  }

  // 删除未被引用的 paste
  const pasteStoreDir = PASTE_STORE_DIR
  const entries = await readdir(pasteStoreDir)

  for (const entry of entries) {
    const hash = entry.replace('.txt', '')
    if (!referencedHashes.has(hash)) {
      await unlink(join(pasteStoreDir, entry))
    }
  }
}
```

## 10. 创新点与反思

### 10.1 创新设计

1. **粘贴引用系统**：将大内容哈希存储，引用中只保存短哈希
2. **双向行读取**：支持从后向前读取历史，优化最近优先场景
3. **原子写入**：临时文件 + rename 保证数据完整性

### 10.2 架构权衡

**JSONL vs SQLite**

当前使用 JSON Lines 格式：
- ✅ 人类可读
- ✅ 易于追加
- ✅ 无需依赖
- ❌ 查询效率低
- ❌ 没有索引

对于大量历史数据，SQLite 可能更合适。

### 10.3 未来演进

1. **全文搜索**：集成 Meilisearch 或 SQLite FTS
2. **历史同步**：跨设备历史同步
3. **语义搜索**：基于 embedding 的相似历史查找

---

Claude Code 的 History 系统是一个轻量级但功能完整的历史管理方案。通过粘贴引用、反向读取、文件锁等设计，实现了高效的历史存储和检索。理解其设计，对于构建任何需要历史功能的 CLI 工具都具有参考价值。
