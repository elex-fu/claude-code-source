# Claude Code Plugin 系统技术分析

## 1. 模块总结介绍

Plugin 系统是 Claude Code 的「功能扩展框架」，允许第三方开发者为 Claude Code 添加新功能。该系统支持从多个源安装插件，并提供完整的生命周期管理。

核心特性：
- **多源安装**：用户级、项目级、本地级、托管级
- **依赖解析**：自动处理插件间依赖
- **版本管理**：语义化版本控制
- **热加载**：无需重启即可加载新插件

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Plugin System                            │
├─────────────────────────────────────────────────────────────┤
│  Installation Sources                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │  User    │ │ Project  │ │  Local   │ │   Managed    │   │
│  │  ~/.claude│ │ .claude/ │ │  custom  │ │  enterprise  │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘   │
│       └─────────────┴─────────────┴──────────────┘           │
│                        │                                     │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Plugin Loader & Validator                   ││
│  │  • Manifest validation  • Code signing  • Sandbox        ││
│  └─────────────────────────────────────────────────────────┘│
│                        │                                     │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Runtime Integration                         ││
│  │  • Commands  • Tools  • Hooks  • MCP servers             ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## 3. 插件清单格式

### 3.1 Manifest 结构

```typescript
// types/plugin.ts
interface PluginManifest {
  id: string                    // 唯一标识符
  name: string                  // 显示名称
  version: string               // 语义化版本
  description: string
  author: string
  
  // 入口点
  entry: {
    commands?: string           // 命令定义文件
    tools?: string              // 工具定义文件
    hooks?: string              // Hook 定义文件
  }
  
  // 依赖
  dependencies?: Record<string, string>
  
  // 权限声明
  permissions: {
    filesystem?: string[]       // 允许访问的路径
    network?: boolean           // 网络访问
    shell?: boolean             // Shell 执行
  }
  
  // MCP 配置
  mcpServers?: Record<string, MCPServerConfig>
}
```

### 3.2 清单验证

```typescript
// utils/plugins/schemas.ts
import { z } from 'zod'

export const PluginManifestSchema = z.object({
  id: z.string().regex(/^[a-z0-9-]+$/),
  name: z.string().min(1).max(100),
  version: z.string().regex(/^\d+\.\d+\.\d+$/),
  description: z.string().max(500),
  author: z.string(),
  
  entry: z.object({
    commands: z.string().optional(),
    tools: z.string().optional(),
    hooks: z.string().optional(),
  }),
  
  dependencies: z.record(z.string()).optional(),
  
  permissions: z.object({
    filesystem: z.array(z.string()).optional(),
    network: z.boolean().optional(),
    shell: z.boolean().optional(),
  }),
})

export type PluginManifest = z.infer<typeof PluginManifestSchema>
```

## 4. 插件安装

### 4.1 安装流程

```typescript
// services/plugins/pluginOperations.ts
export async function installPlugin(
  identifier: string,
  scope: InstallableScope,
  options: InstallOptions = {}
): Promise<InstallResult> {
  // 1. 解析标识符
  const parsed = parsePluginIdentifier(identifier)
  
  // 2. 检查是否已安装
  const existing = await findInstalledPlugin(parsed.id)
  if (existing && !options.force) {
    return {
      success: false,
      error: `Plugin ${parsed.id} is already installed`,
    }
  }

  // 3. 从市场获取
  const marketplace = await getMarketplace(parsed.source)
  const entry = await marketplace.getPlugin(parsed.id, parsed.version)
  
  if (!entry) {
    return {
      success: false,
      error: `Plugin ${identifier} not found`,
    }
  }

  // 4. 检查策略限制
  if (await isPluginBlockedByPolicy(entry.manifest)) {
    return {
      success: false,
      error: 'Plugin is blocked by organization policy',
    }
  }

  // 5. 下载并验证
  const downloadPath = await downloadPlugin(entry)
  const manifest = await loadPluginManifest(downloadPath)
  
  // 6. 解析依赖
  const depsResult = await resolveDependencies(manifest)
  if (!depsResult.success) {
    return {
      success: false,
      error: `Dependency resolution failed: ${depsResult.error}`,
    }
  }

  // 7. 安装到目标位置
  const installPath = getInstallPath(manifest.id, scope)
  await copyPluginToVersionedCache(downloadPath, installPath)

  // 8. 更新配置
  await addPluginToSettings(manifest.id, scope, {
    version: manifest.version,
    path: installPath,
    enabled: true,
  })

  // 9. 加载插件
  await loadPlugin(manifest.id)

  return {
    success: true,
    manifest,
  }
}
```

### 4.2 依赖解析

```typescript
// utils/plugins/dependencyResolver.ts
export async function resolveDependencies(
  manifest: PluginManifest
): Promise<DependencyResult> {
  if (!manifest.dependencies) {
    return { success: true, dependencies: [] }
  }

  const resolved: ResolvedDependency[] = []
  const conflicts: DependencyConflict[] = []

  for (const [depId, versionRange] of Object.entries(manifest.dependencies)) {
    const installed = await findInstalledPlugin(depId)
    
    if (!installed) {
      // 尝试自动安装
      const installResult = await installPlugin(
        `${depId}@${versionRange}`,
        'user'
      )
      
      if (!installResult.success) {
        conflicts.push({
          id: depId,
          required: versionRange,
          available: null,
          error: installResult.error,
        })
        continue
      }
      
      resolved.push({
        id: depId,
        version: installResult.manifest!.version,
        source: 'installed',
      })
    } else if (!satisfies(installed.version, versionRange)) {
      conflicts.push({
        id: depId,
        required: versionRange,
        available: installed.version,
        error: 'Version mismatch',
      })
    } else {
      resolved.push({
        id: depId,
        version: installed.version,
        source: 'existing',
      })
    }
  }

  if (conflicts.length > 0) {
    return {
      success: false,
      error: formatDependencyConflicts(conflicts),
      conflicts,
    }
  }

  return { success: true, dependencies: resolved }
}
```

## 5. 插件加载

### 5.1 加载流程

```typescript
// utils/plugins/pluginLoader.ts
export async function loadAllPlugins(): Promise<LoadedPlugin[]> {
  const installed = await loadInstalledPluginsFromDisk()
  const loaded: LoadedPlugin[] = []
  const errors: PluginError[] = []

  for (const plugin of installed) {
    if (!plugin.enabled) continue

    try {
      const loadedPlugin = await loadPlugin(plugin)
      loaded.push(loadedPlugin)
    } catch (error) {
      errors.push({
        id: plugin.id,
        error: String(error),
        phase: 'load',
      })
    }
  }

  // 更新状态
  setAppState(prev => ({
    ...prev,
    plugins: {
      ...prev.plugins,
      enabled: loaded,
      errors,
    },
  }))

  return loaded
}

export async function loadPlugin(pluginInfo: PluginInfo): Promise<LoadedPlugin> {
  const manifest = await loadPluginManifest(pluginInfo.path)
  
  // 验证清单
  const validation = PluginManifestSchema.safeParse(manifest)
  if (!validation.success) {
    throw new Error(`Invalid manifest: ${validation.error}`)
  }

  const loaded: LoadedPlugin = {
    id: manifest.id,
    manifest,
    path: pluginInfo.path,
    commands: [],
    tools: [],
    hooks: [],
  }

  // 加载命令
  if (manifest.entry.commands) {
    const commandsPath = join(pluginInfo.path, manifest.entry.commands)
    const commands = await import(commandsPath)
    loaded.commands = Object.values(commands.default || commands)
  }

  // 加载工具
  if (manifest.entry.tools) {
    const toolsPath = join(pluginInfo.path, manifest.entry.tools)
    const tools = await import(toolsPath)
    loaded.tools = Object.values(tools.default || tools)
  }

  // 加载 Hooks
  if (manifest.entry.hooks) {
    const hooksPath = join(pluginInfo.path, manifest.entry.hooks)
    const hooks = await import(hooksPath)
    loaded.hooks = Object.values(hooks.default || hooks)
  }

  // 注册到系统
  registerPluginCommands(loaded.commands)
  registerPluginTools(loaded.tools)
  registerPluginHooks(loaded.hooks)

  return loaded
}
```

### 5.2 缓存机制

```typescript
const pluginCache = new Map<string, LoadedPlugin>()

export async function loadPluginCached(
  pluginInfo: PluginInfo
): Promise<LoadedPlugin> {
  const cacheKey = `${pluginInfo.id}@${pluginInfo.version}`
  
  if (pluginCache.has(cacheKey)) {
    return pluginCache.get(cacheKey)!
  }

  const loaded = await loadPlugin(pluginInfo)
  pluginCache.set(cacheKey, loaded)
  return loaded
}

export function clearPluginCache(): void {
  pluginCache.clear()
}
```

## 6. 插件生命周期

### 6.1 启用/禁用

```typescript
export async function enablePlugin(pluginId: string): Promise<void> {
  await updatePluginSettings(pluginId, { enabled: true })
  
  const plugin = await loadPlugin(await getPluginInfo(pluginId))
  
  setAppState(prev => ({
    ...prev,
    plugins: {
      ...prev.plugins,
      enabled: [...prev.plugins.enabled, plugin],
    },
  }))
}

export async function disablePlugin(pluginId: string): Promise<void> {
  await updatePluginSettings(pluginId, { enabled: false })
  
  // 卸载插件
  unregisterPluginCommands(pluginId)
  unregisterPluginTools(pluginId)
  unregisterPluginHooks(pluginId)
  
  setAppState(prev => ({
    ...prev,
    plugins: {
      ...prev.plugins,
      enabled: prev.plugins.enabled.filter(p => p.id !== pluginId),
    },
  }))
}
```

### 6.2 卸载

```typescript
export async function uninstallPlugin(
  pluginId: string,
  options: { keepData?: boolean } = {}
): Promise<UninstallResult> {
  // 1. 检查反向依赖
  const dependents = await findReverseDependents(pluginId)
  if (dependents.length > 0) {
    return {
      success: false,
      error: `Cannot uninstall: required by ${dependents.join(', ')}`,
    }
  }

  // 2. 禁用插件
  await disablePlugin(pluginId)

  // 3. 删除文件
  const pluginInfo = await getPluginInfo(pluginId)
  await removePluginInstallation(pluginInfo.path)

  // 4. 清理数据（除非保留）
  if (!options.keepData) {
    await deletePluginDataDir(pluginId)
    await deletePluginOptions(pluginId)
  }

  // 5. 从配置移除
  await removePluginFromSettings(pluginId)

  return { success: true }
}
```

## 7. 市场管理

### 7.1 市场配置

```typescript
// utils/plugins/marketplaceManager.ts
interface MarketplaceConfig {
  id: string
  name: string
  url: string
  auth?: {
    type: 'bearer' | 'basic'
    token?: string
  }
}

export async function getMarketplace(
  id: string
): Promise<Marketplace> {
  const config = await loadKnownMarketplacesConfig()
  const marketplaceConfig = config.marketplaces.find(m => m.id === id)
  
  if (!marketplaceConfig) {
    throw new Error(`Unknown marketplace: ${id}`)
  }

  return new MarketplaceClient(marketplaceConfig)
}

class MarketplaceClient {
  constructor(private config: MarketplaceConfig) {}

  async getPlugin(
    id: string,
    version?: string
  ): Promise<MarketplaceEntry | null> {
    const url = new URL(`/plugins/${id}`, this.config.url)
    if (version) {
      url.searchParams.set('version', version)
    }

    const response = await fetch(url, {
      headers: this.getAuthHeaders(),
    })

    if (!response.ok) return null
    return response.json()
  }

  async search(query: string): Promise<MarketplaceEntry[]> {
    const url = new URL('/search', this.config.url)
    url.searchParams.set('q', query)

    const response = await fetch(url, {
      headers: this.getAuthHeaders(),
    })

    return response.json()
  }

  private getAuthHeaders(): Record<string, string> {
    if (this.config.auth?.type === 'bearer') {
      return {
        Authorization: `Bearer ${this.config.auth.token}`,
      }
    }
    return {}
  }
}
```

## 8. 企业策略

### 8.1 策略检查

```typescript
// utils/plugins/pluginPolicy.ts
export async function isPluginBlockedByPolicy(
  manifest: PluginManifest
): Promise<boolean> {
  const policy = await loadOrganizationPolicy()

  // 检查黑名单
  if (policy.blockedPlugins.includes(manifest.id)) {
    return true
  }

  // 检查权限
  for (const permission of Object.keys(manifest.permissions)) {
    if (policy.allowedPermissions.includes(permission)) {
      continue
    }
    // 需要额外审批
    if (!policy.autoApprovePermissions.includes(permission)) {
      return true
    }
  }

  // 检查签名
  if (policy.requireSignature && !(await verifyPluginSignature(manifest))) {
    return true
  }

  return false
}
```

---

Plugin 系统通过多源安装、依赖解析、版本管理、热加载等设计，构建了一个完整的扩展生态。理解其设计，对于构建可扩展的 CLI 工具具有参考价值。
