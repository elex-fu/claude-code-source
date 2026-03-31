# 第 11 章：工具权限模型

> 能力越大，责任越大。工具权限模型是 Claude Code 安全性的核心。

---

## 11.1 为什么需要权限模型

Claude Code 能执行 Shell 命令、修改文件、访问网络。这些能力如果不加控制，可能造成严重后果：

- 误删重要文件
- 执行恶意脚本
- 泄露敏感信息
- 意外修改生产数据库

权限模型的目标是：**在保持 Claude 能力的同时，防止意外或恶意的破坏性操作**。

---

## 11.2 权限模式（PermissionMode）

Claude Code 有四种权限模式：

```typescript
// src/utils/permissions/PermissionMode.ts
type PermissionMode =
  | 'default'                    // 默认：危险操作前询问
  | 'acceptEdits'                // 自动接受文件编辑，其他操作询问
  | 'bypassPermissions'          // 跳过所有权限检查（危险！）
  | 'plan'                       // 计划模式：只能生成计划，不能执行
```

**默认模式**是最安全的，适合日常使用。

**acceptEdits 模式**适合信任 Claude 的文件修改，但仍然对 Shell 命令保持谨慎。

**bypassPermissions 模式**适合完全受信任的自动化场景（如 CI/CD），不应在交互式会话中使用。

**计划模式**是一种特殊的安全模式，Claude 只能描述计划，不能执行任何工具。

---

## 11.3 工具级权限检查

每个工具调用都经过 `canUseTool()` 函数检查：

```typescript
// src/hooks/useCanUseTool.tsx（简化）
export type CanUseToolFn = (
  toolName: string,
  toolInput: unknown,
  context: PermissionContext
) => CanUseToolResult

type CanUseToolResult =
  | { behavior: 'allow' }                    // 允许
  | { behavior: 'deny'; message: string }    // 拒绝
  | { behavior: 'ask'; message: string }     // 询问用户
```

权限检查的决策树：

![权限检查决策树](../diagrams/permission-decision-tree.svg)

---

## 11.4 命令安全分析

BashTool 有专门的命令安全分析模块（`src/utils/bash/`）：

```typescript
// 危险命令检测
const DANGEROUS_PATTERNS = [
  /rm\s+-rf?\s+[\/~]/,          // rm -rf /
  />\s*\/dev\/sd[a-z]/,         // 覆写磁盘
  /mkfs\./,                      // 格式化文件系统
  /dd\s+.*of=\/dev\//,          // dd 写入设备
  /chmod\s+-R\s+777/,           // 危险权限
  /curl.*\|\s*bash/,            // 管道执行远程脚本
  /wget.*\|\s*sh/,              // 同上
]

function analyzeCommandSafety(command: string): SafetyAnalysis {
  for (const pattern of DANGEROUS_PATTERNS) {
    if (pattern.test(command)) {
      return {
        safe: false,
        reason: `检测到危险模式: ${pattern}`,
        requiresConfirmation: true
      }
    }
  }
  return { safe: true }
}
```

这个分析不是万能的（正则表达式无法覆盖所有情况），但能捕获最常见的危险操作。

---

## 11.5 路径权限控制

文件操作工具有路径级别的权限控制：

```typescript
// 允许的路径范围
type PathPermission = {
  allowedPaths: string[]    // 允许访问的路径
  blockedPaths: string[]    // 禁止访问的路径
}

// 检查路径是否在允许范围内
function isPathAllowed(filePath: string, permission: PathPermission): boolean {
  const resolved = path.resolve(filePath)

  // 检查是否在禁止路径中
  for (const blocked of permission.blockedPaths) {
    if (resolved.startsWith(path.resolve(blocked))) {
      return false
    }
  }

  // 检查是否在允许路径中
  for (const allowed of permission.allowedPaths) {
    if (resolved.startsWith(path.resolve(allowed))) {
      return true
    }
  }

  return false
}
```

默认情况下，Claude Code 只能访问当前工作目录及其子目录。

---

## 11.6 用户确认流程

当工具需要用户确认时，Claude Code 会显示一个确认对话框：

```
Claude 想要执行以下命令：

  rm -rf node_modules/

这个操作不可逆。是否允许？

[允许一次]  [始终允许]  [拒绝]  [拒绝并解释]
```

四个选项的设计很精妙：

- **允许一次**：只允许这一次，下次同样操作还需要确认
- **始终允许**：将此操作加入白名单，以后自动允许
- **拒绝**：拒绝这次操作，Claude 会尝试其他方案
- **拒绝并解释**：拒绝并告诉 Claude 为什么，Claude 可以调整策略

---

## 11.7 权限决策的持久化

用户的权限决策可以持久化：

```typescript
// 工具决策追踪
toolDecisions?: Map<string, {
  source: string           // 决策来源（用户、配置文件等）
  decision: 'accept' | 'reject'
  timestamp: number
}>
```

这让用户不需要对同样的操作反复确认。同时，决策记录也提供了审计能力——可以查看哪些操作被允许或拒绝了。

---

## 11.8 沙箱模式

Claude Code 支持沙箱模式，在受限环境中运行：

```typescript
// src/entrypoints/sandboxTypes.ts
type SandboxConfig = {
  allowedCommands: string[]    // 白名单命令
  allowedPaths: string[]       // 白名单路径
  networkAccess: boolean       // 是否允许网络访问
  maxExecutionTime: number     // 最大执行时间
}
```

沙箱模式适合：
- CI/CD 环境中的自动化任务
- 不信任的代码库分析
- 教育场景（限制学生的操作范围）

---

## 11.9 权限模型的设计权衡

权限模型面临一个根本的权衡：**安全性 vs 便利性**。

过于严格的权限会让 Claude Code 变得难用——每次操作都需要确认，用户会感到烦躁。

过于宽松的权限会带来安全风险——Claude 可能执行意外的破坏性操作。

Claude Code 的解决方案是**分层权限**：

```
层次 1：模式级别（bypassPermissions / default / plan）
    ↓
层次 2：工具级别（某些工具默认允许，某些默认询问）
    ↓
层次 3：操作级别（同一工具的不同操作有不同权限）
    ↓
层次 4：路径级别（文件操作的路径范围限制）
    ↓
层次 5：命令级别（Shell 命令的安全分析）
```

用户可以在任何层次调整权限，实现精细化控制。

---

## 11.10 权限拒绝的处理

当工具被拒绝时，Claude 不会简单地停止，而是尝试调整策略：

```
Claude 想要执行：rm -rf dist/
用户：拒绝

Claude：好的，我改用 rimraf dist/ 命令，它更安全。
用户：允许

Claude：执行 rimraf dist/...
```

这种"拒绝后调整"的能力来自 Claude 的推理能力，而不是工具系统本身。工具系统只负责执行权限检查，Claude 负责根据结果调整策略。

---

## 11.11 小结

Claude Code 的权限模型是一个多层次的安全系统：

- **模式级别**：四种权限模式，从完全受限到完全开放
- **工具级别**：每个工具有默认的权限要求
- **操作级别**：同一工具的不同操作有不同风险等级
- **路径级别**：文件操作的路径范围限制
- **命令级别**：Shell 命令的安全分析

这个系统的设计目标是：**让安全成为默认，让便利成为可选**。

---

*下一章：[什么是 Context Engineering](../part5/12-context-what.md)*
