# MCP Specification Cheat Sheet

> Quick reference for the MCP protocol structure, primitives, lifecycle, and spec versions. For wire format details, see [01-json-rpc-cheat-sheet.md](./01-json-rpc-cheat-sheet.md).

---

## Spec Versions

| Version | Status | Notes |
|---------|--------|-------|
| `2024-11-05` | Deprecated | Introduced HTTP+SSE transport (now replaced) |
| `2025-03-26` | Superseded | SSE transport deprecated; OAuth 2.1 added |
| `2025-11-25` | **Current stable** | Primary teaching target for this course |
| `2026-07-28` | Release Candidate | Adds Tasks, Multi-Round-Trip, MCP Apps, Trace Context |

Always send the current protocol version in the HTTP header:
```
MCP-Protocol-Version: 2025-11-25
```

---

## The Three-Actor Model

```
Host Application          MCP Client              MCP Server
(Claude Desktop,          (embedded in            (your code)
 Cursor, Claude Code)     the host)
       │                       │                       │
       │   manages lifecycle   │                       │
       ├──────────────────────►│                       │
       │                       │   JSON-RPC messages   │
       │                       ├──────────────────────►│
       │                       │◄──────────────────────┤
```

- **Host** — the application the user runs (Claude Desktop, Cursor, VS Code, your custom app)
- **Client** — lives inside the host; manages the protocol connection to one server
- **Server** — your MCP server; exposes capabilities

One client connects to **one** server. The host manages multiple clients simultaneously.

---

## Server Primitives (what servers expose)

| Primitive | Who decides to call it | Use case |
|-----------|----------------------|----------|
| **Tool** | The model (AI-controlled) | Actions — write file, call API, run query |
| **Resource** | The user or host (human-controlled) | Context — read file, get schema, browse data |
| **Prompt** | The user or host (human-controlled) | Templates — reusable multi-step instructions |

---

## Client Features (what clients offer, 2025-11-25)

| Feature | What it does | Status |
|---------|-------------|--------|
| **Sampling** | Server requests LLM inference from the client | Deprecated in 2026-07 RC |
| **Roots** | Client declares accessible filesystem paths | Deprecated in 2026-07 RC |
| **Elicitation** | Server pauses to request input from the user | Being replaced in 2026-07 RC |

> **Note:** These features exist in thousands of production servers. Understand them — they will appear in real codebases.

---

## Initialization Lifecycle

```
Client                          Server
  │                               │
  ├── initialize ────────────────►│  {protocolVersion, capabilities, clientInfo}
  │◄──────────────── initialize ──┤  {protocolVersion, capabilities, serverInfo}
  │                               │
  ├── notifications/initialized ─►│  (no response)
  │                               │
  │   [normal operation]          │
  │                               │
  ├── DELETE /mcp (or EOF) ──────►│  session terminated
```

### Initialize request params

```json
{
  "protocolVersion": "2025-11-25",
  "capabilities": {
    "roots": { "listChanged": true },
    "sampling": {}
  },
  "clientInfo": {
    "name": "my-client",
    "version": "1.0.0"
  }
}
```

### Initialize response result

```json
{
  "protocolVersion": "2025-11-25",
  "capabilities": {
    "tools": { "listChanged": true },
    "resources": { "subscribe": true, "listChanged": true },
    "prompts": { "listChanged": true },
    "logging": {}
  },
  "serverInfo": {
    "name": "my-server",
    "version": "1.0.0"
  }
}
```

---

## All MCP Methods (2025-11-25)

### Lifecycle
| Method | Direction | Notes |
|--------|-----------|-------|
| `initialize` | client → server | First message; negotiates capabilities |
| `notifications/initialized` | client → server | Confirms init complete (notification) |
| `ping` | either | Keepalive check |

### Tools
| Method | Direction |
|--------|-----------|
| `tools/list` | client → server |
| `tools/call` | client → server |
| `notifications/tools/list_changed` | server → client (notification) |

### Resources
| Method | Direction |
|--------|-----------|
| `resources/list` | client → server |
| `resources/read` | client → server |
| `resources/subscribe` | client → server |
| `resources/unsubscribe` | client → server |
| `notifications/resources/list_changed` | server → client (notification) |
| `notifications/resources/updated` | server → client (notification) |

### Prompts
| Method | Direction |
|--------|-----------|
| `prompts/list` | client → server |
| `prompts/get` | client → server |
| `notifications/prompts/list_changed` | server → client (notification) |

### Client Features
| Method | Direction |
|--------|-----------|
| `sampling/createMessage` | server → client |
| `roots/list` | server → client |
| `elicitation/create` | server → client |

### Utilities
| Method | Direction |
|--------|-----------|
| `logging/setLevel` | client → server |
| `notifications/message` | server → client (log notification) |
| `notifications/progress` | server → client (notification) |
| `notifications/cancelled` | either (notification) |
| `completion/complete` | client → server |

---

## Tool Schema Structure

```json
{
  "name": "get_weather",
  "title": "Get Weather",
  "description": "Get current weather for a city",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name or zip code"
      }
    },
    "required": ["location"]
  },
  "annotations": {
    "readOnlyHint": true,
    "destructiveHint": false,
    "idempotentHint": true,
    "openWorldHint": true
  }
}
```

### Tool Annotations

| Annotation | Meaning | Default |
|-----------|---------|---------|
| `readOnlyHint` | Does not modify state | `false` |
| `destructiveHint` | May cause irreversible changes | `true` |
| `idempotentHint` | Calling twice = calling once | `false` |
| `openWorldHint` | Reaches external systems | `true` |
| `title` | Human-readable display name | — |

---

## Tool Result Content Types

```json
{ "type": "text", "text": "result string" }
{ "type": "image", "data": "<base64>", "mimeType": "image/png" }
{ "type": "audio", "data": "<base64>", "mimeType": "audio/wav" }
{ "type": "resource_link", "uri": "file:///path", "name": "filename", "mimeType": "text/plain" }
{ "type": "resource", "resource": { "uri": "...", "text": "...", "mimeType": "..." } }
```

For tool execution errors, return a result with `"isError": true`:
```json
{
  "content": [{ "type": "text", "text": "Error: database unavailable" }],
  "isError": true
}
```

---

## Resource URI Schemes (common)

| Scheme | Example | Use case |
|--------|---------|----------|
| `file://` | `file:///home/user/docs/readme.md` | Local filesystem |
| `db://` | `db://mydb/users/schema` | Database metadata |
| `github://` | `github://owner/repo/issues/123` | GitHub content |
| Custom | `weather://london/current` | Domain-specific |

---

## Capability Flags Quick Reference

Declare only the capabilities your server/client actually supports:

```json
{
  "tools": { "listChanged": true },      // server: notify when tool list changes
  "resources": {
    "subscribe": true,                    // server: supports resource subscriptions
    "listChanged": true                   // server: notify when resource list changes
  },
  "prompts": { "listChanged": true },    // server: notify when prompt list changes
  "logging": {}                           // server: accepts logging level changes
}
```

---

*Specification: [MCP 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) | Verified: 2026-06-30*
