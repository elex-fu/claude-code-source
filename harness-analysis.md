# Claude Code Harness 能力深度分析

## 执行摘要

Claude Code 作为生产级 AI Agent 系统，其 harness 能力并非单一模块，而是**分布在整个架构中的系统性设计**。与 Anthropic Labs 文章描述的实验性三 Agent（Planner-Generator-Evaluator）架构不同，Claude Code 采用了更成熟的**分层 harness 模型**：

- **执行层 harness**：StreamingToolExecutor 实现工具级并发与隔离
- **会话层 harness**：AgentTool 实现多代理协作与上下文管理
- **系统层 harness**：AutoCompact + 权限系统实现长期运行的可持续性

---

## 一、Harness 核心实现架构

### 1.1 与参考文档的对比分析

| 维度 | 参考文档（实验性） | Claude Code（生产级） |
|------|------------------|---------------------|
| **架构模式** | 三 Agent 分离（Planner/Generator/Evaluator） | 统一 QueryEngine + 可配置 Agent 类型 |
| **评估机制** | 独立 Evaluator Agent + Playwright MCP | 内置 Verification Agent + 用户确认层 |
| **上下文管理** | 结构化制品传递 | 双层状态（Bootstrap + AppState）+ 自动压缩 |
| **生成-评估分离** | GAN 风格对抗架构 | 同一代码库内的角色分离（Verification Agent） |
| **制品传递** | Markdown 规格 + JSON 任务文件 | 内存状态 + 文件系统快照 + Git 集成 |

### 1.2 Claude Code 的三层 Harness 模型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Harness 架构总览                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      系统层 Harness                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ AutoCompact  │  │ 5层权限系统   │  │ 成本追踪     │              │   │
│  │  │ (长期运行)   │  │ (安全边界)   │  │ (资源控制)   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                   │                                         │
│                                   ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      会话层 Harness                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ AgentTool    │  │ 多代理协作    │  │ 记忆系统     │              │   │
│  │  │ (子代理管理) │  │ (Team/颜色)  │  │ (跨会话)     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                   │                                         │
│                                   ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      执行层 Harness                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ StreamingTool│  │ 并发控制     │  │ 错误级联     │              │   │
│  │  │ Executor     │  │ (安全/独占)  │  │ (Bash取消)   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、执行层 Harness：StreamingToolExecutor

### 2.1 核心实现分析

**源码位置**：`src/services/tools/StreamingToolExecutor.ts`（531 行）

与参考文档中 Generator 和 Evaluator 分离的思路不同，Claude Code 在**单一执行器内实现了生成-执行的统一调度**：

```typescript
// 关键设计：状态机驱动的工具生命周期
private tools: TrackedTool[] = []

// 并发控制逻辑
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

### 2.2 与参考文档的对比

| 参考文档设计 | Claude Code 实现 | 优势 |
|-------------|-----------------|------|
| Generator 产出代码 | Tool.call() 执行操作 | 统一接口，所有工具遵循相同契约 |
| Evaluator 独立评分 | 工具结果直接返回模型 | 模型自主评估，无需额外 Agent |
| 制品传递 | 内存状态 + 流式产出 | 实时反馈，无序列化开销 |

### 2.3 错误级联机制（对应参考文档的"熔断"）

```typescript
// 仅 Bash 错误取消兄弟进程
if (tool.block.name === BASH_TOOL_NAME && isErrorResult) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

**设计洞察**：相比参考文档的"生成器-评估器"循环，Claude Code 采用**即时反馈模式**——工具执行结果立即返回模型，由模型自主决定下一步行动。

---

## 三、会话层 Harness：AgentTool 多代理系统

### 3.1 Agent 定义系统

**源码位置**：`src/tools/AgentTool/loadAgentsDir.ts`

Claude Code 内置了**6 种预定义 Agent 类型**，对应参考文档中的角色分离：

```typescript
// 内置 Agent 类型（builtInAgents.ts）
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  whenToUse: '验证实现工作是否正确...',
  color: 'red',
  background: true,  // 后台运行
  disallowedTools: [
    AGENT_TOOL_NAME,        // 禁止递归创建代理
    FILE_EDIT_TOOL_NAME,    // 禁止修改文件
    FILE_WRITE_TOOL_NAME,   // 禁止写入文件
  ],
  // ...
}
```

| Agent 类型 | 角色定位 | 对应参考文档角色 | 能力限制 |
|-----------|---------|----------------|---------|
| **general-purpose** | 通用执行 | Generator | 完整工具集 |
| **Explore** | 代码探索 | - | 只读工具 |
| **Plan** | 规划生成 | Planner | 只读 + 计划工具 |
| **verification** | 结果验证 | Evaluator | 禁止修改文件 |
| **Test** | 测试执行 | Evaluator | 测试相关工具 |
| **Documentation** | 文档生成 | - | 文档工具 |

### 3.2 上下文隔离机制

与参考文档"结构化制品传递"不同，Claude Code 实现了**内存级上下文隔离**：

```typescript
// forkSubagent.ts - 子代理上下文创建
export async function createSubagentContext(
  parentContext: ToolUseContext,
  agentDefinition: AgentDefinition,
): Promise<ToolUseContext> {
  return {
    ...parentContext,
    // 隔离关键状态
    agentId: createAgentId(),
    messages: [],  // 独立消息历史
    readFileState: cloneFileStateCache(parentContext.readFileState),
    // 权限可继承或重置
    toolPermissionContext: agentDefinition.permissionMode === 'bubble'
      ? parentContext.toolPermissionContext
      : createNewPermissionContext(),
  }
}
```

### 3.3 与参考文档的 Evaluator 对比

**参考文档的 Evaluator**：
- 独立 Agent，使用 Playwright MCP 进行端到端测试
- 明确的通过/失败阈值
- 循环反馈直到通过或达到最大迭代

**Claude Code 的 Verification Agent**：
- 内置 Agent 类型，通过 `AgentTool` 调用
- 禁止文件修改（`disallowedTools` 限制）
- 输出结构化报告（`VERDICT: PASS/FAIL/PARTIAL`）
- 必须由用户触发（非自动循环）

**关键差异**：Claude Code 将评估作为**用户可控的选项**，而非自动循环的一部分，这降低了误报风险。

---

## 四、系统层 Harness：长期运行支持

### 4.1 AutoCompact 自动压缩

**源码位置**：`src/services/compact/autoCompact.ts`

对应参考文档的"上下文重置"策略，但实现更精细：

```typescript
// 自动压缩触发条件
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 默认 13K 缓冲
}

// 熔断机制
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
if (consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { type: 'compact_failed', reason: 'circuit_breaker' }
}
```

| 参考文档策略 | Claude Code 实现 | 说明 |
|-------------|-----------------|------|
| 显式上下文重置 | AutoCompact 自动触发 | 无需用户干预 |
| 结构化制品传递 | 内存摘要替换 | 更快的状态切换 |
| 冲刺边界 | Token 阈值驱动 | 更细粒度的控制 |

### 4.2 ABLATION_BASELINE：科学实验支持

**源码位置**：`src/entrypoints/cli.tsx`（第 16-26 行）

这是 Claude Code 最具特色的 harness 能力——**内置科学实验框架**：

```typescript
// Harness-science L0 ablation baseline
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE',
    'CLAUDE_CODE_DISABLE_THINKING',
    'DISABLE_INTERLEAVED_THINKING',
    'DISABLE_COMPACT',
    'DISABLE_AUTO_COMPACT',
    'CLAUDE_CODE_DISABLE_AUTO_MEMORY',
    'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS'
  ]) {
    process.env[k] ??= '1'
  }
}
```

**能力分析**：
- 通过环境变量启用"消融基线"模式
- 可独立禁用思考、压缩、记忆、后台任务等功能
- 用于对比实验，评估各组件对整体性能的贡献

这与参考文档的"模型能力决定 harness 复杂度"观点完全一致——Claude Code 内置了评估自身 harness 效果的工具。

---

## 五、多代理协作 Harness

### 5.1 TeamCreateTool：团队协作

**源码位置**：`src/tools/TeamCreateTool/`

实现参考文档提到的"多 Agent 架构"：

```typescript
// 创建代理团队
await TeamCreateTool.execute({
  members: [
    { name: 'architect', role: '系统架构师，负责设计方案' },
    { name: 'developer', role: '开发者，负责实现' },
    { name: 'reviewer', role: '代码审查者，负责质量把关' },
  ]
}, context)

// 代理间通信
await SendMessageTool.execute({
  to: 'developer',
  message: '架构方案已确定，请开始实现 UserService'
}, context)
```

### 5.2 颜色系统：可观察性支持

```typescript
// agentColorManager.ts
export type AgentColorName = 'red' | 'blue' | 'green' | 'yellow' | 'purple' | 'orange' | 'pink' | 'cyan'

// UI 中区分不同代理的输出
[蓝色] Explore 代理：发现 47 个 TypeScript 文件...
[绿色] Plan 代理：建议的重构方案是...
[黄色] Review 代理：发现潜在的性能问题...
```

这与参考文档强调的"可观察性"一致，但实现更轻量。

---

## 六、Git 作为状态管理

### 6.1 Worktree 隔离

**源码位置**：`src/tools/AgentTool/loadAgentsDir.ts`（第 94-97 行）

```typescript
isolation: (process.env.USER_TYPE === 'ant'
  ? z.enum(['worktree', 'remote'])
  : z.enum(['worktree'])
).optional(),
```

Agent 可配置在独立 Git worktree 中运行，实现参考文档提到的"可靠性和可恢复性"。

### 6.2 与参考文档的对比

| 参考文档 | Claude Code | 说明 |
|---------|------------|------|
| Git 提交每个冲刺 | Agent 可选择 worktree 隔离 | 更细粒度的隔离 |
| 评估器检查代码状态 | 用户通过 /diff 查看变更 | 用户主导而非自动 |
| 回滚到已知良好提交 | 用户手动 git 操作 | 更灵活但需用户参与 |

---

## 七、关键洞察：Harness 作为产品

### 7.1 模型与 Harness 的协同演进

从源代码可见，Claude Code 的 harness 经历了与参考文档描述类似的演进：

```
早期版本（Sonnet 4.5 时代）
  ├── 需要显式上下文重置
  ├── 依赖冲刺边界
  └── 复杂的权限协商

当前版本（Opus 4.6/Sonnet 4.6）
  ├── AutoCompact 自动处理上下文
  ├── 百万 Token 上下文减少重置需求
  └── 更简洁的权限模型
```

### 7.2 生产级 vs 实验性 Harness 的关键差异

| 维度 | 参考文档（实验性） | Claude Code（生产级） |
|------|------------------|---------------------|
| **目标** | 最大化自主性 | 平衡自主性与可控性 |
| **评估** | 自动循环直到通过 | 用户触发 Verification Agent |
| **错误处理** | 自动重试 | 用户确认后重试 |
| **成本** | 可接受高成本（$200/6小时） | 内置成本追踪和预算控制 |
| **安全** | 依赖评估器 | 5层权限 + 用户确认 |

### 7.3 Harness 的核心设计原则（从源码提炼）

1. **渐进式自主**：从辅助工具到半自主 Agent，再到全自主（通过 `--dangerously-skip-permissions`）

2. **用户始终可控**：所有危险操作需用户确认，评估结果需用户决策

3. **失败可恢复**：AutoCompact、Git worktree、会话持久化确保状态不丢失

4. **可观测内置**：颜色系统、成本追踪、性能指标、详细日志

5. **科学实验友好**：ABLATION_BASELINE 支持系统化评估各组件贡献

---

## 八、结论

Claude Code 的 harness 能力是一个**分层、渐进、用户可控**的系统：

- **执行层**：StreamingToolExecutor 实现工具级并发和错误隔离
- **会话层**：AgentTool + 多代理系统实现角色分离（Explore/Plan/Verification）
- **系统层**：AutoCompact + 权限系统确保长期运行的可持续性

与参考文档的实验性三 Agent 架构相比，Claude Code 选择了**更保守但更安全**的设计：
- 不追求完全自主的生成-评估循环
- 将关键决策（权限、评估、重试）保留给用户
- 通过工具限制（`disallowedTools`）而非 Agent 分离来实现安全边界

这种设计反映了**生产级 AI 系统的核心权衡**：自主性 vs 可控性。Claude Code 选择了后者，这也是其能够成为商业产品的基础。

---

## 附录：关键源码文件

| 文件 | 行数 | 功能 |
|------|------|------|
| `src/services/tools/StreamingToolExecutor.ts` | 531 | 工具执行编排 |
| `src/tools/AgentTool/loadAgentsDir.ts` | 756 | Agent 定义加载 |
| `src/tools/AgentTool/runAgent.ts` | 400+ | Agent 执行逻辑 |
| `src/tools/AgentTool/forkSubagent.ts` | 200+ | 子代理分叉 |
| `src/tools/AgentTool/builtInAgents.ts` | 150+ | 内置 Agent 定义 |
| `src/services/compact/autoCompact.ts` | 300+ | 自动压缩 |
| `src/entrypoints/cli.tsx` | 200+ | 入口与实验框架 |
