# Claude Code MCP 系统技术分析

## 1. 模块总结介绍

MCP（Model Context Protocol）系统是 Claude Code 中负责连接、发现、执行和管理外部 MCP 服务器的核心模块。它的设计目标是将第三方 MCP 服务器的能力无缝集成到 Claude Code 的 agent 工作流中，使 LLM 能够自动发现并调用外部工具、读取资源、执行 prompt 命令。

该模块覆盖了从配置管理到 UI 呈现的完整链路，支持多种传输协议（stdio、SSE、HTTP、WebSocket、IDE Bridge、SDK 内嵌），并内置了企业级策略控制、OAuth 认证、自动重连、大输出持久化、官方 registry 集成等能力。

## 2. 系统架构

MCP 系统采用**四层架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                     UI Layer (React/ink)                    │
│  MCPSettings / MCPListPanel / MCPReconnect / ElicitationDialog│
├─────────────────────────────────────────────────────────────┤
│                     Tool Layer                              │
│  MCPTool / ListMcpResourcesTool / ReadMcpResourceTool       │
│  McpAuthTool / 技能搜索 (MCP_SKILLS)                        │
├─────────────────────────────────────────────────────────────┤
│                     Client Layer                            │
│  MCPConnectionManager (React Context)                       │
│  useManageMCPConnections (hooks)                            │
│  client.ts (connectToServer / fetchTools / callMCPTool)     │
├─────────────────────────────────────────────────────────────┤
│                     Protocol Layer                          │
│  MCP SDK (@modelcontextprotocol/sdk)                        │
│  传输适配: Stdio / SSE / StreamableHTTP / WS / InProcess    │
│  SdkControlTransport / ClaudeAuthProvider                   │
└─────────────────────────────────────────────────────────────┘
```

- **Protocol Layer**：直接与 `@modelcontextprotocol/sdk` 交互，封装不同传输协议的细节，处理连接、认证、消息路由。
- **Client Layer**：管理服务器生命周期（连接、重连、断开、缓存失效），将 SDK 的 client 实例包装成 `ConnectedMCPServer`状态对象。
- **Tool Layer**：将 MCP server 的 tools/resources/prompts 映射为 Claude Code 内部的 `Tool` / `Command`抽象，供 LLM 调用。
- **UI Layer**：提供命令行交互界面（`/mcp`），展示服务器状态、工具列表、认证引导、重连操作。

## 3. 分层说明

### 3.1 Protocol Layer

核心文件：`services/mcp/client.ts`、`auth.ts`、`SdkControlTransport.ts`、`InProcessTransport.ts`

- **传输多样性**：支持 stdio（子进程）、SSE（`SSEClientTransport`）、Streamable HTTP（`StreamableHTTPClientTransport`）、WebSocket（`WebSocketTransport`）、SSE-IDE / WS-IDE（IDE bridge）、claudeai-proxy（代理隧道）、SDK bridge（`SdkControlTransport`）、InProcess（Chrome / Computer Use 优化）。
- **认证体系**：自定义 `ClaudeAuthProvider` 实现 `OAuthClientProvider`，支持：
  - 动态客户端注册（DCR）
  - 预配置 `clientId` + `clientSecret`
  - CIMD（URL-based client_id）
  - Step-up scope 检测（`wrapFetchWithStepUpDetection`）
  - 令牌预刷新（token 过期前 5 分钟主动 refresh）
  - 跨应用访问（XAA / SEP-990）——通过 IdP 的 `id_token` + RFC 8693 token exchange 实现单点登录
- **请求超时**：`wrapFetchWithTimeout` 为每个请求创建独立的 `AbortController`，避免共享 signal 在长连接中过期；对 SSE GET 请求跳过超时，保证长连接存活。

### 3.2 Client Layer

核心文件：`services/mcp/useManageMCPConnections.ts`、`MCPConnectionManager.tsx`、`client.ts`

- **状态机**：每个 MCP server 处于以下状态之一：`pending` → `connected` | `failed` | `needs-auth` | `disabled`。
- **批处理连接**：`getMcpToolsCommandsAndResources` 使用 `p-map` 并发连接，本地服务器（stdio/sdk）与远程服务器分别设置不同并发上限（默认 3 vs 20），避免子进程资源竞争。
- **缓存策略**：
  - `connectToServer` 使用 `lodash.memoize`（key = `name + JSON.stringify(config)`）。
  - `fetchToolsForClient`、`fetchResourcesForClient`、`fetchCommandsForClient` 使用 LRU 缓存（上限 20），keyed by server name。
- **自动重连**：远程 transport（SSE/HTTP/WS/claudeai-proxy）断开后，触发指数退避重连（1s → 2s → 4s → 8s → 16s，最大 30s），最多 5 次尝试（`MAX_RECONNECT_ATTEMPTS`）。
- **Session 过期处理**：检测到 MCP 404 + JSON-RPC `-32001`（Session not found）时，立即清除缓存并触发重新连接；tool call 层对该错误自动重试一次（`MAX_SESSION_RETRIES = 1`）。

### 3.3 Tool Layer

核心文件：`tools/MCPTool/MCPTool.ts`、`ListMcpResourcesTool`、`ReadMcpResourceTool`、`McpAuthTool`

- **工具命名规范**：`mcp__<normalized_server>__<normalized_tool>`，通过 `buildMcpToolName` 生成，避免名称冲突并支持权限规则精确匹配。
- **动态工具生成**：`fetchToolsForClient` 将 MCP 的 `ListToolsResult` 映射为 `Tool` 对象，每个 tool 的 `call()` 实际委托给 `callMCPToolWithUrlElicitationRetry`。支持：
  - `isReadOnly` / `isDestructive` / `isOpenWorld` 从 MCP annotations 推断
  - `searchHint` 与 `alwaysLoad` 支持 anthropic 扩展 meta
  - 自动分类用于折叠（`classifyMcpToolForCollapse`）
- **资源工具**：如果任一 connected server 支持 resources，全局注册 `ListMcpResourcesTool` 与 `ReadMcpResourceTool`，供 LLM 浏览和读取资源。
- **认证伪工具**：对于 needs-auth 的 server，在状态中注入 `McpAuthTool`（`createMcpAuthTool`），LLM 可调用该工具启动 OAuth 流程，完成后自动替换为真实工具集合。

### 3.4 UI Layer

核心文件：`components/mcp/*.tsx`、`commands/mcp/mcp.tsx`

- **命令入口**：`/mcp` 命令支持子参数：`reconnect <server>`、`enable [all/server]`、`disable [all/server]`、`no-redirect`。
- **重连界面**：`MCPReconnect` 组件在重连过程中展示进度与结果。
- **设置面板**：`MCPSettings` / `MCPListPanel` / `MCPToolListView` 展示服务器列表、连接状态、工具详情、认证按钮。
- **URL Elicitation**：当 tool call 返回 `-32042`（UrlElicitationRequired）时，`ElicitationDialog` 弹出授权 URL，用户确认后自动重试工具调用。

## 4. 交互流程

### 4.1 服务器注册与发现

```
1. 配置加载
   ├─ getAllMcpConfigs() 合并多层配置
   │   ├─ enterprise (managed-mcp.json) — 最高优先级且独占
   │   ├─ local / project (.mcp.json, 父目录向上遍历，近者优先)
   │   ├─ user (全局 config)
   │   ├─ plugin (动态加载的插件 MCP servers)
   │   └─ claude.ai (网络 fetch，通过 eligibility 校验)
   └─ 策略过滤: allowedMcpServers / deniedMcpServers (企业策略)

2. 去重 (dedup)
   ├─ plugin servers 与 manual servers 通过签名去重
   │   (stdio: command+args; remote: URL)
   └─ claude.ai connectors 与手动配置通过 URL 去重

3. 状态初始化
   └─ initializeServersAsPending() 将新 server 加入 appState.mcp.clients 为 pending
```

### 4.2 连接与工具发现

```
connectToServer(name, config) [memoized]
  ├─ 根据 config.type 初始化对应 Transport
  ├─ Client.connect(transport) with timeout (默认 30s)
  ├─ 注册 onerror / onclose 监听器（用于连接断开检测与缓存清除）
  ├─ 注册 notification handlers (tools/list_changed, prompts/list_changed, resources/list_changed)
  └─ 返回 ConnectedMCPServer

fetchToolsForClient(client) [LRU cached]
  ├─ client.request({ method: 'tools/list' })
  └─ 将每个 tool 包装为 Tool 对象（含 mcpInfo / checkPermissions / call 等）

fetchResourcesForClient / fetchCommandsForClient
  同理，分别映射 resources/list 与 prompts/list
```

### 4.3 工具执行

```
LLM -> tool_use (mcp__server__tool)
  ↓
Tool.call(args, context)
  ↓
callMCPToolWithUrlElicitationRetry
  ├─ 调用 callMCPTool
  │   ├─ ensureConnectedClient(client) — 若缓存被清除则重新连接
  │   ├─ client.callTool({ name, arguments, _meta })
  │   ├─ 处理 progress 回调 (+ 30s 心跳日志)
  │   └─ transform/processMCPResult
  │       ├─ 图片自动 resize / 压缩
  │       ├─ 大输出文件持久化（超出 token 上限时写入磁盘）
  │       └─ 返回 content blocks
  └─ 若返回 -32042 (UrlElicitationRequired)
      ├─ 运行 elicitation hooks
      ├─ 无 hook 处理则通过 UI/structuredIO 请求用户确认
      └─ 用户 accept 后重试 tool call
```

## 5. 技术原理

### 5.1 传输协议适配

Claude Code 的传输层设计充分考虑了不同运行环境的兼容性：

- **stdio**：使用 `StdioClientTransport`，stderr 被重定向为 pipe 以防止污染 UI；子进程退出时通过 SIGINT → SIGTERM → SIGKILL 的优雅升级策略确保清理彻底。
- **SSE/HTTP**：分别为旧版 SSE 与新 MCP 规范中的 Streamable HTTP 使用不同 SDK transport。HTTP 请求强制注入 `Accept: application/json, text/event-stream`，避免服务器返回 406。
- **WebSocket**：对 Bun 与 Node 分别采用原生 `WebSocket` 或 `ws` 包，支持 proxy、mTLS。
- **SDK Bridge**：`SdkControlClientTransport` 将 MCP JSON-RPC 消息封装为 control request 通过 stdout 发往 SDK 进程，实现跨进程零拷贝通信。
- **InProcess**：针对 Chrome MCP 与 Computer Use 等内置服务器，通过 `createLinkedTransportPair` 避免启动 300MB+ 子进程，提升启动速度与内存效率。

### 5.2 OAuth 与认证

`ClaudeAuthProvider` 是认证核心：

- **缓存架构**：token 存储在系统 keychain（macOS）或等效安全存储中，通过 `SecureStorage` 读写。为避免并发刷新冲突，使用基于文件 lockfile 的互斥机制。
- **Step-up Auth**：当服务器返回 403 `insufficient_scope` 时，`wrapFetchWithStepUpDetection` 捕获并在 provider 上设置 pending scope。此后 `tokens()` 会主动隐藏 `refresh_token`，迫使 SDK 走完整的 authorization code (PKCE) 流程以获取更高权限的 token。
- **跨进程一致性**：`clearKeychainCache` 在重连前被调用，确保其他进程（如 VS Code Extension）修改 token 后，CLI 子进程能读取最新值。
- **XAA（Cross-App Access）**：企业场景下，用户只需在 IdP 登录一次，Claude Code 将 `id_token` 缓存于 keychain。后续每个 XAA-enabled MCP server 通过 RFC 8693 token exchange 静默换得 access_token，无需重复弹窗授权。

### 5.3 结果处理与大输出优化

MCP tool 返回的内容需转换为 Anthropic API 的 `ContentBlockParam[]`。系统支持：

- **图片自动压缩**：`maybeResizeAndDownsampleImageBuffer` 将图片调整至 API 维度限制以内。
- **音频/二进制持久化**：`persistBlobToTextBlock` 将 base64 blob 写入本地文件，仅在上下文中返回文件路径，避免上下文膨胀。
- **大输出截断与文件化**：`processMCPResult` 检测内容是否超出 token 阈值（`ENABLE_MCP_LARGE_OUTPUT_FILES`）。若超出，将完整结果写入 JSON 文件，并返回结构化说明，引导 LLM 按需读取。若内容包含图片，则回退到截断策略以保护图像可查看性。

## 6. 创新点

1. **多源配置合并与企业策略沙箱**：支持 enterprise / local / project / user / plugin / claude.ai 六级配置来源，并通过 allowed/denied list 实现企业级 MCP 访问控制。`doesEnterpriseMcpConfigExist()` 开启时，enterprise 配置具有独占权。

2. **智能去重（Content-based Dedup）**：插件服务器与手动配置服务器即使 key 不同（如 `slack` vs `plugin:marketplace:slack`），也会通过命令签名或 URL 签名去重，避免同一底层服务被重复连接。

3. **连接状态机 + 自动重连 + Session 自愈**：远程服务器断线后自动指数退避重连；检测到 MCP Session 404 时立即清除缓存并重建 session，且 tool call 层对该错误自动重试一次，用户几乎无感知。

4. **Step-up OAuth 与 XAA**：
   - Step-up 解决 OAuth scope 升级难题，通过 403 拦截和 token 隐藏策略，强制走 PKCE 重新授权。
   - XAA 实现企业单点登录形态，一次浏览器授权即可 silent auth 多个下游 MCP server。

5. **InProcess 优化**：将 Chrome MCP 和 Computer Use MCP 从子进程模式改为进程内模式，通过 LinkedTransportPair 直接通信，显著降低启动延迟和内存占用。

6. **官方 Registry 集成**：`officialRegistry.ts` 在后台预取 Anthropic 官方 MCP registry，用于厂商 URL 可信度判断与安全决策。

7. **Channel 通知（KAIROS）**：具备 channel 能力的 MCP server 可向 Claude Code 推送消息（`notifications/claude/channel`），直接入队到用户消息流中，实现 server 到 agent 的主动通信。

## 7. 关键技术

- `@modelcontextprotocol/sdk`：官方 MCP TypeScript SDK，提供 Client、Transport、Auth、Schema 等原语。
- `memoize` + `memoizeWithLRU`：连接与工具/资源/命令 fetch 均使用细粒度缓存，在性能和一致性之间取得平衡；缓存失效与 onclose/onerror 事件联动。
- `p-map`：替换了旧的固定批次顺序执行模型，改为并发槽位动态释放，避免单个慢 server 阻塞整批连接。
- `lockfile` + `SecureStorage`：token 刷新使用跨进程文件锁，token 持久化使用系统安全存储。
- `AbortController` + 独立 timeout：每个 HTTP 请求独立控制超时，避免 `AbortSignal.timeout` 在 Bun 中的内存泄漏问题。
- `normalizeNameForMCP` / `buildMcpToolName`：统一命名规范，确保工具名、权限规则、状态存储之间的互操作性。

## 8. 思考总结

Claude Code 的 MCP 系统设计体现了**企业级稳定性**与**开发者体验**的深度平衡。

从架构上看，四层分离（Protocol / Client / Tool / UI）使得新增传输协议或认证方式时影响面可控；`client.ts` 虽超过 3000 行，但通过功能内聚（连接、fetch、call、transform、util）保持了可读性。状态机 + React hooks（`useManageMCPConnections`）将复杂的异步生命周期封装为声明式的状态更新，UI 与业务逻辑解耦良好。

在工程细节上，团队对边缘问题的处理尤为精细：Bun 与 Node 的 WebSocket 差异、SSE 长连接与 HTTP POST 的超时策略区分、`AbortSignal.timeout` 的内存泄漏规避、OAuth error body 的 200 伪装处理、macOS keychain 的 4096 字节限制导致 metadata 剪裁等，均展现了生产环境打磨的痕迹。

值得关注的演进方向：
- **XAA 跨进程锁**：当前 XAA silent refresh 仅通过 `_refreshInProgress` 去重单进程内并发，TODO 中已标记需要引入类似 `refreshAuthorization` 的 lockfile 跨进程锁。
- **插件 MCP 热更新**：`pluginReconnectKey` 机制已支持通过 `/reload-plugins` 触发重新加载，但配置变更的实时感知仍有优化空间。
- **MCP Skills 与本地搜索集成**：`fetchMcpSkillsForClient` 与 `skillSearch` 联动，预示 MCP 系统未来不仅是工具提供方，也将成为 agent 技能发现与编排的核心基础设施。
