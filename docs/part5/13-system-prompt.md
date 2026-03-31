# 第 13 章：系统提示的构建艺术

> 系统提示是 AI 的"宪法"——它定义了 AI 的角色、能力和边界。

---

## 13.1 系统提示的作用

系统提示（System Prompt）是每次 API 调用时传给 Claude 的"背景设定"。它告诉 Claude：

- 它是谁（角色）
- 它能做什么（能力）
- 它不能做什么（限制）
- 它应该如何行动（行为准则）

系统提示的质量直接决定了 Claude 的行为质量。一个好的系统提示能让 Claude 更准确地理解任务、更合理地使用工具、更安全地执行操作。

---

## 13.2 Claude Code 系统提示的构建流程

系统提示不是静态的，而是在每次对话开始时动态构建的：

![系统提示构建流程](../diagrams/system-prompt-construction.svg)

代码实现（简化）：

```typescript
// 简化的系统提示构建流程
async function fetchSystemPromptParts(config) {
  const parts = []

  // 1. 核心指令（固定部分）
  parts.push(getCoreInstructions())

  // 2. 工具定义（根据当前可用工具动态生成）
  parts.push(getToolDefinitions(config.tools))

  // 3. 用户上下文（CLAUDE.md 内容）
  const userContext = await getUserContext()
  if (userContext.claudeMd) {
    parts.push(formatClaudeMd(userContext.claudeMd))
  }

  // 4. 系统上下文（git 状态等）
  const systemContext = await getSystemContext()
  if (systemContext.gitStatus) {
    parts.push(formatGitStatus(systemContext.gitStatus))
  }

  // 5. 自定义系统提示（用户通过 --system-prompt 传入）
  if (config.customSystemPrompt) {
    parts.push(config.customSystemPrompt)
  }

  // 6. 追加系统提示（用户通过 --append-system-prompt 传入）
  if (config.appendSystemPrompt) {
    parts.push(config.appendSystemPrompt)
  }

  return parts.join('\n\n')
}
```

---

## 13.3 核心指令的设计

Claude Code 的核心指令定义了 Claude 作为编程助手的基本行为准则。虽然完整的系统提示是私有的，但从源码中可以推断出几个关键原则：

**安全优先**：
```
在执行任何可能破坏性的操作之前，必须获得用户确认。
不要执行可能导致数据丢失的操作，除非用户明确要求。
```

**透明操作**：
```
在执行工具调用时，清楚地说明你要做什么和为什么。
如果不确定，先询问而不是猜测。
```

**代码质量**：
```
遵循项目的代码规范（从 CLAUDE.md 中读取）。
优先修改最小必要的代码，不要做不必要的重构。
```

**错误处理**：
```
当工具执行失败时，分析错误原因，尝试替代方案。
不要在没有理解错误的情况下盲目重试。
```

---

## 13.4 工具定义在系统提示中的作用

工具定义是系统提示中最大的部分。每个工具的 `name`、`description` 和 `inputSchema` 都会被序列化到系统提示中：

```json
{
  "name": "FileEditTool",
  "description": "对文件进行精确的字符串替换...",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": {
        "type": "string",
        "description": "要编辑的文件路径"
      },
      "old_string": {
        "type": "string",
        "description": "要替换的内容（必须在文件中唯一存在）"
      },
      "new_string": {
        "type": "string",
        "description": "替换后的内容"
      }
    },
    "required": ["file_path", "old_string", "new_string"]
  }
}
```

Claude 通过这些定义理解每个工具的用途和参数格式。工具描述的质量直接影响 Claude 的工具选择准确性。

---

## 13.5 系统提示的缓存策略

Claude API 支持提示缓存（Prompt Caching）：如果系统提示没有变化，API 会缓存它，减少 token 消耗和延迟。

Claude Code 利用了这个特性：

```typescript
// 系统提示的稳定部分（可缓存）
const stableSystemPrompt = [
  coreInstructions,    // 几乎不变
  toolDefinitions,     // 工具集不变时不变
]

// 系统提示的动态部分（不缓存）
const dynamicSystemPrompt = [
  gitStatus,           // 每次对话可能不同
  claudeMdContent,     // 文件修改后会变
]
```

通过把稳定部分放在前面，动态部分放在后面，最大化缓存命中率。

---

## 13.6 系统提示注入（缓存破坏）

`src/context.ts` 中有一个有趣的功能：

```typescript
// 系统提示注入（ant-only，用于调试）
let systemPromptInjection: string | null = null

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  // 清除上下文缓存，强制重新构建
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

这个功能允许 Anthropic 内部工程师在不重启的情况下修改系统提示，用于调试和实验。注释明确标注了 `ant-only`（仅 Anthropic 内部使用）和 `ephemeral debugging state`（临时调试状态）。

这是一个很好的工程实践：**调试功能要明确标注，防止误用**。

---

## 13.7 多层系统提示的优先级

当有多个系统提示来源时，优先级如下：

```
优先级（从高到低）：
1. --append-system-prompt（用户追加，最高优先级）
2. --system-prompt（用户自定义，完全替换默认提示）
3. CLAUDE.md（项目级配置）
4. 默认系统提示（Claude Code 内置）
```

注意 `--system-prompt` 和 `--append-system-prompt` 的区别：
- `--system-prompt`：**替换**默认系统提示（适合完全自定义场景）
- `--append-system-prompt`：**追加**到默认系统提示（适合在默认基础上扩展）

---

## 13.8 系统提示的长度权衡

系统提示越长，Claude 的理解越完整，但也消耗更多 token。

Claude Code 的权衡策略：

**核心指令**：尽量简洁，只包含最重要的行为准则。

**工具定义**：无法压缩（Claude 需要完整的 schema），但可以通过只注册当前需要的工具来减少长度。

**CLAUDE.md**：用户控制，建议保持在 2000 字以内。

**git 状态**：有 2000 字符的截断限制：
```typescript
const MAX_STATUS_CHARS = 2000
const truncatedStatus = status.length > MAX_STATUS_CHARS
  ? status.substring(0, MAX_STATUS_CHARS) +
    '\n... (truncated. Run "git status" for full output)'
  : status
```

---

## 13.9 系统提示的测试

如何测试系统提示的质量？Claude Code 使用了几种方法：

**行为测试**：给定特定输入，验证 Claude 的行为是否符合预期。

**工具选择测试**：给定特定任务，验证 Claude 是否选择了正确的工具。

**安全测试**：尝试让 Claude 执行危险操作，验证它是否正确拒绝。

**回归测试**：修改系统提示后，运行完整的测试套件，确保没有破坏现有行为。

---

## 13.10 小结

系统提示的构建是一门艺术，也是一门工程：

- **动态构建**：根据当前状态动态生成，不是静态配置
- **分层设计**：核心指令 + 工具定义 + 用户上下文 + 系统上下文
- **缓存优化**：稳定部分前置，最大化缓存命中
- **长度权衡**：在完整性和 token 消耗之间取得平衡
- **可扩展**：支持用户自定义和追加

一个好的系统提示是 Claude Code 高质量输出的基础。

---

*下一章：[Memory 与 CLAUDE.md](./14-memory-claudemd.md)*
