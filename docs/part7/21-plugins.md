# 第 21 章：插件系统

> 插件系统是 Claude Code 开放性的体现——让第三方开发者扩展核心能力。

---

## 21.1 插件系统的设计目标

Claude Code 的插件系统（`src/plugins/`、`src/services/plugins/`）有几个设计目标：

1. **扩展性**：允许第三方添加新功能，而不需要修改核心代码
2. **隔离性**：插件的错误不影响核心系统
3. **安全性**：插件的权限受到限制
4. **可发现性**：用户可以轻松找到和安装插件

---

## 21.2 插件的类型

Claude Code 支持两种插件类型：

**内置插件**（`src/plugins/builtinPlugins.ts`）：
随 Claude Code 一起分发的官方插件，有完整的系统权限。

**第三方插件**（通过 Marketplace）：
用户安装的外部插件，权限受限，需要用户明确授权。

---

## 21.3 插件的 Hook 系统

插件通过 Hook 系统与 Claude Code 集成：

```typescript
// src/types/plugin.ts
type PluginHookMatcher = {
  event: HookEvent          // 触发时机
  matcher?: string          // 匹配条件（可选）
  handler: PluginHandler    // 处理函数
}

type HookEvent =
  | 'PreToolUse'            // 工具调用前
  | 'PostToolUse'           // 工具调用后
  | 'UserPromptSubmit'      // 用户提交消息时
  | 'Stop'                  // 对话结束时
  | 'Notification'          // 发送通知时
```

插件可以在这些关键时机注入自定义逻辑：

```typescript
// 示例：一个记录所有工具调用的插件
const auditPlugin: Plugin = {
  name: 'audit-logger',
  hooks: [
    {
      event: 'PostToolUse',
      handler: async ({ toolName, input, result }) => {
        await appendToAuditLog({
          timestamp: Date.now(),
          tool: toolName,
          input,
          success: !result.is_error,
        })
      }
    }
  ]
}
```

---

## 21.4 插件的配置

插件通过 `settings.json` 配置：

```json
{
  "plugins": {
    "audit-logger": {
      "enabled": true,
      "logFile": "~/.claude/audit.log"
    },
    "custom-formatter": {
      "enabled": true,
      "style": "compact"
    }
  }
}
```

---

## 21.5 插件的安全模型

插件的安全模型基于**最小权限原则**：

```typescript
type PluginPermissions = {
  canReadFiles: boolean      // 是否可以读取文件
  canWriteFiles: boolean     // 是否可以写入文件
  canExecuteCommands: boolean // 是否可以执行命令
  canAccessNetwork: boolean  // 是否可以访问网络
  canModifySettings: boolean // 是否可以修改设置
}
```

用户在安装插件时需要明确授权每种权限。

---

## 21.6 内置插件示例

Claude Code 有几个重要的内置插件：

**AutoUpdater**（`src/components/AutoUpdater.tsx`）：
自动检查和安装 Claude Code 更新。

**PromptSuggestion**（`src/services/PromptSuggestion/`）：
根据当前上下文提供提示建议。

**SessionMemory**（`src/services/SessionMemory/`）：
管理会话记忆的持久化。

**AgentSummary**（`src/services/AgentSummary/`）：
生成代理执行的摘要报告。

---

## 21.7 插件 vs MCP vs Skills

三种扩展机制的对比：

| 维度 | 插件 | MCP | Skills |
|------|------|-----|--------|
| 实现语言 | TypeScript | 任意语言 | Markdown |
| 集成深度 | 深（Hook 系统） | 中（工具/资源） | 浅（提示模板） |
| 开发难度 | 高 | 中 | 低 |
| 适用场景 | 核心功能扩展 | 外部服务集成 | 工作流封装 |
| 权限要求 | 高 | 中 | 低 |

---

## 21.8 小结

Claude Code 的插件系统提供了深度扩展能力：

- **Hook 系统**：在关键时机注入自定义逻辑
- **权限模型**：最小权限原则，用户明确授权
- **两种类型**：内置插件（官方）+ 第三方插件（用户安装）

三种扩展机制（插件、MCP、Skills）覆盖了从深度集成到轻量封装的完整需求。

---

*下一章：[权限模型的分层设计](../part8/22-permission-model.md)*
