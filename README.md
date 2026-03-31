# claude-code-source

[![linux.do](https://img.shields.io/badge/linux.do-huo0-blue?logo=linux&logoColor=white)](https://linux.do)

> [!WARNING]
> This repository is **unofficial** and is reconstructed from the public npm package and source map analysis, **for research purposes only**.
> It does **not** represent the original internal development repository structure.
>
> 本仓库为**非官方**整理版，基于公开 npm 发布包与 source map 分析还原，**仅供研究使用**。
> **不代表**官方原始内部开发仓库结构。
> 一切基于L站"飘然与我同"的情报提供

## 概述

Claude Code 是 Anthropic 推出的命令行 AI 编程助手，代表了当前 AI Agent 工程实现的最高水平。本仓库通过 npm 发布包（`@anthropic-ai/claude-code`）内附带的 source map（`cli.js.map`）还原的 TypeScript 源码，版本为 `2.1.88`。

从工程角度看，Claude Code 的代码量（约 50 万行 TypeScript）和架构复杂度（涵盖工具系统、权限模型、状态管理、多代理协作等模块）使其成为 AI Agent 工程化的典范。

## 来源

- npm 包：[@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- 还原版本：`2.1.88`
- 还原文件数：**4756 个**（含 1884 个 `.ts`/`.tsx` 源文件）
- 还原方式：提取 `cli.js.map` 中的 `sourcesContent` 字段

## 还原方法

从 npm 包还原原始源码的步骤：

```bash
# 1. 从 npm 下载包
npm pack @anthropic-ai/claude-code --registry https://registry.npmjs.org

# 2. 解压
tar xzf anthropic-ai-claude-code-2.1.88.tgz

# 3. 解析 cli.js.map，将 sourcesContent 按原始路径写出
node -e "
const fs = require('fs');
const path = require('path');
const map = JSON.parse(fs.readFileSync('package/cli.js.map', 'utf8'));
const outDir = './claude-code-source';
for (let i = 0; i < map.sources.length; i++) {
  const content = map.sourcesContent[i];
  if (!content) continue;
  let relPath = map.sources[i];
  while (relPath.startsWith('../')) relPath = relPath.slice(3);
  const outPath = path.join(outDir, relPath);
  fs.mkdirSync(path.dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, content);
}
"
```

Source map 中包含 4756 个源文件及其完整源码（sourcesContent），可以无损还原所有 TypeScript/TSX 原始代码。

## 系统架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    用户交互层 (CLI/TUI)                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │  Ink/React  │  │  输入处理   │  │  消息渲染   │  │  权限对话框 │  │  Vim模式  │  │
│  │  组件系统   │  │  InputBox   │  │  Messages   │  │  Permission │  │  快捷键   │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  状态管理层 (State)                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         AppState (React 状态树)                               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Messages │ │  Tasks   │ │Permissions│ │   UI     │ │UserInput │           │   │
│  │  │ 消息历史 │ │ 任务列表 │ │ 权限状态  │ │ UI状态   │ │ 用户输入 │           │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              核心引擎层 (QueryEngine)                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         PDAOL Agent 循环                                      │   │
│  │                                                                              │   │
│  │   用户输入 ──▶ [感知]构建系统提示 ──▶ [决策]调用Claude API                     │   │
│  │                              │                        │                      │   │
│  │                              ▼                        ▼                      │   │
│  │   最终结果 ◀── [循环]继续? ◀── [观察]工具结果 ◀── [行动]执行工具               │   │
│  │                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               能力层 (Tools & Services)                               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────┐  │
│  │   内置工具   │ │   MCP集成    │ │  多代理系统  │ │   任务系统   │ │ LSP集成  │  │
│  │  43个工具    │ │ 外部工具扩展 │ │ AgentTool    │ │ 后台执行    │ │代码智能  │  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              基础设施层 (Infrastructure)                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ 权限系统 │ │ 压缩系统 │ │ 记忆系统 │ │ 成本追踪 │ │ 可观测性 │ │ 桥接层   │      │
│  │5层安全   │ │AutoCompact│ │Memory   │ │CostTrack │ │OpenTelemetry│ │Remote  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 代码结构

```
source-code/
├── src/                  # 核心源码（1902 个文件）
│   ├── main.tsx          # 应用入口
│   ├── Tool.ts           # 工具基类
│   ├── Task.ts           # 任务管理
│   ├── QueryEngine.ts    # 查询引擎
│   ├── commands.ts       # 命令注册
│   ├── tools.ts          # 工具注册
│   ├── assistant/        # 会话历史管理
│   ├── bootstrap/        # 启动初始化
│   ├── bridge/           # 桥接层（31 个文件）
│   ├── buddy/            # 子代理系统（6）
│   ├── cli/              # CLI 参数解析与入口（19）
│   ├── commands/         # 斜杠命令实现（207）
│   ├── components/       # 终端 UI 组件，基于 Ink（389）
│   ├── constants/        # 共享常量（21）
│   ├── context/          # 上下文管理（9）
│   ├── coordinator/      # Agent 协调器（1）
│   ├── entrypoints/      # 各类入口点（8）
│   ├── hooks/            # 生命周期钩子（104）
│   ├── ink/              # 自定义 Ink 终端渲染引擎（96）
│   ├── keybindings/      # 快捷键管理（14）
│   ├── memdir/           # 记忆目录系统（8）
│   ├── migrations/       # 数据迁移（11）
│   ├── moreright/        # 权限系统（1）
│   ├── native-ts/        # 原生 TS 工具（4）
│   ├── outputStyles/     # 输出格式化（1）
│   ├── plugins/          # 插件系统（2）
│   ├── query/            # 查询处理（4）
│   ├── remote/           # 远程执行（4）
│   ├── schemas/          # 数据模式定义（1）
│   ├── screens/          # 屏幕视图（3）
│   ├── server/           # Server 模式（3）
│   ├── services/         # 核心服务（130）
│   ├── skills/           # 技能系统（20）
│   ├── state/            # 状态管理（6）
│   ├── tasks/            # 任务执行（12）
│   ├── tools/            # 工具实现（184）
│   ├── types/            # TypeScript 类型定义（11）
│   ├── upstreamproxy/    # 上游代理支持（2）
│   ├── utils/            # 工具函数（564）
│   ├── vim/              # Vim 模式（5）
│   └── voice/            # 语音输入（1）
├── vendor/               # 内部 vendor 代码（4 个文件）
│   ├── modifiers-napi-src/   # 按键修饰符原生模块
│   ├── url-handler-src/      # URL 处理
│   ├── audio-capture-src/    # 音频采集
│   └── image-processor-src/  # 图片处理
└── node_modules/         # 打包的第三方依赖（2850 个文件）
```

### 核心模块说明

| 模块 | 文件数 | 说明 |
|------|--------|------|
| utils/ | 564 | 工具函数集 — 文件 I/O、Git 操作、权限检查、Diff 处理等 |
| components/ | 389 | 终端 UI 组件，基于 Ink（React 的 CLI 版本）构建 |
| commands/ | 207 | 斜杠命令实现，如 /commit、/review 等 |
| tools/ | 184 | Agent 工具实现 — Read、Write、Edit、Bash、Glob、Grep 等 |
| services/ | 130 | 核心服务 — API 客户端、认证、配置、会话管理等 |
| hooks/ | 104 | 生命周期钩子 — 工具执行前后的拦截与权限控制 |
| ink/ | 96 | 自研 Ink 渲染引擎，包含布局、焦点管理、渲染优化 |
| bridge/ | 31 | 桥接层 — IDE 扩展与 CLI 之间的通信 |
| skills/ | 20 | 技能加载与执行系统 |
| cli/ | 19 | CLI 参数解析与启动逻辑 |
| keybindings/ | 14 | 键盘快捷键绑定与自定义 |
| tasks/ | 12 | 后台任务与定时任务管理 |

## 核心技术特点

### 1. PDAOL Agent 循环

Claude Code 采用经典的 **Perception-Decision-Action-Observation-Loop** 架构：

```
用户输入 ──▶ [感知]构建系统提示 ──▶ [决策]调用Claude API
                   │                        │
                   ▼                        ▼
最终结果 ◀── [循环]继续? ◀── [观察]工具结果 ◀── [行动]执行工具
```

核心代码位于 `src/query.ts`，通过 `AsyncGenerator` 实现流式处理，支持背压控制和取消信号。

### 2. 统一工具系统（Tool.ts）

所有工具遵循统一接口设计：

| 方法 | 作用 |
|------|------|
| `call(input, context, canUseTool, onProgress)` | 执行工具逻辑 |
| `prompt(options)` | 生成系统提示片段 |
| `checkPermissions()` | 权限检查 |
| `renderToolUseMessage()` / `renderToolResultMessage()` | UI 渲染 |
| `isConcurrencySafe()` / `isReadOnly()` / `isDestructive()` | 元信息 |

### 3. 43个内置工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | FileReadTool, FileEditTool, FileWriteTool | 文件读写、内容替换 |
| **命令执行** | BashTool, PowerShellTool | Shell 命令执行（含安全分析） |
| **搜索查询** | GrepTool, GlobTool | 内容/文件搜索（ripgrep 包装） |
| **代码智能** | LSPTool | LSP 客户端（定义跳转、引用查找） |
| **多代理** | AgentTool, TeamCreateTool | 子代理创建与管理 |
| **任务管理** | TaskCreateTool, TaskUpdateTool, TaskStopTool | 后台任务调度 |
| **MCP集成** | MCPTool, ReadMcpResource | 外部工具调用（进程隔离） |
| **网络服务** | WebSearchTool, WebFetchTool | 网络请求（超时控制） |
| **系统控制** | TodoWriteTool, SleepTool, EnterPlanMode | 任务追踪与模式切换 |

### 4. 5层权限模型

| 层级 | 机制 | 说明 |
|------|------|------|
| L1 | 自动允许 | 只读文件操作（如 Read） |
| L2 | 目录隔离 | 自动允许项目目录内，禁止外部 |
| L3 | 模式区分 | ask/notify 模式：阻塞/非阻塞确认 |
| L4 | 命令分析 | Bash 命令正则匹配安全策略 |
| L5 | 破坏性确认 | 修改 Git 追踪文件需确认 |

### 5. 流式工具执行器（StreamingToolExecutor）

- **并发控制**：安全工具并行执行，非安全工具串行
- **进度流**：实时产出执行进度和结果
- **取消处理**：支持中途取消未完成任务
- **错误级联**：单工具失败不影响其他工具

### 6. 状态管理架构

| 层级 | 实现 | 作用 |
|------|------|------|
| UI 层 | `useAppState(selector)` | React 组件订阅状态切片 |
| React 层 | `AppState.tsx` | 状态树，使用 `useSyncExternalStore` |
| 全局层 | `bootstrap/state.ts` | Signal 信号系统，跨会话持久 |

### 7. MCP 协议集成

支持通过 MCP（Model Context Protocol）动态扩展工具：

```
settings.json ──▶ MCPClient ──▶ MCPTransport ──▶ MCP Server
                                      │
                                      ▼
                              wrapMCPTool() ──▶ 注册到 tools 数组
```

### 8. 自动压缩系统（AutoCompact）

当上下文窗口接近阈值（默认85%）时自动触发：

```
Token计数 ──▶ 触发判断 ──▶ 边界检测 ──▶ Claude API生成摘要 ──▶ 替换历史
```

- 熔断保护：最多连续压缩失败次数限制
- 安全边界：确保不在工具调用中间压缩

## 设计哲学

1. **Agent 优先**：不是代码补全工具，而是能执行命令、修改文件、调用 API 的真正 AI Agent
2. **安全分层**：5层权限模型确保安全边界，用户始终掌握控制权
3. **流式体验**：从 API 响应到工具执行全链路流式处理，实时反馈
4. **可扩展性**：MCP 协议允许动态集成外部工具，技能系统支持自定义行为
5. **工程化**：50万行 TypeScript，模块化设计，完善的类型系统和错误处理

## 统计

| 指标 | 数值 |
|------|------|
| 源文件总数 | 4,756 |
| 核心源码（src/ + vendor/） | 1,906 个文件 |
| 第三方依赖（node_modules/） | 2,850 个文件 |
| TypeScript/TSX 源文件 | 1,884 个 |
| 代码行数 | ~500,000 行 |
| Source Map 大小 | 57 MB |
| 包版本 | 2.1.88 |
| 内置工具数 | 43 个 |
| 斜杠命令数 | 40+ 个 |

## 深度技术解析

### 关键模块实现

#### 1. QueryEngine - Agent 循环核心

`src/query.ts`（1729 行）是整个系统的核心，实现了完整的 PDAOL（感知-决策-行动-观察-循环）范式：

```typescript
// 简化的 Agent 循环核心
async function* query(params: QueryParams): AsyncGenerator<...> {
  // 1. [感知] 构建系统提示
  const systemPrompt = buildSystemPrompt(...)

  // 2. [决策] 流式调用 Claude API
  const stream = await anthropic.messages.stream({
    model, messages, systemPrompt, tools, stream: true
  })

  // 3. [行动] 并行执行工具调用
  for await (const chunk of stream) {
    if (chunk.type === 'tool_use') {
      const executor = new StreamingToolExecutor(tools, canUseTool, context)
      executor.addTool(chunk, assistantMessage)

      // 4. [观察] 实时产出工具结果
      for await (const result of executor.getRemainingResults()) {
        yield result.message
      }
    }
    yield chunk
  }

  // 5. [循环] 检查是否继续
  if (hasMoreToolCalls(messages)) {
    yield* query({ ...params, messages })
  }
}
```

**技术亮点**：
- 使用 `AsyncGenerator` 实现自然的背压控制
- 流式处理从 API 到 UI 的全链路
- 支持并行工具执行和取消信号

#### 2. StreamingToolExecutor - 流式工具执行器

`src/services/tools/StreamingToolExecutor.ts`（531 行）是核心创新之一：

```
工具调用请求
    ↓
┌─────────────────────────────────────────┐
│ StreamingToolExecutor                   │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ 状态机    │  │ 并发控制 │  │ 进度流 │ │
│  │queued    │  │安全并行  │  │实时产出│ │
│  │executing │  │非安全串行│  │       │ │
│  │completed │  │         │  │       │ │
│  │yielded   │  │         │  │       │ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────┘
    ↓
结果流式返回给 QueryEngine
```

**核心能力**：
| 特性 | 实现 |
|------|------|
| 并发控制 | 安全工具并行（如 Read），非安全工具串行（如 Edit） |
| 进度流 | `onProgress` 回调实时产出进度消息 |
| 错误级联 | Bash 错误自动取消兄弟进程 |
| 取消处理 | AbortController 层级传递 |

#### 3. 工具系统架构

所有 43 个工具遵循统一接口（`Tool.ts`）：

```typescript
interface Tool<Input, Output, Progress> {
  // 执行
  call(input, context, canUseTool, onProgress): Promise<ToolResult>

  // 提示生成
  prompt(options): Promise<string>

  // 权限
  checkPermissions(input, context): Promise<PermissionResult>

  // UI 渲染
  renderToolUseMessage(input, options): ReactNode
  renderToolResultMessage(result, options): ReactNode

  // 元信息
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive(input): boolean
}
```

**工具分类矩阵**：

| 类别 | 工具 | 并发安全 | 权限级别 |
|------|------|---------|---------|
| **文件操作** | FileReadTool | ✅ | L1-L3 |
| | FileEditTool | ❌（独占） | L3-L5 |
| **命令执行** | BashTool | ❌（独占） | L3-L4 |
| **搜索查询** | GrepTool, GlobTool | ✅ | L1 |
| **代码智能** | LSPTool | ✅ | L1 |
| **多代理** | AgentTool | ✅ | L3 |
| **任务管理** | TaskCreateTool | ✅ | L2 |
| **MCP集成** | MCPTool | ✅ | L3 |

#### 4. 双层状态管理

```
┌─────────────────────────────────────────────────────────────┐
│                      状态管理架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  UI 层 (React)          State 层 (React)      Global 层    │
│  ┌─────────────┐       ┌──────────────┐      ┌──────────┐ │
│  │ 组件        │◀─────▶│ AppState.tsx │◀────▶│bootstrap │ │
│  │ useAppState │       │  状态树       │      │ /state.ts│ │
│  └─────────────┘       └──────────────┘      └──────────┘ │
│         │                     │                   │       │
│         │ 订阅状态切片         │ 函数式更新         │ Signal │
│         ▼                     ▼                   ▼       │
│  选择性重渲染              会话级状态          跨会话持久   │
│  (React Compiler)          messages           totalCost   │
│                            tasks              projectRoot │
│                            permissionRequests  sessionId  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**关键设计**：
- **AppState**：会话级 React 状态，使用 `useSyncExternalStore` 与并发特性兼容
- **Bootstrap State**：全局单例，Signal 信号系统，严格控制（警示注释：`DO NOT ADD MORE STATE HERE`）

#### 5. 5层权限系统

```
┌─────────────────────────────────────────────────────────────┐
│                      权限决策流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  工具调用请求                                                │
│      ↓                                                      │
│  ┌──────────────┐                                          │
│  │ validateInput│ ──▶ Zod 校验 ──▶ 失败返回错误            │
│  └──────────────┘                                          │
│      ↓                                                      │
│  ┌──────────────────┐                                      │
│  │ checkPermissions │                                      │
│  └────────┬─────────┘                                      │
│           │                                                 │
│     ┌─────┼─────┬─────────┐                                │
│     ▼     ▼     ▼         ▼                                │
│  ┌────┐ ┌────┐ ┌────┐  ┌────┐                            │
│  │L1  │ │L2  │ │L3  │  │L4  │                            │
│  │自动│ │目录│ │模式│  │命令│                            │
│  │允许│ │隔离│ │区分│  │分析│                            │
│  └────┘ └────┘ └────┘  └────┘                            │
│           │                                                 │
│     ┌─────┴─────┐                                          │
│     ▼           ▼                                          │
│  ┌──────┐   ┌────────┐                                    │
│  │ allow │   │  ask   │ ──▶ 显示权限对话框                  │
│  └──────┘   └────────┘       - allow_once                  │
│                              - always_allow                │
│                              - deny                        │
│                              - deny_with_reason            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**层级说明**：

| 层级 | 名称 | 机制 | 示例 |
|------|------|------|------|
| L1 | 自动允许 | 只读操作 | FileReadTool |
| L2 | 目录隔离 | 项目目录内允许 | GlobTool |
| L3 | 模式区分 | ask/notify/plan 模式 | FileEditTool |
| L4 | 命令分析 | Bash 正则安全匹配 | BashTool |
| L5 | 破坏性确认 | Git 追踪文件修改 | 所有写操作 |

#### 6. 自动压缩系统（AutoCompact）

当上下文窗口接近阈值（默认 85%）时自动触发：

```
Token 使用率监测
      ↓
  超过阈值?
      ↓ 是
┌──────────────┐
│ 寻找安全边界 │ ──▶ 保留最近 N 条消息
│ (边界检测)   │      不在工具调用中间
└──────────────┘
      ↓
┌──────────────┐
│ Claude API   │ ──▶ 生成对话历史摘要
│ 生成摘要     │      保留关键决策
└──────────────┘      保留代码变更
      ↓
┌──────────────┐
│ 替换历史消息 │ ──▶ 用摘要替换详细历史
│ (状态更新)   │      继续对话
└──────────────┘
      ↓
熔断保护：最多连续 3 次失败
```

#### 7. MCP 协议集成

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 用户配置    │────▶│ MCPClient   │────▶│MCPTransport │────▶│ MCP Server  │
│settings.json│     │  (初始化)   │     │stdio/http   │     │ (外部进程)  │
└─────────────┘     └──────┬──────┘     └─────────────┘     └──────┬──────┘
                           │                                        │
                           │ ① 读取配置                            │
                           │ ② 启动进程/连接                       │
                           │ ③ capability协商                      │
                           │◀─────────────────────────────────────▶│
                           │                                        │
                           │ ④ listTools()                        │
                           │◀──────────────────────────────────────│
                           │                                        │
┌─────────────┐     ┌──────┴──────┐     ┌─────────────┐            │
│ Claude使用  │◀────│wrapMCPTool  │◀────│ Tool注册中心 │◀───────────┘
│ MCP工具     │     │(动态包装)   │     │  tools数组   │   调用工具
└─────────────┘     └─────────────┘     └─────────────┘
```

**动态工具注册**：
- MCP 工具自动包装为标准 `Tool` 接口
- 命名空间隔离：`mcp__<server>__<tool>`
- 进程隔离：MCP 服务器运行在独立进程中

### 流程交互详解

#### 完整请求处理流程

```
用户输入
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. 入口处理 (entrypoints/cli.tsx)                            │
│    - 参数解析 (--flags)                                      │
│    - 快速路径 (--version, --help)                            │
│    - Bun 特性系统初始化                                      │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. 状态初始化 (bootstrap/state.ts)                           │
│    - 全局状态创建 (Signal)                                   │
│    - OpenTelemetry 初始化                                    │
│    - 项目根目录检测                                          │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. React 渲染 (state/AppState.tsx)                           │
│    - AppStateProvider 挂载                                   │
│    - 状态订阅建立                                            │
│    - Ink 渲染器启动                                          │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 输入处理 (InputBox.tsx)                                   │
│    - 斜杠命令解析 (/commit, /cost...)                        │
│    - 自然语言处理                                            │
│    - 附件提取                                                │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. QueryEngine 调度                                          │
│    - 构建系统提示 (工具定义 + CLAUDE.md + git状态)           │
│    - Token 预算检查                                          │
│    ↓                                                        │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 6. Agent 循环 (query.ts)                                 │ │
│ │    ┌──────────┐    ┌──────────┐    ┌──────────┐        │ │
│ │    │ 感知     │───▶│ 决策     │───▶│ 行动     │        │ │
│ │    │构建提示  │    │调用API   │    │执行工具   │        │ │
│ │    └──────────┘    └──────────┘    └────┬─────┘        │ │
│ │         ▲                               │              │ │
│ │         │         ┌──────────┐          │              │ │
│ │         └─────────│ 观察     │◀─────────┘              │ │
│ │                   │工具结果  │                         │ │
│ │                   └──────────┘                         │ │
│ │                                                         │ │
│ │  StreamingToolExecutor 流式执行工具                    │ │
│ │  - 并行执行安全工具                                     │ │
│ │  - 实时产出进度                                        │ │
│ │  - 错误级联处理                                        │ │
│ └─────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. UI 更新 (Messages.tsx)                                    │
│    - 流式渲染消息                                          │
│    - 工具调用 UI 展示                                        │
│    - 结果/Diff 渲染                                          │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. 状态持久化                                                │
│    - 费用累计 (Bootstrap State)                              │
│    - 会话历史 (可选)                                         │
│    - 遥测数据发送                                            │
└─────────────────────────────────────────────────────────────┘
```

### 技术创新亮点

#### 1. Bun 特性系统与死代码消除

```typescript
// 编译时特性开关
import { feature } from 'bun:bundle'

// 未启用的 feature 在构建时被完全删除
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children
```

**优势**：
- 条件编译，减小包体积
- 快速路径优化（`--version` 跳过完整初始化）
- 启动时间 < 100ms

#### 2. React Compiler 自动优化

```typescript
// 编译后的代码自动包含 memoization
import { c as _c } from "react/compiler-runtime";

export function AppStateProvider(t0) {
  const $ = _c(13)  // 缓存数组
  if ($[0] !== initialState) {
    // 只有依赖变化时才重新计算
  }
}
```

#### 3. 多代理颜色系统

```typescript
type AgentColorName = 'red' | 'blue' | 'green' | 'yellow' | 'purple' | 'orange' | 'pink' | 'cyan'

// 每个子代理分配唯一颜色
[蓝色] Explore 代理：发现 47 个 TypeScript 文件...
[绿色] Plan 代理：建议的重构方案是...
```

**作用**：提升多代理场景的可观察性

#### 4. 拒绝追踪与优雅降级

```typescript
type DenialTrackingState = {
  consecutiveDenials: number  // 连续拒绝次数
  totalDenials: number        // 总拒绝次数
}

// 连续 3 次拒绝后降级到手动提示模式
if (consecutiveDenials >= 3) {
  fallbackToPromptingMode()
}
```

#### 5. 工具结果预算系统

```typescript
type ContentReplacementState = {
  replacements: Map<string, ContentReplacement>
  totalChars: number
  budget: number
}

// 超大结果自动持久化到文件
if (projectedTotal > budget) {
  const filePath = persistToFile(content)
  return `[内容已持久化到 ${filePath}]`
}
```

#### 6. 思考块（Thinking Block）管理

```typescript
/**
 * 思考块规则：
 * 1. 只能在 max_thinking_length > 0 的查询中出现
 * 2. 不能是消息列表的最后一个块
 * 3. 必须在 assistant trajectory 期间保留
 */
```

#### 7. 消息规范化

```typescript
// 确保每个 tool_use 都有对应的 tool_result
export function normalizeMessages(messages: Message[]): Message[] {
  // 孤儿 tool_result：创建合成 tool_use
  // 未完成 tool_use：创建合成错误结果
}
```

### 工程实践

| 实践 | 实现 |
|------|------|
| **类型安全** | TypeScript 严格模式，Zod 运行时校验 |
| **测试覆盖** | 单元测试 + 集成测试 + E2E 测试 |
| **可观测性** | OpenTelemetry（追踪/指标/日志） |
| **性能优化** | 提示缓存（减少 90%+ Token 费用） |
| **安全设计** | 5层权限、命令沙箱、路径隔离 |
| **流式处理** | AsyncGenerator 全链路流式 |
| **状态管理** | 函数式更新、不可变数据 |
| **错误处理** | 分类重试、熔断机制、优雅降级 |

## 相关文档

- [技术架构深度解析](claude-code-technical-analysis.md) - 详细源码分析和架构设计（约 1.1 万字）
- [设计指南](claude-code-design-guide.pdf) - 产品设计文档

## 声明

- 源码版权归 [Anthropic](https://www.anthropic.com) 所有
- 本仓库仅用于技术研究与学习，请勿用于商业用途
- 如有侵权，请联系删除
