# MCP System 深度技术思考

## 一、协议抽象的边界工程

### 1.1 传输协议的统一幻象

MCP（Model Context Protocol）承诺了「一次编写，到处运行」的愿景，但 Claude Code 的实现揭示了协议抽象背后的复杂性：

**stdio 的进程边界**
- 优点：完全隔离，崩溃不影响主进程
- 缺点：启动开销（300MB+ 内存）、序列化成本、双向通信复杂

**SSE/HTTP 的网络边界**
- 优点：跨机器、易扩展
- 缺点：防火墙/NAT 穿透、重连逻辑、会话状态管理

**WebSocket 的双向流**
- 优点：实时推送、低延迟
- 缺点：代理兼容性、心跳维护、关闭握手可靠性

Claude Code 的解决方案不是「选择最好的」，而是「支持所有，自动适配」：
```
Server Config ──▶ Transport Selector ──▶ 具体 Transport 实现
                     │
                     ├── stdio（本地工具）
                     ├── SSE（传统 HTTP）
                     ├── StreamableHTTP（新版规范）
                     └── WebSocket（实时推送）
```

这种设计哲学是「实用主义优先」——不等待完美的标准，而是支持实践中遇到的所有场景。

### 1.2 InProcess 优化的反模式突破

通常 MCP 强调「进程隔离」，但 Claude Code 针对高频内置服务器（Chrome、Computer Use）打破了这一规则：

**LinkedTransportPair 模式**
```
主进程 ──▶ LinkedTransportPair ──▶ 同进程 MCP Server
            │
            └── 绕过 JSON-RPC 序列化
            └── 直接方法调用
            └── 共享内存传递大数据
```

这种「反模式」带来了 10x 的性能提升：
- 启动时间：从秒级降至毫秒级
- 内存占用：节省一个完整进程的开销
- 通信延迟：消除 IPC 和序列化成本

关键洞察：**架构原则（隔离）应该服务于目标（性能/可靠性），而非相反**。当隔离带来的成本超过收益时，应该有勇气打破原则。

## 二、认证系统的信任网络

### 2.1 OAuth 的分布式状态挑战

MCP 的 OAuth 实现面临一个经典分布式系统问题：token 状态如何在多个进程间同步？

**问题场景**
1. 用户授权获得 token
2. Token 存储在系统 keychain
3. VS Code Extension（另一个进程）更新了 token
4. Claude Code CLI 进程使用 stale token 导致 401

**解决方案：文件锁 + 缓存失效**
```
Token 刷新前 ──▶ 获取 lockfile
                     │
                     ▼
              读取 keychain（最新值）
                     │
                     ▼
              执行 refresh
                     │
                     ▼
              写入 keychain
                     │
                     ▼
              释放 lockfile
```

`clearKeychainCache()` 在重连前被调用，确保读取到最新值。这种设计牺牲了部分性能（额外的文件锁），换取了正确性。

### 2.2 Step-up Auth 的用户体验优化

OAuth scope 升级是一个常见痛点：
- 初始授权只需要 `read` 权限
- 后来需要 `write` 权限
- 传统方案：重新走完整授权流程

Claude Code 的 Step-up Auth 实现了「增量授权」：
```
Server 返回 403 insufficient_scope
           │
           ▼
wrapFetchWithStepUpDetection 捕获
           │
           ▼
标记 pending scope，隐藏 refresh_token
           │
           ▼
下次 tokens() 调用触发完整 PKCE 流程
           │
           ▼
用户仅需确认新增的 scope
```

这种设计将「重新授权」转化为「授权升级」，用户体验从「中断-重开」变为「无缝-确认」。

### 2.3 XAA 的企业单点登录

Cross-App Access（XAA）是面向企业场景的优雅设计：

**传统方案的问题**
- 每个 MCP Server 独立 OAuth
- 用户需要为每个 server 单独登录
- 在大型企业中，这可能是数十次登录

**XAA 的解决方案**
```
用户第一次登录 IdP
           │
           ▼
Claude Code 缓存 id_token（长期有效）
           │
           ▼
后续每个 XAA-enabled MCP Server
           │
           ▼
通过 RFC 8693 Token Exchange 换得 access_token
           │
           ▼
用户无感知，server 获得独立 token
```

这实现了「一次登录，处处访问」的企业 SSO 体验，同时保持了每个 server token 的独立性（安全隔离）。

## 三、连接状态机的容错设计

### 3.1 会话过期的自愈机制

MCP 远程会话可能因各种原因过期：
- 服务器重启
- 负载均衡器重新调度
- 空闲超时

Claude Code 实现了「透明自愈」：
```
Tool Call 返回 404 + code -32001 (Session Not Found)
                    │
                    ▼
            isMcpSessionExpiredError() 识别
                    │
                    ▼
            清除 client 缓存
                    │
                    ▼
            重新建立连接（指数退避）
                    │
                    ▼
            重试原请求（用户无感知）
```

这种「let it crash」哲学的温和版本——接受失败，快速恢复，对用户透明。

### 3.2 连接池的动态管理

MCP Client 不是「连接即持有」，而是「按需连接，空闲释放」：

```
闲置连接 ──▶ 空闲检测 ──▶ 超过阈值 ──▶ 优雅关闭
                              │
                              ▼
                        保留配置，快速重建
```

这种设计平衡了：
- **延迟**：避免频繁重连
- **资源**：不长期占用服务器资源
- **容错**：定期重建连接，避免 staleness

## 四、工具映射的类型系统挑战

### 4.1 动态 Schema 的类型擦除

MCP Server 在连接时才暴露其工具 Schema，这与 TypeScript 的静态类型系统存在张力：

```typescript
// 编译时未知
type MCPTools = ???

// 运行时通过 listTools() 获得
const tools = await client.listTools()

// 映射为 Tool<any, any, any>
```

解决方案是「运行时类型守护 + 边界校验」：
- 输入时：Zod Schema 验证
- 输出时：结构检查与转换
- 内部：any 类型，信任 MCP SDK

这种「类型擦除」虽然牺牲了部分编译时安全，但获得了运行时的灵活性。

### 4.2 命名空间的隔离策略

不同 MCP Server 可能提供同名工具（如两个 server 都有 `search`）。Claude Code 采用「命名空间前缀」策略：
```
原始工具名: search
Server 标识: github-mcp
映射后工具名: mcp__github_mcp__search
```

这种设计确保了：
- 工具名全局唯一
- 用户可以通过前缀识别工具来源
- 权限规则可以针对特定 server 的工具

## 五、结果治理的数据流工程

### 5.1 大结果的背压机制

MCP 工具可能返回巨大结果（数据库查询、大文件读取）。Claude Code 实现了「背压」机制防止上下文爆炸：

```
结果大小检测
       │
       ├── 小结果 ──▶ 直接返回
       │
       └── 大结果 ──▶ 持久化到磁盘
                         │
                         ▼
                    返回引用 + 元数据
                         │
                         ▼
                    LLM 按需读取（通过 FileReadTool）
```

这种「流式卸载」确保了：
- 上下文窗口不被撑爆
- 数据完整性不丢失
- 模型可以按需访问细节

### 5.2 多模态结果的统一抽象

MCP 结果可能包含文本、图片、音频等多种模态。Claude Code 将其统一为 Anthropic API 的 ContentBlock：
```
MCP Result ──▶ processMCPResult()
                     │
                     ├── text ──▶ TextBlock
                     ├── image ──▶ ImageBlock（自动 resize）
                     └── binary ──▶ 持久化 + TextBlock（路径）
```

这种「归一化」简化了上层逻辑——Query Loop 无需关心数据来源，统一处理 ContentBlock。

## 六、反思与未来演进

### 6.1 当前架构的权衡

**配置复杂性的权衡**
支持 6 级配置来源（enterprise/local/project/user/plugin/claude.ai）提供了灵活性，但也增加了理解成本。用户可能困惑「为什么这个 server 被禁用了？」。

**连接状态的可见性**
虽然自动重连提升了可靠性，但也可能掩盖问题。用户可能不知道某个 server 已经重连了 5 次，直到询问才意识到网络不稳定。

### 6.2 未来演进方向

**MCP Server 的能力发现**
当前需要手动配置 server URL。未来可以探索自动发现机制（类似 DNS-SD），让工具能自动找到局域网或云端的 MCP Server。

**细粒度的权限控制**
当前权限控制是 server 级别。未来可以探索 tool-level、甚至 argument-level 的权限控制（如「允许 read，但禁止 write」）。

**MCP 的编排与组合**
多个 MCP Server 之间可能存在依赖关系（如 Server A 的输出是 Server B 的输入）。可以探索「MCP 工作流」机制，让用户定义跨 server 的调用链。

---

Claude Code 的 MCP System 是一个生产级的协议实现，它不仅遵循规范，更在认证、容错、性能等方面做了大量工程创新。理解其设计，对于任何需要构建可扩展 AI 插件系统的团队都具有重要参考价值。
