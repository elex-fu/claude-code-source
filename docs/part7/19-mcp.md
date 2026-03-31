# 第 19 章：MCP 协议 —— 工具的互联网

> MCP 之于 AI 工具，就像 HTTP 之于网页——一个开放协议，让工具可以互联互通。

---

## 19.1 MCP 是什么

MCP（Model Context Protocol）是 Anthropic 在 2024 年提出的开放协议，定义了 AI 模型与外部工具/资源之间的标准通信方式。

在 MCP 之前，每个 AI 工具都有自己的集成方式：
- GitHub Copilot 有自己的 API
- Cursor 有自己的插件系统
- Claude Code 有自己的工具定义

这导致了碎片化：为一个 AI 工具开发的集成，无法直接用于另一个 AI 工具。

MCP 的目标是：**让任何工具都能被任何支持 MCP 的 AI 使用**。

---

## 19.2 MCP 的架构

![MCP 架构：AI 工具的互联网](../diagrams/mcp-architecture.svg)

MCP 服务器可以提供三种资源：
- **Tools**：可以被 Claude 调用的函数
- **Resources**：可以被 Claude 读取的数据（文件、数据库记录等）
- **Prompts**：预定义的提示模板

---

## 19.3 Claude Code 的 MCP 客户端

`src/services/mcp/` 实现了完整的 MCP 客户端：

```typescript
// MCP 服务器连接
type MCPServerConnection = {
  name: string              // 服务器名称
  transport: MCPTransport   // 传输方式（stdio 或 HTTP）
  tools: MCPTool[]          // 服务器提供的工具
  resources: MCPResource[]  // 服务器提供的资源
  prompts: MCPPrompt[]      // 服务器提供的提示
}

// 连接到 MCP 服务器
async function connectMCPServer(config: MCPServerConfig): Promise<MCPServerConnection> {
  const transport = config.type === 'stdio'
    ? new StdioTransport(config.command, config.args)
    : new HTTPTransport(config.url)

  const client = new MCPClient(transport)
  await client.connect()

  return {
    name: config.name,
    transport,
    tools: await client.listTools(),
    resources: await client.listResources(),
    prompts: await client.listPrompts(),
  }
}
```

---

## 19.4 MCP 工具的动态注册

MCP 服务器提供的工具会动态注册到 Claude Code 的工具系统中：

```typescript
// 将 MCP 工具包装成 Claude Code 工具
function wrapMCPTool(mcpTool: MCPTool, server: MCPServerConnection): Tool {
  return {
    name: `mcp__${server.name}__${mcpTool.name}`,  // 命名空间避免冲突
    description: mcpTool.description,
    inputSchema: mcpTool.inputSchema,

    async execute(input, context) {
      // 通过 MCP 协议调用工具
      const result = await server.callTool(mcpTool.name, input)
      return { type: 'tool_result', content: result }
    }
  }
}
```

注意工具名称的命名空间：`mcp__<服务器名>__<工具名>`。这防止了不同 MCP 服务器的工具名冲突。

---

## 19.5 MCP 资源的访问

MCP 资源通过 `ListMcpResourcesTool` 和 `ReadMcpResourceTool` 访问：

```typescript
// 列出所有 MCP 资源
await ListMcpResourcesTool.execute({
  server_name: 'github'  // 可选，不指定则列出所有服务器的资源
}, context)
// 返回：[{ uri: 'github://repos/myorg/myrepo', name: 'myrepo', ... }]

// 读取特定资源
await ReadMcpResourceTool.execute({
  uri: 'github://repos/myorg/myrepo/issues/123'
}, context)
// 返回：Issue #123 的详细内容
```

---

## 19.6 MCP 认证

MCP 服务器可能需要认证。认证流程：

![MCP 认证流程](../diagrams/mcp-auth-flow.svg)

`McpAuthTool` 处理认证流程：

```typescript
// 当 MCP 工具调用返回 -32042 错误（需要认证）时
// McpAuthTool 触发认证流程
await McpAuthTool.execute({
  server_name: 'github',
  auth_url: 'https://github.com/login/oauth/authorize?...'
}, context)
```

---

## 19.7 配置 MCP 服务器

用户通过配置文件添加 MCP 服务器：

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    },
    "postgres": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    },
    "custom-api": {
      "type": "http",
      "url": "http://localhost:3000/mcp"
    }
  }
}
```

---

## 19.8 开发自己的 MCP 服务器

MCP 协议是开放的，任何人都可以开发 MCP 服务器。一个简单的 MCP 服务器：

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new Server({
  name: 'my-tools',
  version: '1.0.0',
})

// 注册工具
server.setRequestHandler('tools/list', async () => ({
  tools: [{
    name: 'get_weather',
    description: '获取指定城市的天气',
    inputSchema: {
      type: 'object',
      properties: {
        city: { type: 'string', description: '城市名称' }
      },
      required: ['city']
    }
  }]
}))

// 处理工具调用
server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'get_weather') {
    const { city } = request.params.arguments
    const weather = await fetchWeather(city)
    return { content: [{ type: 'text', text: JSON.stringify(weather) }] }
  }
})

// 启动服务器
const transport = new StdioServerTransport()
await server.connect(transport)
```

---

## 19.9 MCP 生态系统

MCP 协议发布后，已经有大量的 MCP 服务器：

| 服务器 | 提供的能力 |
|--------|-----------|
| GitHub MCP | 读写 Issues、PR、代码 |
| PostgreSQL MCP | 查询数据库 |
| Filesystem MCP | 访问文件系统（沙箱化） |
| Brave Search MCP | 网络搜索 |
| Slack MCP | 读写 Slack 消息 |
| Google Drive MCP | 访问 Google Drive |
| Puppeteer MCP | 控制浏览器 |

这个生态系统还在快速增长。

---

## 19.10 MCP 的设计哲学

MCP 的设计体现了几个重要原则：

**开放标准**：MCP 是开放协议，不是 Anthropic 的私有 API。任何 AI 工具都可以实现 MCP 客户端，任何服务都可以实现 MCP 服务器。

**关注点分离**：AI 模型不需要知道工具的实现细节，只需要知道工具的接口（name、description、schema）。

**安全边界**：MCP 服务器运行在独立进程中，与 AI 模型隔离。服务器的权限由用户配置，不由 AI 决定。

**可组合性**：多个 MCP 服务器可以同时连接，Claude 可以跨服务器组合工具。

---

## 19.11 小结

MCP 是 Claude Code 扩展能力的核心机制：

- **开放协议**：任何服务都可以成为 MCP 服务器
- **动态注册**：MCP 工具自动注册到 Claude Code 的工具系统
- **三种资源**：Tools（可调用）、Resources（可读取）、Prompts（模板）
- **认证支持**：内置 OAuth 认证流程
- **生态系统**：已有大量现成的 MCP 服务器

MCP 让 Claude Code 的能力边界从"内置工具"扩展到了"整个互联网上的服务"。

---

*下一章：[Skills 系统](./20-skills.md)*
