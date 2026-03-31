# 第 22 章：权限模型的分层设计

> 好的权限系统让安全成为默认，让便利成为可选。

---

## 22.1 权限设计的核心矛盾

AI Agent 的权限设计面临一个根本矛盾：

**能力越强，风险越大**。

Claude Code 能执行 Shell 命令、修改文件、访问网络——这些能力让它非常有用，但也意味着一个错误的操作可能造成严重后果。

解决这个矛盾的方法不是限制能力，而是**精细化控制能力的使用**。

---

## 22.2 五层权限架构

![五层权限架构](../diagrams/permission-model-layers.svg)

Claude Code 的权限系统分为五层，从粗到细：

---

## 22.3 层次 1：会话模式

```typescript
type PermissionMode =
  | 'default'              // 危险操作前询问
  | 'acceptEdits'          // 自动接受文件编辑
  | 'bypassPermissions'    // 跳过所有检查（危险！）
  | 'plan'                 // 只能生成计划
```

会话模式在启动时设置，影响整个会话的权限基调。

**default 模式**是最安全的，适合日常使用。

**acceptEdits 模式**适合信任 Claude 的文件修改，但仍然对 Shell 命令保持谨慎。这是一个常见的中间状态：开发者通常信任 Claude 修改代码，但不信任它执行任意命令。

**bypassPermissions 模式**完全跳过权限检查，适合 CI/CD 等完全受信任的自动化场景。源码中有明确的警告：

```typescript
// 只有在完全受信任的环境中才使用
// 错误使用可能导致数据丢失或安全问题
if (allowDangerouslySkipPermissions) {
  logWarning('Running with --dangerously-skip-permissions. All tool calls will be auto-approved.')
}
```

**plan 模式**是最受限的，Claude 只能生成计划，不能执行任何工具。适合需要人工审查的场景。

---

## 22.4 层次 2：工具白名单/黑名单

用户可以配置哪些工具允许使用，哪些禁止：

```json
// ~/.claude/settings.json
{
  "allowedTools": ["FileReadTool", "GrepTool", "GlobTool"],
  "deniedTools": ["BashTool", "FileWriteTool"]
}
```

或者在 CLAUDE.md 中配置：

```markdown
<!-- CLAUDE.md -->
# 工具限制
只允许使用只读工具，不允许修改文件或执行命令。
```

---

## 22.5 层次 3：工具级权限

每个工具有默认的权限要求：

| 工具 | 默认权限要求 | 原因 |
|------|------------|------|
| FileReadTool | 自动允许 | 只读，无副作用 |
| GrepTool | 自动允许 | 只读，无副作用 |
| GlobTool | 自动允许 | 只读，无副作用 |
| FileEditTool | 询问（默认模式）/ 自动（acceptEdits） | 修改文件 |
| FileWriteTool | 询问 | 创建/覆写文件 |
| BashTool | 询问 | 执行任意命令 |
| WebFetchTool | 询问 | 网络访问 |

---

## 22.6 层次 4：操作级权限

同一个工具的不同操作可能有不同的风险等级：

```typescript
// BashTool 的操作级权限分析
function analyzeBashCommand(command: string): RiskLevel {
  if (isDangerousCommand(command)) {
    return 'high'    // 需要明确确认
  }
  if (isNetworkCommand(command)) {
    return 'medium'  // 需要确认
  }
  if (isReadOnlyCommand(command)) {
    return 'low'     // 可以自动允许
  }
  return 'medium'    // 默认需要确认
}

// 只读命令（低风险）
const READ_ONLY_COMMANDS = ['ls', 'cat', 'grep', 'find', 'git log', 'git status', ...]

// 危险命令（高风险）
const DANGEROUS_PATTERNS = [/rm\s+-rf/, /mkfs/, /dd\s+.*of=\/dev\//, ...]
```

---

## 22.7 层次 5：路径/命令级权限

最细粒度的控制：

```json
// 允许特定路径的写入
{
  "allowedWritePaths": ["./src/", "./tests/"],
  "deniedWritePaths": ["./config/", "./.env"]
}

// 允许特定命令
{
  "allowedBashCommands": ["npm test", "npm run build", "git status"],
  "deniedBashCommands": ["rm -rf", "sudo"]
}
```

---

## 22.8 权限决策的记录与审计

所有权限决策都被记录：

```typescript
// 工具决策追踪
toolDecisions?: Map<string, {
  source: string           // 决策来源（用户交互、配置文件、白名单等）
  decision: 'accept' | 'reject'
  timestamp: number
}>
```

这提供了完整的审计能力：可以查看哪些操作被允许或拒绝，以及原因。

---

## 22.9 权限的继承与覆盖

子代理继承父代理的权限，但可以被进一步限制：

```typescript
// 父代理启动子代理时，可以限制子代理的权限
await AgentTool.execute({
  prompt: '...',
  // 子代理只能读取文件，不能写入或执行命令
  allowedTools: ['FileReadTool', 'GrepTool', 'GlobTool'],
}, context)
```

权限只能被限制，不能被扩展：子代理不能拥有比父代理更多的权限。

---

## 22.10 权限系统的用户体验

好的权限系统不应该让用户感到烦躁。Claude Code 的设计原则：

**最小化打扰**：只在真正需要时询问，不过度询问。

**记住决策**：用户允许一次后，相同操作不再询问（在同一会话内）。

**清晰的说明**：询问时清楚说明要做什么、为什么需要权限、可能的影响。

**快速响应**：权限检查不应该增加明显的延迟。

---

## 22.11 小结

Claude Code 的权限模型是一个五层的精细化控制系统：

1. **会话模式**：整体权限基调
2. **工具白名单/黑名单**：工具级别的开关
3. **工具级权限**：每个工具的默认行为
4. **操作级权限**：同一工具不同操作的风险分级
5. **路径/命令级权限**：最细粒度的控制

设计原则：**安全是默认，便利是可选，控制在用户手中**。

---

*下一章：[安全设计](./23-security.md)*
