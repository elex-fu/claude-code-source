# Claude Code 诊断追踪系统技术分析

## 1. 模块总结介绍

Diagnostic Tracking 系统是 Claude Code 的「代码健康监测器」，负责实时追踪 IDE 提供的代码诊断信息（错误、警告、提示）。该系统与 LSP/MCP 集成，在文件编辑前后捕获诊断基线，帮助 AI 了解代码变更引入的新问题。

核心特性：
- **诊断基线捕获**：编辑前记录文件诊断状态
- **增量检测**：识别新引入的错误/警告
- **双文件对比**：支持 file:// 和 _claude_fs_right: 协议对比
- **IDE 集成**：通过 MCP 调用 IDE 诊断服务

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                  Diagnostic Tracking System                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │  File Edit  │───▶│   Baseline  │───▶│  New Diagnostic │  │
│  │   Event     │    │   Capture   │    │    Detection    │  │
│  └─────────────┘    └─────────────┘    └─────────────────┘  │
│         │                                            │       │
│         ▼                                            ▼       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              MCP IDE Client                             ││
│  │  • getDiagnostics  • openFile  • Protocol handling     ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  State Management                                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ baseline: Map   │  │ rightFileState: │  │ timestamps: │ │
│  │ <path, diag[]>  │  │     Map         │  │    Map      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心数据类型

### 3.1 诊断结构

```typescript
interface Diagnostic {
  message: string
  severity: 'Error' | 'Warning' | 'Info' | 'Hint'
  range: {
    start: { line: number; character: number }
    end: { line: number; character: number }
  }
  source?: string    // 诊断来源（如 typescript、eslint）
  code?: string      // 错误代码
}

interface DiagnosticFile {
  uri: string              // 文件 URI（file:// 或 _claude_fs_right:）
  diagnostics: Diagnostic[]
}
```

### 3.2 服务状态

```typescript
class DiagnosticTrackingService {
  private baseline: Map<string, Diagnostic[]>     // 基线诊断
  private rightFileDiagnosticsState: Map<string, Diagnostic[]>  // 右侧文件状态
  private lastProcessedTimestamps: Map<string, number>         // 处理时间戳
  private mcpClient: MCPServerConnection | undefined           // MCP 连接
}
```

## 4. 诊断追踪流程

### 4.1 基线捕获流程

```
编辑开始前
    │
    ▼
┌─────────────────────┐
│ beforeFileEdited()  │
│ 编辑前基线捕获       │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ callIdeRpc()        │
│ getDiagnostics      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 存储基线到 baseline │
│ Map<path, diag[]>   │
└─────────────────────┘
```

### 4.2 新诊断检测流程

```typescript
async getNewDiagnostics(): Promise<DiagnosticFile[]> {
  // 1. 获取所有文件的当前诊断
  const allDiagnostics = await callIdeRpc('getDiagnostics', {})
  
  // 2. 筛选有基线的文件
  const filesWithBaselines = allDiagnostics
    .filter(file => this.baseline.has(normalizePath(file.uri)))
  
  // 3. 对比检测新诊断
  const newDiagnosticFiles: DiagnosticFile[] = []
  
  for (const file of filesWithBaselines) {
    const baselineDiags = this.baseline.get(normalizedPath) || []
    
    // 4. 智能选择对比源（file:// 或 _claude_fs_right:）
    const fileToUse = selectBestDiagnosticSource(file, claudeFsRightFile)
    
    // 5. 找出不在基线中的新诊断
    const newDiagnostics = fileToUse.diagnostics.filter(
      d => !baselineDiags.some(b => areDiagnosticsEqual(d, b))
    )
    
    if (newDiagnostics.length > 0) {
      newDiagnosticFiles.push({ uri: file.uri, diagnostics: newDiagnostics })
    }
    
    // 6. 更新基线为当前状态
    this.baseline.set(normalizedPath, fileToUse.diagnostics)
  }
  
  return newDiagnosticFiles
}
```

## 5. 双文件协议处理

### 5.1 协议对比策略

Claude Code 同时处理两种文件协议：
- `file://` - 实际文件系统
- `_claude_fs_right:` - IDE 右侧/预览文件

```typescript
// 智能选择诊断源
function selectBestDiagnosticSource(
  file: DiagnosticFile,                    // file:// 协议
  claudeFsRightFile?: DiagnosticFile       // _claude_fs_right: 协议
): DiagnosticFile {
  if (!claudeFsRightFile) {
    return file
  }
  
  const previousRightDiagnostics = rightFileDiagnosticsState.get(normalizedPath)
  
  // 使用 right 文件当：
  // 1. 从未获取过 right 文件诊断
  // 2. right 文件诊断刚刚发生变化
  if (!previousRightDiagnostics ||
      !areDiagnosticArraysEqual(previousRightDiagnostics, claudeFsRightFile.diagnostics)) {
    rightFileDiagnosticsState.set(normalizedPath, claudeFsRightFile.diagnostics)
    return claudeFsRightFile
  }
  
  return file
}
```

### 5.2 URI 规范化

```typescript
private normalizeFileUri(fileUri: string): string {
  // 移除协议前缀
  const protocolPrefixes = [
    'file://',
    '_claude_fs_right:',
    '_claude_fs_left:',
  ]
  
  let normalized = fileUri
  for (const prefix of protocolPrefixes) {
    if (fileUri.startsWith(prefix)) {
      normalized = fileUri.slice(prefix.length)
      break
    }
  }
  
  // 平台感知路径规范化（处理 Windows 大小写不敏感）
  return normalizePathForComparison(normalized)
}
```

## 6. 诊断对比算法

### 6.1 诊断相等性判断

```typescript
private areDiagnosticsEqual(a: Diagnostic, b: Diagnostic): boolean {
  return (
    a.message === b.message &&
    a.severity === b.severity &&
    a.source === b.source &&
    a.code === b.code &&
    a.range.start.line === b.range.start.line &&
    a.range.start.character === b.range.start.character &&
    a.range.end.line === b.range.end.line &&
    a.range.end.character === b.range.end.character
  )
}

private areDiagnosticArraysEqual(a: Diagnostic[], b: Diagnostic[]): boolean {
  if (a.length !== b.length) return false
  
  // 双向检查确保集合相等
  return (
    a.every(diagA => b.some(diagB => areDiagnosticsEqual(diagA, diagB))) &&
    b.every(diagB => a.some(diagA => areDiagnosticsEqual(diagA, diagB)))
  )
}
```

## 7. 与 Agent 循环集成

### 7.1 查询生命周期

```typescript
async handleQueryStart(clients: MCPServerConnection[]): Promise<void> {
  if (!this.initialized) {
    // 首次查询：初始化诊断追踪
    const connectedIdeClient = getConnectedIdeClient(clients)
    if (connectedIdeClient) {
      this.initialize(connectedIdeClient)
    }
  } else {
    // 后续查询：重置追踪状态
    this.reset()
  }
}
```

### 7.2 编辑前钩子

```typescript
// 在 FileEditTool 调用前自动触发
await diagnosticTracker.beforeFileEdited(filePath)

// 执行文件编辑...

// 在查询结束时获取新诊断
const newDiagnostics = await diagnosticTracker.getNewDiagnostics()
if (newDiagnostics.length > 0) {
  // 将新诊断反馈给 AI
  const summary = DiagnosticTrackingService.formatDiagnosticsSummary(newDiagnostics)
  // 添加到上下文...
}
```

## 8. 诊断格式化输出

### 8.1 格式化实现

```typescript
static formatDiagnosticsSummary(files: DiagnosticFile[]): string {
  const MAX_CHARS = 4000
  const truncationMarker = '…[truncated]'
  
  const result = files
    .map(file => {
      const filename = file.uri.split('/').pop() || file.uri
      const diagnostics = file.diagnostics
        .map(d => {
          const symbol = getSeveritySymbol(d.severity)
          return `  ${symbol} [Line ${d.range.start.line + 1}] ${d.message}`
        })
        .join('\n')
      
      return `${filename}:\n${diagnostics}`
    })
    .join('\n\n')
  
  // 截断处理
  if (result.length > MAX_CHARS) {
    return result.slice(0, MAX_CHARS - truncationMarker.length) + truncationMarker
  }
  
  return result
}

static getSeveritySymbol(severity: Diagnostic['severity']): string {
  return {
    Error: '✖',      // figures.cross
    Warning: '⚠',    // figures.warning
    Info: 'ℹ',       // figures.info
    Hint: '★',       // figures.star
  }[severity] || '•'
}
```

## 9. 容错与降级

### 9.1 静默失败策略

```typescript
async beforeFileEdited(filePath: string): Promise<void> {
  if (!this.initialized || !this.mcpClient) {
    return  // 静默返回，不阻塞编辑
  }
  
  try {
    const result = await callIdeRpc('getDiagnostics', { uri: `file://${filePath}` })
    // 处理结果...
  } catch (_error) {
    // 静默失败 - IDE 可能不支持诊断服务
  }
}
```

### 9.2 单例模式保证

```typescript
export class DiagnosticTrackingService {
  private static instance: DiagnosticTrackingService | undefined
  
  static getInstance(): DiagnosticTrackingService {
    if (!DiagnosticTrackingService.instance) {
      DiagnosticTrackingService.instance = new DiagnosticTrackingService()
    }
    return DiagnosticTrackingService.instance
  }
}

export const diagnosticTracker = DiagnosticTrackingService.getInstance()
```

## 10. 技术创新点

1. **双协议智能选择**：根据 _claude_fs_right 状态变化动态选择诊断源，确保获取最新分析结果

2. **增量追踪设计**：通过基线对比而非全量比较，减少噪声并聚焦于新引入的问题

3. **平台感知规范化**：统一处理不同协议前缀和 Windows 大小写不敏感路径

4. **零侵入集成**：诊断追踪自动在文件编辑前后触发，无需修改现有工具逻辑

5. **优雅降级**：IDE 不支持诊断服务时静默失败，不影响核心功能

---

诊断追踪系统通过基线对比、双协议处理、增量检测等设计，实现了代码编辑的实时质量反馈。理解其设计，对于构建 IDE 集成的 AI 工具具有参考价值。
