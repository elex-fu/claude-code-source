# Claude Code Bash 命令系统技术分析

## 1. 模块总结介绍

Bash Command 系统是 Claude Code 的「命令解析与安全中枢」，负责解析、分析和安全执行 Bash 命令。该系统采用纯 TypeScript 实现的 Tree-sitter 兼容解析器，提供精确的 AST 分析和安全检测。

核心特性：
- **纯 TS 解析器**：无需 WASM/Native 依赖的 Bash AST 解析
- **安全 walker**：检测危险命令模式（eval、subshell 等）
- **资源限制**：50ms 超时 + 50k 节点上限防止攻击
- **多 Shell 支持**：Bash、Zsh、PowerShell 统一抽象

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                   Bash Command System                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Parser Layer                                               │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│   │  bashParser.ts  │  │   parser.ts     │  │   ast.ts    │ │
│   │  • Tokenizer    │  │  • Command parse│  │  • Security │ │
│   │  • Recursive    │  │  • Env extract  │  │    walker   │ │
│   │    descent      │  │  • PARSE_ABORTED│  │  • Danger   │ │
│   └─────────────────┘  └─────────────────┘  │    detect   │ │
│                                             └─────────────┘ │
│                                                              │
│   Shell Abstraction                                          │
│   ┌──────────────────────────────────────────────────────┐  │
│   │              ShellProvider Interface                 │  │
│   │  • BashProvider    • PowerShellProvider             │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                              │
│   Execution Layer                                            │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│   │   Shell.ts      │  │ ShellCommand.ts │  │   wrapSpawn │ │
│   │  • exec()       │  │  • Result type  │  │  • Timeout  │ │
│   │  • findShell()  │  │  • Streaming    │  │  • Sandbox  │ │
│   └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 解析器核心

### 3.1 纯 TypeScript 解析器

```typescript
// bashParser.ts - 纯 TS 实现，无需 WASM
export type TsNode = {
  type: string
  text: string
  startIndex: number   // UTF-8 字节偏移
  endIndex: number
  children: TsNode[]
}

interface ParserModule {
  parse: (source: string, timeoutMs?: number) => TsNode | null
}
```

### 3.2 资源限制

```typescript
const PARSE_TIMEOUT_MS = 50     // 50ms 超时
const MAX_NODES = 50_000        // 5万节点上限
const MAX_COMMAND_LENGTH = 10000 // 1万字符上限

function parseSource(source: string, timeoutMs = PARSE_TIMEOUT_MS): TsNode | null {
  // 1. 长度检查
  if (source.length > MAX_COMMAND_LENGTH) return null
  
  // 2. 启动解析（带超时机制）
  const startTime = Date.now()
  const nodeCount = { value: 0 }
  
  function checkLimits(): boolean {
    if (Date.now() - startTime > timeoutMs) return false
    if (nodeCount.value > MAX_NODES) return false
    return true
  }
  
  // 3. 递归下降解析
  return parseProgram(source, checkLimits, nodeCount)
}
```

### 3.3 Tokenizer 设计

```typescript
type TokenType =
  | 'WORD'           // 普通单词
  | 'NUMBER'         // 数字
  | 'OP'             // 运算符 (|, &&, ||, ;)
  | 'NEWLINE'
  | 'COMMENT'
  | 'DQUOTE'         // 双引号字符串
  | 'SQUOTE'         // 单引号字符串
  | 'DOLLAR'         // $变量
  | 'DOLLAR_PAREN'   // $(命令替换)
  | 'DOLLAR_BRACE'   // ${参数扩展}
  | 'BACKTICK'       // `命令替换`
  | 'EOF'

interface Token {
  type: TokenType
  value: string
  start: number   // UTF-8 字节偏移
  end: number
}
```

## 4. AST 节点类型

### 4.1 核心节点类型

```typescript
// program: 整个脚本
type ProgramNode = {
  type: 'program'
  children: (CommandNode | PipelineNode | CommentNode)[]
}

// command: 简单命令
type CommandNode = {
  type: 'command'
  children: [
    CommandNameNode,
    ...ArgumentNode[]
  ]
}

// pipeline: 管道命令 (cmd1 | cmd2 | cmd3)
type PipelineNode = {
  type: 'pipeline'
  children: CommandNode[]
}

// redirected_statement: 重定向命令 (cmd > file)
type RedirectedStatementNode = {
  type: 'redirected_statement'
  children: [CommandNode, ...RedirectNode[]]
}
```

### 4.2 变量与替换

```typescript
// variable_assignment: 变量赋值 (VAR=value)
type VariableAssignmentNode = {
  type: 'variable_assignment'
  text: string  // "VAR=value"
}

// command_substitution: 命令替换 ($(...) 或 `...`)
type CommandSubstitutionNode = {
  type: 'command_substitution'
  children: [ProgramNode]
}

// process_substitution: 进程替换 (<(...) 或 >(...))
type ProcessSubstitutionNode = {
  type: 'process_substitution'
  children: [ProgramNode]
}

// string: 带展开的字符串 ("hello $name")
type StringNode = {
  type: 'string'
  children: (TextNode | ExpansionNode)[]
}
```

## 5. 命令解析流程

### 5.1 解析入口

```typescript
// parser.ts
export async function parseCommand(
  command: string
): Promise<ParsedCommandData | null> {
  if (!command || command.length > MAX_COMMAND_LENGTH) return null
  
  // Feature gate: TREE_SITTER_BASH
  if (!feature('TREE_SITTER_BASH')) return null
  
  await ensureParserInitialized()
  const mod = getParserModule()
  if (!mod) return null
  
  try {
    const rootNode = mod.parse(command)
    if (!rootNode) return null
    
    // 提取命令节点
    const commandNode = findCommandNode(rootNode, null)
    
    // 提取环境变量
    const envVars = extractEnvVars(commandNode)
    
    return { rootNode, envVars, commandNode, originalCommand: command }
  } catch {
    return null
  }
}
```

### 5.2 命令节点查找

```typescript
function findCommandNode(node: Node, parent: Node | null): Node | null {
  // 直接是命令类型
  if (COMMAND_TYPES.has(node.type)) return node
  
  // 变量赋值后的命令 (VAR=value cmd)
  if (node.type === 'variable_assignment' && parent) {
    return parent.children.find(
      c => COMMAND_TYPES.has(c.type) && c.startIndex > node.startIndex
    ) ?? null
  }
  
  // 管道：递归查找第一个命令
  if (node.type === 'pipeline') {
    for (const child of node.children) {
      const result = findCommandNode(child, node)
      if (result) return result
    }
  }
  
  // 重定向语句：查找内部命令
  if (node.type === 'redirected_statement') {
    return node.children.find(c => COMMAND_TYPES.has(c.type)) ?? null
  }
  
  // 递归搜索
  for (const child of node.children) {
    const result = findCommandNode(child, node)
    if (result) return result
  }
  
  return null
}
```

## 6. 安全 Walker

### 6.1 危险命令检测

```typescript
// ast.ts - 安全 walker
const EVAL_LIKE_BUILTINS = new Set([
  'eval',        // eval "..."
  'source',      // source file
  '.',           // . file (source 的别名)
  'trap',        // trap 'cmd' SIGNAL
  'enable',      // enable -n builtin
  'hash',        // hash -r
  'caller',      // caller
  'coproc',      // coproc
  'builtin',     // builtin cmd
  'command',     // command -v
  'help',        // help
])

const DANGEROUS_PATTERNS = [
  { type: 'command_substitution', risk: 'high' },    // $(...)
  { type: 'process_substitution', risk: 'high' },    // <(...)
  { type: 'subshell', risk: 'high' },                // (...)
]
```

### 6.2 AST 安全遍历

```typescript
export function walkForSecurity(node: Node): SecurityReport {
  const risks: Risk[] = []
  
  function walk(node: Node, depth: number): void {
    // 深度限制
    if (depth > 100) {
      risks.push({ type: 'depth_exceeded', level: 'error' })
      return
    }
    
    // 检测危险节点类型
    if (node.type === 'command_substitution') {
      risks.push({
        type: 'command_substitution',
        level: 'high',
        text: node.text,
      })
    }
    
    if (node.type === 'process_substitution') {
      risks.push({
        type: 'process_substitution',
        level: 'high',
        text: node.text,
      })
    }
    
    // 检测危险内置命令
    if (node.type === 'command_name') {
      const cmd = node.text
      if (EVAL_LIKE_BUILTINS.has(cmd)) {
        risks.push({
          type: 'dangerous_builtin',
          level: 'high',
          command: cmd,
        })
      }
    }
    
    // 递归遍历子节点
    for (const child of node.children) {
      walk(child, depth + 1)
    }
  }
  
  walk(node, 0)
  return { risks, safe: risks.length === 0 }
}
```

### 6.3 解析中止信号

```typescript
// SECURITY: 解析被中止的特殊信号
export const PARSE_ABORTED = Symbol('parse-aborted')

export async function parseCommandRaw(
  command: string
): Promise<Node | null | typeof PARSE_ABORTED> {
  // ... 解析逻辑
  
  const result = mod.parse(command)
  
  // 模块已加载但解析返回 null = 超时/节点上限
  if (result === null) {
    logEvent('tengu_tree_sitter_parse_abort', {
      cmdLength: command.length,
    })
    return PARSE_ABORTED  // 调用者必须视为失败关闭
  }
  
  return result
}
```

## 7. Shell 抽象层

### 7.1 ShellProvider 接口

```typescript
// shell/shellProvider.ts
export interface ShellProvider {
  readonly type: ShellType
  
  // 执行命令
  exec(command: string, options: ExecOptions): Promise<ExecResult>
  
  // 构建环境
  buildEnvironment(): Record<string, string>
  
  // 检测当前目录
  detectCwd(output: string): string | null
  
  // 生成提示符检测命令
  getPromptCommand(): string
}

export type ShellType = 'bash' | 'powershell'
```

### 7.2 Bash Provider

```typescript
// shell/bashProvider.ts
export async function createBashShellProvider(
  shellPath: string
): Promise<ShellProvider> {
  return {
    type: 'bash',
    
    async exec(command, options) {
      const subprocess = spawn(shellPath, ['-c', command], {
        env: buildBashEnvironment(),
        cwd: options.cwd,
      })
      
      return wrapSpawn(subprocess, options)
    },
    
    buildEnvironment() {
      return {
        ...process.env,
        PS1: '\x01\x02',  // 特殊标记便于检测 cwd
        PS0: '\x01\x02',
      }
    },
    
    detectCwd(output) {
      // 解析 PS1/PS0 标记提取 cwd
      const match = output.match(/\x01\x02(.+?)\x01\x02/)
      return match?.[1] ?? null
    },
  }
}
```

### 7.3 Shell 自动检测

```typescript
// Shell.ts
export async function findSuitableShell(): Promise<string> {
  // 1. 检查覆盖变量
  const shellOverride = process.env.CLAUDE_CODE_SHELL
  if (shellOverride && isExecutable(shellOverride)) {
    return shellOverride
  }
  
  // 2. 检查 SHELL 环境变量
  const envShell = process.env.SHELL
  const isSupported = envShell && 
    (envShell.includes('bash') || envShell.includes('zsh'))
  
  // 3. 使用 which 查找
  const [zshPath, bashPath] = await Promise.all([
    which('zsh'),
    which('bash'),
  ])
  
  // 4. 按优先级排序
  const candidates = [
    isSupported ? envShell : null,
    zshPath,
    bashPath,
    '/bin/zsh',
    '/bin/bash',
    '/usr/bin/zsh',
    '/usr/bin/bash',
  ].filter(Boolean)
  
  const shellPath = candidates.find(isExecutable)
  
  if (!shellPath) {
    throw new Error(
      'No suitable shell found. Claude CLI requires a Posix shell environment.'
    )
  }
  
  return shellPath
}
```

## 8. 执行层

### 8.1 执行配置

```typescript
export type ExecOptions = {
  timeout?: number           // 超时（默认 30 分钟）
  onProgress?: (            // 进度回调
    lastLines: string,
    allLines: string,
    totalLines: number,
    totalBytes: number,
    isIncomplete: boolean
  ) => void
  preventCwdChanges?: boolean   // 阻止目录变更
  shouldUseSandbox?: boolean    // 使用沙箱
  shouldAutoBackground?: boolean // 自动后台化
  onStdout?: (data: string) => void  // stdout 流回调
}
```

### 8.2 执行流程

```typescript
export async function exec(
  command: string,
  abortSignal: AbortSignal,
  shellType: ShellType,
  options?: ExecOptions
): Promise<ShellCommand> {
  const timeout = options?.timeout || DEFAULT_TIMEOUT // 30min
  
  // 1. 获取 Shell Provider
  const provider = await resolveProvider[shellType]()
  
  // 2. 生成命令 ID
  const id = Math.floor(Math.random() * 0x10000).toString(16)
  
  // 3. 解析命令（安全分析）
  const parseResult = await parseCommandRaw(command)
  if (parseResult === PARSE_ABORTED) {
    // 解析被中止 = 命令过于复杂，拒绝执行
    return createFailedCommand(command, 'Parse aborted: command too complex')
  }
  
  // 4. 安全 walker 检查
  if (parseResult) {
    const securityReport = walkForSecurity(parseResult)
    if (!securityReport.safe) {
      // 危险命令需要额外确认
      const confirmed = await confirmDangerousCommand(securityReport)
      if (!confirmed) {
        return createAbortedCommand(command, 'User declined dangerous command')
      }
    }
  }
  
  // 5. 执行命令
  const result = await provider.exec(command, {
    ...options,
    timeout,
    signal: abortSignal,
  })
  
  return result
}
```

## 9. 与子进程集成

### 9.1 wrapSpawn 包装器

```typescript
// ShellCommand.ts
export function wrapSpawn(
  subprocess: ChildProcess,
  options: WrapOptions
): Promise<ShellCommand> {
  return new Promise((resolve, reject) => {
    const stdout: string[] = []
    const stderr: string[] = []
    let killed = false
    
    // 设置超时
    const timeoutId = options.timeout ? 
      setTimeout(() => {
        killed = true
        subprocess.kill('SIGTERM')
        reject(new Error(`Command timed out after ${options.timeout}ms`))
      }, options.timeout) : null
    
    // 处理 stdout
    subprocess.stdout?.on('data', (data: Buffer) => {
      const chunk = data.toString('utf-8')
      stdout.push(chunk)
      options.onStdout?.(chunk)
    })
    
    // 处理 stderr
    subprocess.stderr?.on('data', (data: Buffer) => {
      stderr.push(data.toString('utf-8'))
    })
    
    // 进程结束
    subprocess.on('close', (code, signal) => {
      if (timeoutId) clearTimeout(timeoutId)
      
      resolve({
        command: options.command,
        stdout: stdout.join(''),
        stderr: stderr.join(''),
        exitCode: code ?? undefined,
        signal: signal ?? undefined,
        killed,
      })
    })
    
    // 错误处理
    subprocess.on('error', (error) => {
      if (timeoutId) clearTimeout(timeoutId)
      reject(error)
    })
  })
}
```

## 10. 技术创新点

1. **纯 TypeScript 解析器**：不依赖 WASM/Native 模块，实现 Tree-sitter 兼容的 Bash AST 解析

2. **资源限制防护**：50ms 超时 + 50k 节点上限防止解析器被恶意输入攻击

3. **PARSE_ABORTED 信号**：区分"解析器不可用"和"解析被中止"，实现失败关闭的安全策略

4. **安全 Walker 模式**：AST 遍历检测 eval、subshell 等危险模式，支持细粒度风险评估

5. **ShellProvider 抽象**：统一接口支持 Bash、Zsh、PowerShell 等多种 Shell

6. **UTF-8 字节偏移**：使用字节偏移而非字符索引，兼容多字节字符和 ANSI 序列

7. **特性门控**：TREE_SITTER_BASH 特性开关控制解析器启用，便于灰度发布

---

Bash Command 系统通过纯 TS 解析器、安全 Walker、资源限制等设计，实现了精确且安全的命令解析。理解其设计，对于构建安全的 AI 命令执行系统具有参考价值。
