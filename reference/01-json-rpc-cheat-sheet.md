# JSON-RPC 2.0 Cheat Sheet

> JSON-RPC 2.0 is the wire format for all MCP communication. Every message you send or receive is one of four types: Request, Response, Error, or Notification.

---

## The Four Message Types

### Request (client → server, expects a reply)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "London" }
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `jsonrpc` | `"2.0"` | Yes | Always exactly the string `"2.0"` |
| `id` | string \| number \| null | Yes | Must be unique per in-flight request |
| `method` | string | Yes | The operation name |
| `params` | object \| array | No | Omit if method takes no parameters |

---

### Response (server → client, success)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "Temperature: 18°C" }],
    "isError": false
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | matches request id | Links response to the original request |
| `result` | any | The method's return value; mutually exclusive with `error` |

---

### Error Response (server → client, failure)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Unknown tool: invalid_tool_name",
    "data": { "tool": "invalid_tool_name" }
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `code` | integer | See error codes reference |
| `message` | string | Human-readable description |
| `data` | any | Optional additional context |

---

### Notification (either direction, no reply)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

No `id` field — notifications never receive a response.

---

## Rules That Catch Beginners

| Rule | Wrong | Right |
|------|-------|-------|
| Version string | `"jsonrpc": 2` (number) | `"jsonrpc": "2.0"` (string) |
| ID types | `"id": { "key": 1 }` (object) | `"id": 1` or `"id": "abc"` |
| No reply needed | Include `id` anyway | Omit `id` entirely for notifications |
| Null result | `"result": {}` | `"result": null` is valid |
| Error vs result | Include both fields | `result` and `error` are mutually exclusive |
| Embedded newlines | Multi-line JSON in stdio | Single-line JSON terminated with `\n` |

---

## Batch Requests

Send multiple requests in one HTTP call:

```json
[
  { "jsonrpc": "2.0", "id": 1, "method": "tools/list" },
  { "jsonrpc": "2.0", "id": 2, "method": "resources/list" }
]
```

Responses return as an array, in any order. Match by `id`.

> **MCP Note:** Batch requests are defined by JSON-RPC 2.0 but are rarely used in MCP implementations. Most MCP transports handle each request individually.

---

## Message Framing by Transport

### stdio
- One JSON object per line
- Terminated by `\n`
- **No embedded newlines** inside the JSON
- Read `stdin` line by line; write to `stdout` line by line

```
{"jsonrpc":"2.0","id":1,"method":"tools/list"}\n
{"jsonrpc":"2.0","id":1,"result":{"tools":[...]}}\n
```

### Streamable HTTP
- Client sends requests via HTTP POST to the `/mcp` endpoint
- Server responds with either:
  - `Content-Type: application/json` for single responses
  - `Content-Type: text/event-stream` (SSE) for streaming
- SSE format: `data: {"jsonrpc":"2.0",...}\n\n`

---

## ID Assignment Rules

- IDs must be unique **per in-flight request** (not globally unique forever)
- Once a response arrives, that ID can be reused
- Use integers (1, 2, 3...) or UUIDs — both are fine
- `null` is technically allowed but creates ambiguous responses; avoid it

---

## Quick Debugging Checklist

When a JSON-RPC message fails to parse or process:

1. Is `"jsonrpc": "2.0"` present as a string, not a number?
2. Does the request have an `id` field (not a notification)?
3. Is the JSON on a single line (stdio transport)?
4. Are `result` and `error` mutually exclusive in the response?
5. Does the response `id` match the request `id`?
6. Is the `params` field an object (not an array) for MCP methods?

---

*Specification: [JSON-RPC 2.0](https://www.jsonrpc.org/specification) | Verified: 2026-06-30*
