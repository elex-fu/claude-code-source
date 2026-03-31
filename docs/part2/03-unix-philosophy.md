# 第 3 章：Unix 哲学与 CLI 的传统

> "Write programs that do one thing and do it well.
>  Write programs that work together.
>  Write programs that handle text streams."
> —— Doug McIlroy，Unix 管道的发明者

---

## 3.1 Unix 哲学：50 年前的设计智慧

1969 年，Ken Thompson 和 Dennis Ritchie 在贝尔实验室创造了 Unix。这不只是一个操作系统，更是一套影响深远的**设计哲学**。

Unix 哲学的核心是三条原则：

1. **单一职责**：每个程序只做一件事，并把它做好
2. **组合性**：程序之间通过标准接口（文本流）协作
3. **透明性**：程序的行为可预测、可观察

这三条原则，在 50 年后的 Claude Code 里依然清晰可见。

---

## 3.2 管道：最早的"工具调用"

Unix 管道（`|`）是人类历史上最早的"工具调用"机制之一：

```bash
# 找出最常用的 10 个命令
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10
```

这条命令做了什么？

1. `history` 输出命令历史
2. `awk` 提取第二列（命令名）
3. `sort` 排序
4. `uniq -c` 统计重复次数
5. `sort -rn` 按数字倒序排列
6. `head -10` 取前 10 条

每个程序只做一件事，通过管道串联起来完成复杂任务。

**这和 Claude Code 的工具调用链惊人地相似**：

```
用户请求 → GrepTool（搜索） → FileReadTool（读取） → FileEditTool（修改） → BashTool（验证）
```

---

## 3.3 文件系统：统一的抽象

Unix 的另一个伟大设计是"一切皆文件"。网络连接、设备、进程信息——都通过文件系统接口访问。

这个抽象的价值在于：**统一的接口降低了认知负担**。

Claude Code 继承了这个思想。它的工具系统也是一套统一的接口：无论是读本地文件、调用 MCP 服务器、还是执行远程任务，都通过相同的工具调用机制完成。

---

## 3.4 标准输入输出：接口即契约

Unix 程序通过 stdin/stdout/stderr 通信。这个设计的精妙之处在于：

- **解耦**：程序不需要知道数据从哪里来、到哪里去
- **可组合**：任何程序都可以和任何程序组合
- **可测试**：用文件模拟输入，捕获输出验证结果

Claude Code 的工具系统也遵循类似的契约：

```typescript
// 每个工具都有统一的接口
type Tool = {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema  // 输入契约
  execute(input: I, context: ToolUseContext): Promise<ToolResult<O>>  // 输出契约
}
```

输入有 schema 约束，输出有统一的 `ToolResult` 格式。这就是现代版的 stdin/stdout。

---

## 3.5 CLI 的黄金时代

1970-2000 年代，CLI 工具统治了软件世界。`vim`、`emacs`、`make`、`grep`、`sed`、`awk`——这些工具至今仍在使用，因为它们的设计足够好。

CLI 工具的优势：
- **可脚本化**：可以被其他程序调用
- **可组合**：通过管道和重定向组合
- **低开销**：不需要 GUI 框架
- **可远程**：通过 SSH 在任何地方使用

Claude Code 选择 CLI 作为主要界面，不是因为 GUI 不好，而是因为 CLI 的这些特性对 AI Agent 来说至关重要：**AI 需要能被脚本调用、能与其他工具组合、能在服务器上运行**。

---

## 3.6 make：最早的"任务编排"

`make` 是 1976 年发明的构建工具，它的核心思想是**依赖图**：

```makefile
# 目标：依赖
app: main.o utils.o
    gcc -o app main.o utils.o

main.o: main.c
    gcc -c main.c

utils.o: utils.c
    gcc -c utils.c
```

`make` 会分析依赖关系，只重新构建需要更新的部分。

这个思想在 Claude Code 的任务系统里有直接的体现：任务有依赖关系，有状态（pending/running/completed/failed），有并行执行的能力。

---

## 3.7 从 CLI 到 TUI：终端的进化

随着终端能力的增强，出现了 TUI（Terminal User Interface）——在终端里渲染复杂界面的技术。

`vim`、`htop`、`ncurses` 应用——这些工具证明了终端不只能显示文本，还能有丰富的交互界面。

Claude Code 使用 **Ink**（React for CLI）构建 TUI 界面。这是一个重要的技术选择：

```tsx
// Claude Code 的 UI 组件，用 React 写的
function App() {
  return (
    <Box flexDirection="column">
      <Header />
      <MessageList messages={messages} />
      <InputBox onSubmit={handleSubmit} />
    </Box>
  )
}
```

用 React 写终端 UI，意味着：
- 组件化、可复用
- 声明式渲染
- 状态驱动
- 开发者熟悉的编程模型

---

## 3.8 Unix 哲学在 Claude Code 中的体现

| Unix 原则 | Claude Code 的实现 |
|----------|------------------|
| 单一职责 | 每个工具只做一件事（`GrepTool` 只搜索，`FileEditTool` 只编辑） |
| 组合性 | 工具可以链式调用，输出作为下一个工具的输入 |
| 透明性 | 每次工具调用都显示给用户，可以审查和中断 |
| 文本流 | 工具通过结构化文本（JSON）通信 |
| 可脚本化 | 支持非交互模式，可被脚本调用 |
| 管道 | 支持 stdin 输入（`cat file | claude "分析这个"`） |

---

## 3.9 小结

Unix 哲学给了我们三个永恒的设计原则：单一职责、组合性、透明性。

50 年后，这些原则依然是构建好工具的基础。Claude Code 不是凭空发明的，它站在 Unix 传统的肩膀上，把这些原则应用到了 AI Agent 的设计中。

理解这个传统，你就理解了为什么 Claude Code 选择 CLI 而不是 GUI，为什么工具系统这样设计，为什么每个工具都有清晰的单一职责。

---

*下一章：[REPL 的演化史](./04-repl-evolution.md)*
