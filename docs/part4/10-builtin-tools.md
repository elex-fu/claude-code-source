# 第 10 章：43 个内置工具全览

> 工具集的设计反映了系统对"AI 能做什么"的判断。

---

## 10.1 工具分类概览

Claude Code 内置 43 个工具，按功能分为 8 大类：

| 类别 | 工具数 | 核心职责 |
|------|--------|---------|
| 文件操作 | 5 | 读写文件、搜索内容和路径 |
| Shell 执行 | 3 | 执行系统命令 |
| 代码智能 | 2 | LSP 集成、Notebook 编辑 |
| 任务管理 | 6 | 后台任务的完整生命周期 |
| 多代理协作 | 4 | 子代理、团队、消息传递 |
| 计划模式 | 4 | 进入/退出计划和 Worktree |
| MCP 集成 | 4 | MCP 协议工具和资源 |
| 扩展与其他 | 15 | Skills、定时任务、Web、配置等 |

---

## 10.2 文件操作工具

### FileReadTool
**职责**：读取文件内容

```typescript
// 输入
{
  file_path: string      // 文件路径
  offset?: number        // 起始行（用于大文件分段读取）
  limit?: number         // 读取行数
}

// 输出：文件内容（带行号）
```

**设计亮点**：支持分段读取（`offset` + `limit`），避免一次性读取大文件导致 token 超限。Claude 可以先读前 100 行，根据需要再读后续内容。

---

### FileEditTool
**职责**：精确字符串替换

```typescript
// 输入
{
  file_path: string      // 文件路径
  old_string: string     // 要替换的内容（必须唯一）
  new_string: string     // 替换后的内容
}
```

**设计亮点**：要求 `old_string` 在文件中唯一存在。这个约束看起来严格，实际上是安全保障——防止 Claude 意外修改了错误的位置。如果 `old_string` 不唯一，工具会报错，Claude 需要提供更多上下文来精确定位。

---

### FileWriteTool
**职责**：创建或完全覆写文件

```typescript
// 输入
{
  file_path: string      // 文件路径
  content: string        // 文件内容
}
```

**与 FileEditTool 的区别**：`FileWriteTool` 覆写整个文件，`FileEditTool` 只修改特定内容。对于新文件用 `FileWriteTool`，对于修改现有文件优先用 `FileEditTool`（更安全，更精确）。

---

### GlobTool
**职责**：按模式搜索文件路径

```typescript
// 输入
{
  pattern: string        // glob 模式，如 "**/*.ts"
  path?: string          // 搜索根目录
}

// 输出：匹配的文件路径列表（按修改时间排序）
```

**设计亮点**：结果按修改时间排序，最近修改的文件排在前面。这对 Claude 很有用——通常最近修改的文件是最相关的。

---

### GrepTool
**职责**：搜索文件内容

```typescript
// 输入
{
  pattern: string        // 正则表达式
  path?: string          // 搜索目录
  glob?: string          // 文件过滤模式
  output_mode?: 'content' | 'files_with_matches' | 'count'
  context?: number       // 显示匹配行前后的行数
}
```

**设计亮点**：基于 ripgrep，速度极快。支持三种输出模式，Claude 可以根据需要选择：只需要知道哪些文件有匹配（`files_with_matches`），还是需要看具体内容（`content`）。

---

## 10.3 Shell 执行工具

### BashTool
**职责**：执行 Shell 命令

```typescript
// 输入
{
  command: string        // Shell 命令
  timeout?: number       // 超时时间（毫秒）
  description?: string   // 命令描述（显示给用户）
}

// 输出：stdout + stderr + 退出码
```

**安全设计**：BashTool 是最危险的工具，因为它可以执行任意命令。Claude Code 对 BashTool 有特殊的安全处理：
- 默认需要用户确认
- 有命令安全分析（检测危险命令如 `rm -rf`）
- 支持超时防止命令挂起
- 在沙箱模式下受限

---

### PowerShellTool
**职责**：执行 PowerShell 命令（Windows）

与 BashTool 类似，但针对 Windows 环境。

---

### REPLTool
**职责**：交互式 REPL 执行

```typescript
// 输入
{
  code: string           // 要执行的代码
  language?: string      // 语言（python、node 等）
}
```

**与 BashTool 的区别**：REPLTool 维护一个持久的 REPL 会话，变量在多次调用之间保持。适合需要多步骤计算的场景。

---

## 10.4 代码智能工具

### LSPTool
**职责**：Language Server Protocol 集成

```typescript
// 支持的操作
type LSPOperation =
  | 'goToDefinition'      // 跳转到定义
  | 'findReferences'      // 查找引用
  | 'hover'               // 悬停文档
  | 'documentSymbol'      // 文档符号列表
  | 'workspaceSymbol'     // 工作区符号搜索
  | 'goToImplementation'  // 跳转到实现
  | 'prepareCallHierarchy'// 调用层次
  | 'incomingCalls'       // 入调用
  | 'outgoingCalls'       // 出调用
```

**设计亮点**：LSP 集成让 Claude 能做真正的代码理解，而不只是文本搜索。"找出所有调用 `getUserById` 的地方"用 `findReferences` 比用 `GrepTool` 更准确（能处理重命名、别名等情况）。

---

### NotebookEditTool
**职责**：编辑 Jupyter Notebook

```typescript
// 输入
{
  notebook_path: string  // Notebook 路径
  cell_number?: number   // 目标 cell（0-indexed）
  new_source: string     // 新的 cell 内容
  edit_mode?: 'replace' | 'insert' | 'delete'
  cell_type?: 'code' | 'markdown'
}
```

---

## 10.5 任务管理工具

任务管理工具是 Claude Code 后台任务系统的接口：

| 工具 | 职责 |
|------|------|
| `TaskCreateTool` | 创建后台任务（bash 命令或子代理） |
| `TaskGetTool` | 获取单个任务状态 |
| `TaskListTool` | 列出所有任务 |
| `TaskOutputTool` | 读取任务输出 |
| `TaskStopTool` | 停止任务 |
| `TaskUpdateTool` | 更新任务描述 |

**使用场景**：长时间运行的任务（如构建、测试、数据处理）可以作为后台任务执行，Claude 可以继续处理其他事情，定期检查任务状态。

---

## 10.6 多代理协作工具

### AgentTool
**职责**：启动子代理

```typescript
// 输入
{
  description: string    // 子代理的任务描述
  prompt: string         // 子代理的初始提示
  subagent_type?: string // 代理类型（general-purpose、Explore 等）
  isolation?: 'worktree' // 是否在独立 worktree 中运行
  model?: string         // 子代理使用的模型
  run_in_background?: boolean // 是否后台运行
}
```

**设计亮点**：子代理有自己独立的工具集、上下文和执行环境。父代理可以并行启动多个子代理，实现真正的并行处理。

---

### TeamCreateTool / TeamDeleteTool
**职责**：创建/删除协作代理团队

团队是多个代理的集合，可以通过 `SendMessageTool` 相互通信。

---

### SendMessageTool
**职责**：向其他代理发送消息

实现代理间的异步通信，是多代理协作的基础。

---

## 10.7 计划模式工具

### EnterPlanModeTool / ExitPlanModeTool
**职责**：进入/退出计划模式

计划模式下，Claude 只能生成计划，不能执行工具。用于需要用户审查计划再执行的场景。

```
用户：帮我重构整个认证模块

Claude（计划模式）：
  我的计划是：
  1. 分析现有认证流程
  2. 设计新的接口
  3. 逐步迁移

  是否批准执行？

用户：批准

Claude（执行模式）：开始执行...
```

---

### EnterWorktreeTool / ExitWorktreeTool
**职责**：进入/退出 Git Worktree

在独立的 worktree 中工作，不影响主分支。适合实验性修改或并行开发。

---

## 10.8 MCP 集成工具

| 工具 | 职责 |
|------|------|
| `MCPTool` | 调用 MCP 服务器提供的工具 |
| `McpAuthTool` | MCP 服务器认证 |
| `ListMcpResourcesTool` | 列出 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，允许外部服务器向 Claude 提供工具和资源。详见第 19 章。

---

## 10.9 其他重要工具

### WebFetchTool
**职责**：抓取网页内容

```typescript
// 输入
{
  url: string            // URL
  prompt: string         // 从页面提取什么信息
}
```

**设计亮点**：不是简单返回 HTML，而是用 AI 处理页面内容，提取用户需要的信息。

---

### WebSearchTool
**职责**：搜索互联网

```typescript
// 输入
{
  query: string          // 搜索查询
  allowed_domains?: string[]  // 只搜索这些域名
  blocked_domains?: string[]  // 排除这些域名
}
```

---

### TodoWriteTool
**职责**：管理任务列表

```typescript
// 输入
{
  todos: Array<{
    content: string
    status: 'pending' | 'in_progress' | 'completed'
    activeForm: string
  }>
}
```

**设计亮点**：TodoWriteTool 是 Claude 的"工作记忆"。对于复杂的多步骤任务，Claude 会用 TodoWriteTool 记录进度，确保不遗漏任何步骤。

---

### AskUserQuestionTool
**职责**：向用户提问

```typescript
// 输入
{
  questions: Array<{
    question: string
    header: string
    options: Array<{ label: string; description: string }>
    multiSelect?: boolean
  }>
}
```

**设计亮点**：结构化的问题格式，让用户可以通过选项回答，而不只是自由文本。这减少了歧义，提高了交互效率。

---

### SkillTool
**职责**：执行预定义的 Skill

Skills 是可复用的提示模板，详见第 20 章。

---

### ScheduleCronTool / RemoteTriggerTool
**职责**：定时任务和远程触发

允许 Claude 创建定时执行的任务，或触发远程 Agent 执行。

---

## 10.10 工具的演化

Claude Code 的工具集不是一成不变的。从源码的迁移文件（`src/migrations/`）可以看到工具的演化历史：

- 模型从 Opus 迁移到 Sonnet 4.5，再到 Sonnet 4.6
- 权限系统的多次重构
- 新工具的持续添加

这种演化能力来自工具系统的良好抽象：添加新工具只需要实现 `Tool` 接口，不需要修改核心系统。

---

## 10.11 小结

43 个内置工具覆盖了软件开发的完整工作流：

- **探索**：GlobTool、GrepTool、LSPTool
- **理解**：FileReadTool、WebFetchTool
- **修改**：FileEditTool、FileWriteTool
- **执行**：BashTool、REPLTool
- **协作**：AgentTool、TeamCreateTool、SendMessageTool
- **管理**：TaskCreateTool、TodoWriteTool、ScheduleCronTool
- **扩展**：MCPTool、SkillTool

这个工具集的设计原则是：**每个工具做一件事，Claude 负责编排**。

---

*下一章：[工具权限模型](./11-tool-permission.md)*
