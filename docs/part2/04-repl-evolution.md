# 第 4 章：REPL 的演化史

> "The read-eval-print loop is the simplest possible interactive programming environment."

---

## 4.1 什么是 REPL

REPL 是 **Read-Eval-Print Loop** 的缩写：

1. **Read**：读取用户输入
2. **Eval**：执行/求值
3. **Print**：打印结果
4. **Loop**：回到第一步

这是最简单的交互式编程环境。你输入一行代码，立刻看到结果。

```python
>>> 1 + 1
2
>>> "hello".upper()
'HELLO'
>>> [x**2 for x in range(5)]
[0, 1, 4, 9, 16]
```

---

## 4.2 REPL 的起源：Lisp（1958）

REPL 的概念来自 Lisp，由 John McCarthy 在 1958 年发明。

Lisp 的 REPL 是革命性的，因为它打破了"编写-编译-运行"的传统循环，让程序员可以**即时探索**。

```lisp
; Lisp REPL，1960 年代
> (+ 1 2)
3
> (defun square (x) (* x x))
SQUARE
> (square 5)
25
```

这个"即时反馈"的思想，是所有现代交互式工具的基础。

---

## 4.3 Shell：操作系统的 REPL

Unix Shell（sh、bash、zsh）是操作系统层面的 REPL：

```bash
$ ls -la
$ cd src
$ grep -r "TODO" .
$ git status
```

Shell 的 REPL 循环：
- **Read**：读取命令行输入
- **Eval**：解析命令，fork 子进程执行
- **Print**：显示输出
- **Loop**：显示新的提示符

Shell 的伟大之处在于它把**操作系统的能力**暴露给了用户，通过一个简单的文本界面。

---

## 4.4 Node.js REPL：JavaScript 的即时环境

2009 年，Node.js 带来了服务端 JavaScript，也带来了 Node.js REPL：

```javascript
$ node
> const arr = [1, 2, 3, 4, 5]
undefined
> arr.filter(x => x % 2 === 0)
[ 2, 4 ]
> arr.reduce((sum, x) => sum + x, 0)
15
```

Node.js REPL 的特点：
- 支持多行输入
- 自动补全
- 历史记录
- 可以 `require` 模块

---

## 4.5 IPython/Jupyter：科学计算的 REPL

2001 年，Fernando Pérez 创建了 IPython，后来演化为 Jupyter Notebook。

Jupyter 把 REPL 的概念推向了新高度：

```python
# Cell 1
import pandas as pd
df = pd.read_csv('data.csv')
df.head()
# 立刻显示表格

# Cell 2
df.describe()
# 立刻显示统计信息

# Cell 3
df.plot(kind='bar')
# 立刻显示图表
```

Jupyter 的创新：
- **Cell 模型**：代码分块执行，每块有独立输出
- **富媒体输出**：不只是文本，还有图表、表格、HTML
- **持久状态**：变量在 cells 之间共享
- **叙事性**：代码和文档混合

Claude Code 支持 `NotebookEditTool`，可以直接编辑 Jupyter Notebook，正是因为 Notebook 已经成为数据科学工作流的核心。

---

## 4.6 ChatGPT：对话即 REPL

2022 年，ChatGPT 的出现把 REPL 的概念带到了自然语言层面：

```
用户：帮我写一个快速排序
ChatGPT：[给出代码]

用户：改成支持自定义比较函数
ChatGPT：[修改代码]

用户：加上单元测试
ChatGPT：[添加测试]
```

这是一个新型的 REPL：
- **Read**：读取自然语言输入
- **Eval**：LLM 理解并生成响应
- **Print**：输出文本/代码
- **Loop**：继续对话

但 ChatGPT 的 REPL 有一个根本限制：**它只能生成文本，不能执行动作**。

---

## 4.7 Claude Code：行动即 REPL

Claude Code 是 REPL 演化的下一步：**把自然语言 REPL 和工具执行结合起来**。

```
用户：帮我找出项目里所有未使用的导入并删除

Claude Code：
  → GlobTool: 找到所有 .ts 文件
  → FileReadTool: 读取每个文件
  → 分析未使用的导入
  → FileEditTool: 删除未使用的导入
  → BashTool: 运行 tsc 验证编译通过
  → 完成，共修改 23 个文件
```

这个 REPL 的特点：
- **Read**：读取自然语言意图
- **Eval**：规划并执行工具调用链
- **Print**：显示执行过程和结果
- **Loop**：等待下一个指令

---

## 4.8 REPL 演化的关键维度

| 维度 | Lisp REPL | Shell | Jupyter | ChatGPT | Claude Code |
|------|-----------|-------|---------|---------|-------------|
| 输入类型 | 代码 | 命令 | 代码 | 自然语言 | 自然语言 |
| 执行能力 | 计算 | 系统操作 | 计算+可视化 | 无 | 系统操作+计算+网络 |
| 状态持久 | 会话内 | 会话内 | Notebook | 对话内 | 对话内+文件系统 |
| 输出类型 | 值 | 文本 | 富媒体 | 文本 | 文本+动作结果 |
| 可组合性 | 低 | 高（管道） | 中 | 低 | 高（工具链） |

---

## 4.9 Claude Code 的 REPL 设计

Claude Code 的 REPL 实现在 `src/entrypoints/cli.tsx` 中，核心循环大致是：

```
1. 显示提示符，等待用户输入
2. 解析输入（斜杠命令 or 自然语言）
3. 如果是斜杠命令，直接执行
4. 如果是自然语言，提交给 QueryEngine
5. QueryEngine 调用 Claude API，获取流式响应
6. 响应中包含工具调用时，执行工具
7. 工具结果回填到对话，继续生成
8. 显示最终结果
9. 回到第 1 步
```

这个循环有几个关键设计：

**流式显示**：Claude 的响应是流式的，用户不需要等待完整响应才能看到内容。

**工具调用透明**：每次工具调用都显示给用户，用户可以看到 Claude 在做什么。

**可中断**：用户随时可以按 `Ctrl+C` 中断当前操作。

**历史记录**：对话历史保存在本地，可以恢复之前的会话。

---

## 4.10 小结

REPL 的演化史是一部**降低交互门槛**的历史：

- Lisp REPL：让程序员可以即时探索代码
- Shell：让用户可以即时操作操作系统
- Jupyter：让数据科学家可以即时探索数据
- ChatGPT：让普通人可以用自然语言交互
- Claude Code：让开发者可以用自然语言操作代码库

每一步都在降低"从想法到执行"的摩擦。Claude Code 是这条路上目前最远的一步。

---

*下一章：[从聊天机器人到 Agent](./05-from-chatbot-to-agent.md)*
