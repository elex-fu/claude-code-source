# 第 17 章：多代理架构

> 一个 Claude 能做很多事，多个 Claude 能做更多事。

---

## 17.1 为什么需要多代理

单个 Agent 有几个固有限制：

**上下文限制**：一个 Agent 的上下文窗口是有限的。对于超大型任务（分析整个代码库、处理大量文件），单个 Agent 无法在一个上下文中完成。

**并行限制**：单个 Agent 是串行的，一次只能做一件事。对于可以并行的任务（同时分析多个模块、同时运行多个测试），单个 Agent 效率低下。

**专业化限制**：不同的任务需要不同的专业知识和工具集。一个通用 Agent 不如多个专业 Agent 的组合。

多代理架构解决了这三个问题。

---

## 17.2 Claude Code 的三种代理模式

![多代理架构：三种模式](../diagrams/multi-agent-modes.svg)

### 模式一：子代理（AgentTool）

父代理启动一个独立的子代理，子代理有自己的上下文和工具集：

```typescript
// 父代理调用 AgentTool
await AgentTool.execute({
  description: '分析 src/auth/ 模块',
  prompt: '详细分析认证模块的代码结构、安全性和潜在问题',
  subagent_type: 'Explore',  // 专门用于代码探索的代理类型
  model: 'claude-opus-4-6',  // 可以为子代理指定不同的模型
}, context)
```

子代理的特点：
- **独立上下文**：子代理有自己的消息历史，不占用父代理的上下文
- **独立工具集**：可以为子代理配置不同的工具
- **结果汇报**：子代理完成后，结果返回给父代理

### 模式二：后台代理（run_in_background）

子代理在后台运行，父代理不等待结果：

```typescript
await AgentTool.execute({
  description: '后台分析安全漏洞',
  prompt: '...',
  run_in_background: true,  // 后台运行
}, context)
// 立即返回，不等待子代理完成
```

### 模式三：Worktree 隔离代理

子代理在独立的 Git Worktree 中运行，完全隔离：

```typescript
await AgentTool.execute({
  description: '在独立分支上实验新功能',
  prompt: '...',
  isolation: 'worktree',  // 创建独立 worktree
}, context)
```

---

## 17.3 代理类型系统

Claude Code 定义了多种专业化代理类型：

```typescript
// src/tools/AgentTool/loadAgentsDir.ts
type AgentDefinition = {
  name: string
  description: string
  systemPrompt: string
  tools: string[]        // 该代理可用的工具列表
  model?: string         // 默认模型
}
```

内置代理类型：
- **general-purpose**：通用代理，有完整工具集
- **Explore**：代码探索专用，只有读取工具（无写入权限）
- **Plan**：规划专用，只能生成计划，不能执行

用户也可以在 `.claude/agents/` 目录中定义自定义代理类型。

---

## 17.4 团队协作模式

`TeamCreateTool` 和 `SendMessageTool` 实现了更复杂的多代理协作：

```typescript
// 创建一个代理团队
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

团队模式适合需要多角色协作的复杂任务：
- 架构师设计，开发者实现，审查者检查
- 前端代理和后端代理并行开发
- 测试代理和开发代理协作

---

## 17.5 代理间的上下文共享

多代理系统面临一个核心问题：**代理之间如何共享信息？**

Claude Code 使用了几种机制：

**文件系统共享**：最简单的方式，代理通过读写文件交换信息。

```
代理 A 写入：/tmp/analysis_result.md
代理 B 读取：/tmp/analysis_result.md
```

**消息传递**：通过 `SendMessageTool` 直接传递消息。

**任务输出**：父代理通过 `TaskOutputTool` 读取子代理的输出。

**共享状态**：`in_process_teammate` 类型的代理共享同一个 AppState。

---

## 17.6 代理的颜色系统

Claude Code 有一个有趣的设计：每个代理有一个颜色标识（`src/tools/AgentTool/agentColorManager.ts`）：

```typescript
type AgentColorName =
  | 'blue' | 'green' | 'yellow' | 'red'
  | 'cyan' | 'magenta' | 'white'
```

在 UI 中，不同代理的输出用不同颜色显示，让用户一眼就能区分哪个代理在说话：

```
[蓝色] 主代理：我来分析这个项目...
[绿色] 子代理（Explore）：发现了 47 个 TypeScript 文件...
[黄色] 子代理（Plan）：建议的重构方案是...
```

这是一个小细节，但对多代理场景的可观察性很重要。

---

## 17.7 代理的资源限制

为了防止代理失控，Claude Code 对代理有资源限制：

```typescript
type QueryEngineConfig = {
  maxTurns?: number        // 最大轮次（防止无限循环）
  maxBudgetUsd?: number    // 最大费用（防止意外高额账单）
  taskBudget?: { total: number }  // token 预算
}
```

父代理可以为子代理设置更严格的限制：

```typescript
// 子代理只允许 10 轮对话，最多花费 $0.5
await AgentTool.execute({
  prompt: '...',
  maxTurns: 10,
  maxBudgetUsd: 0.5,
}, context)
```

---

## 17.8 多代理的调试挑战

多代理系统的调试比单代理复杂得多：

**问题 1：哪个代理出了问题？**
解决：每个代理有唯一 ID 和颜色，日志中标注代理来源。

**问题 2：代理间的通信是否正确？**
解决：`SendMessageTool` 的调用记录在对话历史中，可以追溯。

**问题 3：子代理的上下文是什么？**
解决：子代理的完整对话历史保存在任务输出文件中。

**问题 4：代理是否在循环？**
解决：`maxTurns` 限制防止无限循环，超出时自动终止。

---

## 17.9 多代理的适用场景

多代理不是万能的，有其适用场景：

**适合多代理**：
- 任务可以明确分解为独立子任务
- 子任务可以并行执行
- 不同子任务需要不同的专业知识
- 任务超出单个上下文窗口

**不适合多代理**：
- 任务高度依赖顺序执行
- 子任务之间有复杂的依赖关系
- 任务简单，多代理的协调开销大于收益

---

## 17.10 小结

Claude Code 的多代理架构提供了三种协作模式：

- **子代理**：独立上下文，结果汇报给父代理
- **后台代理**：异步执行，不阻塞父代理
- **团队协作**：多代理通过消息传递协作

关键设计：颜色系统（可观察性）、资源限制（安全性）、多种共享机制（灵活性）。

---

*下一章：[协调器模式](./18-coordinator.md)*
