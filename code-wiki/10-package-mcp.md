# 10 `@earendil-works/pi-mcp`

## 1. 职责

`pi-mcp` 是一个独立、轻量的 Model Context Protocol 客户端，不依赖官方 MCP SDK 或其他 Pi 包：

- 传输无关的客户端核心。
- stdio 与 Streamable HTTP 传输。
- JSON-RPC 关联、初始化、超时、取消、进度、服务端请求处理。
- MCP OAuth 子集（发现、PKCE、动态客户端注册、刷新、step-up）。
- 把 MCP 工具转换为 `pi-agent-core` 的 `AgentTool`。

源码：[packages/mcp/src](../packages/mcp/src/)

## 2. 模块

| 文件/目录 | 职责 |
|---|---|
| [client.ts](../packages/mcp/src/client.ts) | `McpClient`：协议状态机与请求关联 |
| [transports/stdio.ts](../packages/mcp/src/transports/stdio.ts) | 子进程 stdio 传输 |
| [transports/streamable-http.ts](../packages/mcp/src/transports/streamable-http.ts) | Streamable HTTP、会话、GET 流、断线续传 |
| [transports/transport.ts](../packages/mcp/src/transports/transport.ts) | `McpTransport` 接口 |
| [protocol/jsonrpc.ts](../packages/mcp/src/protocol/jsonrpc.ts) | JSON-RPC 类型、错误、解析 |
| [protocol/types.ts](../packages/mcp/src/protocol/types.ts) | MCP 协议类型与版本 |
| [protocol/content.ts](../packages/mcp/src/protocol/content.ts) | 内容块转换 `toLlmContent()` |
| [oauth/](../packages/mcp/src/oauth/) | MCP OAuth 实现 |
| [testing/](../packages/mcp/src/testing/) | 内存传输对，便于测试 |

## 3. 快速使用

```ts
import { McpClient, StdioTransport } from "@earendil-works/pi-mcp";

const transport = new StdioTransport({
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
});

const client = new McpClient({
  name: "my-client",
  version: "1.0.0",
  roots: [{ uri: "file:///workspace", name: "workspace" }],
});

await client.connect(transport);
const tools = await client.listTools();
const result = await client.callTool("search", { query: "MCP" });
await client.close();
```

HTTP：

```ts
import { StreamableHttpTransport } from "@earendil-works/pi-mcp";
const transport = new StreamableHttpTransport({ url, headers });
```

## 4. McpClient API

| 方法 | 说明 |
|---|---|
| `connect(transport)` | 发送 `initialize`、协商版本、发送 `notifications/initialized` |
| `request(method, params?, options?)` | 泛化请求，支持 `signal`、`timeoutMs`、`onProgress` |
| `notify(method, params?)` | 发送通知 |
| `ping(options?)` | 心跳 |
| `listTools(options?)` | 分页拉取全部工具 |
| `listResources(options?)` / `listResourcesPage()` | 资源列表/单页 |
| `listResourceTemplates()` / `listResourceTemplatesPage()` | 资源模板 |
| `readResource(uri, options?)` | 读取资源 |
| `callTool(name, args, options?)` | 调用工具；支持进度与取消 |
| `setRequestHandler(method, handler)` | 处理服务端请求（默认处理 `ping`、`roots/list`） |
| `onNotification(method, listener)` | 订阅通知 |
| `onError(listener)` / `onClose(listener)` | 错误/关闭 |
| `close()` | 按协议关闭（stdio：关 stdin → SIGTERM → SIGKILL，作用于进程组） |

`McpRequestOptions`：`signal`、`timeoutMs`、`onProgress`。进度通知会续期超时。

## 5. 协议支持

- 协议版本 `2025-11-25`，兼容 `2025-06-18`、`2025-03-26`、`2024-11-05`。
- 初始化、`notifications/initialized`、ping。
- 分页 `tools/list`、`tools/call`（含 `structuredContent`）。
- 进度通知、超时续期、请求取消。
- Streamable HTTP：会话、服务端 → 客户端 GET 流、重连、`Last-Event-ID` 续传。
- 服务端请求：`ping`、`roots/list`。
- 日志、工具列表变更通知。
- OAuth：protected-resource/authorization-server 发现、PKCE、动态注册、token 刷新、`insufficient_scope` step-up。

不支持：批量 JSON-RPC、旧 HTTP+SSE、服务端模式、sampling、tasks。

## 6. 内容转换

`toLlmContent(result)` 把 `CallToolResult` 转成 `pi-ai` 的 `TextContent | ImageContent[]`：

- 文本、图片直接通过。
- 内嵌文本/图片资源解包。
- 音频、资源链接、二进制资源转成简短文本占位。
- 只有 `structuredContent` 的结果转为 JSON 文本。

把 MCP 工具包装为 `AgentTool`：

```ts
const tools: AgentTool[] = (await client.listTools()).map((tool) => ({
  name: `mcp_${tool.name}`.replace(/[^A-Za-z0-9_-]/g, "_").slice(0, 64),
  label: tool.title ?? tool.name,
  description: tool.description ?? tool.name,
  parameters: Type.Unsafe({ ...tool.inputSchema, type: "object", properties: tool.inputSchema.properties ?? {} }),
  execute: async (_id, params, signal) => {
    const result = await client.callTool(tool.name, params as Record<string, unknown>, { signal });
    return { content: toLlmContent(result), details: undefined, isError: result.isError === true };
  },
}));
```

## 7. OAuth

`@earendil-works/pi-mcp/oauth` 提供：

- `McpOAuthProvider`
- `adaptOAuthProvider()`
- `authorizeMcp()`
- `OAuthCallbackServer`
- `McpOAuthAuthorizationRequiredError`
- `McpOAuthStateStore`（可注入持久化）

包本身不打开浏览器，也不决定凭据存储位置。

## 8. 在 Pi 中的使用

coding-agent 的 `mcp` 内置扩展：

- 读取 `<agent-dir>/mcp.json` 与 `.pi/mcp.json`。
- 支持 stdio 与 HTTP server。
- 按 `exposure` 注册工具：`direct`、`codemode`、`deferred`。
- 连接/重连、OAuth、资源、日志、工具列表变更。
- MCP 工具名加 `mcp__` 前缀，避免与内置工具冲突。

## 9. 依赖与测试

依赖：`cross-spawn`。无内部依赖。

```bash
cd packages/mcp
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/client.test.ts
```

CI 另运行官方 MCP conformance 套件（`npm run test:mcp-conformance`），并与 baseline 比较。
