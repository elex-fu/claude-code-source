# Claude Code OAuth 系统技术分析

## 1. 模块总结介绍
OAuth 系统是 Claude Code 的「身份认证中枢」，负责管理用户与 Anthropic API 的认证流程。该系统实现了完整的 OAuth 2.0 + PKCE 流程，支持多种身份提供商和设备授权。

核心特性：- **PKCE 增强安全**：防止授权码拦截攻击- **本地回调服务器**：自动捕获授权回调- **Token 刷新**：自动处理 Token 过期- **多账户支持**：切换不同身份

## 2. 系统架构```┌─────────────────────────────────────────────────────────────┐│                      OAuth System                            │├─────────────────────────────────────────────────────────────┤│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ ││  │   Auth Flow     │  │   Token Store   │  │   Profile   │ ││  │                 │  │                 │  │   Mgmt      │ ││  │ • PKCE Gen      │──▶ • Keychain      │──▶ • User Info│ ││  │ • Callback      │    • File Backup    │    • Orgs     │ ││  │ • Token Exchange│    • Memory Cache   │    • Quotas   │ ││  └─────────────────┘  └─────────────────┘  └─────────────┘ │└─────────────────────────────────────────────────────────────┘```

## 3. PKCE 流程实现

### 3.1 PKCE 参数生成
```typescript
// oauth/crypto.ts
import { randomBytes, createHash } from 'crypto'

export interface PKCEParams {
  codeVerifier: string
  codeChallenge: string
  codeChallengeMethod: 'S256'
}

export function generatePKCE(): PKCEParams {
  // 生成随机 code_verifier (43-128 字符)
  const codeVerifier = randomBytes(32)
    .toString('base64url')
    .slice(0, 128)

  // 计算 code_challenge = BASE64URL(SHA256(code_verifier))
  const codeChallenge = createHash('sha256')
    .update(codeVerifier)
    .digest('base64url')

  return {
    codeVerifier,
    codeChallenge,
    codeChallengeMethod: 'S256',
  }
}
```

### 3.2 授权 URL 构建
```typescript
// oauth/client.ts
export function buildAuthURL(
  config: OAuthConfig,
  pkce: PKCEParams,
  state: string
): string {
  const params = new URLSearchParams({
    client_id: config.clientId,
    response_type: 'code',
    redirect_uri: config.redirectUri,
    code_challenge: pkce.codeChallenge,
    code_challenge_method: pkce.codeChallengeMethod,
    state,
    scope: 'openid profile email',
  })

  return `${config.authEndpoint}?${params.toString()}`
}
```

## 4. 本地回调服务器

### 4.1 HTTP 回调服务
```typescript
// oauth/auth-code-listener.ts
export function startAuthCallbackServer(
  port: number,
  onCode: (code: string, state: string) => void
): Promise<Server> {
  return new Promise((resolve) => {
    const server = createServer((req, res) => {
      const url = new URL(req.url || '', `http://localhost:${port}`)
      
      const code = url.searchParams.get('code')
      const state = url.searchParams.get('state')
      const error = url.searchParams.get('error')

      if (error) {
        res.end(`<h1>Auth Failed</h1><p>${error}</p>`)
        return
      }

      if (code && state) {
        onCode(code, state)
        res.end(`<h1>Auth Success</h1><p>You can close this window.</p>`)
      } else {
        res.end('<h1>Invalid Request</h1>')
      }
    })

    server.listen(port, () => resolve(server))
  })
}
```

### 4.2 端口选择
```typescript
export async function findAvailablePort(
  preferredPorts: number[]
): Promise<number> {
  for (const port of preferredPorts) {
    try {
      const server = createServer()
      await new Promise<void>((resolve, reject) => {
        server.listen(port, () => {
          server.close()
          resolve()
        })
        server.on('error', reject)
      })
      return port
    } catch {
      continue
    }
  }
  throw new Error('No available port found')
}
```

## 5. Token 交换与存储

### 5.1 Token 交换
```typescript
export async function exchangeCodeForTokens(
  code: string,
  pkce: PKCEParams,
  config: OAuthConfig
): Promise<TokenResponse> {
  const response = await fetch(config.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      client_id: config.clientId,
      code,
      redirect_uri: config.redirectUri,
      code_verifier: pkce.codeVerifier,
    }),
  })

  if (!response.ok) {
    throw new Error(`Token exchange failed: ${response.statusText}`)
  }

  return response.json()
}
```

### 5.2 Token 存储
```typescript
// utils/auth.ts
const KEYCHAIN_SERVICE = 'claude-code-auth'
const KEYCHAIN_ACCOUNT = 'default'

export async function saveTokens(tokens: TokenResponse): Promise<void> {
  // 1. 存储到系统 keychain
  await keychain.setPassword(
    KEYCHAIN_SERVICE,
    KEYCHAIN_ACCOUNT,
    JSON.stringify(tokens)
  )

  // 2. 备份到文件（keychain 失败时使用）
  const backupPath = getAuthBackupPath()
  await writeFile(
    backupPath,
    JSON.stringify(tokens),
    { mode: 0o600 } // 仅限当前用户读取
  )
}

export async function loadTokens(): Promise<TokenResponse | null> {
  try {
    // 优先从 keychain 读取
    const data = await keychain.getPassword(
      KEYCHAIN_SERVICE,
      KEYCHAIN_ACCOUNT
    )
    if (data) {
      return JSON.parse(data)
    }
  } catch {
    // 回退到文件
    const backupPath = getAuthBackupPath()
    const data = await readFile(backupPath, 'utf-8')
    return JSON.parse(data)
  }
  return null
}
```

## 6. Token 刷新

### 6.1 自动刷新
```typescript
export async function getValidAccessToken(): Promise<string> {
  const tokens = await loadTokens()
  if (!tokens) {
    throw new Error('Not authenticated')
  }

  // 检查是否过期
  if (isTokenExpired(tokens)) {
    const newTokens = await refreshTokens(tokens.refresh_token)
    await saveTokens(newTokens)
    return newTokens.access_token
  }

  return tokens.access_token
}

function isTokenExpired(tokens: TokenResponse): boolean {
  // 提前 5 分钟刷新
  const expiresAt = tokens.issued_at + tokens.expires_in * 1000
  return Date.now() > expiresAt - 5 * 60 * 1000
}
```

### 6.2 刷新请求
```typescript
async function refreshTokens(
  refreshToken: string
): Promise<TokenResponse> {
  const response = await fetch(config.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      client_id: config.clientId,
      refresh_token: refreshToken,
    }),
  })

  if (!response.ok) {
    throw new Error('Token refresh failed')
  }

  return response.json()
}
```

## 7. 用户资料管理

### 7.1 获取用户信息
```typescript
// oauth/getOauthProfile.ts
export async function getOAuthProfile(
  accessToken: string
): Promise<UserProfile> {
  const response = await fetch(config.userInfoEndpoint, {
    headers: { Authorization: `Bearer ${accessToken}` },
  })

  const data = await response.json()

  return {
    id: data.sub,
    email: data.email,
    name: data.name,
    organizations: data.organizations || [],
    quotaStatus: data.quota_status,
  }
}
```

### 7.2 组织信息
```typescript
export interface Organization {
  id: string
  name: string
  role: 'admin' | 'member'
  quota: {
    used: number
    total: number
    resetsAt: Date
  }
}

export async function getUserOrganizations(
  accessToken: string
): Promise<Organization[]> {
  const profile = await getOAuthProfile(accessToken)
  return profile.organizations
}
```

## 8. 多账户支持

### 8.1 账户切换
```typescript
const ACCOUNTS_DIR = '~/.claude/accounts/'

export async function switchAccount(accountId: string): Promise<void> {
  // 保存当前账户
  const currentTokens = await loadTokens()
  if (currentTokens) {
    const currentId = getCurrentAccountId()
    await saveAccountTokens(currentId, currentTokens)
  }

  // 加载新账户
  const newTokens = await loadAccountTokens(accountId)
  await saveTokens(newTokens)
  setCurrentAccountId(accountId)
}

export async function listAccounts(): Promise<AccountInfo[]> {
  const accountsDir = await readdir(ACCOUNTS_DIR)
  return Promise.all(
    accountsDir
      .filter((f) => f.endsWith('.json'))
      .map(async (f) => {
        const id = f.replace('.json', '')
        const info = await loadAccountInfo(id)
        return {
          id,
          email: info.email,
          isActive: id === getCurrentAccountId(),
        }
      })
  )
}
```

## 9. 错误处理

### 9.1 认证错误分类
```typescript
export class AuthError extends Error {
  constructor(
    message: string,
    public code: 
      | 'cancelled'
      | 'timeout'
      | 'network_error'
      | 'invalid_state'
      | 'token_expired'
  ) {
    super(message)
  }
}

export function handleAuthError(error: unknown): never {
  if (error instanceof AuthError) {
    throw error
  }

  if (error instanceof Error) {
    if (error.message.includes('ECONNREFUSED')) {
      throw new AuthError('Network error', 'network_error')
    }
    if (error.message.includes('invalid_grant')) {
      throw new AuthError('Session expired, please login again', 'token_expired')
    }
  }

  throw new AuthError('Unknown auth error', 'network_error')
}
```

## 10. 安全最佳实践

### 10.1 PKCE 验证
```typescript
// 验证 state 防止 CSRF
export function validateState(
  receivedState: string,
  storedState: string
): void {
  if (!timingSafeEqual(
    Buffer.from(receivedState),
    Buffer.from(storedState)
  )) {
    throw new AuthError('Invalid state', 'invalid_state')
  }
}
```

### 10.2 令牌安全
- 仅通过 HTTPS 传输
- 存储在系统 keychain
- 内存中仅保留访问令牌
- 刷新令牌绝不暴露

---

Claude Code 的 OAuth 系统实现了生产级的身份认证方案。通过 PKCE、本地回调、Token 刷新等设计，提供了安全且流畅的用户体验。
