# Claude Code Bridge 系统技术分析

## 1. 模块总结介绍

Bridge 系统是 Claude Code 的「远程控制中枢」，实现了终端会话与 claude.ai Web 界面的双向实时同步。该系统让用户可以在 Web 端查看和控制本地终端会话，是 Claude Code 区别于传统 CLI 工具的关键特性。

核心功能包括：
- **双向消息同步**：本地消息实时推送到 Web，Web 指令实时同步到本地
- **断线重连**：网络波动时自动恢复连接，不丢失会话状态
- **权限桥接**：Web 端的权限请求转发到本地处理
- **多传输协议**：支持 SSE、WebSocket、HTTP 多种传输方式

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code CLI                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  REPL 会话    │  │  Bridge Core │  │   Transport Layer    │  │
│  │              │◀─▶│  (状态管理)  │◀─▶│  (SSE/WS/HTTP)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ WebSocket/SSE
┌─────────────────────────────────────────────────────────────────┐
│                      claude.ai (Cloud)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Web UI     │◀─▶│   Session   │◀─▶│   Ingress API        │  │
│  │              │   │   Manager   │   │   (WebSocket)        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP/REST
┌─────────────────────────────────────────────────────────────────┐
│                      Bridge API Server                          │
│         (Env Registration / Session Creation / Polling)         │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 核心组件

### 3.1 Bridge Core (`replBridge.ts`)

Bridge Core 是 Bridge 系统的核心协调器，职责包括：

```typescript
export type BridgeCoreHandle = {
  bridgeSessionId: string
  environmentId: string
  sessionIngressUrl: string
  writeMessages(messages: Message[]): void
  writeSdkMessages(messages: SDKMessage[]): void
  sendControlRequest(request: SDKControlRequest): void
  sendControlResponse(response: SDKControlResponse): void
  teardown(): Promise<void>
}
```

核心功能：
- 环境注册（Environment Registration）
- 会话创建与管理
- 消息双向转发
- 控制指令处理

### 3.2 Transport Layer

```typescript
export type ReplBridgeTransport = {
  connect(): Promise<void>
  disconnect(): void
  send(message: unknown): void
  onMessage: (handler: (msg: unknown) => void) => void
}

// V1: 基于 SSE 的长轮询
export function createV1ReplTransport(config: V1Config): ReplBridgeTransport

// V2: 基于 WebSocket 的双向实时
export function createV2ReplTransport(config: V2Config): ReplBridgeTransport
```

Transport 层抽象支持多种传输协议，自动降级：
1. **WebSocket**：首选，双向实时
2. **SSE**：Server-Sent Events，单向推送
3. **长轮询**：最终 fallback

### 3.3 Hybrid Transport

```typescript
export class HybridTransport implements ReplBridgeTransport {
  private v1Transport: ReplBridgeTransport
  private v2Transport: ReplBridgeTransport
  private activeTransport: ReplBridgeTransport

  async connect(): Promise<void> {
    // 优先尝试 V2 (WebSocket)
    try {
      await this.v2Transport.connect()
      this.activeTransport = this.v2Transport
      return
    } catch (error) {
      logForDebugging('V2 transport failed, falling back to V1')
    }

    // Fallback 到 V1 (SSE)
    await this.v1Transport.connect()
    this.activeTransport = this.v1Transport
  }
}
```

Hybrid Transport 实现了自动降级：先尝试 WebSocket，失败后自动切换到 SSE。

## 4. 连接生命周期

### 4.1 初始化流程

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  CLI 启动 │───▶│ 注册环境      │───▶│ 创建会话      │───▶│ 连接 Ingress │
└──────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                      │                    │                    │
                      ▼                    ▼                    ▼
                POST /v1/envs       POST /v1/sessions     WebSocket/SSE
                获取 envId          获取 sessionId        建立长连接
```

### 4.2 消息同步流程

```
本地 REPL                    Bridge Core                   Web UI
    │                            │                           │
    │── 新消息 ─────────────────▶│                           │
    │    (writeMessages)         │                           │
    │                            │── 格式转换 ──────────────▶│
    │                            │   (Message → SDKMessage)  │
    │                            │                           │
    │                            │◀── 用户输入 ──────────────│
    │                            │    (onInboundMessage)     │
    │◀── 处理输入 ───────────────│                           │
```

### 4.3 断线重连机制

```typescript
// 指数退避重连
const POLL_ERROR_INITIAL_DELAY_MS = 2_000
const POLL_ERROR_MAX_DELAY_MS = 60_000
const POLL_ERROR_GIVE_UP_MS = 15 * 60 * 1000

async function reconnectWithBackoff(): Promise<void> {
  let delay = POLL_ERROR_INITIAL_DELAY_MS
  const startTime = Date.now()

  while (Date.now() - startTime < POLL_ERROR_GIVE_UP_MS) {
    try {
      await transport.connect()
      return // 连接成功
    } catch (error) {
      logForDebugging(`Reconnect failed, retrying in ${delay}ms`)
      await sleep(delay)
      delay = Math.min(delay * 2, POLL_ERROR_MAX_DELAY_MS)
    }
  }

  throw new Error('Reconnect attempts exhausted')
}
```

重连策略：
- 初始延迟 2 秒
- 最大延迟 60 秒
- 15 分钟后放弃

## 5. 控制协议

### 5.1 控制请求类型

```typescript
export type SDKControlRequest =
  | { type: 'interrupt' }                    // 中断当前操作
  | { type: 'set_permission_mode'; mode: PermissionMode }
  | { type: 'set_model'; model: string }
  | { type: 'permission_response'; approved: boolean }
  | { type: 'set_max_thinking_tokens'; maxTokens: number | null }
```

### 5.2 控制响应类型

```typescript
export type SDKControlResponse =
  | { type: 'interrupt_ack' }
  | { type: 'permission_mode_set'; ok: true } | { type: 'permission_mode_set'; ok: false; error: string }
  | { type: 'model_set' }
  | { type: 'permission_granted' } | { type: 'permission_denied' }
  | { type: 'max_thinking_tokens_set' }
  | { type: 'error'; message: string }
```

### 5.3 消息处理流程

```typescript
// bridgeMessaging.ts
export function handleServerControlRequest(
  request: SDKControlRequest,
  handlers: {
    onInterrupt?: () => void
    onSetPermissionMode?: (mode: PermissionMode) => void
    onSetModel?: (model: string) => void
    onPermissionResponse?: (approved: boolean) => void
  }
): SDKControlResponse {
  switch (request.type) {
    case 'interrupt':
      handlers.onInterrupt?.()
      return { type: 'interrupt_ack' }

    case 'set_permission_mode':
      const result = handlers.onSetPermissionMode?.(request.mode)
      return result?.ok
        ? { type: 'permission_mode_set', ok: true }
        : { type: 'permission_mode_set', ok: false, error: result?.error || 'Unknown error' }

    // ... 其他 case
  }
}
```

## 6. 安全与认证

### 6.1 Work Secret 机制

```typescript
// workSecret.ts
export function decodeWorkSecret(workSecret: string): {
  bridgeId: string
  workId: string
  envSecret: string
} {
  const decoded = base64UrlDecode(workSecret)
  // 解析 bridgeId, workId, envSecret
  return { bridgeId, workId, envSecret }
}
```

Work Secret 是 Bridge 连接的凭证，包含：
- `bridgeId`：桥接器标识
- `workId`：工作单元标识
- `envSecret`：环境密钥

### 6.2 受信任设备

```typescript
// trustedDevice.ts
export async function getTrustedDeviceToken(): Promise<string | undefined> {
  // 从系统 keychain 获取受信任设备令牌
  return keychain.get('claude-trusted-device-token')
}
```

受信任设备令牌存储在系统密钥链中，用于设备身份验证。

### 6.3 JWT 处理

```typescript
// jwtUtils.ts
export function parseJWT(token: string): JWTClaims {
  const [header, payload, signature] = token.split('.')
  return JSON.parse(base64UrlDecode(payload))
}

export function isJWTExpired(token: string): boolean {
  const claims = parseJWT(token)
  return claims.exp * 1000 < Date.now()
}
```

## 7. 消息映射与格式转换

### 7.1 内部消息转 SDK 消息

```typescript
// utils/messages/mappers.ts
export function toSDKMessages(messages: Message[]): SDKMessage[] {
  return messages.map((msg) => {
    switch (msg.type) {
      case 'user':
        return {
          role: 'user',
          content: msg.content,
          uuid: msg.uuid,
        }
      case 'assistant':
        return {
          role: 'assistant',
          content: msg.content,
          tool_calls: msg.toolCalls,
          uuid: msg.uuid,
        }
      case 'tool_result':
        return {
          role: 'tool_result',
          tool_use_id: msg.toolUseId,
          content: msg.content,
          uuid: msg.uuid,
        }
    }
  })
}
```

### 7.2 本地输出转换

```typescript
export function localCommandOutputToSDKAssistantMessage(
  stdout: string,
  stderr: string
): SDKMessage {
  return {
    role: 'assistant',
    content: [
      stdout && { type: 'local_command_stdout', content: stdout },
      stderr && { type: 'local_command_stderr', content: stderr },
    ].filter(Boolean),
  }
}
```

## 8. 容量管理

### 8.1 Capacity Wake 机制

```typescript
// capacityWake.ts
export function createCapacityWake(): CapacitySignal {
  const abortController = new AbortController()
  let pendingResolve: (() => void) | null = null

  return {
    // 等待容量可用
    waitForCapacity(): Promise<void> {
      if (hasCapacity()) return Promise.resolve()
      return new Promise((resolve) => {
        pendingResolve = resolve
      })
    },

    // 释放容量信号
    signalCapacityAvailable(): void {
      pendingResolve?.()
      pendingResolve = null
    },

    abortSignal: abortController.signal,
  }
}
```

Capacity Wake 防止消息洪泛：当 Web 端处理能力不足时，本地暂停发送。

### 8.2 Flush Gate

```typescript
// flushGate.ts
export class FlushGate {
  private flushPromise: Promise<void> | null = null
  private resolveFlush: (() => void) | null = null

  // 等待门打开
  async waitForFlush(): Promise<void> {
    if (!this.flushPromise) {
      this.flushPromise = new Promise((resolve) => {
        this.resolveFlush = resolve
      })
    }
    return this.flushPromise
  }

  // 打开门，让所有等待继续
  open(): void {
    this.resolveFlush?.()
    this.flushPromise = null
    this.resolveFlush = null
  }
}
```

Flush Gate 用于批量消息的原子发送。

## 9. 调试与诊断

### 9.1 Bridge 调试工具

```typescript
// bridgeDebug.ts
export function wrapApiForFaultInjection<T extends object>(
  api: T,
  faultConfig: FaultConfig
): T {
  return new Proxy(api, {
    get(target, prop) {
      if (faultConfig.shouldFail(prop as string)) {
        throw new Error(`Injected fault: ${String(prop)}`)
      }
      return target[prop as keyof T]
    },
  })
}

export function injectBridgeFault(faultType: BridgeFaultType): void {
  activeFaults.add(faultType)
}
```

支持注入故障进行测试：
- 网络断开
- 认证失败
- 消息丢失

### 9.2 诊断日志

```typescript
// debugUtils.ts
export function describeAxiosError(error: unknown): string {
  if (!isAxiosError(error)) return String(error)

  return [
    `Status: ${error.response?.status}`,
    `Code: ${error.code}`,
    `Message: ${error.message}`,
    `Response: ${JSON.stringify(error.response?.data)}`,
  ].join('\n')
}
```

## 10. 创新点与反思

### 10.1 创新点

1. **Hybrid Transport**：自动降级的传输层设计
2. **Capacity Wake**：背压机制防止消息洪泛
3. **Bridge Pointer**：持久化的 Bridge 会话指针
4. **Flush Gate**：批量消息原子发送

### 10.2 架构权衡

**实时性 vs 可靠性**

- WebSocket：实时性好，但连接不稳定
- SSE：可靠性好，但单向通信
- 长轮询：最可靠，但延迟高

Hybrid Transport 通过自动降级平衡了这两者。

### 10.3 生产经验

1. **指数退避重连**：避免对服务器造成压力
2. **消息去重**：使用 UUID 集合防止重复发送
3. **优雅关闭**：session 归档后再断开连接

### 10.4 未来演进

1. **Edge Relay**：通过 CDN Edge 节点加速连接
2. **QUIC 传输**：减少连接建立延迟
3. **Local-First 模式**：支持离线编辑后同步

---

Claude Code 的 Bridge 系统是一个生产级的远程控制实现，它解决了 CLI 工具与 Web 界面实时同步的复杂问题。通过 Hybrid Transport、Capacity Wake、Work Secret 等创新设计，实现了低延迟、高可靠的双向同步。理解其设计，对于构建任何需要远程控制的 CLI 工具都具有重要参考价值。
