# Claude Code 多Agent系统技术实现深度分析

## 一、概述

Claude Code 的多Agent系统是一个生产级的分布式AI协作架构，通过类型安全的Agent定义、灵活的上下文隔离机制和高效的并发控制，实现了复杂软件工程任务的分解与并行执行。

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户交互层 (CLI/TUI)                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  Ink/React  │  │  输入处理    │  │  消息渲染   │  │  多Agent视图  │   │
│  │  组件系统   │  │  InputBox   │  │  Messages   │  │  AgentColors │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Agent调度层                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     AgentTool (工具入口)                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │  类型选择    │  │  权限检查    │  │  隔离级别   │             │   │
│  │  │ subagent_type│  │ canUseTool  │  │ isolation   │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌───────────────────────┐ ┌──────────────┐ ┌──────────────────────┐
│    协调器模式          │ │   Fork子Agent │ │     后台任务         │
│  (Coordinator Mode)   │ │  (默认模式)    │ │  (Background Task)   │
│  ┌─────────────────┐  │ │ ┌──────────┐ │ │ ┌─────────────────┐  │
│  │  Worker池管理    │  │ │ │内存隔离  │ │ │ │ 异步执行         │  │
│  │  任务分解与合成  │  │ │ │Worktree  │ │ │ │ 任务通知         │  │
│  │  SendMessage通信 │  │ │ │Remote    │ │ │ │ 进度追踪         │  │
│  └─────────────────┘  │ │ └──────────┘ │ │ └─────────────────┘  │
└───────────────────────┘ └──────────────┘ └──────────────────────┘
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        执行编排层                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                 StreamingToolExecutor                            │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │   │
│  │  │ 状态机     │  │ 并发控制    │  │ 错误级联   │  │ 进度流    │ │   │
│  │  │queued      │  │安全并行     │  │Bash取消    │  │实时产出   │ │   │
│  │  │executing   │  │独占执行     │  │兄弟进程    │  │onProgress │ │   │
│  │  │completed   │  │            │  │            │  │           │ │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └───────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│   内置工具     │         │   MCP工具      │         │   子Agent工具  │
│  Read/Edit/   │         │  外部服务调用  │         │  递归调用Agent │
│  Bash/Glob    │         │  进程隔离      │         │  嵌套执行      │
└───────────────┘         └───────────────┘         └───────────────┘
```

### 1.2 核心组件关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Agent定义层                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Built-in     │  │ Custom       │  │ Plugin       │  │ Coordinator│ │
│  │ (6种预定义)   │  │ (用户定义)    │  │ (插件扩展)    │  │ (协调器)    │ │
│  │ Explore      │  │ userSettings │  │ skill工具    │  │ worker      │ │
│  │ Plan         │  │ projectSet   │  │ 扩展         │  │ pool        │ │
│  │ Verification │  │ policySet    │  │              │  │             │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼  loadAgentsDir.ts
┌─────────────────────────────────────────────────────────────────────────┐
│                         Agent选择器                                      │
│         ┌───────────────┐      ┌───────────────┐      ┌──────────┐     │
│         │  subagent_type│ ──▶  │  查找AgentDef │ ──▶  │ 过滤权限 │     │
│         └───────────────┘      └───────────────┘      └──────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌───────────────────────┐ ┌──────────────┐ ┌──────────────────────┐
│      父Agent          │ │   子Agent     │ │      子Agent          │
│  ┌─────────────────┐  │ │ ┌──────────┐ │ │  ┌─────────────────┐  │
│  │ context         │  │ │ │context   │ │ │  │ context         │  │
│  │ ├── messages    │  │ │ ├── fork  │ │ │  │ ├── messages    │  │
│  │ ├── fileState   │  │ │ ├── state │ │ │  │ ├── fileState   │  │
│  │ └── permissions │──┼─▶ │         │ │ └──│  └── permissions │  │
│  └─────────────────┘  │ │ └──────────┘ │ │  └─────────────────┘  │
│        ▲              │ └──────────────┘ │           ▲           │
│        │              │         │        │           │           │
│  ┌─────┴─────┐        │         ▼        │     ┌─────┴─────┐     │
│  │ 结果聚合   │◀───────┼─── 通知/消息 ─────┴────▶│ 结果聚合   │     │
│  └───────────┘        │                      └───────────┘     │
└───────────────────────┘                      └───────────────────────┘
```

## 二、Agent定义系统

### 2.1 类型架构

Agent定义采用分层类型设计，位于 `src/tools/AgentTool/loadAgentsDir.ts`：

```typescript
// 基础Agent定义 - 所有Agent类型共享的属性
type BaseAgentDefinition = {
  agentType: string              // 唯一标识符
  whenToUse: string             // 使用场景描述（用于模型选择）
  tools?: string[]              // 允许的工具列表
  disallowedTools?: string[]    // 禁止的工具列表
  color?: AgentColorName        // UI颜色标识
  model?: string               // 使用的模型（inherit/haiku/sonnet/opus）
  effort?: EffortValue         // 努力程度设置
  permissionMode?: PermissionMode  // 权限模式（default/ask/notify/bubble/plan）
  maxTurns?: number            // 最大轮数限制
  memory?: AgentMemoryScope    // 记忆作用域（user/project/local）
  isolation?: 'worktree' | 'remote'  // 隔离模式
  background?: boolean         // 是否后台运行
  omitClaudeMd?: boolean       // 是否省略CLAUDE.md节省token
  hooks?: HooksSettings        // 会话级钩子
  mcpServers?: AgentMcpServerSpec[]  // Agent专属MCP服务器
}

// 内置Agent - 动态生成系统提示
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// 自定义Agent - 静态提示
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // userSettings/projectSettings/policySettings
  filename?: string
  baseDir?: string
}

// 插件Agent
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  filename?: string
  plugin: string
}
```

### 2.2 内置Agent类型

系统内置6种预定义Agent，定义于 `src/tools/AgentTool/builtInAgents.ts`：

| Agent类型 | 定位 | 模型 | 关键限制 | 特殊能力 |
|-----------|------|------|----------|----------|
| **general-purpose** | 通用执行 | inherit | 无 | 完整工具集 |
| **Explore** | 代码探索 | haiku(外部)/inherit(ant) | 禁止写操作 | 快速搜索代码库 |
| **Plan** | 架构规划 | inherit | 禁止写操作 | 生成实现计划 |
| **verification** | 结果验证 | inherit | 禁止修改文件 | PASS/FAIL/PARTIAL评判 |
| **statusline-setup** | 状态栏配置 | inherit | - | 配置状态栏设置 |
| **claude-code-guide** | 使用指南 | inherit | - | 回答CC使用问题 |

**Explore Agent示例**（`src/tools/AgentTool/built-in/exploreAgent.ts`）：

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases...',
  disallowedTools: [
    AGENT_TOOL_NAME,        // 禁止递归创建代理
    FILE_EDIT_TOOL_NAME,    // 禁止修改文件
    FILE_WRITE_TOOL_NAME,   // 禁止写入文件
    NOTEBOOK_EDIT_TOOL_NAME,// 禁止编辑notebook
  ],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,      // 省略CLAUDE.md节省约5-15K token/周
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

**Verification Agent示例**（`src/tools/AgentTool/built-in/verificationAgent.ts`）：

```typescript
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  whenToUse: 'Use this agent to verify that implementation work is correct...',
  color: 'red',            // 红色标识验证Agent
  background: true,        // 始终在后台运行
  disallowedTools: [
    AGENT_TOOL_NAME,        // 禁止递归创建
    FILE_EDIT_TOOL_NAME,    // 禁止修改（纯验证）
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  model: 'inherit',
  criticalSystemReminder_EXPERIMENTAL: 'CRITICAL: This is a VERIFICATION-ONLY task...',
}
```

### 2.3 能力限制机制

通过 `disallowedTools` 实现安全的能力限制：

```typescript
// Explore/Plan Agent：只读模式
disallowedTools: [
  AGENT_TOOL_NAME,        // 禁止创建子Agent（防止无限嵌套）
  FILE_EDIT_TOOL_NAME,    // 禁止文件编辑
  FILE_WRITE_TOOL_NAME,   // 禁止文件写入
  BASH_TOOL_NAME,         // 可配置限制危险命令
]

// Verification Agent：验证模式
disallowedTools: [
  AGENT_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,    // 关键：禁止修改被验证的代码
  FILE_WRITE_TOOL_NAME,
]
```

## 三、子Agent创建机制

### 3.1 AgentTool调用流程

AgentTool是创建子Agent的核心工具（`src/tools/AgentTool/AgentTool.tsx:196-400`）：

```typescript
export const AgentTool = buildTool({
  name: AGENT_TOOL_NAME,
  inputSchema: lazySchema(() => z.object({
    description: z.string().describe('A short (3-5 word) description'),
    prompt: z.string().describe('The task for the agent to perform'),
    subagent_type: z.string().optional(),  // Agent类型
    model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
    run_in_background: z.boolean().optional(),
    name: z.string().optional(),           // 多Agent场景下的名称
    team_name: z.string().optional(),      // 所属团队
    isolation: z.enum(['worktree', 'remote']).optional(),
  })),

  async call(input, toolUseContext, canUseTool, assistantMessage, onProgress) {
    // 1. 解析Agent类型
    const effectiveType = input.subagent_type ??
      (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)

    // 2. 查找Agent定义
    const selectedAgent = findAgentDefinition(effectiveType, toolUseContext)

    // 3. 决定执行模式
    if (shouldRunAsync) {
      return runAsyncAgentLifecycle(...)  // 后台执行
    } else {
      return runAgent({...})  // 同步执行
    }
  }
})
```

### 3.2 Fork Subagent机制

Fork是最常用的子Agent创建方式（`src/tools/AgentTool/forkSubagent.ts`）：

```typescript
// Fork特性开关
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // 与协调器模式互斥
    if (getIsNonInteractiveSession()) return false
    return true
  }
  return false
}

// Fork Agent定义 - 继承父级完整能力
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  whenToUse: 'Implicit fork — inherits full conversation context',
  tools: ['*'],              // 继承父级全部工具
  maxTurns: 200,
  model: 'inherit',          // 继承父级模型
  permissionMode: 'bubble',  // 权限上冒到父级终端
  useExactTools: true,       // 工具定义字节级一致（缓存优化）
}
```

### 3.3 递归Fork防护

防止无限递归创建（`src/tools/AgentTool/forkSubagent.ts:78-89`）：

```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}

// 调用时检查
if (isInForkChild(toolUseContext.messages)) {
  throw new Error('Fork is not available inside a forked worker...')
}
```

### 3.4 Fork子Agent创建流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Fork子Agent创建完整流程                                │
└─────────────────────────────────────────────────────────────────────────┘

用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. AgentTool.call() 接收参数                                     │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ description: "搜索代码"                                   │  │
│    │ prompt: "查找所有API端点"                                  │  │
│    │ subagent_type: undefined → 触发Fork路径                   │  │
│    │ model: undefined → 继承父级                               │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 递归防护检查                                                  │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ isInForkChild(messages) ?                               │  │
│    │ ├── 检查messages中是否包含<FORK_BOILERPLATE_TAG>         │  │
│    │ └── 若存在 → 抛出错误（禁止嵌套Fork）                     │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 选择Agent定义                                                 │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ effectiveType = undefined                               │  │
│    │ isForkPath = true                                       │  │
│    │ selectedAgent = FORK_AGENT                              │  │
│    │                                                         │  │
│    │ FORK_AGENT配置:                                         │  │
│    │ ├── tools: ['*'] (继承全部工具)                          │  │
│    │ ├── model: 'inherit'                                    │  │
│    │ ├── permissionMode: 'bubble' (权限上冒)                  │  │
│    │ └── useExactTools: true (缓存优化)                       │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 构建Fork消息上下文 (缓存优化关键)                              │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │                                                         │  │
│    │  父级Assistant消息                                       │  │
│    │  ┌─────────────────────────────────────────────────┐   │  │
│    │  │ content: [                                      │   │  │
    │  │  │   {type: "thinking", ...},                    │   │  │
    │  │  │   {type: "text", ...},                        │   │  │
    │  │  │   {type: "tool_use", id: "tool_1", ...}, ◀──┐ │   │  │
    │  │  │   {type: "tool_use", id: "tool_2", ...}, ◀──┼─┼───┼──┤
│    │  │ ]                                               │ │   │  │
│    │  └─────────────────────────────────────────────────┘ │   │  │
│    │                                                      │   │  │
│    │  子级User消息 (工具结果)                               │   │  │
│    │  ┌─────────────────────────────────────────────────┐ │   │  │
│    │  │ content: [                                      │ │   │  │
│    │  │   {type: "tool_result",                         │ │   │  │
│    │  │    tool_use_id: "tool_1",                       │ │   │  │
│    │  │    content: "Fork started..."}, ◀───────────────┘ │   │  │
│    │  │   {type: "tool_result",                         │     │  │
│    │  │    tool_use_id: "tool_2",                       │     │  │
│    │  │    content: "Fork started..."}, ◀───────────────┘     │  │
│    │  │   {type: "text",                                   │  │
│    │  │    text: "<FORK_BOILERPLATE>...指令..."}  ◀── 唯一差异 │  │
│    │  │ ]                                               │    │  │
│    │  └─────────────────────────────────────────────────┘    │  │
│    │                                                          │  │
│    │  ★ 所有子Agent共享相同前缀，只有最后text块不同            │  │
│    │  ★ 最大化Claude API的prompt缓存命中率                     │  │
│    │                                                         │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 创建子Agent上下文                                            │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ createSubagentContext(parentContext, FORK_AGENT)        │  │
│    │                                                         │  │
│    │ 继承:                                                   │  │
│    │ ├── options (工具列表、配置等)                           │  │
│    │ ├── abortController (取消信号)                           │  │
│    │ └── fileState (文件状态缓存)                             │  │
│    │                                                         │  │
│    │ 新建/隔离:                                              │  │
│    │ ├── agentId (新UUID)                                    │  │
│    │ ├── messages (空数组，从forkMessages开始)                │  │
│    │ └── permissionMode='bubble' → 权限继承父级              │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 执行模式决策                                                  │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ run_in_background = true ?                              │  │
│    │ ├── YES → runAsyncAgentLifecycle() → 后台异步执行        │  │
│    │ └── NO  → runAgent() → 同步阻塞执行                      │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. 启动子Agent执行 (runAgent)                                    │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ a. 初始化MCP服务器 (若有配置)                              │  │
│    │ b. 组装工具池 (assembleToolPool)                          │  │
│    │ c. 构建系统提示 (buildEffectiveSystemPrompt)               │  │
│    │ d. 注册任务状态 (registerAsyncAgent)                       │  │
│    │ e. 启动query()循环                                        │  │
│    │                                                         │  │
│    │ ┌─────────────┐     ┌─────────────┐     ┌──────────┐   │  │
│    │ │   query()   │────▶│ API Stream  │────▶│ Tool执行 │   │  │
│    │ │             │◀────│   流式响应   │◀────│          │   │  │
│    │ └─────────────┘     └─────────────┘     └──────────┘   │  │
│    │        │                                               │  │
│    │        └───────────────────────────────────────────────┘  │
│    │                    (循环直到完成或出错)                      │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. 结果返回                                                      │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 同步模式:                                               │  │
│    │ ├── 等待runAgent完成                                    │  │
│    │ ├── 收集所有messages                                    │  │
│    │ └── 返回最终结果                                        │  │
│    │                                                         │  │
│    │ 异步模式:                                               │  │
│    │ ├── 立即返回async_launched状态                          │  │
│    │ ├── 提供outputFile路径供查询                            │  │
│    │ └── 后台完成后发送task-notification                      │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 四、上下文隔离机制

### 4.1 上下文构建策略

Fork机制的核心优化在于最大化Prompt缓存命中率（`src/tools/AgentTool/forkSubagent.ts:107-169`）：

```typescript
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'

export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆完整的父级assistant消息（包含所有tool_use块）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 收集所有tool_use块
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use',
  )

  // 3. 所有tool_result使用相同的placeholder文本
  // 这是缓存优化的关键：确保所有fork子进程的前缀字节一致
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{
      type: 'text' as const,
      text: FORK_PLACEHOLDER_RESULT,  // 固定文本
    }],
  }))

  // 4. 构建用户消息：placeholder结果 + 子任务指令
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text' as const, text: buildChildMessage(directive) },
    ],
  })

  // 结果: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
  // 只有最后的文本块在不同子进程间不同
  return [fullAssistantMessage, toolResultMessage]
}
```

**缓存优化原理**：
- 所有fork子进程共享相同的前缀（assistant消息 + placeholder结果）
- 只有最后的指令文本不同
- Claude API的prompt缓存机制可以复用前面大部分token

### 4.2 子Agent上下文创建

创建隔离的子Agent上下文（`src/tools/AgentTool/forkSubagent.ts` 及相关代码）：

```typescript
export async function createSubagentContext(
  parentContext: ToolUseContext,
  agentDefinition: AgentDefinition,
): Promise<ToolUseContext> {
  return {
    ...parentContext,
    // 关键隔离点
    agentId: createAgentId(),           // 独立Agent ID
    messages: [],                       // 独立消息历史
    readFileState: cloneFileStateCache(parentContext.readFileState),

    // 权限上下文：继承或重置
    toolPermissionContext: agentDefinition.permissionMode === 'bubble'
      ? parentContext.toolPermissionContext  // bubble模式继承
      : createNewPermissionContext(),         // 其他模式新建
  }
}
```

### 4.3 三种隔离级别

| 隔离级别 | 机制 | 适用场景 | 实现位置 |
|---------|------|---------|---------|
| **内存隔离** | 独立messages/state | 默认子Agent | `createSubagentContext` |
| **Worktree隔离** | Git worktree独立副本 | 需要文件隔离 | `createAgentWorktree` |
| **Remote隔离** | 远程CCR环境 | 完全隔离任务 | `registerRemoteAgentTask` |

**Worktree隔离示例**：

```typescript
// src/utils/worktree.ts
export async function createAgentWorktree(agentId: string): Promise<{
  worktreePath: string
  branchName: string
  cleanup: () => Promise<void>
}> {
  const branchName = `agent-${agentId}`
  const worktreePath = path.join(os.tmpdir(), 'claude-code-worktrees', agentId)

  // 创建独立worktree
  await exec(`git worktree add -b ${branchName} ${worktreePath}`)

  return {
    worktreePath,
    branchName,
    cleanup: async () => {
      await exec(`git worktree remove ${worktreePath}`)
      await exec(`git branch -D ${branchName}`)
    }
  }
}
```

## 五、Agent间通信机制

### 5.1 通知系统

子Agent通过任务通知与父Agent通信（`src/tasks/LocalAgentTask/LocalAgentTask.tsx:197-260`）：

```typescript
export function enqueueAgentNotification({
  taskId,
  description,
  status,           // 'completed' | 'failed' | 'killed'
  error,
  finalMessage,
  usage,
  worktreePath,
}: {
  taskId: string
  description: string
  status: 'completed' | 'failed' | 'killed'
  // ...其他参数
}): void {
  // 构建XML格式的任务通知
  const notification = `
<task-notification>
  <task-id>${taskId}</task-id>
  <status>${status}</status>
  <summary>Agent "${description}" ${status}</summary>
  ${finalMessage ? `<result>${finalMessage}</result>` : ''}
  ${usage ? `
  <usage>
    <total_tokens>${usage.totalTokens}</total_tokens>
    <tool_uses>${usage.toolUses}</tool_uses>
    <duration_ms>${usage.durationMs}</duration_ms>
  </usage>
  ` : ''}
  ${worktreePath ? `<worktree-path>${worktreePath}</worktree-path>` : ''}
</task-notification>`

  // 添加到消息队列
  enqueuePendingNotification(notification, setAppState)
}
```

### 5.2 SendMessage工具

允许向运行中的Agent发送消息（协调器模式核心）：

```typescript
// 伪代码示意
SendMessageTool.execute({
  to: 'agent-abc-123',  // 目标Agent ID
  message: '继续完成剩下的任务'
})
```

### 5.3 任务状态共享

通过全局状态管理任务（`src/tasks/LocalAgentTask/LocalAgentTask.tsx:116-148`）：

```typescript
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  agentType: string
  model?: string
  abortController?: AbortController
  progress?: AgentProgress        // 进度追踪
  messages?: Message[]            // 完整对话历史
  pendingMessages: string[]       // 待处理消息队列
  isBackgrounded: boolean         // 是否后台运行
  retain: boolean                 // UI是否保持
}

// 更新任务状态
export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T
): void {
  setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [taskId]: updater(prev.tasks[taskId] as T)
    }
  }))
}
```

## 六、并发控制与执行编排

### 6.1 StreamingToolExecutor架构

执行编排的核心（`src/services/tools/StreamingToolExecutor.ts:40-62`）：

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private toolUseContext: ToolUseContext
  private hasErrored = false
  private erroredToolDescription = ''
  // 兄弟进程取消控制器 - Bash错误时取消并行进程
  private siblingAbortController: AbortController
  private discarded = false

  constructor(
    private readonly toolDefinitions: Tools,
    private readonly canUseTool: CanUseToolFn,
    toolUseContext: ToolUseContext,
  ) {
    this.toolUseContext = toolUseContext
    this.siblingAbortController = createChildAbortController(
      toolUseContext.abortController,
    )
  }
}
```

### 6.2 工具跟踪状态机

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean    // 是否可并发
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]    // 进度消息
}
```

### 6.3 并发控制逻辑

```typescript
// 检查当前是否可以执行工具
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||  // 无执行中工具
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}

// 处理队列
private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue

    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)
    } else {
      // 非并发安全工具需要独占执行
      if (!tool.isConcurrencySafe) break
    }
  }
}
```

### 6.4 并发策略矩阵

| 工具类型 | 并发安全 | 策略 | 示例 |
|---------|---------|------|------|
| 只读文件操作 | ✅ 是 | 并行执行 | Read, Glob, Grep |
| 写文件操作 | ❌ 否 | 独占执行 | Edit, Write |
| Bash命令 | ❌ 否 | 独占执行 + 错误级联 | Bash |
| MCP工具 | ✅ 是 | 并行执行 | MCPTool |
| Agent工具 | ✅ 是 | 并行执行（后台） | AgentTool |

### 6.5 错误级联机制

Bash错误时自动取消兄弟进程（`src/services/tools/StreamingToolExecutor.ts`）：

```typescript
// 仅Bash错误触发兄弟进程取消
if (tool.block.name === BASH_TOOL_NAME && isErrorResult) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

**设计原理**：Bash命令通常有副作用，一个失败可能导致后续命令产生不一致状态。

### 6.6 StreamingToolExecutor执行流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  StreamingToolExecutor并发执行流程                        │
└─────────────────────────────────────────────────────────────────────────┘

QueryEngine                         StreamingToolExecutor
    │                                      │
    │  1. 收到API响应流 (tool_use块)         │
    │─────────────────────────────────────▶│
    │                                      │
    │                              ┌───────┴───────┐
    │                              │   addTool()   │
    │                              │               │
    │                              │ ┌───────────┐ │
    │                              │ │ 解析输入   │ │
    │                              │ │ 检查安全性 │ │
    │                              │ │ isSafe = ? │ │
    │                              │ └─────┬─────┘ │
    │                              │       │       │
    │                              │ ┌─────┴─────┐ │
    │                              │ │ 创建TrackedTool│
    │                              │ │ status='queued'│
    │                              │ │ isConcurrencySafe│
    │                              │ │   = isSafe    │
    │                              │ └─────────────┘ │
    │                              │       │         │
    │                              └───────┼─────────┘
    │                                      │
    │                              ┌───────┴───────┐
    │                              │  processQueue()│
    │                              │               │
    │                              │ for each tool: │
    │                              │   canExecute?  │
    │                              └───────┬───────┘
    │                                      │
    │                    ┌─────────────────┼─────────────────┐
    │                    │                 │                 │
    │                    ▼                 ▼                 ▼
    │            ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │            │ 安全工具     │   │ 安全工具     │   │ 非安全工具   │
    │            │ Read        │   │ Grep        │   │ Edit        │
    │            │ isSafe=true │   │ isSafe=true │   │ isSafe=false│
    │            └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
    │                   │                 │                 │
    │                   ▼                 ▼                 │
    │            ┌─────────────┐   ┌─────────────┐         │
    │            │ 并行执行     │   │ 并行执行     │         │
    │            │ status='exe'│   │ status='exe'│         │
    │            │ 同时运行     │   │ 同时运行     │         │
    │            └──────┬──────┘   └──────┬──────┘         │
    │                   │                 │                 │
    │                   ▼                 ▼                 │
    │            ┌─────────────┐   ┌─────────────┐         │
    │            │ 完成        │   │ 完成        │         │
    │            │ status='comp'│   │ status='comp'│         │
    │            └──────┬──────┘   └──────┬──────┘         │
    │                   │                 │                 │
    │                   └────────┬────────┘                 │
    │                            │                          │
    │                            ▼                          │
    │                   ┌─────────────────┐                 │
    │                   │ 检查队列下一个    │◀────────────────┘
    │                   │ 非安全工具?       │
    │                   │ 前面有执行中?     │
    │                   └────────┬────────┘
    │                            │
    │              ┌─────────────┼─────────────┐
    │              │             │             │
    │              ▼             │             ▼
    │       ┌──────────┐        │      ┌──────────┐
    │       │ 有执行中 │        │      │ 队列为空 │
    │       │ 等待...  │        │      │ 或全部完成│
    │       └────┬─────┘        │      └────┬─────┘
    │            │              │           │
    │            │              ▼           │
    │            │       ┌──────────┐       │
    │            │       │ 可以执行 │       │
    │            │       │ status='exe'     │
    │            │       └────┬─────┘       │
    │            │            │             │
    │            │            ▼             │
    │            │       ┌──────────┐       │
    │            │       │ 独占执行 │       │
    │            │       │ 完成前阻塞队列   │
    │            │       └────┬─────┘       │
    │            │            │             │
    │            └────────────┼─────────────┘
    │                         │
    │                         ▼
    │                ┌─────────────────┐
    │                │  yieldResults()  │
    │                │ 按原始顺序产出结果 │
    │                │ (queued→completed)│
    │                └────────┬────────┘
    │                         │
    │  2. 产出工具结果          │
    │◀────────────────────────│
    │                         │
    │  3. 继续循环或结束        │
    │                         │
    ▼                         ▼


错误级联场景 (Bash错误):

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Worker-1 (Bash: npm install)                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐                 │
│  │ 执行中   │      │ 执行中   │      │ 等待中   │                 │
│  │ status  │      │ status  │      │ status  │                 │
│  │ ='exe'  │      │ ='exe'  │      │ ='queued'│                 │
│  └────┬────┘      └────┬────┘      └────┬────┘                 │
│       │                │                │                       │
│       ▼                │                │                       │
│   ┌───────┐            │                │                       │
│   │ ERROR │            │                │                       │
│   └───┬───┘            │                │                       │
│       │                │                │                       │
│       ▼                ▼                ▼                       │
│  ┌─────────────────────────────────────────┐                   │
│  │ siblingAbortController.abort()          │                   │
│  │ 取消所有执行中的兄弟进程                  │                   │
│  └─────────────────────────────────────────┘                   │
│       │                │                │                       │
│       ▼                ▼                ▼                       │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐                 │
│  │ 返回错误 │      │ 被取消   │      │ 未开始   │                 │
│  │ is_error│      │ is_error│      │ (无影响) │                 │
│  │ =true   │      │ =true   │      │         │                 │
│  └─────────┘      └─────────┘      └─────────┘                 │
│                                                                 │
│  说明: Bash错误会触发级联取消，防止不一致状态                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 七、协调器模式（Coordinator Mode）

### 7.1 模式定义

高级多Agent编排模式（`src/coordinator/coordinatorMode.ts:36-41`）：

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

### 7.2 协调器角色定义

协调器系统提示核心（`src/coordinator/coordinatorMode.ts:111-369`）：

```
## 1. Your Role

You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible

Every message you send is to the user. Worker results and system notifications
are internal signals — never thank or acknowledge them.

## 2. Your Tools

- **AgentTool** - Spawn a new worker
- **SendMessageTool** - Continue an existing worker
- **TaskStopTool** - Stop a running worker
```

### 7.3 任务工作流

标准4阶段工作流：

| 阶段 | 执行者 | 目的 |
|------|--------|------|
| Research | Workers（并行） | 调研代码库、理解问题 |
| Synthesis | **协调器** | 综合发现、制定实现规范 |
| Implementation | Workers | 按规范执行修改 |
| Verification | Workers | 验证修改正确性 |

### 7.4 并发策略

```
**Parallelism is your superpower.** Workers are async. Launch independent
workers concurrently whenever possible — don't serialize work that can run
simultaneously.

Manage concurrency:
- **Read-only tasks** (research) — run in parallel freely
- **Write-heavy tasks** (implementation) — one at a time per set of files
- **Verification** can sometimes run alongside implementation
```

### 7.5 Worker Prompt编写规范

**关键原则：综合理解，不懒惰委托**

```typescript
// 反模式：懒惰委托（错误）
${AGENT_TOOL_NAME}({
  prompt: "Based on your findings, fix the auth bug",
  ...
})

// 正确：综合后的具体规范
${AGENT_TOOL_NAME}({
  prompt: "Fix the null pointer in src/auth/validate.ts:42.
    The user field on Session (src/auth/types.ts:15) is undefined
    when sessions expire but the token remains cached.
    Add a null check before user.id access — if null, return 401
    with 'Session expired'. Commit and report the hash."
})
```

### 7.6 Continue vs Spawn决策

| 情况 | 机制 | 原因 |
|------|------|------|
| 研究恰好覆盖需编辑的文件 | **Continue** | Worker已有文件上下文 |
| 研究广泛但实现聚焦 | **Spawn fresh** | 避免噪音，专注上下文 |
| 修正失败或扩展近期工作 | **Continue** | 保留错误上下文 |
| 验证其他Worker写的代码 | **Spawn fresh** | 新视角，无实现偏见 |

### 7.7 协调器模式交互流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    协调器模式 - 典型任务处理流程                           │
│                      (研究 → 综合 → 实现 → 验证)                          │
└─────────────────────────────────────────────────────────────────────────┘

用户
 │
 │  "修复auth模块的null pointer问题"
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         协调器 (Coordinator)                      │
│                     CLAUDE_CODE_COORDINATOR_MODE=1                │
└─────────────────────────────────────────────────────────────────┘
 │
 │ 分析: 需要研究 + 定位问题
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: RESEARCH (并行执行)                                      │
│                                                                  │
│  协调器发起多个并行Agent调用 (同一条消息中多个工具调用):            │
│                                                                  │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────┐ │
│  │ Agent-1 (Explore) │  │ Agent-2 (Explore) │  │ Agent-3      │ │
│  │                   │  │                   │  │ (Explore)    │ │
│  │ 调研auth模块       │  │ 查找相关测试文件   │  │ 查看近期提交  │ │
│  │ src/auth/         │  │ *auth*.test.*     │  │ git log      │ │
│  └─────────┬─────────┘  └─────────┬─────────┘  └──────┬───────┘ │
│            │                      │                   │         │
│            │ 并行执行              │ 并行执行          │ 并行执行 │
│            ▼                      ▼                   ▼         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   StreamingToolExecutor                     │ │
│  │              (并发执行3个AgentTool - 安全工具)               │ │
│  └────────────────────────────────────────────────────────────┘ │
│            │                      │                   │         │
│            │ 返回结果              │ 返回结果          │ 返回结果 │
│            ▼                      ▼                   ▼         │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────┐ │
│  │ 发现validate.ts   │  │ 找到auth.test.ts │  │ 发现近期相关  │ │
│  │ 第42行null指针    │  │ 测试覆盖情况      │  │ 重构提交      │ │
│  │ 用户字段未检查    │  │ 3个测试文件       │  │ abc123       │ │
│  └─────────┬─────────┘  └─────────┬─────────┘  └──────┬───────┘ │
│            │                      │                   │         │
└────────────┼──────────────────────┼───────────────────┼─────────┘
             │                      │                   │
             └──────────────────────┼───────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 2: SYNTHESIS (协调器执行)                                   │
│                                                                  │
│  协调器综合所有Worker的发现:                                       │
│                                                                  │
│  1. 阅读结果 → 理解问题                                           │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 问题: validate.ts:42 访问user.id前未检查null              │ │
│     │       Session过期时user字段为undefined                    │ │
│     │ 相关: types.ts:15 Session定义                             │ │
│     │       auth.test.ts 有3个测试用例                          │ │
│     └─────────────────────────────────────────────────────────┘ │
│                                                                  │
│  2. 制定具体实现规范 (关键: 不依赖Worker理解，自己综合):            │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 修复方案:                                                │ │
│     │ 1. 在validate.ts:42添加null检查                          │ │
│     │ 2. 若user为null，返回401 + "Session expired"             │ │
│     │ 3. 更新auth.test.ts第58行预期错误消息                     │ │
│     │ 4. 运行测试确保通过                                       │ │
│     │ 5. 提交并报告commit hash                                 │ │
│     └─────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
 │
 │ 决策: 需要修改代码
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 3: IMPLEMENTATION                                          │
│                                                                  │
│  选择: Continue vs Spawn?                                        │
│                                                                  │
│  分析: Agent-1已加载validate.ts和相关文件 → 高上下文重叠           │
│  决策: 使用SendMessage继续Agent-1                                  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ SendMessageTool.call()                                      ││
│  │                                                             ││
│  │ to: "agent-1-xyz"                                           ││
│  │ message: "Fix the null pointer in src/auth/validate.ts:42.   ││
│  │          The user field on Session (src/auth/types.ts:15)    ││
│  │          is undefined when sessions expire...                 ││
│  │          Add a null check before user.id access...           ││
│  │          Run tests and commit."                              ││
│  │                                                             ││
│  │ ★ 规范包含具体文件路径、行号、预期行为                         ││
│  │ ★ 没有说"Based on your findings" (这是懒惰委托)               ││
│  │ ★ 明确"done"的标准 (测试通过、提交)                           ││
│  └─────────────────────────────────────────────────────────────┘│
 │                                                                  │
 │  Worker执行                                                       │
 │                                                                  │
 ▼                                                                  │
┌─────────────────────────────────────────────────────────────────┐│
│                         Agent-1 (Worker)                          ││
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐││
│  │ 读取validate │  │ 添加null检查 │  │ 运行测试     │  │ git提交  │││
│  │ .ts:42      │  │ if (!user)   │  │ npm test     │  │ commit   │││
│  │             │  │  return 401  │  │ (3/3 pass)   │  │ hash:def456│
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘││
│       │                │                │                │       ││
│       └────────────────┴────────────────┴────────────────┘       ││
│                              │                                    ││
│                              ▼                                    ││
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 返回结果:                                                     ││
│  │ - 修改了src/auth/validate.ts (添加null检查)                   ││
│  │ - 更新了auth.test.ts (修正预期错误消息)                       ││
│  │ - 所有测试通过                                                ││
│  │ - Commit: def456789                                          ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘│
                              │
                              │ <task-notification>
                              │ status: completed
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 4: VERIFICATION (可选，重要任务)                             │
│                                                                  │
│  决策: 重要修改，需要独立验证                                      │
│  策略: Spawn fresh verifier (新视角，无实现偏见)                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ AgentTool.call()                                             ││
│  │                                                              ││
│  │ subagent_type: "verification"                                ││
│  │ prompt: "Verify the null pointer fix in src/auth/validate.ts ││
│  │         commit def456789. Original issue: user.id access     ││
│  │         without null check. Check:                           ││
│  │         1. Code has proper null check                        ││
│  │         2. Tests cover the fix                               ││
│  │         3. No regressions                                    ││
│  │         End with VERDICT: PASS/FAIL/PARTIAL"                 ││
│  └─────────────────────────────────────────────────────────────┘│
 │                                                                  │
 ▼                                                                  │
┌─────────────────────────────────────────────────────────────────┐│
│                     Verification Agent                            ││
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              ││
│  │ 读取validate │  │ 检查测试覆盖  │  │ 边界测试     │              ││
│  │ .ts:42      │  │ grep null    │  │ user=null    │              ││
│  │ 确认null检查 │  │ check分支    │  │ user=undefined│             ││
│  └─────────────┘  └─────────────┘  └─────────────┘              ││
│       │                │                │                        ││
│       └────────────────┴────────────────┘                        ││
│                              │                                   ││
│                              ▼                                   ││
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ VERDICT: PASS                                                ││
│  │                                                              ││
│  │ ### Check: Null check exists                                 ││
│  │ **Command run:** grep -n "if (!user)" src/auth/validate.ts    ││
│  │ **Output observed:** 42:    if (!user) {                     ││
│  │ **Result: PASS**                                             ││
│  │                                                              ││
│  │ ### Check: Test coverage                                     ││
│  │ **Command run:** npm test -- auth.test.ts                    ││
│  │ **Output observed:** 3 passing                               ││
│  │ **Result: PASS**                                             ││
│  │                                                              ││
│  │ ### Check: Adversarial probe (boundary value)                ││
│  │ **Command run:** curl -H "Authorization: invalid" ...        ││
│  │ **Output observed:** 401 Session expired                     ││
│  │ **Result: PASS**                                             ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          向用户报告                                │
│                                                                  │
│  "已完成auth模块null pointer修复:                                  │
│                                                                  │
│   1. 在src/auth/validate.ts:42添加null检查                        │
│   2. 当user为null时返回401 'Session expired'                     │
│   3. 更新了相关测试                                               │
│   4. 所有测试通过 (3/3)                                           │
│   5. 提交: def456789                                              │
│                                                                  │
│   验证结果: VERDICT: PASS (所有检查通过)"                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.8 Agent通信时序图

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  User   │     │ Coordinator │     │  Worker-1   │     │  Worker-2   │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
     │                 │                   │                   │
     │  "Fix auth bug" │                   │                   │
     │────────────────▶│                   │                   │
     │                 │                   │                   │
     │                 │ AgentTool(desc="Research auth",        │
     │                 │              type="Explore",            │
     │                 │              prompt="...") ──────────────▶
     │                 │                   │                   │
     │                 │ AgentTool(desc="Find tests",            │
     │                 │              type="Explore",            │
     │                 │              prompt="...") ─────────────────────────▶
     │                 │                   │                   │
     │                 │                   │                   │
     │                 │                   │ (并行执行中)        │
     │                 │                   │                   │
     │                 │◀───────────────── │                   │
     │                 │ <task-notification>                   │
     │                 │ task-id: worker-1                     │
     │                 │ status: completed                     │
     │                 │ result: "Found null pointer..."       │
     │                 │                   │                   │
     │                 │◀──────────────────────────────────────│
     │                 │ <task-notification>                   │
     │                 │ task-id: worker-2                     │
     │                 │ status: completed                     │
     │                 │ result: "Found 3 test files..."       │
     │                 │                   │                   │
     │                 │ (综合结果，制定规范)                     │
     │                 │                   │                   │
     │                 │ SendMessage(to: worker-1,             │
     │                 │              message: "Fix...") ───────▶
     │                 │                   │                   │
     │                 │                   │ (执行修改)         │
     │                 │                   │                   │
     │                 │◀───────────────── │                   │
     │                 │ <task-notification>                   │
     │                 │ status: completed                     │
     │                 │ result: "Fixed. Commit: def456"       │
     │                 │                   │                   │
     │                 │ (决定验证)                               │
     │                 │                   │                   │
     │                 │ AgentTool(desc="Verify fix",           │
     │                 │              type="verification",       │
     │                 │              prompt="...") ─────────────▶
     │                 │                   │                   │
     │                 │                   │                   │
     │                 │◀───────────────── │                   │
     │                 │ <task-notification>                   │
     │                 │ status: completed                     │
     │                 │ result: "VERDICT: PASS"               │
     │                 │                   │                   │
     │  "修复完成，验证通过"  │                   │                   │
     │◀────────────────│                   │                   │
     │                 │                   │                   │
```

## 八、多Agent颜色系统

### 8.1 颜色定义

为可观察性设计（`src/tools/AgentTool/agentColorManager.ts`）：

```typescript
export type AgentColorName =
  | 'red' | 'blue' | 'green' | 'yellow'
  | 'purple' | 'orange' | 'pink' | 'cyan'

export const AGENT_COLORS: readonly AgentColorName[] = [
  'red', 'blue', 'green', 'yellow',
  'purple', 'orange', 'pink', 'cyan',
]

export const AGENT_COLOR_TO_THEME_COLOR = {
  red: 'red_FOR_SUBAGENTS_ONLY',
  blue: 'blue_FOR_SUBAGENTS_ONLY',
  green: 'green_FOR_SUBAGENTS_ONLY',
  // ... 其他颜色
}
```

### 8.2 颜色分配与管理

```typescript
export function setAgentColor(
  agentType: string,
  color: AgentColorName | undefined,
): void {
  const agentColorMap = getAgentColorMap()  // 全局颜色映射

  if (!color) {
    agentColorMap.delete(agentType)
    return
  }

  if (AGENT_COLORS.includes(color)) {
    agentColorMap.set(agentType, color)
  }
}

export function getAgentColor(agentType: string): keyof Theme | undefined {
  if (agentType === 'general-purpose') {
    return undefined  // 通用Agent无色
  }

  const agentColorMap = getAgentColorMap()
  const existingColor = agentColorMap.get(agentType)
  if (existingColor && AGENT_COLORS.includes(existingColor)) {
    return AGENT_COLOR_TO_THEME_COLOR[existingColor]
  }

  return undefined
}
```

### 8.3 UI展示效果

```
[蓝色] Explore 代理：发现 47 个 TypeScript 文件...
[绿色] Plan 代理：建议的重构方案是...
[红色] Verification 代理：VERDICT: PASS
[黄色] Worker-A：正在执行修改...
```

## 九、任务管理

### 9.0 后台任务生命周期流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      后台Agent任务完整生命周期                            │
└─────────────────────────────────────────────────────────────────────────┘

用户/父Agent
     │
     │ AgentTool.call()
     │ ├── description: "Long running analysis"
     │ ├── prompt: "Analyze all API endpoints..."
     │ └── run_in_background: true  ◀── 关键参数
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. AgentTool.call() 检测到 background=true                      │
│                                                                  │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ isBackgroundTasksDisabled ?                             │  │
│    │ ├── YES → 转为同步执行                                  │  │
│    │ └── NO  → 启动后台生命周期                               │  │
│    └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. runAsyncAgentLifecycle() 启动                                 │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ a. registerAsyncAgent() 注册任务                           │  │
│  │    ├── 生成 agentId                                       │  │
│  │    ├── 创建 LocalAgentTaskState                           │  │
│  │    │   ├── type: 'local_agent'                             │  │
│  │    │   ├── status: 'pending'                               │  │
│  │    │   ├── isBackgrounded: true                            │  │
│  │    │   └── progress: { toolUseCount: 0, ... }              │  │
│  │    └── 添加到 AppState.tasks                               │  │
│  │                                                            │  │
│  │ b. initTaskOutputAsSymlink() 创建输出文件                   │  │
│  │    └── outputFile: ~/.claude/tasks/{taskId}/output.jsonl   │  │
│  │                                                            │  │
│  │ c. 启动 runAgent() 在Promise中异步执行                      │  │
│  │    └── 不等待，立即返回                                     │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
     │
     │ 立即返回 (不等待执行完成)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 返回 async_launched 状态                                      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ {                                                        │   │
│  │   status: 'async_launched',                              │   │
│  │   agentId: 'agent-xyz-123',                              │   │
│  │   description: 'Long running analysis',                  │   │
│  │   prompt: 'Analyze all API endpoints...',                │   │
│  │   outputFile: '~/.claude/tasks/xyz/output.jsonl',        │   │
│  │   canReadOutputFile: true                                │   │
│  │ }                                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
     │
     │ (同时，后台任务开始执行)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 后台任务执行 (runAgent)                                       │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ while (turnCount < maxTurns):                              │  │
│  │                                                            │  │
│  │   a. query() 调用API                                       │  │
│  │      ├── 产出流式响应                                      │  │
│  │      └── 写入 output.jsonl                                │  │
│  │                                                            │  │
│  │   b. 遇到 tool_use → StreamingToolExecutor                │  │
│  │      ├── 执行工具                                          │  │
│  │      ├── 更新 progress                                     │  │
│  │      │   ├── toolUseCount++                               │  │
│  │      │   └── tokenCount += usage                          │  │
│  │      └── 更新 AppState.tasks[agentId].progress            │  │
│  │                                                            │  │
│  │   c. 写入工具结果到 output.jsonl                          │  │
│  │                                                            │  │
│  │   d. turnCount++                                          │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ 执行完成或出错                      │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 完成处理:                                                  │  │
│  │                                                            │  │
│  │ 成功: completeAsyncAgent()                                │  │
│  │ ├── status → 'completed'                                  │  │
│  │ ├── result → 最终结果                                      │  │
│  │ ├── finalMessage → 摘要                                    │  │
│  │ └── enqueueAgentNotification()                             │  │
│  │                                                            │  │
│  │ 失败: failAsyncAgent()                                    │  │
│  │ ├── status → 'failed'                                     │  │
│  │ ├── error → 错误信息                                       │  │
│  │ └── enqueueAgentNotification()                             │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 任务通知发送 (enqueueAgentNotification)                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 构建XML通知:                                              │   │
│  │                                                          │   │
│  │ <task-notification>                                      │   │
│  │   <task-id>agent-xyz-123</task-id>                       │   │
│  │   <status>completed</status>                             │   │
│  │   <summary>Agent "Long running analysis" completed</summary>│ │
│  │   <result>Found 47 endpoints... Total 234 issues</result> │   │
│  │   <usage>                                                │   │
│  │     <total_tokens>45231</total_tokens>                   │   │
│  │     <tool_uses>23</tool_uses>                            │   │
│  │     <duration_ms>125000</duration_ms>                    │   │
│  │   </usage>                                               │   │
│  │   <output-file>~/.claude/tasks/xyz/output.jsonl</output-file>││
│  │ </task-notification>                                     │   │
│  │                                                          │   │
│  │ 添加到消息队列 → 父Agent下轮可见                          │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 父Agent收到通知                                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 解析 <task-notification>:                                │   │
│  │                                                          │   │
│  │ if (status === 'completed') {                            │   │
│  │   // 读取结果                                             │   │
│  │   ReadTool.call(output-file) ◀── 可选，获取完整日志       │   │
│  │   // 或直接使用 <result> 摘要                             │   │
│  │ } else if (status === 'failed') {                        │   │
│  │   // 决定重试或报告用户                                    │   │
│  │   SendMessageTool.call(to: taskId, message: "Retry...")   │   │
│  │ }                                                         │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘


任务停止流程:

用户/父Agent
     │
     │ TaskStopTool.call()
     │ └── task_id: 'agent-xyz-123'
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ killAsyncAgent(taskId)                                          │
│                                                                  │
│ 1. 查找 AppState.tasks[taskId]                                  │
│ 2. 调用 abortController.abort('user_stopped')                   │
│ 3. 发送 kill信号到子进程 (如果是隔离模式)                         │
│ 4. 更新状态: status='killed'                                     │
│ 5. 发送通知: <task-notification>status='killed'</...>            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.1 任务状态定义

（`src/tasks/LocalAgentTask/LocalAgentTask.tsx:116-148`）：

```typescript
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean
  pendingMessages: string[]
  retain: boolean
  diskLoaded: boolean
  evictAfter?: number
}
```

### 9.2 进度追踪

```typescript
export type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number      // 最新输入token（累计）
  cumulativeOutputTokens: number // 累计输出token
  recentActivities: ToolActivity[]
}

export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: Message,
  resolveActivityDescription?: ActivityDescriptionResolver,
  tools?: Tools
): void {
  if (message.type !== 'assistant') return

  const usage = message.message.usage
  tracker.latestInputTokens = usage.input_tokens +
    (usage.cache_creation_input_tokens ?? 0) +
    (usage.cache_read_input_tokens ?? 0)
  tracker.cumulativeOutputTokens += usage.output_tokens

  for (const content of message.message.content) {
    if (content.type === 'tool_use') {
      tracker.toolUseCount++
      tracker.recentActivities.push({
        toolName: content.name,
        input: content.input,
        activityDescription: resolveActivityDescription?.(content.name, content.input),
      })
    }
  }
}
```

### 9.3 后台任务生命周期

```typescript
async function runAsyncAgentLifecycle(params): Promise<AsyncResult> {
  // 1. 注册任务
  const taskId = registerAsyncAgent({
    agentId,
    description,
    prompt,
    // ...
  })

  // 2. 启动后台执行
  const executionPromise = runAgent({
    agentDefinition,
    promptMessages,
    toolUseContext: subagentContext,
    isAsync: true,
    // ...
  })

  // 3. 处理结果
  executionPromise.then(
    result => completeAsyncAgent(taskId, result),
    error => failAsyncAgent(taskId, error)
  )

  // 4. 立即返回任务信息
  return {
    status: 'async_launched',
    agentId,
    description,
    outputFile: getTaskOutputPath(taskId),
  }
}
```

## 十、关键技术创新点

### 10.1 Prompt缓存优化

Fork机制的核心优化：

```typescript
// 所有fork子进程使用相同的placeholder结果
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
```

这使得多个子Agent共享API请求前缀，最大化缓存命中率。

### 10.2 能力安全隔离

通过`disallowedTools`而非沙箱实现轻量级隔离：
- 精确控制每个Agent的工具访问权限
- 禁止递归创建防止无限嵌套
- 验证Agent禁止修改文件确保公正性

### 10.3 渐进式上下文继承

三种继承模式：

| 模式 | 权限 | 消息历史 | 文件状态 |
|------|------|---------|---------|
| bubble | 继承 | 继承 | 继承 |
| reset | 新建 | 新建 | 克隆 |
| worktree | 继承 | 新建 | 隔离 |

### 10.4 实时进度流

通过`onProgress`回调实现实时进度更新：

```typescript
for await (const result of executor.getRemainingResults()) {
  onProgress?.({
    type: 'agent_progress',
    agentId,
    toolUseCount: tracker.toolUseCount,
    tokenCount: getTokenCountFromTracker(tracker),
    lastActivity: tracker.recentActivities[tracker.recentActivities.length - 1],
  })
  yield result.message
}
```

### 10.5 多Agent协作协议

基于XML的任务通知协议：

```xml
<task-notification>
  <task-id>agent-xyz-123</task-id>
  <status>completed</status>
  <summary>Agent "Investigate auth bug" completed</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  <usage>
    <total_tokens>15234</total_tokens>
    <tool_uses>15</tool_uses>
    <duration_ms>45000</duration_ms>
  </usage>
</task-notification>
```

## 十一、总结

Claude Code的多Agent系统通过以下技术实现高效协作：

1. **类型安全的Agent定义**：分层类型系统支持内置、自定义、插件三种Agent
2. **灵活的上下文隔离**：内存/Worktree/Remote三级隔离满足不同需求
3. **高效的并发控制**：StreamingToolExecutor实现智能的工具编排
4. **优雅的通信机制**：任务通知 + SendMessage实现父子/同级通信
5. **可观察的颜色系统**：8色方案支持多Agent场景的可视化区分
6. **协调器模式**：高级编排实现复杂任务的分解与合成

这套系统的核心设计哲学是：**在自主性与可控性之间找到平衡**——既让Agent能够并行高效工作，又保持用户对关键决策的控制权。

---

## 附录：完整系统交互总图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Claude Code 多Agent系统 - 完整架构                       │
└─────────────────────────────────────────────────────────────────────────────────┘

                                    用户层
                                      │
    ┌─────────────────────────────────┼─────────────────────────────────┐
    │                                 │                                 │
    ▼                                 ▼                                 ▼
┌──────────┐                   ┌──────────────┐                   ┌──────────┐
│  CLI/TUI │                   │  SDK/API     │                   │  IDE插件  │
│  交互终端 │                   │  程序化接口   │                   │  VSCode  │
└────┬─────┘                   └──────┬───────┘                   └────┬─────┘
     │                                │                                │
     └────────────────────────────────┼────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              调度与协调层                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  AgentTool (src/tools/AgentTool/AgentTool.tsx)                          │   │
│  │  ├── 输入解析: description, prompt, subagent_type, model, isolation     │   │
│  │  ├── 类型选择: Fork(默认) | Built-in | Custom                           │   │
│  │  ├── 权限检查: canUseTool()                                             │   │
│  │  └── 执行决策: Sync | Async | Teammate | Remote                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│                    ┌─────────────────┼─────────────────┐                        │
│                    │                 │                 │                        │
│                    ▼                 ▼                 ▼                        │
│  ┌───────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐       │
│  │  Coordinator Mode     │ │  Standard Mode   │ │  Background Mode     │       │
│  │  (协调器模式)          │ │  (标准Fork模式)   │ │  (后台任务模式)       │       │
│  │                       │ │                  │ │                      │       │
│  │ ┌───────────────────┐ │ │ ┌──────────────┐ │ │ ┌──────────────────┐ │       │
│  │ │ 任务分解与合成     │ │ │ │ 内存隔离      │ │ │ │ 异步执行          │ │       │
│  │ │ Worker池管理       │ │ │ │ 上下文继承    │ │ │ │ 任务通知          │ │       │
│  │ │ SendMessage通信    │ │ │ │ Prompt缓存   │ │ │ │ 进度追踪          │ │       │
│  │ └───────────────────┘ │ │ └──────────────┘ │ │ └──────────────────┘ │       │
│  └───────────────────────┘ └──────────────────┘ └──────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Agent执行层                                          │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ runAgent() (src/tools/AgentTool/runAgent.ts)                            │   │
│  │                                                                         │   │
│  │ 初始化:                                                                 │   │
│  │ ├── createSubagentContext() 创建隔离上下文                               │   │
│  │ │   ├── agentId (新UUID)                                                │   │
│  │ │   ├── messages (fork/empty)                                           │   │
│  │ │   ├── readFileState (clone)                                           │   │
│  │ │   └── permissionContext (inherit/new)                                 │   │
│  │ ├── initializeAgentMcpServers() 启动专属MCP                             │   │
│  │ ├── assembleToolPool() 组装工具集                                       │   │
│  │ └── buildEffectiveSystemPrompt() 构建系统提示                            │   │
│  │                                                                         │   │
│  │ 执行循环 (while turnCount < maxTurns):                                  │   │
│  │ └── query() ◀─────────────────────────────────────────────────────┐    │   │
│  │     │                                                              │    │   │
│  │     ▼                                                              │    │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐│    │   │
│  │ │                    StreamingToolExecutor                         ││    │   │
│  │ │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  ││    │   │
│  │ │  │ 工具队列    │  │ 并发控制    │  │ 错误处理    │  │ 进度产出  │  ││    │   │
│  │ │  │ queued     │  │ 安全并行    │  │ Bash级联    │  │ onProgress│  ││    │   │
│  │ │  │ executing  │  │ 独占串行    │  │ 取消兄弟     │  │ 实时流    │  ││    │   │
│  │ │  │ completed  │  │            │  │            │  │          │  ││    │   │
│  │ │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  ││    │   │
│  │ └─────────────────────────────────────────────────────────────────┘│    │   │
│  │     │                                                              │    │   │
│  │     │ 产出结果                                                      │    │   │
│  │     ▼                                                              │    │   │
│  │ 工具执行 ◀─────────────────────────────────────────────────────────┘    │   │
│  │     │                                                                    │   │
│  │     ▼                                                                    │   │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │ │Read      │ │Edit      │ │Bash      │ │Grep/Glob │ │Agent     │ ...   │   │
│  │ │(安全并行)│ │(独占)    │ │(独占)    │ │(安全并行)│ │(后台)    │       │   │
│  │ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  │                                                                         │   │
│  │ 完成:                                                                   │   │
│  │ ├── 更新AppState.tasks[taskId].result                                  │   │
│  │ ├── 写入output.jsonl                                                   │   │
│  │ └── enqueueAgentNotification() 发送通知                                 │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              状态管理层                                           │
│                                                                                   │
│  ┌──────────────────────────┐  ┌──────────────────────────┐  ┌──────────────┐  │
│  │    AppState (React)      │  │   Bootstrap State        │  │  持久化存储   │  │
│  │    (会话级)               │  │   (全局单例)              │  │              │  │
│  │                          │  │                          │  │              │  │
│  │ ┌──────────────────────┐ │  │ ┌──────────────────────┐ │  │ ┌──────────┐ │  │
│  │ │ tasks: {             │ │  │ │ agentColorMap        │ │  │ │ output/  │ │  │
│  │ │   [taskId]: {        │ │  │ │ sessionId            │ │  │ │ jsonl    │ │  │
│  │ │     type: 'local_agent'│ │  │ │ totalCost            │ │  │ │ metadata │ │  │
│  │ │     agentId          │ │  │ │ projectRoot          │ │  │ │ transcript││  │
│  │ │     status           │ │  │ └──────────────────────┘ │  │ └──────────┘ │  │
│  │ │     progress         │ │  │                          │  │              │  │
│  │ │     messages         │ │  │                          │  │              │  │
│  │ │   }                  │ │  │                          │  │              │  │
│  │ │ }                    │ │  │                          │  │              │  │
│  │ └──────────────────────┘ │  │                          │  │              │  │
│  └──────────────────────────┘  └──────────────────────────┘  └──────────────┘  │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘


关键数据流:

1. 创建Agent:
   AgentTool.call() → runAgent() → createSubagentContext() → query()循环

2. 工具执行:
   query() → StreamingToolExecutor.addTool() → processQueue() → executeTool()
         → 并发控制(安全/独占) → onProgress()实时更新

3. 后台任务:
   runAsyncAgentLifecycle() → registerAsyncAgent() → runAgent()在Promise中
                           → 立即返回async_launched → 完成后enqueueAgentNotification()

4. 任务通知:
   enqueueAgentNotification() → 构建<task-notification>XML → 添加到消息队列
                             → 父Agent下轮query()收到 → 解析并处理

5. 进度追踪:
   Tool执行 → updateProgressFromMessage() → 更新AppState.tasks[taskId].progress
          → onProgress()回调 → 写入output.jsonl → UI实时显示


核心文件索引:

| 文件路径 | 功能 | 关键导出 |
|---------|------|---------|
| src/tools/AgentTool/AgentTool.tsx | Agent工具入口 | AgentTool, inputSchema |
| src/tools/AgentTool/runAgent.ts | Agent执行核心 | runAgent |
| src/tools/AgentTool/forkSubagent.ts | Fork机制 | buildForkedMessages, FORK_AGENT |
| src/tools/AgentTool/loadAgentsDir.ts | Agent定义加载 | AgentDefinition, loadAgentsDir |
| src/tools/AgentTool/builtInAgents.ts | 内置Agent | getBuiltInAgents |
| src/tools/AgentTool/agentColorManager.ts | 颜色管理 | AGENT_COLORS, setAgentColor |
| src/services/tools/StreamingToolExecutor.ts | 并发执行 | StreamingToolExecutor |
| src/coordinator/coordinatorMode.ts | 协调器模式 | isCoordinatorMode, getCoordinatorSystemPrompt |
| src/tasks/LocalAgentTask/LocalAgentTask.tsx | 后台任务 | runAsyncAgentLifecycle, enqueueAgentNotification |
| src/query.ts | QueryEngine | query, QueryParams |
```
