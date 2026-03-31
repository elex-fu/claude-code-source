# 第 1 章：Claude Code 是什么

> "It's not a chatbot. It's an agent."
> —— Anthropic 工程师

---

## 1.1 一句话解释

Claude Code 是一个运行在终端里的 AI 编程助手，它能**真正操作你的代码库**——读文件、写文件、执行命令、搜索代码、调用 API——而不只是给你看代码片段。

这个区别很关键。

传统的 AI 编程助手（比如早期的 GitHub Copilot）是**补全工具**：你写代码，它帮你补全下一行。Claude Code 是**执行工具**：你描述目标，它帮你完成整个任务。

---

## 1.2 一个具体的例子

假设你说："帮我把项目里所有用 `var` 声明的变量改成 `const` 或 `let`，并跑一遍测试确认没有问题。"

一个普通的 AI 聊天工具会：
- 给你一段解释
- 也许给你一个正则表达式
- 让你自己去执行

Claude Code 会：
1. 用 `GlobTool` 找到所有 `.js` / `.ts` 文件
2. 用 `GrepTool` 搜索所有 `var` 声明
3. 用 `FileEditTool` 逐个修改
4. 用 `BashTool` 执行 `npm test`
5. 如果测试失败，分析错误，回滚或修复
6. 告诉你结果

这就是 Agent 和聊天机器人的本质区别：**能不能真正执行动作**。

---

## 1.3 Claude Code 的核心能力

### 文件操作
- 读取任意文件（`FileReadTool`）
- 精确编辑文件（`FileEditTool`）——不是替换整个文件，而是精确的字符串替换
- 创建新文件（`FileWriteTool`）
- 搜索文件名（`GlobTool`）
- 搜索文件内容（`GrepTool`）

### 命令执行
- 执行任意 Shell 命令（`BashTool`）
- 执行 PowerShell（`PowerShellTool`）
- 交互式 REPL（`REPLTool`）

### 代码智能
- LSP 集成（`LSPTool`）——跳转定义、查找引用、悬停文档
- Jupyter Notebook 编辑（`NotebookEditTool`）

### 网络能力
- 抓取网页（`WebFetchTool`）
- 搜索互联网（`WebSearchTool`）

### 任务管理
- 创建后台任务（`TaskCreateTool`）
- 查询任务状态（`TaskGetTool`、`TaskListTool`、`TaskOutputTool`）
- 停止任务（`TaskStopTool`）

### 多代理协作
- 启动子代理（`AgentTool`）
- 创建协作团队（`TeamCreateTool`）
- 代理间通信（`SendMessageTool`）

### 扩展能力
- MCP 工具调用（`MCPTool`）
- Skills 执行（`SkillTool`）
- 定时任务（`ScheduleCronTool`）

---

## 1.4 核心架构一览

![Claude Code 核心架构](../diagrams/claude-code-architecture.svg)

Claude Code 的架构分为几个关键层次：

**用户层**：通过 CLI、Desktop 或 Web 界面与系统交互。

**QueryEngine**：核心引擎，负责消息循环、Agent 执行和流式响应处理。

**工具系统**：提供原子操作能力（文件读写、命令执行、代码搜索等）。

**权限系统**：五层权限架构，从会话模式到具体路径/命令级别的细粒度控制。

**MCP 客户端**：连接外部 MCP 服务器（GitHub、Slack、Database 等），扩展工具能力。

**状态管理**：Bootstrap State（全局单例）和 AppState（会话级响应式状态）。

**Context Engineering**：系统提示构建、Memory 系统、CLAUDE.md 项目上下文、Auto-Compact 压缩。

这些组件协同工作，让 Claude Code 能够理解你的意图、安全地执行操作、管理长期上下文。

---

## 1.5 它长什么样

Claude Code 是一个命令行工具，安装后通过 `claude` 命令启动：

```bash
# 安装
npm install -g @anthropic-ai/claude-code

# 启动交互模式
claude

# 直接执行任务
claude "帮我写一个 README"

# 在特定目录工作
claude --cwd /path/to/project
```

启动后，你会看到一个简洁的终端界面：

```
╭─────────────────────────────────────────╮
│ Claude Code                             │
│ ✓ Connected to claude-sonnet-4-6        │
╰─────────────────────────────────────────╯

> _
```

就这样。没有复杂的配置，没有 GUI，只有一个对话框。

---

## 1.5 它和其他工具的区别

| 工具 | 类型 | 能执行动作？ | 能访问文件系统？ | 能运行命令？ |
|------|------|------------|----------------|------------|
| GitHub Copilot | 补全工具 | 否 | 否（只读当前文件） | 否 |
| ChatGPT | 聊天工具 | 否 | 否 | 否 |
| Cursor | IDE 集成 | 部分 | 是 | 部分 |
| **Claude Code** | **Agent** | **是** | **是** | **是** |

---

## 1.6 它的边界

Claude Code 不是万能的。它有几个重要的边界：

**权限边界**：默认情况下，执行危险操作（删除文件、运行未知脚本）前会询问你。你可以配置权限级别。

**上下文边界**：每次对话有 token 限制。Claude Code 有自动压缩机制，但超长的对话仍然会丢失早期信息。

**能力边界**：它依赖 Claude 模型的能力。模型不会的事，工具也做不到。

**网络边界**：默认情况下，它只能访问你明确允许的网络资源。

---

## 1.7 小结

Claude Code 是：
- 一个**终端 CLI 工具**
- 一个**AI Agent**，能真正执行动作
- 一个**工程平台**，有完整的工具系统、权限模型、扩展机制

Claude Code 不是：
- 一个聊天机器人
- 一个代码补全工具
- 一个 IDE 替代品

理解了这个定位，你就理解了为什么它的设计如此复杂——因为它要解决的问题，本质上是**如何让 AI 安全、可靠地操作真实的计算机系统**。

---

*下一章：[快速上手](./02-quickstart.md)*
