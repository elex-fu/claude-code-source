# Claude Code 系统架构总览

## 一、系统概述

Claude Code 是一个基于终端的 AI 编程助手，它通过深度集成 Anthropic 的 Claude 模型，为开发者提供自然语言驱动的代码编辑、文件操作、命令执行和项目管理能力。其架构设计融合了现代 CLI 工具、React 式 UI 框架、流式 API 处理和分布式 Agent 协调等多项先进技术。

---

## 二、整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           User Interface Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Terminal   │  │   Ink/React  │  │  Keybinding  │  │   Prompt     │ │
│  │   Input      │  │   Renderer   │  │   System     │  │   Input      │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘ │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          State Management Layer                         │
│                    (AppState + Zustand-style Store)                     │
│  · Message History  · Tool Permission Context  · MCP Client Registry   │
│  · Agent Registry   · Theme/Settings           · Notification Queue    │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Core Engine Layer                             │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Query Engine (query.ts)                      │  │
│  │   · AsyncGenerator-based streaming loop                         │  │
│  │   · Multi-turn conversation state machine                       │  │
│  │   · Auto-compact & context collapse                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────┐  ┌─────────────────────────────────────┐  │
│  │   Tool Execution System  │  │       Multi-Agent Coordination      │  │
│  │   · StreamingToolExecutor│  │   · Agent Tool (sub-agents)         │  │
│  │   · Concurrent/Serial    │  │   · Task Scheduling                 │  │
│  │   · Permission Control   │  │   · Inter-agent Communication       │  │
│  └──────────────────────────┘  └─────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Integration Layer                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐ │
│  │   MCP System     │  │   Skill System   │  │   Memory System        │ │
│  │   · Server Conn  │  │   · Bundled      │  │   · Session Memory     │ │
│  │   · OAuth/Auth   │  │   · Custom       │  │   · CLAUDE.md          │ │
│  │   · Tool Mapping │  │   · Auto-load    │  │   · Attachments        │ │
│  └──────────────────┘  └──────────────────┘  └────────────────────────┘ │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐ │
│  │   API Bridge     │  │   Hooks System   │  │   Vim Mode             │ │
│  │   · Remote CCR   │  │   · Stop Hooks   │  │   · Keybindings        │ │
│  │   · Desktop SDK  │  │   · Post-samp.   │  │   · Motions/Operators  │ │
│  └──────────────────┘  └──────────────────┘  └────────────────────────┘ │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          External Services                              │
│     Anthropic API    ·    MCP Servers    ·    Git/GitHub    ·    LSP   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三、核心子系统详解

### 3.1 前端渲染层 (Ink + React)

Claude Code 使用 **Ink**（一个基于 React 的终端 UI 框架）构建其交互界面。这是其最独特的技术选择之一：

- **自定义 Reconciler**: `ink/reconciler.ts` 实现了一个完整的 React reconciler，针对终端输出优化
- **Yoga Layout Engine**: 使用 Facebook 的 Yoga 引擎进行 CSS-like 布局计算（Flexbox）
- **组件化架构**: Box、Text、Spinner、ScrollBox 等终端组件
- **事件系统**: 完整的键盘事件、鼠标点击事件处理

这种架构让 Claude Code 能够呈现复杂的交互式 UI（如权限确认对话框、设置面板、差异对比），同时保持在终端中运行。

### 3.2 状态管理层 (AppState)

采用类似 Zustand 的轻量级状态管理：

```typescript
// 核心状态结构
interface AppState {
  messages: Message[]              // 对话历史
  toolPermissionContext: {...}     // 权限上下文
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
  }
  agents: AgentDefinitionsResult   // Agent 定义
  theme: Theme                     // 主题配置
  // ... 更多
}
```

- **跨层共享**: CLI 逻辑和 UI 组件共享同一状态
- **持久化**: 支持会话恢复（resume）
- **派生状态**: 通过 hooks（如 `useAppState(selector)`）实现精确订阅

### 3.3 Query Engine 核心循环

这是 Claude Code 的心脏，一个基于 AsyncGenerator 的复杂状态机：

1. **上下文准备**: 注入 CLAUDE.md、git status、system prompt
2. **流式 API 调用**: 通过 Anthropic SDK 发起 SSE 请求
3. **实时工具检测**: 在模型输出流中检测 tool_use 块
4. **并发工具执行**: StreamingToolExecutor 立即启动并发安全的工具
5. **Stop Hooks**: 在 turn 结束时执行用户自定义脚本
6. **循环或终止**: 根据模型响应决定继续下一轮或结束

关键创新：**流式工具执行**让工具在模型还没说完话时就开始运行，极大降低延迟。

### 3.4 工具系统

统一的工具抽象支持 40+ 种操作：

| 类别 | 工具示例 | 特点 |
|------|----------|------|
| 文件操作 | FileRead, FileEdit, FileWrite, Glob, Grep | 带权限控制 |
| 系统命令 | Bash, PowerShell | 并发安全检测 |
| 代码智能 | LSP, LSPTool | 语言服务器协议 |
| 网络 | WebFetch, WebSearch | 结果截断保护 |
| Agent | AgentTool | 递归创建子 Agent |
| MCP | MCPTool | 外部服务集成 |
| 任务 | TaskCreate, TaskUpdate, TaskOutput | 任务生命周期 |

### 3.5 MCP 集成系统

Model Context Protocol 是 Claude Code 的插件架构：

- **多传输支持**: stdio、SSE、StreamableHTTP、WebSocket
- **OAuth 认证**: 完整的授权码流程 + Token 刷新
- **动态发现**: 连接后自动拉取 tools、resources、prompts
- **结果治理**: 大结果截断、二进制持久化、图像缩放

### 3.6 多级上下文压缩

为应对长对话的 token 限制，实现了四层压缩策略：

1. **Snip Compact**: 移除被显式引用之外的老旧内容
2. **Micro Compact**: 单 turn 内小工具链快速摘要
3. **Auto Compact**: Fork 专门 Agent 生成历史摘要
4. **Context Collapse**: 阶段性对话折叠存储

### 3.7 多 Agent 协调

支持复杂的 Agent 协作场景：

- **Agent as Tool**: 将 Agent 抽象为可调用的 Tool
- **嵌套层级**: 支持父子孙多级嵌套
- **本地/远程混合**: 透明调度本地和远程 Agent
- **Mailbox 通信**: 异步消息队列解耦父子 Agent

---

## 四、数据流向

### 4.1 单次用户请求流程

```
用户输入 → Prompt Input Component
    ↓
AppState.messages 更新
    ↓
QueryEngine.submitMessage() / query()
    ↓
上下文组装 (Context + Messages + Attachments)
    ↓
Auto-compact 检查 (若 token 超限)
    ↓
Anthropic API 流式请求
    ↓
流式响应解析 (text | tool_use | thinking)
    ↓
StreamingToolExecutor 执行工具
    ↓
工具结果归队 → 更新 messages
    ↓
Stop Hooks 执行
    ↓
新一轮循环或结束
    ↓
UI 渲染最终响应
```

### 4.2 工具调用详细流程

```
模型输出 tool_use JSON
    ↓
解析 input 参数
    ↓
isConcurrencySafe() 检测
    ↓
权限检查 (canUseTool)
    ↓
   ├─ 拒绝 → 生成拒绝消息
   └─ 允许 → 执行工具 call()
              ↓
        进度消息流 (progress)
              ↓
        结果格式化
              ↓
        tool_result 块生成
              ↓
        重新进入 query loop
```

---

## 五、关键技术选型

| 领域 | 技术 | 选型理由 |
|------|------|----------|
| 运行时 | Bun | 高性能 JS 运行时，内置打包 |
| UI 框架 | Ink (React) | 终端内的 React 体验 |
| 状态管理 | Zustand-like | 轻量、跨层共享 |
| 布局引擎 | Yoga | 成熟的 Flexbox 实现 |
| API SDK | @anthropic-ai/sdk | 官方 SDK，SSE 支持 |
| 校验 | Zod v4 | 类型安全 + 运行时校验 |
| 终端处理 | xterm.js concepts | ANSI、光标、事件 |
| 打包 | bun:bundle | 单文件可执行 |

---

## 六、架构亮点

### 6.1 终端原生设计

不同于基于 Web 的 AI 助手，Claude Code 完全在终端中运行：
- 直接访问本地 shell、文件系统、git
- 与开发者现有工作流无缝集成
- 通过 SSH 可在远程服务器运行

### 6.2 流式优先

从 API 到 UI 的全链路流式处理：
- 模型输出流式解析
- 工具流式执行
- UI 增量渲染
- 零等待体验

### 6.3 安全分层

多层权限控制：
- 全局权限模式 (default/auto/plan/accept/deny)
- 规则系统 (alwaysAllow/alwaysDeny)
- 自动分类器 (yoloClassifier)
- 显式确认对话框

### 6.4 可扩展性

- **MCP**: 动态接入外部服务
- **Skills**: 自定义命令和提示词
- **Plugins**: 运行时扩展
- **Hooks**: 用户自定义脚本集成

---

## 七、模块依赖关系

```
main.tsx (入口)
    ├── CLI/Commands
    ├── Bootstrap (init)
    │       └── AppStateProvider
    │               ├── QueryEngine
    │               │       ├── Tool System
    │               │       │       ├── MCP System
    │               │       │       ├── Agent System
    │               │       │       └── Skill System
    │               │       ├── Prompt System
    │               │       │       └── Memory System
    │               │       └── Token Management
    │               ├── Ink UI
    │               │       ├── Components
    │               │       └── Hooks
    │               └── Hooks System
    └── Bridge (远程模式)
```

---

## 八、未来演进方向

1. **更强的 Agent 协作**: 从层级调用向网状协作演进
2. **智能上下文管理**: 基于注意力的动态上下文加载
3. **可视化增强**: 终端内的 richer rendering（图像、图表）
4. **分布式执行**: 更强的 remote/local 混合调度
5. **垂直领域扩展**: 针对特定语言/框架的深度集成

---

## 九、总结

Claude Code 的架构是一个精心设计的分层系统：

- **底层**: 高效的终端 UI 引擎（Ink）
- **核心**: 流式 Query Engine 驱动的 Agent 循环
- **扩展**: MCP + Skills + Hooks 构成的可插拔生态
- **体验**: 原生终端集成 + 现代交互设计

这种架构使其在性能、可扩展性和用户体验之间取得了优秀的平衡，代表了 AI 编程助手类产品的工程巅峰。
