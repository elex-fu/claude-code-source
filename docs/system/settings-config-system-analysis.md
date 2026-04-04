# Claude Code Settings & Config 系统技术分析

## 1. 模块总结介绍

Settings & Config 系统是 Claude Code 的「配置管理中心」，负责管理用户设置、项目配置和运行时配置。该系统支持多级配置覆盖、动态更新和验证。

核心特性：
- **多级配置**：Global → User → Project → Local 层级
- **动态更新**：运行时配置变更无需重启
- **类型安全**：完整的 TypeScript 类型定义
- **迁移支持**：配置版本自动迁移

## 2. 配置层级架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Config Hierarchy                          │
├─────────────────────────────────────────────────────────────┤
│  Priority (High → Low)                                      │
│                                                             │
│  ┌─────────────┐  1. CLI Arguments (--model, --permission)  │
│  │   CLI Args  │     命令行参数，最高优先级                  │
│  └──────┬──────┘                                            │
│         │ 2. Local Config (.claude/settings.local.json)     │
│  ┌──────▼──────┐     本地覆盖，不提交到版本控制              │
│  │    Local    │                                            │
│  └──────┬──────┘  3. Project Config (.claude/settings.json) │
│         │     项目级配置，随仓库共享                         │
│  ┌──────▼──────┐                                            │
│  │   Project   │  4. User Config (~/.claude/settings.json) │
│  └──────┬──────┘     用户级配置，跨项目共享                  │
│         │ 5. Global Defaults                                │
│  ┌──────▼──────┐     内置默认值，最低优先级                  │
│  │   Global    │                                            │
│  └─────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心类型定义

### 3.1 设置结构

```typescript
// utils/settings/types.ts
export interface SettingsJson {
  // 模型设置
  defaultModel?: ModelSetting
  preferredModels?: ModelOption[]
  
  // 权限设置
  permissionMode?: PermissionMode
  allowedTools?: string[]
  disallowedTools?: string[]
  
  // 行为设置
  autoCompact?: boolean
  enableTelemetry?: boolean
  theme?: ThemeSetting
  
  // 编辑器设置
  vimMode?: boolean
  autoSave?: boolean
  
  // 插件设置
  plugins?: PluginSettings
  
  // MCP 设置
  mcpServers?: Record<string, MCPServerConfig>
  
  // Hook 设置
  hooks?: HookConfig[]
  
  // 版本
  version: number
}

export interface PluginSettings {
  enabled: string[]
  disabled: string[]
  configurations: Record<string, unknown>
}

export type SettingSource = 
  | 'cli' 
  | 'local' 
  | 'project' 
  | 'user' 
  | 'default'
```

### 3.2 配置 schema

```typescript
// utils/settings/schema.ts
import { z } from 'zod'

export const SettingsSchema = z.object({
  defaultModel: z.enum(['opus', 'sonnet', 'haiku']).optional(),
  
  permissionMode: z.enum(['auto', 'plan', 'ask']).optional(),
  
  allowedTools: z.array(z.string()).optional(),
  disallowedTools: z.array(z.string()).optional(),
  
  autoCompact: z.boolean().optional(),
  enableTelemetry: z.boolean().optional(),
  
  theme: z.enum(['light', 'dark', 'system']).optional(),
  vimMode: z.boolean().optional(),
  
  plugins: z.object({
    enabled: z.array(z.string()),
    disabled: z.array(z.string()),
    configurations: z.record(z.unknown()),
  }).optional(),
  
  mcpServers: z.record(z.object({
    url: z.string(),
    auth: z.object({
      type: z.enum(['bearer', 'basic']),
      token: z.string(),
    }).optional(),
  })).optional(),
  
  version: z.number().default(1),
})

export type Settings = z.infer<typeof SettingsSchema>
```

## 4. 配置加载

### 4.1 分层加载

```typescript
// utils/settings/settings.ts
export async function loadAllSettings(
  cwd: string
): Promise<SettingsJson> {
  // 1. 从低到高加载各层
  const globalDefaults = getGlobalDefaults()
  const userSettings = await loadUserSettings()
  const projectSettings = await loadProjectSettings(cwd)
  const localSettings = await loadLocalSettings(cwd)
  
  // 2. 合并（高优先级覆盖低优先级）
  const merged: SettingsJson = {
    ...globalDefaults,
    ...userSettings,
    ...projectSettings,
    ...localSettings,
    // 数组类型特殊处理（合并而非替换）
    allowedTools: [
      ...(globalDefaults.allowedTools || []),
      ...(userSettings.allowedTools || []),
      ...(projectSettings.allowedTools || []),
      ...(localSettings.allowedTools || []),
    ],
  }
  
  // 3. 验证
  const result = SettingsSchema.safeParse(merged)
  if (!result.success) {
    logError('Settings validation failed:', result.error)
    return globalDefaults
  }
  
  return result.data
}

async function loadUserSettings(): Promise<Partial<SettingsJson>> {
  const path = getGlobalClaudeFile('settings.json')
  return loadSettingsFile(path)
}

async function loadProjectSettings(
  cwd: string
): Promise<Partial<SettingsJson>> {
  const path = join(cwd, '.claude', 'settings.json')
  return loadSettingsFile(path)
}

async function loadLocalSettings(
  cwd: string
): Promise<Partial<SettingsJson>> {
  const path = join(cwd, '.claude', 'settings.local.json')
  return loadSettingsFile(path)
}

async function loadSettingsFile(
  path: string
): Promise<Partial<SettingsJson>> {
  try {
    const content = await readFile(path, 'utf-8')
    return JSON.parse(content)
  } catch {
    return {}
  }
}
```

### 4.2 CLI 参数覆盖

```typescript
// main.tsx
program
  .option('-m, --model <model>', 'Model to use')
  .option('--permission-mode <mode>', 'Permission mode')
  .option('--allowed-tools <tools>', 'Comma-separated list of allowed tools')
  .action(async (prompt, options) => {
    // CLI 参数覆盖配置
    const cliSettings: Partial<SettingsJson> = {}
    
    if (options.model) {
      cliSettings.defaultModel = options.model as ModelSetting
    }
    
    if (options.permissionMode) {
      cliSettings.permissionMode = options.permissionMode as PermissionMode
    }
    
    if (options.allowedTools) {
      cliSettings.allowedTools = options.allowedTools.split(',')
    }
    
    // 合并到设置
    applyCLIOptions(cliSettings)
  })
```

## 5. 配置更新

### 5.1 更新设置

```typescript
export async function updateSettings(
  updates: Partial<SettingsJson>,
  source: SettingSource
): Promise<void> {
  const current = await loadSettingsForSource(source)
  const merged = { ...current, ...updates }
  
  // 验证
  const result = SettingsSchema.safeParse(merged)
  if (!result.success) {
    throw new Error(`Invalid settings: ${result.error.message}`)
  }
  
  // 保存
  const path = getSettingsPathForSource(source)
  await writeFile(path, JSON.stringify(merged, null, 2))
  
  // 通知变更
  notifySettingsChange(source, updates)
}

function getSettingsPathForSource(source: SettingSource): string {
  switch (source) {
    case 'user':
      return getGlobalClaudeFile('settings.json')
    case 'project':
      return join(getCwd(), '.claude', 'settings.json')
    case 'local':
      return join(getCwd(), '.claude', 'settings.local.json')
    default:
      throw new Error(`Cannot save to source: ${source}`)
  }
}
```

### 5.2 运行时更新

```typescript
// utils/settings/applySettingsChange.ts
export function applySettingsChange(
  source: SettingSource,
  setState: SetAppStateFn
): void {
  setState(prev => {
    const newSettings = {
      ...prev.settings,
      ...getSettingsForSource(source),
    }
    
    return {
      ...prev,
      settings: newSettings,
    }
  })
  
  // 应用特殊设置
  if (updates.theme) {
    applyTheme(updates.theme)
  }
  
  if (updates.vimMode !== undefined) {
    setInputMode(updates.vimMode ? 'vim' : 'emacs')
  }
}
```

## 6. 配置验证

### 6.1 验证器

```typescript
// utils/settings/validation.ts
export function validateSettings(
  settings: unknown
): ValidationResult {
  const result = SettingsSchema.safeParse(settings)
  
  if (result.success) {
    return { valid: true }
  }
  
  const errors = result.error.errors.map(e => ({
    path: e.path.join('.'),
    message: e.message,
  }))
  
  return { valid: false, errors }
}

export function validateToolList(tools: string[]): ValidationError[] {
  const errors: ValidationError[] = []
  const knownTools = getAllToolNames()
  
  for (const tool of tools) {
    if (!knownTools.includes(tool)) {
      // 检查是否为通配符模式
      if (!tool.includes('*')) {
        errors.push({
          path: `tools.${tool}`,
          message: `Unknown tool: ${tool}`,
        })
      }
    }
  }
  
  return errors
}
```

### 6.2 配置提示

```typescript
// utils/settings/validationTips.ts
export function getValidationTip(
  error: ValidationError
): string {
  const tips: Record<string, string> = {
    'defaultModel': 'Valid models: opus, sonnet, haiku',
    'permissionMode': 'Valid modes: auto, plan, ask',
    'theme': 'Valid themes: light, dark, system',
  }
  
  return tips[error.path] || error.message
}
```

## 7. 配置迁移

### 7.1 版本迁移

```typescript
// migrations/settings.ts
const MIGRATIONS: Record<number, (settings: any) => any> = {
  1: (s) => s, // 初始版本
  
  2: (s) => ({
    ...s,
    // v1 -> v2: 重命名字段
    permissionMode: s.permissions?.mode,
    allowedTools: s.permissions?.allowed,
    disallowedTools: s.permissions?.denied,
    permissions: undefined,
  }),
  
  3: (s) => ({
    ...s,
    // v2 -> v3: MCP 配置结构调整
    mcpServers: Object.entries(s.mcpServers || {}).reduce(
      (acc, [key, value]) => ({
        ...acc,
        [key]: {
          url: (value as any).url,
          auth: (value as any).auth,
        },
      }),
      {}
    ),
  }),
}

export async function migrateSettings(
  settings: any
): Promise<SettingsJson> {
  const currentVersion = settings.version || 1
  const latestVersion = Object.keys(MIGRATIONS).length
  
  if (currentVersion >= latestVersion) {
    return settings
  }
  
  let migrated = { ...settings }
  
  for (let v = currentVersion; v < latestVersion; v++) {
    if (MIGRATIONS[v]) {
      migrated = MIGRATIONS[v](migrated)
      migrated.version = v + 1
    }
  }
  
  return migrated
}
```

## 8. 缓存与性能

### 8.1 设置缓存

```typescript
// utils/settings/settingsCache.ts
const settingsCache = new Map<string, {
  settings: SettingsJson
  mtime: number
}>()

export async function loadSettingsCached(
  path: string
): Promise<SettingsJson> {
  const stats = await stat(path).catch(() => null)
  
  if (!stats) {
    return getGlobalDefaults()
  }
  
  const cached = settingsCache.get(path)
  if (cached && cached.mtime === stats.mtimeMs) {
    return cached.settings
  }
  
  const settings = await loadSettingsFile(path)
  settingsCache.set(path, {
    settings,
    mtime: stats.mtimeMs,
  })
  
  return settings
}

export function invalidateSettingsCache(path?: string): void {
  if (path) {
    settingsCache.delete(path)
  } else {
    settingsCache.clear()
  }
}
```

### 8.2 文件监听

```typescript
export function watchSettingsFile(
  path: string,
  onChange: () => void
): () => void {
  const watcher = watchFile(path, { interval: 1000 }, () => {
    invalidateSettingsCache(path)
    onChange()
  })
  
  return () => unwatchFile(path)
}
```

---

Settings & Config 系统通过层级合并、类型验证、自动迁移等设计，提供了灵活且可靠的配置管理。理解其设计，对于构建可配置的 CLI 工具具有参考价值。
