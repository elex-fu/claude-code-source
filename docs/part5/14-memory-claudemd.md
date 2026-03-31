# 第 14 章：Memory 与 CLAUDE.md

> CLAUDE.md 是你给 Claude 的"项目说明书"，Memory 系统是 Claude 的"长期记忆"。

---

## 14.1 AI 的记忆问题

LLM 天生是无状态的：每次 API 调用都是独立的，模型不记得上次对话的内容。

这对 AI 编程助手来说是个大问题：
- 每次新会话，Claude 都不知道项目背景
- 每次都需要重新解释代码规范
- 每次都需要重新说明团队约定

Claude Code 通过两种机制解决这个问题：**CLAUDE.md**（显式记忆）和 **Memory 系统**（自动记忆）。

---

## 14.2 CLAUDE.md：显式项目记忆

CLAUDE.md 是一个 Markdown 文件，放在项目根目录（或任何子目录），Claude Code 每次启动时自动读取。

**发现机制**（`src/utils/claudemd.ts`）：

```typescript
// 从当前目录向上查找 CLAUDE.md
async function getMemoryFiles(): Promise<string[]> {
  const files = []
  let dir = cwd

  while (dir !== path.dirname(dir)) {  // 直到根目录
    const claudeMd = path.join(dir, 'CLAUDE.md')
    if (await fileExists(claudeMd)) {
      files.push(claudeMd)
    }
    dir = path.dirname(dir)
  }

  // 也检查 ~/.claude/CLAUDE.md（全局配置）
  const globalClaudeMd = path.join(os.homedir(), '.claude', 'CLAUDE.md')
  if (await fileExists(globalClaudeMd)) {
    files.push(globalClaudeMd)
  }

  return files
}
```

**多级 CLAUDE.md**：
```
~/.claude/CLAUDE.md          # 全局配置（所有项目共享）
/project/CLAUDE.md           # 项目级配置
/project/src/CLAUDE.md       # 子目录级配置（更具体）
/project/src/auth/CLAUDE.md  # 模块级配置（最具体）
```

所有找到的 CLAUDE.md 都会被读取，按从全局到具体的顺序合并。

---

## 14.3 CLAUDE.md 的最佳实践

一个好的 CLAUDE.md 应该包含：

```markdown
# 项目名称

## 项目概述
简短描述项目是什么、做什么。

## 技术栈
- 语言：TypeScript 5.x
- 框架：Next.js 14
- 数据库：PostgreSQL + Prisma
- 测试：Jest + Testing Library

## 目录结构
- src/app/        Next.js App Router 页面
- src/components/ 可复用组件
- src/lib/        工具函数和服务
- src/types/      TypeScript 类型定义

## 代码规范
- 使用函数式组件，不用 class 组件
- 所有函数必须有 TypeScript 类型
- 禁止使用 any 类型
- 文件命名：kebab-case
- 组件命名：PascalCase

## 常用命令
- npm run dev      启动开发服务器（端口 3000）
- npm test         运行测试
- npm run build    构建生产版本
- npm run lint     运行 ESLint

## 重要约定
- 不要直接修改 src/generated/ 目录
- 数据库迁移必须先在 staging 验证
- API 路由必须有认证中间件
- 所有用户输入必须验证

## 当前工作
- 正在重构认证模块（见 src/auth/）
- 待完成：用户权限系统
```

---

## 14.4 CLAUDE.md 的 @ 引用语法

CLAUDE.md 支持 `@` 语法引用其他文件：

```markdown
# 项目配置

## API 规范
@docs/api-spec.md

## 数据库 Schema
@prisma/schema.prisma

## 环境变量说明
@.env.example
```

这让 CLAUDE.md 可以引用项目中已有的文档，避免重复维护。

---

## 14.5 Memory 系统：自动记忆

除了手动维护的 CLAUDE.md，Claude Code 还有一个自动 Memory 系统（`src/memdir/`）。

**Memory 文件的存储位置**：
```
~/.claude/projects/<project-hash>/memory/
├── user_role.md          # 用户角色信息
├── feedback_testing.md   # 测试相关反馈
├── project_context.md    # 项目上下文
└── MEMORY.md             # Memory 索引
```

**Memory 的类型**（`src/memdir/memoryTypes.ts`）：

```typescript
type MemoryType =
  | 'user'       // 用户信息（角色、偏好、知识背景）
  | 'feedback'   // 用户反馈（做什么、不做什么）
  | 'project'    // 项目信息（目标、约束、决策）
  | 'reference'  // 外部资源引用
```

---

## 14.6 Memory 的自动提取

Claude Code 可以自动从对话中提取 Memory（`src/services/extractMemories/`）：

```typescript
// 当用户说"记住这个"时，自动保存
用户：记住：我们的 API 使用 JWT 认证，token 有效期 24 小时

// Claude 会创建一个 Memory 文件
// ~/.claude/projects/.../memory/project_auth.md
---
name: API 认证配置
type: project
---
API 使用 JWT 认证，token 有效期 24 小时。
```

**触发条件**：
- 用户明确说"记住"、"记录"
- 用户纠正 Claude 的行为（"不要这样做"）
- 用户确认 Claude 的非显而易见的选择

---

## 14.7 Memory 的相关性搜索

不是所有 Memory 都会在每次对话中加载。Claude Code 使用相关性搜索（`src/memdir/findRelevantMemories.ts`）：

```typescript
// 根据当前任务找出相关的 Memory
async function findRelevantMemories(
  currentTask: string,
  allMemories: Memory[]
): Promise<Memory[]> {
  // 简单的关键词匹配
  // 或者使用嵌入向量相似度（如果启用）
  return allMemories.filter(memory =>
    isRelevant(memory, currentTask)
  )
}
```

这避免了把所有 Memory 都塞进上下文（浪费 token），只加载与当前任务相关的 Memory。

---

## 14.8 嵌套 Memory：动态加载

Claude Code 支持嵌套 Memory 的动态加载（`loadedNestedMemoryPaths`）：

```typescript
// QueryEngine 追踪已加载的 Memory 路径
private loadedNestedMemoryPaths = new Set<string>()

// 当 Claude 访问新目录时，检查是否有相关 Memory
// 如果有，动态注入到上下文
```

这让 Memory 系统能够根据 Claude 的工作位置动态调整上下文。

---

## 14.9 团队 Memory 同步

Claude Code 支持团队 Memory 同步（`src/services/teamMemorySync/`）：

```
团队成员 A 的 Memory → 同步到团队共享 Memory
团队成员 B 的 Claude Code → 读取团队共享 Memory
```

这让团队可以共享项目约定、最佳实践、常见问题解决方案，而不需要每个人都手动维护 CLAUDE.md。

---

## 14.10 CLAUDE.md vs Memory：如何选择

| 场景 | 推荐方式 |
|------|---------|
| 项目技术栈说明 | CLAUDE.md |
| 代码规范 | CLAUDE.md |
| 常用命令 | CLAUDE.md |
| 个人工作偏好 | Memory（user 类型） |
| 项目决策记录 | Memory（project 类型） |
| Claude 的行为反馈 | Memory（feedback 类型） |
| 外部文档引用 | CLAUDE.md（@ 语法） |
| 临时上下文 | 直接在对话中说明 |

---

## 14.11 Memory 系统的设计原则

Claude Code 的 Memory 系统遵循几个设计原则：

**不保存可推导的信息**：代码结构、文件路径、git 历史——这些可以通过读取代码库获得，不需要保存到 Memory。

**保存非显而易见的信息**：团队约定、历史决策、个人偏好——这些无法从代码中推导，值得保存。

**Memory 会过时**：保存时间戳，定期检查 Memory 是否仍然有效。

**Memory 不是日志**：不保存"做了什么"，而是保存"为什么这样做"。

---

## 14.12 小结

Claude Code 的记忆系统是两层的：

- **CLAUDE.md**：显式的、手动维护的项目记忆，适合稳定的项目信息
- **Memory 系统**：自动的、动态的个人/团队记忆，适合偏好和决策记录

两者结合，让 Claude 在每次会话中都能快速进入工作状态，不需要重复解释背景。

---

*下一章：[上下文压缩（Auto-Compact）](./15-compact.md)*
