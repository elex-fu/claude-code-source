# claude-code-sourcemap

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

## 相关文档

- [技术架构深度解析](docs/claude-code-technical-analysis.md) - 详细源码分析和架构设计
- [设计指南](claude-code-design-guide.pdf) - 产品设计文档

## 声明

- 源码版权归 [Anthropic](https://www.anthropic.com) 所有
- 本仓库仅用于技术研究与学习，请勿用于商业用途
- 如有侵权，请联系删除
