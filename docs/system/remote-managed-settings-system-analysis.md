# Claude Code 远程托管设置系统技术分析

## 1. 模块总结介绍

Remote Managed Settings 系统是 Claude Code 的「企业策略控制中心」，允许组织管理员通过云端统一管理团队成员的 Claude Code 配置。该系统支持策略下发、安全检查、自动更新，确保企业级的合规性和一致性。

核心特性：
- **云端策略管理**：企业管理员远程配置团队设置
- **Checksum 缓存**：基于 ETag 的缓存机制减少网络流量
- **后台轮询**：每小时检查策略更新
- **安全检查**：危险设置变更需要用户确认
- **失败开放**：获取失败时使用缓存或继续无策略运行

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│              Remote Managed Settings System                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Enterprise Admin Console                                   │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  • 设置策略规则  • 分配团队配置  • 查看合规状态       │  │
│   └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│   ┌──────────────────────────────────────────────────────┐  │
│   │           Anthropic API Server                       │  │
│   │  /api/claude_code/settings                           │  │
│   │  • 认证  • Checksum  • 304 Not Modified              │  │
│   └──────────────────────────┬───────────────────────────┘  │
│                              │                               │
│          ┌───────────────────┼───────────────────┐           │
│          ▼                   ▼                   ▼           │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│   │   Console   │    │  Claude.ai  │    │    CCR      │     │
│   │   API Key   │    │    OAuth    │    │   Headless  │     │
│   │  (All orgs) │    │ (Ent/Team)  │    │             │     │
│   └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
│   Local Cache Layer                                          │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│   │  Persistent     │  │   Session       │  │  Checksum   │ │
│   │  (sync.json)    │  │   (Memory)      │  │  (SHA256)   │ │
│   └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心数据模型

### 3.1 远程设置响应

```typescript
interface RemoteManagedSettingsResponse {
  settings: SettingsJson      // 实际设置内容
  checksum: string            // SHA256 校验和
}

// Zod Schema 验证
const RemoteManagedSettingsResponseSchema = () =>
  z.object({
    settings: SettingsSchema(),
    checksum: z.string(),
  })
```

### 3.2 获取结果类型

```typescript
interface RemoteManagedSettingsFetchResult {
  success: boolean
  settings?: SettingsJson | null  // null 表示 304 Not Modified
  checksum?: string
  error?: string
  skipRetry?: boolean              // 是否跳过重试
}
```

## 4. Checksum 缓存机制

### 4.1 Checksum 计算

```typescript
/**
 * 计算设置内容的 Checksum
 * 必须与服务器 Python 实现一致：json.dumps(sort_keys=True, separators=(",", ":"))
 */
export function computeChecksumFromSettings(settings: SettingsJson): string {
  // 1. 递归排序所有键
  const sorted = sortKeysDeep(settings)
  
  // 2. 紧凑 JSON 序列化（无空格）
  const normalized = jsonStringify(sorted)  // separators=(",", ":")
  
  // 3. SHA256 哈希
  const hash = createHash('sha256')
    .update(normalized)
    .digest('hex')
  
  return `sha256:${hash}`
}

// 递归深度排序对象键
function sortKeysDeep(obj: unknown): unknown {
  if (Array.isArray(obj)) {
    return obj.map(sortKeysDeep)
  }
  if (obj !== null && typeof obj === 'object') {
    const sorted: Record<string, unknown> = {}
    for (const key of Object.keys(obj).sort()) {
      sorted[key] = sortKeysDeep((obj as Record<string, unknown>)[key])
    }
    return sorted
  }
  return obj
}
```

### 4.2 ETag 缓存流程

```typescript
async function fetchRemoteManagedSettings(
  cachedChecksum?: string
): Promise<RemoteManagedSettingsFetchResult> {
  const headers: Record<string, string> = {
    ...authHeaders,
    'User-Agent': getClaudeCodeUserAgent(),
  }
  
  // 添加 If-None-Match 头启用 ETag 缓存
  if (cachedChecksum) {
    headers['If-None-Match'] = `"${cachedChecksum}"`
  }
  
  const response = await axios.get(endpoint, {
    headers,
    timeout: 10000,
    validateStatus: status => 
      status === 200 || status === 204 || status === 304 || status === 404,
  })
  
  // 304 Not Modified - 缓存仍有效
  if (response.status === 304) {
    return {
      success: true,
      settings: null,  // 信号：使用缓存
      checksum: cachedChecksum,
    }
  }
  
  // 200 OK - 新设置内容
  if (response.status === 200) {
    const parsed = RemoteManagedSettingsResponseSchema().safeParse(response.data)
    return {
      success: true,
      settings: parsed.data.settings,
      checksum: parsed.data.checksum,
    }
  }
  
  // 204/404 - 无远程设置
  return { success: true, settings: {}, checksum: undefined }
}
```

## 5. 加载与同步流程

### 5.1 初始化加载流程

```typescript
export async function loadRemoteManagedSettings(): Promise<void> {
  // 1. 设置加载完成 Promise
  if (isRemoteManagedSettingsEligible() && !loadingCompletePromise) {
    loadingCompletePromise = new Promise(resolve => {
      loadingCompleteResolve = resolve
    })
  }
  
  // 2. 缓存优先：如有磁盘缓存，立即应用并解阻塞
  if (getRemoteManagedSettingsSyncFromCache() && loadingCompleteResolve) {
    loadingCompleteResolve()
    loadingCompleteResolve = null
  }
  
  try {
    // 3. 获取远程设置（带重试）
    const settings = await fetchAndLoadRemoteManagedSettings()
    
    // 4. 启动后台轮询
    if (isRemoteManagedSettingsEligible()) {
      startBackgroundPolling()
    }
    
    // 5. 触发设置热重载
    if (settings !== null) {
      settingsChangeDetector.notifyChange('policySettings')
    }
  } finally {
    // 6. 始终解阻塞等待者
    if (loadingCompleteResolve) {
      loadingCompleteResolve()
      loadingCompleteResolve = null
    }
  }
}
```

### 5.2 获取并加载

```typescript
async function fetchAndLoadRemoteManagedSettings(): Promise<SettingsJson | null> {
  if (!isRemoteManagedSettingsEligible()) {
    return null
  }
  
  // 1. 加载磁盘缓存
  const cachedSettings = getRemoteManagedSettingsSyncFromCache()
  const cachedChecksum = cachedSettings
    ? computeChecksumFromSettings(cachedSettings)
    : undefined
  
  try {
    // 2. 带重试获取远程设置
    const result = await fetchWithRetry(cachedChecksum)
    
    if (!result.success) {
      // 获取失败，使用过期缓存（优雅降级）
      if (cachedSettings) {
        setSessionCache(cachedSettings)
        return cachedSettings
      }
      return null
    }
    
    // 3. 处理 304 Not Modified
    if (result.settings === null && cachedSettings) {
      setSessionCache(cachedSettings)
      return cachedSettings
    }
    
    // 4. 应用新设置
    const newSettings = result.settings || {}
    
    if (Object.keys(newSettings).length > 0) {
      // 安全检查
      const securityResult = await checkManagedSettingsSecurity(
        cachedSettings,
        newSettings
      )
      if (!handleSecurityCheckResult(securityResult)) {
        // 用户拒绝，使用缓存
        return cachedSettings
      }
      
      // 保存到磁盘和会话缓存
      setSessionCache(newSettings)
      await saveSettings(newSettings)
      return newSettings
    }
    
    // 5. 空设置（404）- 删除缓存文件
    await clearRemoteManagedSettingsCache()
    return newSettings
    
  } catch {
    // 使用过期缓存降级
    if (cachedSettings) {
      setSessionCache(cachedSettings)
      return cachedSettings
    }
    return null
  }
}
```

## 6. 资格控制

### 6.1 资格检查逻辑

```typescript
function isRemoteManagedSettingsEligible(): boolean {
  // Console 用户（API Key）：所有组织都 eligible
  const apiKey = getAnthropicApiKeyWithSource({ 
    skipRetrievingKeyFromApiKeyHelper: true 
  })
  if (apiKey.key) {
    return true
  }
  
  // OAuth 用户：仅 Enterprise/C4E 和 Team 订阅者 eligible
  const oauthTokens = getClaudeAIOAuthTokens()
  if (oauthTokens?.accessToken) {
    // 检查用户订阅级别
    return isEnterpriseOrTeamSubscriber(oauthTokens)
  }
  
  return false
}
```

### 6.2 多认证方式支持

```typescript
function getRemoteSettingsAuthHeaders(): {
  headers: Record<string, string>
  error?: string
} {
  // 优先 API Key（Console 用户）
  try {
    const { key: apiKey } = getAnthropicApiKeyWithSource({
      skipRetrievingKeyFromApiKeyHelper: true,
    })
    if (apiKey) {
      return { headers: { 'x-api-key': apiKey } }
    }
  } catch {
    // 无 API Key，继续检查 OAuth
  }
  
  // 回退到 OAuth
  const oauthTokens = getClaudeAIOAuthTokens()
  if (oauthTokens?.accessToken) {
    return {
      headers: {
        Authorization: `Bearer ${oauthTokens.accessToken}`,
        'anthropic-beta': OAUTH_BETA_HEADER,
      },
    }
  }
  
  return { headers: {}, error: 'No authentication available' }
}
```

## 7. 后台轮询

### 7.1 轮询机制

```typescript
const POLLING_INTERVAL_MS = 60 * 60 * 1000  // 1 小时
let pollingIntervalId: ReturnType<typeof setInterval> | null = null

export function startBackgroundPolling(): void {
  if (pollingIntervalId !== null) return
  if (!isRemoteManagedSettingsEligible()) return
  
  pollingIntervalId = setInterval(() => {
    void pollRemoteSettings()
  }, POLLING_INTERVAL_MS)
  
  // 不阻塞进程退出
  pollingIntervalId.unref()
  
  // 注册清理
  registerCleanup(async () => stopBackgroundPolling())
}

async function pollRemoteSettings(): Promise<void> {
  if (!isRemoteManagedSettingsEligible()) return
  
  const prevCache = getRemoteManagedSettingsSyncFromCache()
  const previousSettings = prevCache ? jsonStringify(prevCache) : null
  
  try {
    await fetchAndLoadRemoteManagedSettings()
    
    // 检查设置是否实际变化
    const newCache = getRemoteManagedSettingsSyncFromCache()
    const newSettings = newCache ? jsonStringify(newCache) : null
    
    if (newSettings !== previousSettings) {
      settingsChangeDetector.notifyChange('policySettings')
    }
  } catch {
    // 后台轮询失败不阻塞
  }
}
```

## 8. 安全检查

### 8.1 危险设置检测

```typescript
// securityCheck.tsx
export async function checkManagedSettingsSecurity(
  oldSettings: SettingsJson | null,
  newSettings: SettingsJson
): Promise<SecurityCheckResult> {
  const risks: SettingRisk[] = []
  
  // 检查环境变量变更
  if (newSettings.env && Object.keys(newSettings.env).length > 0) {
    const addedEnv = oldSettings?.env 
      ? Object.keys(newSettings.env).filter(k => !(k in oldSettings.env!))
      : Object.keys(newSettings.env)
    
    if (addedEnv.length > 0) {
      risks.push({
        type: 'env_added',
        severity: 'warning',
        message: `Environment variables will be added: ${addedEnv.join(', ')}`,
      })
    }
  }
  
  // 检查权限模式降级
  if (newSettings.permissionMode === 'auto' && oldSettings?.permissionMode === 'ask') {
    risks.push({
      type: 'permission_downgrade',
      severity: 'critical',
      message: 'Permission mode will change from "ask" to "auto"',
    })
  }
  
  // 检查 MCP 服务器变更
  if (newSettings.mcpServers) {
    const newServers = Object.keys(newSettings.mcpServers)
    const oldServers = oldSettings?.mcpServers 
      ? Object.keys(oldSettings.mcpServers) 
      : []
    const added = newServers.filter(s => !oldServers.includes(s))
    
    if (added.length > 0) {
      risks.push({
        type: 'mcp_servers_added',
        severity: 'warning',
        message: `New MCP servers will be added: ${added.join(', ')}`,
      })
    }
  }
  
  return {
    safe: risks.length === 0,
    risks,
  }
}
```

### 8.2 用户确认流程

```typescript
export function handleSecurityCheckResult(
  result: SecurityCheckResult
): boolean {
  if (result.safe) return true
  
  // 显示确认对话框
  const confirmed = showSecurityConfirmationDialog(result.risks)
  
  // 记录用户选择
  logEvent('remote_settings_security_confirmation', {
    confirmed,
    riskCount: result.risks.length,
  })
  
  return confirmed
}
```

## 9. 等待机制

### 9.1 初始化 Promise

```typescript
let loadingCompletePromise: Promise<void> | null = null
let loadingCompleteResolve: (() => void) | null = null

export function initializeRemoteManagedSettingsLoadingPromise(): void {
  if (loadingCompletePromise) return
  
  if (isRemoteManagedSettingsEligible()) {
    loadingCompletePromise = new Promise(resolve => {
      loadingCompleteResolve = resolve
      
      // 30 秒超时防止死锁（测试环境）
      setTimeout(() => {
        if (loadingCompleteResolve) {
          loadingCompleteResolve()
          loadingCompleteResolve = null
        }
      }, 30000)
    })
  }
}

export async function waitForRemoteManagedSettingsToLoad(): Promise<void> {
  if (loadingCompletePromise) {
    await loadingCompletePromise
  }
}
```

### 9.2 使用场景

```typescript
// 其他系统等待远程设置加载完成
async function initializeSomeSystem() {
  // 等待远程策略设置
  await waitForRemoteManagedSettingsToLoad()
  
  // 现在可以安全读取设置
  const settings = getSettings()
  // ...
}
```

## 10. 技术创新点

1. **企业级策略管理**：支持组织级统一配置，满足企业合规需求

2. **Checksum ETag 缓存**：通过 SHA256 校验和实现高效的 HTTP 缓存，减少不必要的传输

3. **双认证支持**：同时支持 API Key 和 OAuth，覆盖 Console 和 Claude.ai 用户

4. **安全检查机制**：敏感设置变更需要用户确认，防止恶意策略注入

5. **多级缓存策略**：会话内存 + 磁盘缓存 + 过期缓存降级，确保高可用性

6. **后台轮询更新**：每小时检查策略变更，支持动态策略更新

7. **等待 Promise 模式**：允许其他系统等待远程设置加载完成，避免竞态条件

8. **失败开放设计**：任何失败都不阻塞核心功能，优先保证用户体验

---

Remote Managed Settings 系统通过云端策略管理、安全检查、多级缓存等设计，实现了企业级的 Claude Code 配置统一管理。理解其设计，对于构建企业级 SaaS 产品的策略下发系统具有参考价值。
