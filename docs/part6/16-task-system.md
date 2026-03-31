# 第 16 章：任务系统设计

> 后台任务是 Agent 从"同步助手"进化为"异步协作者"的关键。

---

## 16.1 为什么需要任务系统

早期的 AI 助手是完全同步的：用户问，AI 答，用户等待。

但真实的工程工作不是这样的：
- 构建一个大型项目可能需要 10 分钟
- 运行完整的测试套件可能需要 30 分钟
- 数据处理任务可能需要几个小时

如果 Claude 在执行这些任务时用户什么都不能做，体验会很差。

任务系统解决了这个问题：**让 Claude 能够在后台执行长时间任务，同时继续响应用户的其他请求**。

---

## 16.2 任务类型

`src/Task.ts` 定义了 7 种任务类型：

```typescript
type TaskType =
  | 'local_bash'          // 本地 Shell 命令（最常用）
  | 'local_agent'         // 本地子代理（独立的 Claude 实例）
  | 'remote_agent'        // 远程代理（在 CCR 上运行）
  | 'in_process_teammate' // 进程内协作代理（共享内存）
  | 'local_workflow'      // 本地工作流（多步骤任务）
  | 'monitor_mcp'         // MCP 监控任务
  | 'dream'               // 自动梦境模式（实验性）
```

每种类型有不同的执行环境和能力：

| 类型 | 执行位置 | 隔离级别 | 通信方式 |
|------|---------|---------|---------|
| local_bash | 本地进程 | 低 | stdout/stderr |
| local_agent | 本地子进程 | 中 | 文件 + 消息 |
| remote_agent | 远程服务器 | 高 | HTTP API |
| in_process_teammate | 同一进程 | 无 | 共享内存 |

---

## 16.3 任务状态机

每个任务都有明确的状态机：

```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

```typescript
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

// 终态判断
function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

终态是不可逆的：一旦任务完成、失败或被杀死，就不能再转换到其他状态。这个设计防止了状态机的混乱。

---

## 16.4 任务 ID 的设计

任务 ID 的设计很有意思：

```typescript
// 任务 ID 前缀
const TASK_ID_PREFIXES = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

// 生成任务 ID：前缀 + 8 位随机字符
// 例如：b3k9x2mf（本地 bash 任务）
//       a7p1n4qz（本地代理任务）
function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type]
  const bytes = randomBytes(8)
  // 使用 36 进制（数字 + 小写字母）
  // 36^8 ≈ 2.8 万亿种组合，足以防止暴力枚举
  return prefix + encode(bytes, TASK_ID_ALPHABET)
}
```

前缀让人一眼就能看出任务类型，随机后缀保证唯一性。注释中明确说明了安全考量：**防止暴力枚举符号链接攻击**。

---

## 16.5 任务输出的持久化

每个任务的输出都写入磁盘文件：

```typescript
type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  outputFile: string    // 输出文件路径
  outputOffset: number  // 已读取的偏移量（用于增量读取）
  startTime: number
  endTime?: number
  notified: boolean     // 是否已发送完成通知
}
```

输出文件的好处：
- **持久化**：任务输出不会因为进程重启而丢失
- **增量读取**：`TaskOutputTool` 可以从 `outputOffset` 开始读取，避免重复读取
- **大输出支持**：不受内存限制，可以处理 GB 级别的输出

---

## 16.6 任务的生命周期管理

```typescript
// 创建任务
const taskId = await TaskCreateTool.execute({
  command: 'npm run build',
  description: '构建生产版本'
}, context)
// 返回：{ taskId: 'b3k9x2mf' }

// 检查状态
const status = await TaskGetTool.execute({ taskId }, context)
// 返回：{ status: 'running', outputOffset: 1024 }

// 读取输出（增量）
const output = await TaskOutputTool.execute({
  taskId,
  block: false  // 非阻塞，立即返回当前输出
}, context)

// 等待完成（阻塞）
const result = await TaskOutputTool.execute({
  taskId,
  block: true,  // 阻塞，等待任务完成
  timeout: 300000  // 5 分钟超时
}, context)

// 停止任务
await TaskStopTool.execute({ taskId }, context)
```

---

## 16.7 任务与主对话的协调

任务在后台运行时，主对话仍然可以继续。Claude 可以：

1. 启动多个后台任务
2. 继续处理用户的其他请求
3. 定期检查任务状态
4. 任务完成时汇报结果

```
用户：同时运行前端和后端的测试

Claude：好的，我来并行运行两个测试套件。

  → TaskCreateTool: npm run test:frontend（任务 ID: b1a2b3c4）
  → TaskCreateTool: npm run test:backend（任务 ID: b5d6e7f8）

  两个测试正在后台运行。我来检查进度...

  → TaskGetTool: b1a2b3c4 → running（已运行 30 秒）
  → TaskGetTool: b5d6e7f8 → running（已运行 30 秒）

  [2 分钟后]

  → TaskGetTool: b1a2b3c4 → completed ✓
  → TaskGetTool: b5d6e7f8 → failed ✗

  前端测试通过！后端测试失败，让我看看错误...
  → TaskOutputTool: b5d6e7f8 → [错误输出]
```

---

## 16.8 任务的清理机制

任务完成后需要清理资源：

```typescript
type TaskHandle = {
  taskId: string
  cleanup?: () => void  // 清理函数（关闭进程、释放资源等）
}
```

`cleanup` 函数在任务终止时调用，确保：
- 子进程被正确终止
- 临时文件被删除
- 网络连接被关闭

---

## 16.9 小结

任务系统让 Claude Code 从同步助手进化为异步协作者：

- **7 种任务类型**：覆盖从简单 Shell 命令到复杂多代理协作
- **清晰的状态机**：pending → running → terminal
- **持久化输出**：任务输出写入磁盘，支持增量读取
- **并行执行**：多个任务可以同时运行
- **生命周期管理**：创建、监控、读取、停止的完整工具集

---

*下一章：[多代理架构](./17-multi-agent.md)*
