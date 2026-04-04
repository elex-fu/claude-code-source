# Claude Code 权限系统技术分析

## 1. 模块总结介绍

权限系统是 Claude Code 的「安全守门员」，控制 AI 对敏感操作的访问。它实现了多层次的权限模型，从粗粒度的自动/询问模式到细粒度的工具级控制。

核心特性：
- **多层级权限**：Auto / Plan / Ask 三种主要模式
- **工具级控制**：每个工具可独立配置 allow/deny/ask
- **分类器预检**：AI 自动判断是否需要权限
- **权限委托**：多 Agent 场景下的权限桥接

## 2. 权限模型架构

```
┌─────────────────────────────────────────────────────────────┐
│                   Permission System                          │
├─────────────────────────────────────────────────────────────┤
│  Permission Mode                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │    Auto     │  │    Plan     │  │        Ask          │ │
│  │             │  │             │  │                     │ │
│  │ 自动执行    │  │ 计划审批    │  │ 每次询问            │ │
│  │ 高风险确认  │  │ 批量执行    │  │ 用户确认            │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                                                              │
│  Tool-Level Control                                         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Tool Pattern: allow | deny | ask                       ││
│  │  Example: FileEditTool=ask, BashTool=deny               ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心类型定义

### 3.1 权限模式

```typescript
// utils/permissions/PermissionMode.ts
export type PermissionMode = 'auto' | 'plan' | 'ask'

export interface PermissionModeConfig {
  mode: PermissionMode
  // Auto 模式下的子配置
  auto?: {
    requireConfirmationFor: string[] // 需要确认的工具模式
  }
  // Plan 模式下的子配置
  plan?: {
    maxIterations: number
    autoApproveSamePlan: boolean
  }
}
```

### 3.2 工具权限

```typescript
// utils/permissions/permissionSetup.ts
export type ToolPermission = 'allow' | 'deny' | 'ask'

export interface ToolPermissionRule {
  pattern: string | RegExp  // 工具名或模式
  permission: ToolPermission
  description?: string
}

export interface ToolPermissionContext {
  rules: ToolPermissionRule[]
  mode: PermissionMode
  isBypassAvailable: boolean
  isBypassEnabled: boolean
}
```

## 4. 权限决策流程

### 4.1 决策流程图

```
Tool Use Request
       │
       ▼
┌─────────────────┐
│  Check Global   │──── Denied ────▶ Block
│    Deny List    │
└────────┬────────┘
         │ Allowed
         ▼
┌─────────────────┐
│  Check Tool     │──── Denied ────▶ Block
│   Permission    │
└────────┬────────┘
         │ Ask
         ▼
┌─────────────────┐
│  Classifier     │──── Low Risk ──▶ Allow
│   Pre-check     │
└────────┬────────┘
         │ High Risk
         ▼
┌─────────────────┐
│  Permission     │──── Approved ──▶ Allow
│    Dialog       │
└────────┬────────┘
         │ Denied
         ▼
       Block
```

### 4.2 核心决策函数

```typescript
// utils/permissions/hasPermissionsToUseTool.ts
export function hasPermissionsToUseTool(
  toolName: string,
  toolInput: Record<string, unknown>,
  context: ToolPermissionContext
): ToolPermission {
  // 1. 检查全局拒绝列表
  if (isGloballyDenied(toolName)) {
    return 'deny'
  }

  // 2. 检查工具特定规则
  const rule = findMatchingRule(toolName, context.rules)
  if (rule) {
    return rule.permission
  }

  // 3. 根据模式默认行为
  switch (context.mode) {
    case 'auto':
      return checkAutoModePermission(toolName, toolInput, context)
    case 'plan':
      return 'ask' // Plan 模式需要显式批准
    case 'ask':
    default:
      return 'ask'
  }
}

function checkAutoModePermission(
  toolName: string,
  toolInput: Record<string, unknown>,
  context: ToolPermissionContext
): ToolPermission {
  // 检查是否需要高权限确认
  if (context.auto?.requireConfirmationFor.some(p => 
    matchesPattern(toolName, p)
  )) {
    return 'ask'
  }

  // 检查是否为高风险操作
  if (isHighRiskOperation(toolName, toolInput)) {
    return 'ask'
  }

  return 'allow'
}
```

## 5. 分类器预检

### 5.1 分类器实现

```typescript
// utils/classifierApprovals.ts
export async function classifierApprovesToolUse(
  toolName: string,
  toolInput: Record<string, unknown>,
  context: ExecutionContext
): Promise<boolean> {
  // 使用轻量模型快速判断
  const response = await queryHaiku({
    system: `
You are a security classifier. Determine if this tool use is safe.
Allow: read-only operations, safe file reads
Ask: file modifications, network requests
Deny: destructive operations without context
`,
    messages: [{
      role: 'user',
      content: `Tool: ${toolName}
Input: ${JSON.stringify(toolInput, null, 2)}

Is this safe to auto-approve? Answer yes/no only.`
    }]
  })

  return response.content.toLowerCase().includes('yes')
}
```

### 5.2 分类器缓存

```typescript
const classifierCache = new Map<string, boolean>()

function getClassifierCacheKey(
  toolName: string,
  input: Record<string, unknown>
): string {
  // 对输入进行规范化后哈希
  const normalized = normalizeToolInput(input)
  return `${toolName}:${hashObject(normalized)}`
}

export async function cachedClassifierApproves(
  toolName: string,
  input: Record<string, unknown>
): Promise<boolean> {
  const key = getClassifierCacheKey(toolName, input)
  
  if (classifierCache.has(key)) {
    return classifierCache.get(key)!
  }

  const result = await classifierApprovesToolUse(toolName, input)
  classifierCache.set(key, result)
  
  // 限制缓存大小
  if (classifierCache.size > 1000) {
    const firstKey = classifierCache.keys().next().value
    classifierCache.delete(firstKey)
  }

  return result
}
```

## 6. 权限对话框

### 6.1 对话框组件

```typescript
// components/PermissionRequestDialog.tsx
export function PermissionRequestDialog({
  request,
  onApprove,
  onDeny,
  onApproveAll,
}: PermissionDialogProps) {
  const [showDetails, setShowDetails] = useState(false)
  const [dontAskAgain, setDontAskAgain] = useState(false)

  return (
    <Dialog>
      <Text bold>Permission Required</Text>
      
      <Box marginY={1}>
        <Text>Tool: {request.toolName}</Text>
        <Text>Action: {getActionDescription(request)}</Text>
      </Box>

      {showDetails && (
        <Box borderStyle="round" padding={1}>
          <Text dimColor>Input:</Text>
          <Text>{JSON.stringify(request.toolInput, null, 2)}</Text>
        </Box>
      )}

      <Box>
        <Button onClick={() => onApprove(dontAskAgain)}>
          Approve
        </Button>
        <Button onClick={onDeny}>Deny</Button>
        <Button onClick={() => setShowDetails(!showDetails)}>
          {showDetails ? 'Hide' : 'Show'} Details
        </Button>
      </Box>

      <Checkbox
        checked={dontAskAgain}
        onChange={setDontAskAgain}
      >
        Don't ask again for similar operations
      </Checkbox>
    </Dialog>
  )
}
```

### 6.2 对话框状态管理

```typescript
// hooks/useCanUseTool.tsx
type PermissionRequest = {
  id: string
  toolName: string
  toolInput: Record<string, unknown>
  resolve: (result: PermissionResult) => void
}

const permissionQueue: PermissionRequest[] = []

export function useCanUseTool() {
  const appState = useAppState()

  return async (
    toolName: string,
    toolInput: Record<string, unknown>
  ): Promise<boolean> => {
    const permission = hasPermissionsToUseTool(
      toolName,
      toolInput,
      appState.toolPermissionContext
    )

    if (permission === 'deny') {
      return false
    }

    if (permission === 'allow') {
      return true
    }

    // 需要询问用户
    return new Promise((resolve) => {
      const request: PermissionRequest = {
        id: generateUUID(),
        toolName,
        toolInput,
        resolve,
      }

      permissionQueue.push(request)
      
      // 触发 UI 更新
      setAppState(prev => ({
        ...prev,
        permissionQueue: [...prev.permissionQueue, request],
      }))
    })
  }
}
```

## 7. 权限委托

### 7.1 多 Agent 权限桥接

```typescript
// utils/swarm/leaderPermissionBridge.ts
export function bridgeTeammatePermissionToLeader(
  teammateTaskId: string,
  request: PermissionRequest
): Promise<PermissionResult> {
  return new Promise((resolve) => {
    // 将请求加入领导者的权限队列
    setAppState(prev => ({
      ...prev,
      teammatePermissionQueue: [
        ...prev.teammatePermissionQueue,
        {
          taskId: teammateTaskId,
          request,
          resolve,
        }
      ]
    }))
  })
}

// Leader 处理函数
export async function handleTeammatePermissionResponse(
  taskId: string,
  approved: boolean
): Promise<void> {
  const queue = getAppState().teammatePermissionQueue
  const pending = queue.find(p => p.taskId === taskId)
  
  if (pending) {
    pending.resolve({ approved })
    
    // 从队列移除
    setAppState(prev => ({
      ...prev,
      teammatePermissionQueue: prev.teammatePermissionQueue.filter(
        p => p.taskId !== taskId
      )
    }))
  }
}
```

## 8. 权限持久化

### 8.1 保存用户选择

```typescript
export async function savePermissionPreference(
  pattern: string,
  permission: ToolPermission
): Promise<void> {
  const config = await loadProjectConfig()
  
  config.permissions = config.permissions || []
  
  // 更新或添加规则
  const existingIndex = config.permissions.findIndex(
    r => r.pattern === pattern
  )
  
  if (existingIndex >= 0) {
    config.permissions[existingIndex].permission = permission
  } else {
    config.permissions.push({ pattern, permission })
  }

  await saveProjectConfig(config)
}
```

## 9. 安全审计

### 9.1 权限使用日志

```typescript
export function logPermissionDecision(
  toolName: string,
  decision: ToolPermission,
  context: {
    mode: PermissionMode
    classified?: boolean
    userConfirmed?: boolean
  }
): void {
  logEvent('permission_decision', {
    tool_name: toolName,
    decision,
    mode: context.mode,
    auto_classified: context.classified,
    user_confirmed: context.userConfirmed,
  })
}
```

---

权限系统通过多层决策、分类器预检、权限委托等设计，在便利性和安全性之间取得平衡。理解其设计，对于构建安全的 AI 工具具有参考价值。
