# Transport Comparison

> MCP has two active transports and one deprecated transport. Choose based on where your server runs and who needs to reach it.

---

## Decision: Which Transport Do I Use?

```
Is your server running on the same machine as the client?
├── YES → Use stdio
└── NO  → Use Streamable HTTP

Is your server a legacy server using the old HTTP+SSE pattern?
└── YES → Plan migration to Streamable HTTP; support both during transition
```

---

## Comparison Table

| Dimension | stdio | Streamable HTTP | HTTP+SSE (deprecated) |
|-----------|-------|----------------|----------------------|
| **Status** | Active | Active | Deprecated (March 2025) |
| **Location** | Local only | Remote + local | Remote + local |
| **Protocol** | Pipe (stdin/stdout) | HTTP POST + SSE | GET (SSE) + POST |
| **Multiple clients** | No (1:1 process) | Yes | Yes |
| **Horizontal scaling** | No | Yes (stateless) | No (stateful SSE) |
| **Authentication** | Process-level (OS) | OAuth 2.1 / Bearer | OAuth 2.1 / Bearer |
| **Session management** | Process lifetime | `Mcp-Session-Id` header | SSE connection |
| **Latency** | ~0ms (IPC) | HTTP round-trip | HTTP + SSE overhead |
| **Connection setup** | Subprocess launch | HTTP request | SSE + POST endpoint |
| **Firewall/NAT** | Not applicable | Needs open port | Needs open port |
| **Reverse proxy** | Not applicable | nginx / Cloudflare | Requires sticky sessions |
| **Best for** | Claude Code, Cursor, Claude Desktop | Remote APIs, shared servers | Legacy servers only |

---

## stdio Transport

### How it works

The client launches the server as a child process. Messages are newline-delimited JSON on stdin/stdout. Stderr is available for logging.

```
Claude Code
    │
    ├── spawn: python server.py
    │          ↕ stdin/stdout (JSON lines)
    └── server.py (your MCP server)
```

### Setup — Python (FastMCP)

```python
from fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run()  # defaults to stdio
```

### Setup — TypeScript

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({ name: "my-server", version: "1.0.0" });

server.registerTool("hello", {
  description: "Say hello",
  inputSchema: { name: { type: z.string() } },
}, async ({ name }) => ({
  content: [{ type: "text", text: `Hello, ${name}!` }]
}));

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Claude Code config (`~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

### Claude Desktop config (`claude_desktop_config.json`)

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/path/to/server.py"]
    }
  }
}
```

### Key constraints

- Server process is launched fresh per host startup; not per conversation
- Server exits when the host closes or the client disconnects
- Only one client can connect (the process that spawned it)
- Write logs to **stderr only** — stdout is reserved for MCP messages
- No authentication needed; OS process isolation is sufficient

---

## Streamable HTTP Transport

### How it works

The server runs as an independent HTTP service. Clients POST requests to `/mcp`. Server can respond with JSON (single response) or SSE stream (streaming response). A GET to `/mcp` opens a server-push SSE stream for server-initiated messages.

```
Claude Code / Any MCP Client
    │
    ├── POST https://your-server.com/mcp   ← requests
    │◄── SSE stream or JSON response       ← responses + notifications
    │
    └── GET https://your-server.com/mcp    ← server-initiated messages
        ◄── SSE stream
```

### Setup — Python (FastMCP)

```python
from fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
    # Server available at http://localhost:8000/mcp
```

### Setup — TypeScript (with Express)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";
import { randomUUID } from "crypto";

const app = express();
app.use(express.json());

const server = new McpServer({ name: "my-server", version: "1.0.0" });
// ... register tools/resources/prompts ...

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: () => randomUUID(),
  });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.get("/mcp", async (req, res) => {
  // Handle server-sent events stream
});

app.listen(8000);
```

### Client config (Claude Code)

```json
{
  "mcpServers": {
    "my-remote-server": {
      "url": "https://your-server.com/mcp",
      "headers": {
        "Authorization": "Bearer your-token"
      }
    }
  }
}
```

### Required HTTP headers

| Header | Direction | Value |
|--------|-----------|-------|
| `Accept` | client → server | `application/json, text/event-stream` |
| `Content-Type` | client → server | `application/json` |
| `MCP-Protocol-Version` | client → server | `2025-11-25` |
| `Mcp-Session-Id` | client → server | Session ID from initialize response |
| `Origin` | client → server | Must be validated by server (DNS rebinding protection) |

---

## Deprecated: HTTP+SSE Transport

> **Status: Deprecated since March 2025.** Still present in many production servers built before mid-2025. Understand it for debugging legacy servers; do not build new servers with it.

### How it differed

- Client opens a persistent **GET** connection to receive SSE events
- Server sends an `endpoint` event on the SSE stream with a URL for client-to-server POST
- Client sends all requests to that POST URL
- This required **sticky sessions** behind load balancers (SSE connection had to reach the same instance)

### Migration to Streamable HTTP

```python
# Old pattern (SSE) — do not use for new servers
mcp.run(transport="sse", host="0.0.0.0", port=8000)

# New pattern (Streamable HTTP)
mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
```

The server exposes the same `/mcp` endpoint. Clients that use the old SSE pattern will attempt a GET first and get the SSE endpoint event; clients that use Streamable HTTP will POST to initialize directly. Both can coexist on the same server during migration.

---

## Performance Characteristics

| Scenario | stdio | Streamable HTTP |
|----------|-------|----------------|
| First request latency | ~0ms (IPC) | 1–5ms (local network) to 50–200ms (remote) |
| Subsequent request latency | ~0ms | ~1–5ms (local) |
| Concurrent clients | 1 | Unlimited |
| Horizontal scaling | Not applicable | Load balancer + stateless instances |
| Memory per connection | Process memory | HTTP keep-alive pool |

---

*Specification: [MCP Transports](https://modelcontextprotocol.io/docs/concepts/transports) | Verified: 2026-06-30*
