# Claude Code 后台远程会话系统技术分析

## 1. 模块总结介绍

Background Remote Session 系统是 Claude Code 的「云端执行引擎」，支持在远程环境中异步执行长时间运行的任务。该系统通过 CCR (Claude Code Remote) 模式，让用户可以将耗时操作（如大规模重构、测试套件运行）卸载到云端。

核心特性：
- **云端执行**：任务在远程容器/环境中运行
- **异步模式**：本地关闭后任务继续执行
- **GitHub 集成**：通过 GitHub App 或 Token 同步代码
- **Bundle 模式**：本地打包上传，无需 GitHub 访问
- **策略控制**：企业管理员可限制远程会话使用

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│           Background Remote Session System                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Local CLI                                                  │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  1. Precondition Checks                              │  │
│   │     • Login status      • Git repo                   │  │
│   │     • Remote env        • GitHub access              │  │
│   └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  2. Bundle / Git Push                                │  │
│   │     • Git push (normal)   • Tar upload (bundle)      │  │
│   └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  3. Teleport API                                     │  │
│   │     /api/teleport/session                            │  │
│   └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│   Remote Environment (CCR)                                   │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  4. Remote Agent                                     │  │
│   │     • Execute task      • Stream output              │  │
│   │     • Sync state        • Store logs                 │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 3. 会话数据模型

### 3.1 远程会话类型

```typescript
// remoteSession.ts
export type BackgroundRemoteSession = {
  id: string                    // 会话唯一 ID
  command: string               // 用户命令
  startTime: number             // 开始时间戳
  status: 'starting' | 'running' | 'completed' | 'failed' | 'killed'
  todoList: TodoList            // 关联的 todo 列表
  title: string                 // 会话标题
  type: 'remote_session'        // 类型标识
  log: SDKMessage[]             // 执行日志
}

// 前置条件失败类型
export type BackgroundRemoteSessionPrecondition =
  | { type: 'not_logged_in' }
  | { type: 'no_remote_environment' }
  | { type: 'not_in_git_repo' }
  | { type: 'no_git_remote' }
  | { type: 'github_app_not_installed' }
  | { type: 'policy_blocked' }
```

### 3.2 前置条件检查

```typescript
// preconditions.ts
export interface RemotePreconditions {
  // 认证相关
  needsLogin: boolean           // 需要登录 Claude.ai
  
  // 环境相关
  hasRemoteEnv: boolean         // 有可用远程环境
  
  // Git 相关
  isInGitRepo: boolean          // 在 Git 仓库中
  hasGitRemote: boolean         // 有 GitHub 远程仓库
  
  // GitHub 访问
  githubAppInstalled: boolean   // GitHub App 已安装
  githubTokenSynced: boolean    // GitHub Token 已同步
}

// 访问方式
export type RepoAccessMethod = 'github-app' | 'token-sync' | 'none'
```

## 4. 资格检查流程

### 4.1 前置条件检查流程

```typescript
// remoteSession.ts
export async function checkBackgroundRemoteSessionEligibility({
  skipBundle = false,
}: { skipBundle?: boolean } = {}): Promise<BackgroundRemoteSessionPrecondition[]> {
  const errors: BackgroundRemoteSessionPrecondition[] = []

  // 1. 策略检查（最高优先级）
  if (!isPolicyAllowed('allow_remote_sessions')) {
    errors.push({ type: 'policy_blocked' })
    return errors
  }

  // 2. 并行检查基础条件
  const [needsLogin, hasRemoteEnv, repository] = await Promise.all([
    checkNeedsClaudeAiLogin(),
    checkHasRemoteEnvironment(),
    detectCurrentRepositoryWithHost(),
  ])

  if (needsLogin) {
    errors.push({ type: 'not_logged_in' })
  }

  if (!hasRemoteEnv) {
    errors.push({ type: 'no_remote_environment' })
  }

  // 3. Bundle 模式检查
  const bundleSeedGateOn =
    !skipBundle &&
    (isEnvTruthy(process.env.CCR_FORCE_BUNDLE) ||
      isEnvTruthy(process.env.CCR_ENABLE_BUNDLE) ||
      (await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bundle_seed_enabled')))

  // 4. Git 相关检查
  if (!checkIsInGitRepo()) {
    errors.push({ type: 'not_in_git_repo' })
  } else if (bundleSeedGateOn) {
    // Bundle 模式：只需 .git/ 目录，无需远程仓库
  } else if (repository === null) {
    errors.push({ type: 'no_git_remote' })
  } else if (repository.host === 'github.com') {
    // 5. GitHub App 检查
    const hasGithubApp = await checkGithubAppInstalled(
      repository.owner,
      repository.name,
    )
    if (!hasGithubApp) {
      errors.push({ type: 'github_app_not_installed' })
    }
  }

  return errors
}
```

### 4.2 GitHub 访问检查

```typescript
// preconditions.ts
export async function checkRepoForRemoteAccess(
  owner: string,
  repo: string,
): Promise<{ hasAccess: boolean; method: RepoAccessMethod }> {
  // 层级 1: GitHub App 安装检查
  if (await checkGithubAppInstalled(owner, repo)) {
    return { hasAccess: true, method: 'github-app' }
  }
  
  // 层级 2: GitHub Token 同步检查
  if (
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_lantern', false) &&
    (await checkGithubTokenSynced())
  ) {
    return { hasAccess: true, method: 'token-sync' }
  }
  
  // 层级 3: 无访问权限
  return { hasAccess: false, method: 'none' }
}
```

### 4.3 GitHub App 安装检查

```typescript
export async function checkGithubAppInstalled(
  owner: string,
  repo: string,
  signal?: AbortSignal,
): Promise<boolean> {
  const accessToken = getClaudeAIOAuthTokens()?.accessToken
  if (!accessToken) return false

  const orgUUID = await getOrganizationUUID()
  if (!orgUUID) return false

  const url = `${getOauthConfig().BASE_API_URL}/api/oauth/organizations/${orgUUID}/code/repos/${owner}/${repo}`
  
  const response = await axios.get(url, {
    headers: {
      ...getOAuthHeaders(accessToken),
      'x-organization-uuid': orgUUID,
    },
    timeout: 15000,
    signal,
  })

  if (response.status === 200 && response.data.status) {
    return response.data.status.app_installed
  }
  
  return false
}
```

## 5. 两种代码同步模式

### 5.1 GitHub 推送模式

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Local     │───▶│   GitHub    │───▶│   Remote    │
│   Git Repo  │    │   Remote    │    │   CCR Env   │
└─────────────┘    └─────────────┘    └─────────────┘

流程:
1. 本地创建临时分支
2. 提交当前修改
3. 推送到 GitHub
4. 远程环境拉取代码
5. 执行用户命令
6. 结果推送回 GitHub
7. 本地拉取更新
```

### 5.2 Bundle 打包模式

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Local     │───▶│   Bundle    │───▶│   Remote    │
│   Git Repo  │    │   Upload    │    │   CCR Env   │
└─────────────┘    └─────────────┘    └─────────────┘

流程:
1. 本地 git bundle create
2. 压缩打包整个仓库
3. 上传到 Teleport API
4. 远程环境解压恢复
5. 执行用户命令
6. 返回结果 bundle
```

### 5.3 Bundle 模式启用条件

```typescript
function isBundleModeEnabled(): boolean {
  // 强制启用
  if (isEnvTruthy(process.env.CCR_FORCE_BUNDLE)) return true
  
  // 显式启用
  if (isEnvTruthy(process.env.CCR_ENABLE_BUNDLE)) return true
  
  // 特性门控
  return checkGate_CACHED_OR_BLOCKING('tengu_ccr_bundle_seed_enabled')
}
```

## 6. 会话生命周期

### 6.1 创建会话

```typescript
async function createRemoteSession(
  command: string,
  title: string,
  todoList: TodoList,
): Promise<BackgroundRemoteSession> {
  // 1. 检查前置条件
  const preconditions = await checkBackgroundRemoteSessionEligibility()
  if (preconditions.length > 0) {
    throw new PreconditionError(preconditions)
  }

  // 2. 准备代码同步
  const isBundleMode = isBundleModeEnabled()
  let syncData: SyncData
  
  if (isBundleMode) {
    syncData = await prepareBundleUpload()
  } else {
    syncData = await prepareGitPush()
  }

  // 3. 调用 Teleport API 创建会话
  const session = await teleportApi.createSession({
    command,
    title,
    syncData,
    bundleMode: isBundleMode,
  })

  // 4. 本地存储会话信息
  const remoteSession: BackgroundRemoteSession = {
    id: session.id,
    command,
    startTime: Date.now(),
    status: 'starting',
    todoList,
    title,
    type: 'remote_session',
    log: [],
  }

  await saveRemoteSession(remoteSession)
  
  return remoteSession
}
```

### 6.2 状态同步

```typescript
async function syncRemoteSessionStatus(
  sessionId: string
): Promise<void> {
  const session = await getRemoteSession(sessionId)
  if (!session) return

  // 从远程获取最新状态
  const remoteStatus = await teleportApi.getSessionStatus(sessionId)
  
  // 更新本地状态
  session.status = remoteStatus.status
  session.log = remoteStatus.log
  
  // 如果完成，拉取结果
  if (remoteStatus.status === 'completed') {
    await pullRemoteResults(session)
  }
  
  await saveRemoteSession(session)
}
```

## 7. 策略控制

### 7.1 企业策略限制

```typescript
// policyLimits/index.ts
export function isPolicyAllowed(action: string): boolean {
  const policy = getCurrentPolicy()
  
  switch (action) {
    case 'allow_remote_sessions':
      // 检查远程会话策略
      return policy.remoteSessions !== false
      
    case 'allow_bundle_upload':
      // 检查 bundle 上传策略
      return policy.bundleUpload !== false
      
    default:
      return true
  }
}
```

### 7.2 远程托管设置集成

```typescript
// 等待远程策略加载
await waitForRemoteManagedSettingsToLoad()

// 检查是否允许远程会话
if (!isPolicyAllowed('allow_remote_sessions')) {
  return {
    errors: [{ type: 'policy_blocked' }]
  }
}
```

## 8. 错误处理与降级

### 8.1 前置条件错误处理

```typescript
function handlePreconditionErrors(
  errors: BackgroundRemoteSessionPrecondition[]
): void {
  for (const error of errors) {
    switch (error.type) {
      case 'not_logged_in':
        showLoginPrompt()
        break
        
      case 'github_app_not_installed':
        showGitHubAppInstallPrompt()
        break
        
      case 'no_remote_environment':
        showNoEnvironmentError()
        break
        
      case 'policy_blocked':
        showPolicyBlockedError()
        break
    }
  }
}
```

### 8.2 连接失败降级

```typescript
async function executeWithFallback(
  command: string,
  options: ExecOptions
): Promise<ExecResult> {
  try {
    // 尝试远程执行
    if (await canUseRemoteSession()) {
      return await executeRemote(command, options)
    }
  } catch (error) {
    logForDebugging(`Remote execution failed: ${error}`)
    // 继续本地执行
  }
  
  // 回退到本地执行
  return await executeLocal(command, options)
}
```

## 9. 技术创新点

1. **双模式代码同步**：GitHub 推送和 Bundle 打包两种模式，适应不同网络环境

2. **分层前置条件检查**：从策略到认证到环境的层级检查，快速失败减少等待

3. **Bundle Seed 优化**：本地打包上传避免 GitHub 依赖，提高内网/企业环境兼容性

4. **异步会话管理**：会话状态持久化，支持断开重连后继续监控

5. **策略集成**：与企业远程托管设置集成，支持组织级访问控制

6. **访问方式降级**：GitHub App → Token Sync → Bundle 的层级降级策略

---

Background Remote Session 系统通过云端执行、代码同步、策略控制等设计，实现了耗时长任务的远程卸载。理解其设计，对于构建云端开发环境具有参考价值。
