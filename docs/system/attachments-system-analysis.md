# Claude Code Attachments 系统技术分析

## 1. 模块总结介绍

Attachments 系统是 Claude Code 的「上下文注入器」，负责自动或手动将相关文件、图片、记忆等内容附加到对话中。该系统确保模型在处理用户请求时能访问必要的上下文信息。

核心特性：
- **自动附件**：智能识别并附加相关文件
- **图片处理**：自动调整大小和格式转换
- **Token 预算管理**：确保附件不超过上下文限制
- **记忆注入**：CLAUDE.md 和 Session Memory 自动附加

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Attachments System                        │
├─────────────────────────────────────────────────────────────┤
│  Input Sources                                                │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌───────────┐ │
│  │ File Read  │ │   Image    │ │  Memory    │ │  Context  │ │
│  │  (@file)   │ │   Paste    │ │  Files     │ │  Suggestions│ │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬─────┘ │
│        └──────────────┼──────────────┼──────────────┘         │
│                       │              │                        │
│                       ▼              ▼                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Attachment Processor                        ││
│  │  • Token budget  • Image resize  • Content filter       ││
│  └─────────────────────────────────────────────────────────┘│
│                       │                                       │
│                       ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Message Assembly                            ││
│  │  • Reorder (attachments first)  • Deduplicate           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## 3. 附件类型

### 3.1 附件消息格式

```typescript
// types/message.ts
type AttachmentMessage = {
  type: 'attachment'
  uuid: string
  content: ContentBlockParam[]
  source: AttachmentSource
  metadata: {
    filePath?: string
    fileType?: string
    tokenCount: number
    isImage?: boolean
  }
}

type AttachmentSource =
  | 'file_read'      // @file 引用
  | 'image_paste'    // 粘贴的图片
  | 'memory'         // CLAUDE.md / Session Memory
  | 'context_suggestion'  // 上下文建议
  | 'tool_result'    // 工具结果引用
```

### 3.2 附件生成

```typescript
// utils/attachments.ts
export async function generateFileAttachment(
  filePath: string,
  options: AttachmentOptions = {}
): Promise<AttachmentMessage | null> {
  const resolvedPath = await expandPath(filePath)
  
  // 检查文件类型
  const isImage = isImageFile(resolvedPath)
  
  if (isImage) {
    return generateImageAttachment(resolvedPath, options)
  }

  // 文本文件
  const content = await readFileInRange(resolvedPath, {
    maxTokens: options.maxTokens || 8000,
  })

  if (!content) {
    return null
  }

  return createAttachmentMessage({
    source: 'file_read',
    content: [{ type: 'text', text: content }],
    metadata: {
      filePath: resolvedPath,
      fileType: 'text',
      tokenCount: estimateTokenCount(content),
    },
  })
}

async function generateImageAttachment(
  filePath: string,
  options: AttachmentOptions
): Promise<AttachmentMessage> {
  const imageBuffer = await readFile(filePath)
  
  // 调整大小和压缩
  const processed = await maybeResizeAndDownsampleImageBlock({
    source: {
      type: 'base64',
      media_type: getMimeType(filePath),
      data: imageBuffer.toString('base64'),
    },
  }, {
    maxWidth: 1568,
    maxHeight: 1568,
  })

  return createAttachmentMessage({
    source: 'image_paste',
    content: [processed],
    metadata: {
      filePath,
      fileType: 'image',
      isImage: true,
      tokenCount: calculateImageTokens(processed),
    },
  })
}
```

## 4. Token 预算管理

### 4.1 预算分配

```typescript
const TOKEN_BUDGETS = {
  TOTAL_ATTACHMENTS: 80000,      // 附件总预算
  PER_FILE: 10000,               // 单个文件上限
  PER_IMAGE: 16000,              // 单张图片上限
  MEMORY_FILES: 25000,           // 记忆文件预算
  CONTEXT_SUGGESTIONS: 10000,    // 上下文建议预算
}

export async function getAttachmentsWithinBudget(
  requestedFiles: string[],
  totalBudget: number = TOKEN_BUDGETS.TOTAL_ATTACHMENTS
): Promise<AttachmentMessage[]> {
  const attachments: AttachmentMessage[] = []
  let remainingBudget = totalBudget

  // 按重要性排序
  const prioritized = prioritizeFiles(requestedFiles)

  for (const filePath of prioritized) {
    const maxTokens = Math.min(
      TOKEN_BUDGETS.PER_FILE,
      remainingBudget
    )

    if (maxTokens <= 0) break

    const attachment = await generateFileAttachment(filePath, {
      maxTokens,
    })

    if (attachment) {
      attachments.push(attachment)
      remainingBudget -= attachment.metadata.tokenCount
    }
  }

  return attachments
}
```

### 4.2 文件范围读取

```typescript
// utils/processUserInput/readFileInRange.ts
export async function readFileInRange(
  filePath: string,
  options: {
    maxTokens?: number
    offset?: number
  }
): Promise<string | null> {
  const stats = await stat(filePath)
  
  // 小文件：直接读取
  if (stats.size < 10 * 1024) {
    return readFile(filePath, 'utf-8')
  }

  // 大文件：智能切片
  const maxBytes = (options.maxTokens || 8000) * 4  // 估算 4 字符/token
  
  const fd = await open(filePath, 'r')
  try {
    const buffer = Buffer.alloc(maxBytes)
    const { bytesRead } = await fd.read(
      buffer,
      0,
      maxBytes,
      options.offset || 0
    )

    let content = buffer.toString('utf-8', 0, bytesRead)

    // 如果截断了，添加省略标记
    if (bytesRead < stats.size) {
      content += '\n\n[...truncated]'
    }

    return content
  } finally {
    await fd.close()
  }
}
```

## 5. 智能附件建议

### 5.1 相关文件检测

```typescript
export async function suggestRelevantFiles(
  userInput: string,
  context: SuggestionContext
): Promise<SuggestedAttachment[]> {
  const suggestions: SuggestedAttachment[] = []

  // 1. 提取输入中的文件引用
  const mentionedFiles = extractFileReferences(userInput)
  for (const file of mentionedFiles) {
    suggestions.push({
      filePath: file,
      reason: 'explicitly_mentioned',
      priority: 1,
    })
  }

  // 2. 基于 Git 状态
  const gitFiles = await getGitModifiedFiles()
  for (const file of gitFiles) {
    suggestions.push({
      filePath: file,
      reason: 'git_modified',
      priority: 2,
    })
  }

  // 3. 基于最近访问
  const recentFiles = await getRecentlyAccessedFiles(5)
  for (const file of recentFiles) {
    if (!suggestions.find(s => s.filePath === file)) {
      suggestions.push({
        filePath: file,
        reason: 'recently_accessed',
        priority: 3,
      })
    }
  }

  // 4. 语义搜索（如果有嵌入）
  const semanticFiles = await semanticFileSearch(userInput)
  for (const file of semanticFiles) {
    if (!suggestions.find(s => s.filePath === file.path)) {
      suggestions.push({
        filePath: file.path,
        reason: 'semantic_match',
        score: file.score,
        priority: 4,
      })
    }
  }

  return suggestions.sort((a, b) => a.priority - b.priority)
}
```

### 5.2 上下文感知注入

```typescript
export async function injectContextAwareAttachments(
  messages: Message[],
  userInput: string
): Promise<Message[]> {
  // 1. 获取建议
  const suggestions = await suggestRelevantFiles(userInput, {
    currentFiles: getOpenFiles(),
    recentEdits: getRecentEdits(),
  })

  // 2. 过滤已存在的
  const existingPaths = new Set(
    messages
      .filter(m => m.type === 'attachment')
      .map(m => m.metadata.filePath)
  )

  const newSuggestions = suggestions.filter(
    s => !existingPaths.has(s.filePath)
  )

  // 3. 生成附件
  const attachments = await getAttachmentsWithinBudget(
    newSuggestions.map(s => s.filePath)
  )

  // 4. 插入到用户消息前
  if (attachments.length > 0) {
    const lastUserIndex = messages.findLastIndex(m => m.type === 'user')
    messages.splice(lastUserIndex, 0, ...attachments)
  }

  return messages
}
```

## 6. 记忆文件注入

### 6.1 CLAUDE.md 加载

```typescript
export async function loadMemoryPrompt(cwd: string): Promise<string | null> {
  const memoryFiles: string[] = []

  // 向上遍历目录树收集 CLAUDE.md
  let currentDir = cwd
  while (true) {
    const claudeMdPath = join(currentDir, '.claude', 'CLAUDE.md')
    
    if (await fileExists(claudeMdPath)) {
      memoryFiles.unshift(claudeMdPath) // 父目录在前
    }

    const parentDir = dirname(currentDir)
    if (parentDir === currentDir) break
    currentDir = parentDir
  }

  // 检查用户主目录
  const homeClaudeMd = getGlobalClaudeFile('CLAUDE.md')
  if (await fileExists(homeClaudeMd)) {
    memoryFiles.unshift(homeClaudeMd)
  }

  if (memoryFiles.length === 0) {
    return null
  }

  // 读取并合并
  const contents = await Promise.all(
    memoryFiles.map(async (path) => {
      const content = await readFile(path, 'utf-8')
      return `<!-- ${path} -->\n${content}`
    })
  )

  return contents.join('\n\n')
}
```

### 6.2 Session Memory

```typescript
export async function getSessionMemoryAttachments(): Promise<AttachmentMessage[]> {
  const memoryDir = getMemoryPath()
  const files = await getMemoryFiles(memoryDir)

  const attachments: AttachmentMessage[] = []
  let budget = TOKEN_BUDGETS.MEMORY_FILES

  for (const file of files) {
    const content = await readFile(file.path, 'utf-8')
    const tokens = estimateTokenCount(content)

    if (tokens > budget) {
      // 截断或跳过
      continue
    }

    attachments.push(
      createAttachmentMessage({
        source: 'memory',
        content: [{ type: 'text', text: content }],
        metadata: {
          filePath: file.path,
          fileType: 'memory',
          tokenCount: tokens,
        },
      })
    )

    budget -= tokens
  }

  return attachments
}
```

## 7. 图片处理

### 7.1 图片大小调整

```typescript
// utils/imageResizer.ts
export async function maybeResizeAndDownsampleImageBlock(
  image: ImageBlockParam,
  options: {
    maxWidth?: number
    maxHeight?: number
    quality?: number
  }
): Promise<ImageBlockParam> {
  if (image.source.type !== 'base64') {
    return image
  }

  const { width, height } = await getImageDimensions(image.source.data)

  // 检查是否需要调整
  if (
    width <= (options.maxWidth || 1568) &&
    height <= (options.maxHeight || 1568)
  ) {
    return image
  }

  // 计算新尺寸
  const ratio = Math.min(
    (options.maxWidth || 1568) / width,
    (options.maxHeight || 1568) / height
  )

  const newWidth = Math.floor(width * ratio)
  const newHeight = Math.floor(height * ratio)

  // 调整图片
  const resized = await resizeImage(image.source.data, {
    width: newWidth,
    height: newHeight,
    quality: options.quality || 0.85,
  })

  return {
    type: 'image',
    source: {
      type: 'base64',
      media_type: image.source.media_type,
      data: resized,
    },
  }
}
```

## 8. 附件排序与去重

### 8.1 消息重排序

```typescript
export function reorderMessagesForAPI(
  messages: Message[]
): Message[] {
  // 附件消息移到用户消息之前
  const attachments = messages.filter(m => m.type === 'attachment')
  const others = messages.filter(m => m.type !== 'attachment')

  // 在附件内部按来源排序
  const sortedAttachments = attachments.sort((a, b) => {
    const priority = {
      memory: 1,
      context_suggestion: 2,
      file_read: 3,
      tool_result: 4,
      image_paste: 5,
    }
    return priority[a.source] - priority[b.source]
  })

  return [...sortedAttachments, ...others]
}
```

### 8.2 去重

```typescript
export function deduplicateAttachments(
  messages: Message[]
): Message[] {
  const seen = new Set<string>()
  const result: Message[] = []

  for (const message of messages) {
    if (message.type === 'attachment') {
      const key = message.metadata.filePath || message.uuid
      
      if (seen.has(key)) {
        continue // 跳过重复
      }
      
      seen.add(key)
    }
    
    result.push(message)
  }

  return result
}
```

---

Attachments 系统通过智能建议、Token 预算管理、图片处理等设计，确保模型始终拥有必要的上下文信息。理解其设计，对于构建上下文感知的 AI 系统具有参考价值。
