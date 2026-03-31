# 第 24 章：性能优化

> 性能不是优化出来的，是设计出来的。

---

## 24.1 Claude Code 的性能挑战

Claude Code 面临独特的性能挑战：

**启动延迟**：用户期望 CLI 工具快速启动。但 Claude Code 需要加载大量模块、初始化状态、连接 MCP 服务器。

**API 延迟**：每次 API 调用都有网络延迟。对于多轮工具调用，延迟会累积。

**大文件处理**：代码库可能有大量文件，搜索和读取需要高效。

**流式渲染**：终端 UI 需要流畅地渲染流式输出，不能有明显的卡顿。

**内存使用**：长对话的消息历史可能很大，需要控制内存使用。

---

## 24.2 启动性能优化

Claude Code 有专门的启动性能分析工具（`src/utils/startupProfiler.ts`）：

```typescript
// 在关键启动步骤打点
profileCheckpoint('imports_loaded')
profileCheckpoint('config_read')
profileCheckpoint('mcp_connected')
profileCheckpoint('repl_ready')

// 输出启动时间分析
// imports_loaded: 120ms
// config_read: 45ms
// mcp_connected: 230ms
// repl_ready: 395ms（总计）
```

**快速路径优化**：对于简单命令（`--version`、`--help`），跳过完整初始化：

```typescript
// main.tsx 中的快速路径
if (args.includes('--version')) {
  console.log(VERSION)
  process.exit(0)  // 不初始化任何东西
}
```

**预取优化**：在启动时并行预取需要的资源：

```typescript
// 并行预取，不阻塞主流程
Promise.all([
  prefetchApiKeyFromApiKeyHelperIfSafe(),  // 预取 API Key
  preconnectToAPI(),                        // 预建立 API 连接
  prefetchGitStatus(),                      // 预取 git 状态
])
```

---

## 24.3 API 调用优化

**提示缓存（Prompt Caching）**：
系统提示的稳定部分会被 API 缓存，后续请求不需要重新传输：

```typescript
// 稳定部分（可缓存）放在前面
const systemPrompt = [
  coreInstructions,    // 几乎不变 → 缓存命中率高
  toolDefinitions,     // 工具集不变时不变 → 缓存命中率高
  claudeMdContent,     // 文件不变时不变 → 缓存命中率中
  gitStatus,           // 每次可能不同 → 不缓存
]
```

提示缓存可以减少 90% 以上的输入 token 费用，同时降低延迟。

**流式处理**：
使用流式 API，用户不需要等待完整响应：

```typescript
// 流式 API 调用
const stream = await anthropic.messages.stream({
  model: 'claude-sonnet-4-6',
  messages,
  stream: true,  // 启用流式
})

// 立即开始处理响应
for await (const chunk of stream) {
  yield chunk  // 实时输出给用户
}
```

**并行工具执行**：
多个工具调用并行执行，而不是串行：

```typescript
// 并行执行所有工具调用
const results = await Promise.all(
  toolCalls.map(call => executeTool(call, context))
)
```

---

## 24.4 文件系统优化

**文件读取缓存**（`src/utils/fileStateCache.ts`）：
同一文件在一次对话中只读取一次，后续使用缓存：

```typescript
type FileStateCache = Map<string, {
  content: string
  mtime: number      // 文件修改时间
  readTime: number   // 读取时间
}>

// 读取文件时检查缓存
async function readFileWithCache(path: string, cache: FileStateCache) {
  const cached = cache.get(path)
  const mtime = await getFileMtime(path)

  if (cached && cached.mtime === mtime) {
    return cached.content  // 缓存命中
  }

  const content = await readFile(path)
  cache.set(path, { content, mtime, readTime: Date.now() })
  return content
}
```

**ripgrep 集成**：
GrepTool 使用 ripgrep（而不是 Node.js 的 fs 模块）进行文件搜索，速度快 10-100 倍。

**glob 优化**：
GlobTool 结果按修改时间排序，最相关的文件排在前面，减少后续读取的数量。

---

## 24.5 内存优化

**消息截断**：
工具结果可能很大（如读取了一个大文件），超过限制时自动截断：

```typescript
// 工具结果大小限制
const MAX_TOOL_RESULT_TOKENS = 25000

function truncateToolResult(result: string, maxTokens: number): string {
  const tokens = estimateTokens(result)
  if (tokens <= maxTokens) return result

  // 截断并添加说明
  const truncated = result.substring(0, estimateChars(maxTokens))
  return truncated + '\n\n[内容已截断，原始大小超过限制]'
}
```

**自动压缩**：
当消息历史超过 token 限制时，自动压缩（见第 15 章）。

**循环缓冲区**（`src/utils/CircularBuffer.ts`）：
用于存储有限数量的历史记录，自动丢弃最旧的条目。

---

## 24.6 渲染性能

Claude Code 使用 Ink（React for CLI）渲染 UI。React 的虚拟 DOM 机制保证了高效的增量更新：

```tsx
// 只有变化的部分才会重新渲染
function MessageList({ messages }) {
  return messages.map(msg => (
    <Message key={msg.uuid} message={msg} />
  ))
}
```

**React Compiler 优化**：
从源码可以看到 React Compiler 的痕迹：

```typescript
// src/state/AppState.tsx
import { c as _c } from "react/compiler-runtime";

export function AppStateProvider(t0) {
  const $ = _c(13)  // React Compiler 生成的缓存
  // ...
}
```

React Compiler 自动添加 memoization，减少不必要的重渲染。

---

## 24.7 Bun 运行时的性能优势

Claude Code 使用 Bun 作为运行时，而不是 Node.js：

| 维度 | Node.js | Bun |
|------|---------|-----|
| 启动时间 | ~100ms | ~10ms |
| 模块加载 | 慢（CommonJS） | 快（原生 ESM） |
| TypeScript | 需要编译 | 原生支持 |
| 包管理 | npm（慢） | bun（快 10-25x） |
| 内置工具 | 少 | 多（测试、打包等） |

Bun 的启动速度优势对 CLI 工具特别重要——用户期望 CLI 工具几乎瞬间启动。

---

## 24.8 Feature Flags 与死代码消除

Claude Code 使用 `bun:bundle` 的 `feature()` 函数实现编译时死代码消除：

```typescript
// 只有在启用 VOICE_MODE 时才包含语音相关代码
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children  // 空实现

// 构建时，未启用的 feature 对应的代码会被完全删除
// 减小 bundle 大小，提升加载速度
```

这让 Claude Code 可以针对不同场景（标准版、企业版、轻量版）构建不同的 bundle，每个 bundle 只包含必要的代码。

---

## 24.9 性能监控

Claude Code 有内置的性能监控：

```typescript
// API 调用延迟追踪
pushApiMetricsEntry?.(ttftMs)  // TTFT: Time To First Token

// 工具执行时间追踪
const toolStart = Date.now()
const result = await tool.execute(input, context)
const toolDuration = Date.now() - toolStart

// 每轮统计
turnToolDurationMs += toolDuration
turnToolCount++
```

这些指标帮助识别性能瓶颈：是 API 延迟高？还是工具执行慢？

---

## 24.10 小结

Claude Code 的性能优化是系统性的：

- **启动优化**：快速路径 + 并行预取 + 启动分析
- **API 优化**：提示缓存 + 流式处理 + 并行工具执行
- **文件系统优化**：读取缓存 + ripgrep + 智能排序
- **内存优化**：结果截断 + 自动压缩 + 循环缓冲区
- **渲染优化**：React Compiler + 增量更新
- **运行时优化**：Bun 的原生性能优势
- **构建优化**：Feature flags + 死代码消除

性能优化贯穿了整个系统设计，而不是事后的补丁。

---

*下一章：[Claude Code 的设计原则](../part9/25-design-principles.md)*
