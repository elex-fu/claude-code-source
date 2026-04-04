# Hooks Design 深度技术思考

## 一、钩子系统的哲学定位

### 1.1 扩展点 vs 控制流

Claude Code 的 hooks 系统实现了「开放封闭原则」的极致表达：

```
封闭：核心 Agent 循环的稳定性
  └── 查询-采样-工具执行的主流程不可变

开放：扩展点的无限可能
  ├── SessionStart: 初始化逻辑
  ├── ToolUse: 拦截和修改工具调用
  ├── Stop: 终止条件判断
  └── Sampling: 后处理采样结果
```

这种设计让核心引擎保持精简，而将千变万化的业务逻辑外置到 hooks。

### 1.2 五类钩子的能力谱系

```
能力递增：

command ──▶ 执行 shell 命令，获取文本输出
   │
   ▼
prompt ───▶ 调用轻量模型做条件判断
   │
   ▼
agent ────▶ 启动子智能体，最多 50 轮工具调用
   │
   ▼
http ─────▶ HTTP POST 请求，可集成外部服务
   │
   ▼
function ─▶ 内存中的 TypeScript 回调，零延迟
```

每种类型对应不同的「能力-成本」权衡：从简单的命令执行到复杂的 Agent 推理，再到直接的代码回调。

## 二、权限系统的竞态艺术

### 2.1 多源决策的复杂性

当工具权限请求发生时，可能有多个来源同时响应：

```
                    ┌─────────────────┐
                    │  Permission     │
                    │   Request       │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  Hook Handler │    │  Classifier   │    │  UI Dialog    │
│  (自动规则)    │    │  (AI 判断)    │    │  (用户确认)   │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        │    ┌───────────────┴────────────────────┘
        │    │
        ▼    ▼
┌─────────────────────────────────────┐
│       createResolveOnce()           │
│  「先到先得」的原子决议机制            │
└─────────────────────────────────────┘
```

### 2.2 ResolveOnce 的精妙设计

```typescript
// 伪代码示意
function createResolveOnce<T>(resolve: (value: T) => void) {
  let claimed = false
  return {
    claim(): boolean {
      if (claimed) return false
      claimed = true
      return true  // 只有第一个调用者返回 true
    },
    resolve(value: T) {
      if (!claimed) return  // 被拒绝的调用者无法 resolve
      resolve(value)
    }
  }
}
```

这种「原子 claim」模式确保了：
- 只有一个来源能最终决策
- 避免了重复 resolve 导致的竞态条件
- 后续来源自动被忽略，无需复杂的状态同步

### 2.3 分层权限桥接

不同角色（Coordinator/Worker/Main）有各自的权限处理路径：

```
主智能体:
  └─▶ handleInteractivePermission
      └─▶ 显示 UI 弹窗

Coordinator:
  └─▶ 执行 PermissionRequest hooks
      └─▶ 若仍 ask，fallthrough 到 Classifier

Swarm Worker:
  └─▶ 尝试 Classifier
      └─▶ 若仍 ask，通过 Mailbox 请求 Leader
```

这种分层确保了「适合的处理者在适合的层级做出决策」。

## 三、输入编辑的状态机美学

### 3.1 Emacs 模式的编辑语义

`useTextInput` 实现了完整的 Emacs 键绑定语义：

```
光标移动：
  Ctrl+A ──▶ 行首 (Home)
  Ctrl+E ──▶ 行尾 (End)
  Meta+F ──▶ 下一个单词
  Meta+B ──▶ 上一个单词

编辑操作：
  Ctrl+K ──▶ Kill to EOL (剪切到行尾)
  Ctrl+Y ──▶ Yank (粘贴)
  Meta+Y ──▶ Yank Pop (循环粘贴历史)

Kill Ring：
  多次 Ctrl+K 累积内容
  形成可循环的粘贴历史
```

这些不是简单的快捷键，而是遵循 Emacs 的「编辑语言」哲学。

### 3.2 Vim 模式的状态转换

`useVimInput` 在 Emacs 基础上叠加了 Vim 的状态机：

```
┌──────────┐     i/a/o     ┌──────────┐
│  NORMAL  │◀─────────────▶│ INSERT   │
└────┬─────┘               └──────────┘
     │
     │ v/V/Ctrl+V
     ▼
┌──────────┐
│ VISUAL   │
└────┬─────┘
     │
     │ d/y/c/
     ▼
┌──────────┐
│OPERATOR- │
│PENDING   │
└──────────┘
```

状态间的转换通过 `vim/transitions.js` 和 `vim/operators.js` 严格定义。

### 3.3 输入缓冲的防抖哲学

```
用户输入 ──▶ 缓冲区 ──▶ debounce(50ms) ──▶ 提交
                │
                └── 期间可撤销 (Ctrl+Z)
```

防抖不是简单的延迟，而是创造了「后悔窗口」——在提交前用户可以撤销。

## 四、Agent 作为钩子的元认知

### 4.1 execAgentHook： hook 的终极形态

```
传统 hook: 静态规则
  if (condition) then action

Agent hook: 动态推理
  └─▶ 启动子 Agent
      └─▶ 最多 50 轮工具调用
          └─▶ 访问 SyntheticOutputTool
              └─▶ 返回结构化判断
```

这种设计让 hook 具备「情境感知」能力——它可以：
- 查询代码库状态
- 分析历史提交记录
- 读取配置文件
- 然后做出 informed decision

### 4.2 Post-Sampling Hook 的闭环学习

```
主循环采样完成
       │
       ▼
┌──────────────┐
│ PostSampling │
│    Hooks     │
└──────┬───────┘
       │
       ├── Skill Improvement
       │      └─▶ 分析最近 5 条用户消息
       │      └─▶ 检测 Skill 优化机会
       │      └─▶ 生成修改建议
       │
       └── 其他内部 hook
```

这是「元认知」的体现——系统不仅执行任务，还反思如何改进自身。

## 五、TUI 的基础设施创新

### 5.1 ClockContext：统一的时间源

```
传统方式：                    ClockContext:
每个动画独立计时              全局统一时钟
  │                              │
  ├── Spinner A (setInterval)    ├── 订阅机制
  ├── Spinner B (setInterval)    │      (keepAlive)
  └── Spinner C (setInterval)    │
                                 ▼
                          全局 RAF 时钟
                               │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                Spinner A  Spinner B  Spinner C
```

统一时钟的优势：
- 终端失焦时统一暂停，节省 CPU
- 动画同步，避免视觉撕裂
- 单一控制点，便于调试

### 5.2 useInput 的 Raw Mode 管理

```
组件挂载
    │
    ├── useLayoutEffect (同步)
    │      └─▶ 启用 raw mode
    │
    └── 用户输入
           └─▶ 通过 EventEmitter 分发
           └─▶ stopImmediatePropagation() 支持优先级

组件卸载
    │
    └── useLayoutEffect 清理 (同步)
           └─▶ 禁用 raw mode
```

使用 `useLayoutEffect` 而非 `useEffect` 确保了 raw mode 的启用/禁用精确对应组件生命周期，避免字符泄露。

### 5.3 焦点管理的 DOM 模拟

终端没有原生的「焦点」概念，但 Ink 模拟了完整的焦点系统：

```
FocusManager
    │
    ├── focusable elements (注册)
    ├── focus history (栈结构)
    └── focus events (onFocus/onBlur)
```

这种模拟使得：
- 表单元素可以有「当前输入框」概念
- 对话框可以「捕获」焦点（模态）
- Tab 键可以循环切换

## 六、Session Hooks 的高并发优化

### 6.1 Map 身份稳定性的 O(1) 优化

```typescript
// 高并发场景下的优化

// 不好的做法 (Record):
setSessionHooks(prev => ({
  ...prev,           // O(n) 复制
  [id]: newStore     // 触发所有监听器
}))

// 好的做法 (Map):
setSessionHooks(prev => {
  const next = new Map(prev)  // Map 的浅拷贝
  next.set(id, newStore)      // O(1)
  return next  // 即使返回新 Map，React 也能优化
})
```

使用 `Map` 而非 `Record` 的深层原因：
- `.set()` 是 O(1)，不会触发无关的监听器
- `return prev` 可以短路 React 的 re-render
- 在 `parallel()` 启动 N 个 agent 时，复杂度从 O(N²) 降至 O(1)

### 6.2 Function Hook 的零序列化验证

```
其他 hook 类型:          function hook:
  │                        │
  ├── 序列化到 settings    ├── 内存中的 TS callback
  ├── 持久化到磁盘         ├── 无 I/O 开销
  ├── 运行时反序列化       ├── 直接调用
  └── 执行                 └── 最低延迟
```

`function` 类型的 hooks 不能持久化，但为零延迟验证提供了可能。

## 七、反思与演进

### 7.1 当前架构的权衡

**Hook 配置的复杂性**

支持 command/prompt/agent/http/function 五类 hooks 提供了灵活性，但也增加了学习成本。用户可能困惑「该用哪种类型？」。

**异步 Hook 的不确定性**

`async: true` 的 command hooks 在后台执行，其输出通过轮询获取。这种模式有 15 秒超时限制，且无法实时交互。

### 7.2 未来演进方向

**Predictive Hooks**

基于对话历史预测哪些 hooks 可能触发，提前准备执行环境，减少延迟。

**Hook Composition**

支持 hooks 的链式组合，如 `hook1.then(hook2).catch(hook3)`，形成更复杂的工作流。

**Visual Hook Editor**

提供图形化界面配置 hooks，降低非开发者用户的使用门槛。

---

Claude Code 的 Hooks 设计模块体现了「可扩展架构」的精髓——核心保持精简稳定，扩展点提供无限可能。从简单的命令执行到复杂的 Agent 推理，从同步回调到异步事件，从 UI 交互到后台处理，hooks 系统提供了一个统一的扩展框架。理解其设计，对于构建任何需要高度可扩展性的 AI 系统都具有重要参考价值。
