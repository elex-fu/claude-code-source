# 第 9 章：工具系统的设计哲学

> 工具是 Agent 的手，设计好工具就是设计好 Agent 的能力边界。

---

## 9.1 工具系统的核心问题

设计一个工具系统，需要回答几个根本问题：

1. **工具的接口是什么？** 输入输出如何定义？
2. **工具如何被发现？** Claude 怎么知道有哪些工具可用？
3. **工具如何被选择？** Claude 怎么决定用哪个工具？
4. **工具如何被执行？** 执行时需要什么上下文？
5. **工具的权限如何控制？** 哪些工具需要用户确认？
6. **工具如何扩展？** 如何添加新工具？

Claude Code 的工具系统对这六个问题都有清晰的答案。

---

## 9.2 工具的统一接口

每个工具都实现相同的接口（`src/Tool.ts`）：

```typescript
export type Tool<
  D extends AnyToolDef = AnyToolDef,
  I = D extends AnyToolDef ? D['input'] : never,
  O = D extends AnyToolDef ? D['output'] : never,
> = {
  // 工具名称（Claude 用这个名字调用工具）
  name: string

  // 工具描述（Claude 用这个决定何时使用工具）
  description: string

  // 输入 schema（JSON Schema 格式，用于参数验证和 Claude 理解）
  inputSchema: ToolInputJSONSchema

  // 执行函数
  execute(input: I, context: ToolUseContext): Promise<ToolResult<O>>

  // 可选：是否需要用户确认
  needsPermission?: (input: I) => boolean

  // 可选：工具的 JSX 渲染（在 UI 中显示工具执行状态）
  renderToolUse?: (input: I, context: RenderContext) => React.ReactNode
}
```

这个接口设计的精妙之处：

**`description` 是给 Claude 看的**，不是给用户看的。Claude 通过 description 理解工具的用途，决定何时调用。好的 description 直接影响 Claude 的工具选择质量。

**`inputSchema` 是双重用途的**：一方面用于参数验证（防止 Claude 传入错误参数），另一方面作为 Claude 理解工具参数的文档。

**`execute` 是异步的**：所有工具执行都是异步的，支持 I/O 操作、网络请求等。

---

## 9.3 工具描述的艺术

工具描述是工具系统中最被低估的部分。一个好的描述能让 Claude 准确选择工具，一个差的描述会导致工具被误用或忽略。

以 `FileEditTool` 为例，它的描述大致是：

```
对文件进行精确的字符串替换。
- 使用场景：修改现有文件的特定内容
- 不适用：创建新文件（用 FileWriteTool）、查看文件（用 FileReadTool）
- 重要：old_string 必须在文件中唯一存在，否则会失败
- 重要：必须先用 FileReadTool 读取文件，确认 old_string 的确切内容
```

注意这个描述做了什么：
1. 说明了**适用场景**
2. 说明了**不适用场景**（引导 Claude 选择正确的工具）
3. 说明了**重要约束**（防止常见错误）
4. 说明了**前置条件**（先读后写）

这种描述风格是 Claude Code 工具设计的一个重要模式。

---

## 9.4 ToolUseContext：工具的执行环境

`ToolUseContext` 是工具执行时的完整上下文，包含 30+ 个字段：

```typescript
export type ToolUseContext = {
  // 配置
  options: {
    commands: Command[]
    tools: Tools
    verbose: boolean
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    isNonInteractiveSession: boolean
    // ...
  }

  // 中断控制
  abortController: AbortController

  // 状态读写
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  // UI 交互
  setToolJSX?: SetToolJSXFn          // 设置工具的 UI 渲染
  addNotification?: (n: Notification) => void
  sendOSNotification?: (opts) => void

  // 文件系统
  readFileState: FileStateCache       // 文件读取缓存
  updateFileHistoryState: (updater) => void

  // 消息系统
  messages: Message[]                 // 当前对话历史
  appendSystemMessage?: (msg) => void

  // 权限
  setInProgressToolUseIDs: (f) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void

  // 性能追踪
  setResponseLength: (f) => void
  pushApiMetricsEntry?: (ttftMs: number) => void
  setStreamMode?: (mode: SpinnerMode) => void

  // Memory 系统
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>

  // Skills 系统
  dynamicSkillDirTriggers?: Set<string>
  discoveredSkillNames?: Set<string>

  // 工具决策追踪
  toolDecisions?: Map<string, {
    source: string
    decision: 'accept' | 'reject'
    timestamp: number
  }>
}
```

这个上下文设计体现了一个重要原则：**工具不应该有全局副作用，所有副作用都通过上下文显式传递**。

工具需要更新 UI？通过 `setToolJSX`。
工具需要读取状态？通过 `getAppState`。
工具需要发送通知？通过 `addNotification`。

这让工具的行为完全可预测和可测试。

---

## 9.5 工具注册与发现

工具在 `src/tools.ts` 中注册：

```typescript
// src/tools.ts（简化）
export function getTools(options: GetToolsOptions): Tools {
  const tools: Tool[] = [
    // 文件操作
    new FileReadTool(),
    new FileEditTool(),
    new FileWriteTool(),
    new GlobTool(),
    new GrepTool(),

    // Shell
    new BashTool(),

    // 代理
    new AgentTool(),

    // ... 其他工具
  ]

  // 根据配置过滤工具
  return tools.filter(tool => isToolEnabled(tool, options))
}
```

工具列表在每次会话开始时构建，并通过 API 的 `tools` 参数传给 Claude。Claude 看到的是工具的 `name`、`description` 和 `inputSchema`，不是实现代码。

---

## 9.6 工具的分层设计

![工具系统三层架构](../diagrams/tool-system-hierarchy.svg)

Claude Code 的工具按职责分层：

这种分层设计的好处：**原子工具简单可靠，复杂任务由 Claude 的推理能力编排**，而不是硬编码在工具里。

---

## 9.7 工具结果的格式

工具执行后返回 `ToolResult`：

```typescript
type ToolResult<O> = {
  type: 'tool_result'
  content: string | ContentBlock[]  // 结果内容
  is_error?: boolean                // 是否是错误
  metadata?: {
    tokenCount?: number             // 结果的 token 数
    truncated?: boolean             // 是否被截断
  }
}
```

工具结果会被追加到消息列表，Claude 在下一轮可以看到这些结果并据此决策。

**结果截断**是一个重要的设计考量：文件可能很大，工具结果可能超出 token 限制。Claude Code 会自动截断过大的结果，并在结果中注明截断信息，让 Claude 知道结果是不完整的。

---

## 9.8 工具的幂等性设计

好的工具应该尽量幂等（多次执行结果相同）：

- `FileReadTool`：天然幂等（读操作不改变状态）
- `GrepTool`：天然幂等
- `FileEditTool`：**不幂等**，但有保护机制（`old_string` 必须唯一存在）
- `BashTool`：**不幂等**，需要用户确认

对于不幂等的工具，Claude Code 通过权限系统要求用户确认，防止意外的重复执行。

---

## 9.9 工具的测试策略

`src/tools/testing/` 目录包含工具测试的基础设施：

```typescript
// 工具测试的典型模式
describe('FileEditTool', () => {
  it('should edit file content', async () => {
    // 创建测试文件
    const testFile = createTempFile('hello world')

    // 执行工具
    const result = await FileEditTool.execute({
      file_path: testFile,
      old_string: 'hello',
      new_string: 'goodbye'
    }, mockContext)

    // 验证结果
    expect(result.is_error).toBe(false)
    expect(readFile(testFile)).toBe('goodbye world')
  })
})
```

工具测试的关键是 `mockContext`：通过 mock `ToolUseContext`，可以在不启动完整系统的情况下测试单个工具。

---

## 9.10 工具设计的反模式

在设计工具时，有几个常见的反模式需要避免：

**反模式一：工具做太多事**
```
// 错误：一个工具做了读、分析、写三件事
AnalyzeAndRefactorTool

// 正确：分成三个工具，由 Claude 编排
FileReadTool + (Claude 分析) + FileEditTool
```

**反模式二：工具有隐式依赖**
```
// 错误：工具依赖全局状态
execute(input) {
  const config = globalConfig  // 隐式依赖
}

// 正确：通过 context 显式传递
execute(input, context) {
  const config = context.options.config  // 显式依赖
}
```

**反模式三：工具描述不精确**
```
// 错误：描述太模糊
description: "编辑文件"

// 正确：描述精确，包含约束和使用场景
description: "对文件进行精确的字符串替换。必须先读取文件确认内容..."
```

---

## 9.11 小结

Claude Code 工具系统的设计哲学：

1. **统一接口**：所有工具实现相同的 `Tool` 接口
2. **描述即文档**：工具描述是给 Claude 的使用说明
3. **显式上下文**：所有副作用通过 `ToolUseContext` 显式传递
4. **原子性**：工具只做一件事，复杂任务由 Claude 编排
5. **可测试性**：通过 mock context 独立测试每个工具
6. **权限意识**：不幂等的工具需要权限控制

这些原则共同构成了一个可靠、可扩展、可测试的工具系统。

---

*下一章：[43 个内置工具全览](./10-builtin-tools.md)*
