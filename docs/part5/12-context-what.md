# 第 12 章：什么是 Context Engineering

> "Context is everything. Without context, words and actions have no meaning at all."
> —— Gregory Bateson

---

## 12.1 一个思想实验

想象你雇了一个新员工，第一天上班，你直接把他扔进一个复杂项目，什么都不解释：

- 不告诉他项目背景
- 不告诉他代码规范
- 不告诉他团队约定
- 不告诉他当前状态

他能工作吗？能，但效率极低，错误极多。

现在换一种方式：给他一份详细的入职文档，包含项目背景、技术栈、代码规范、常见问题、当前任务状态。

同样的人，同样的能力，但有了上下文，工作质量天壤之别。

**这就是 Context Engineering 的核心思想：给 AI 提供正确的上下文，是决定 AI 输出质量的关键因素。**

---

## 12.2 什么是 Context Engineering

Context Engineering 是指**系统性地设计、构建和管理 AI 模型的输入上下文**，以最大化模型的输出质量。

它包含三个维度：

**内容维度**：给 AI 什么信息？
- 项目背景（CLAUDE.md）
- 当前状态（git status、文件内容）
- 历史记录（对话历史）
- 外部知识（文档、搜索结果）

**结构维度**：如何组织这些信息？
- 系统提示 vs 用户消息
- 信息的顺序和优先级
- 格式（纯文本、Markdown、XML）

**管理维度**：如何在有限的 token 窗口内管理信息？
- 压缩历史（auto-compact）
- 过滤无关信息
- 动态注入相关信息

---

## 12.3 为什么 Context Engineering 如此重要

LLM 的能力是固定的（由训练决定），但输出质量是可变的（由上下文决定）。

一个简单的实验：

```
# 没有上下文
用户：修复这个 bug

Claude：我需要更多信息...
```

```
# 有上下文
系统提示：你是一个 TypeScript 专家，正在维护一个 React 应用。
          代码规范：使用函数式组件，避免 any 类型。
用户：修复这个 bug
[附上错误信息和相关代码]

Claude：[直接给出精确的修复方案]
```

同样的模型，同样的问题，但有了上下文，输出质量完全不同。

---

## 12.4 Claude Code 的上下文架构

Claude Code 的上下文由多个层次组成：

![Context Engineering：上下文的组成](../diagrams/context-engineering.svg)

每一层都有其作用，缺少任何一层都会降低输出质量。

---

## 12.5 上下文的 token 经济学

Claude 的上下文窗口是有限的（目前最大约 200K tokens）。每个 token 都是宝贵的资源。

**token 的分配**：

```
总 token 预算（200K）
├── 系统提示：~5K（工具定义 + 核心指令）
├── 用户上下文（CLAUDE.md）：~2K（取决于文件大小）
├── 系统上下文（git status 等）：~1K
├── 对话历史：~100K（随对话增长）
├── 当前工具结果：~50K（可能很大）
└── 输出预算：~40K（Claude 的响应）
```

当对话历史增长到接近限制时，需要压缩（compact）。这是 Context Engineering 中最复杂的问题之一。

---

## 12.6 上下文质量的四个维度

**相关性**：上下文是否与当前任务相关？
- 好：当前文件的内容、相关的错误信息
- 差：无关的历史对话、不相关的文件内容

**准确性**：上下文是否准确反映当前状态？
- 好：最新的 git status、实时的文件内容
- 差：过时的信息、缓存的旧状态

**完整性**：上下文是否包含必要的信息？
- 好：包含所有相关的代码、配置、约束
- 差：缺少关键信息，导致 Claude 做出错误假设

**简洁性**：上下文是否简洁，没有冗余？
- 好：精炼的项目说明、关键信息的摘要
- 差：冗长的文档、重复的信息

---

## 12.7 Claude Code 的 Context Engineering 实践

Claude Code 在 Context Engineering 上做了大量工程工作：

**自动注入 git 状态**（`src/context.ts`）：
```typescript
// 每次对话开始时自动获取 git 状态
const gitStatus = await getGitStatus()
// 包含：当前分支、主分支、最近 5 条 commit、工作区状态
```

**CLAUDE.md 自动发现**：
```typescript
// 从当前目录向上查找 CLAUDE.md
// 支持多级目录（项目级、子目录级）
const claudeMds = getClaudeMds(await getMemoryFiles())
```

**自动压缩**（auto-compact）：
```typescript
// 当 token 使用量超过阈值时自动压缩
if (isAutoCompactEnabled() && tokenUsage > threshold) {
  await compactConversation(messages)
}
```

**动态 Memory 注入**：
```typescript
// 根据当前任务动态加载相关的 Memory 文件
const relevantMemories = await findRelevantMemories(currentTask)
```

---

## 12.8 Context Engineering vs Prompt Engineering

很多人混淆 Context Engineering 和 Prompt Engineering。它们的区别：

| 维度 | Prompt Engineering | Context Engineering |
|------|-------------------|---------------------|
| 关注点 | 如何写好单个提示 | 如何管理整个上下文 |
| 范围 | 单次交互 | 整个会话生命周期 |
| 动态性 | 静态（提前写好） | 动态（运行时构建） |
| 工程复杂度 | 低 | 高 |
| 影响范围 | 单次响应质量 | 整体系统能力 |

Prompt Engineering 是 Context Engineering 的子集。Claude Code 的工程重心在 Context Engineering，而不只是写好系统提示。

---

## 12.9 小结

Context Engineering 是 AI Agent 系统工程的核心挑战之一。

关键洞察：
- **上下文决定输出质量**，不只是模型能力
- **上下文是有限资源**，需要精心管理
- **上下文需要动态构建**，不是静态配置
- **Context Engineering 是系统工程**，不只是写提示

接下来的三章，我们深入 Claude Code 的具体实现：系统提示的构建、CLAUDE.md 的设计、以及自动压缩机制。

---

*下一章：[系统提示的构建艺术](./13-system-prompt.md)*
