# MCP Error Codes

> Complete reference for JSON-RPC 2.0 error codes and MCP-specific codes. When debugging, start by matching the numeric code against this table.

---

## Error Object Structure

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

- `code` — numeric; see tables below
- `message` — human-readable string; always present
- `data` — optional additional context; can be any JSON value

---

## Standard JSON-RPC 2.0 Codes

| Code | Name | When It Occurs |
|------|------|----------------|
| `-32700` | **Parse Error** | Server received malformed JSON that could not be parsed |
| `-32600` | **Invalid Request** | JSON is valid but not a valid JSON-RPC request object (missing `jsonrpc`, `method`, etc.) |
| `-32601` | **Method Not Found** | The method does not exist or the server has not declared the required capability |
| `-32602` | **Invalid Params** | Method exists but parameters are malformed, wrong type, or a named resource/tool/prompt is unknown |
| `-32603` | **Internal Error** | Server encountered an unexpected error while processing the request |

### Reserved ranges

| Range | Owner |
|-------|-------|
| `-32768` to `-32000` | Reserved by JSON-RPC 2.0 for protocol-level errors |
| `-32099` to `-32020` | Reserved by MCP specification (see below) |
| `-32019` to `-32000` | Implementation-defined server errors |

---

## MCP-Specific Error Codes

These codes are defined by the MCP specification. Implementations MUST NOT use these codes with meanings other than those listed.

| Code | Name | When It Occurs |
|------|------|----------------|
| `-32002` | **Resource Not Found** | Legacy code (2025-11-25 and earlier). Returned when a resource URI cannot be resolved. Newer spec versions use `-32602` (Invalid Params) instead. Still seen in production servers. |
| `-32020` | **Header Mismatch** | Streamable HTTP: required HTTP headers are missing, mismatched, or malformed (e.g., invalid `Mcp-Session-Id`, missing `Origin`, wrong `MCP-Protocol-Version`) |
| `-32021` | **Missing Required Client Capability** | Server requires a client capability (e.g., `sampling`, `roots`) that the client did not declare during initialization |
| `-32022` | **Unsupported Protocol Version** | Client requested a protocol version the server does not support |
| `-32042` | **URL Elicitation Required** | Server operation requires user input via URL-mode elicitation before it can proceed (2025-11-25 only; being replaced by Multi-Round-Trip in 2026-07 RC) |

---

## Tool Execution Errors vs Protocol Errors

This is one of the most important distinctions in MCP.

### Protocol Error — use `error` field

The request itself was malformed. The model cannot recover by adjusting its inputs.

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32601,
    "message": "Method not found: tools/execute"
  }
}
```

Use for: unknown method, missing capability, malformed JSON, bad parameters.

### Tool Execution Error — use `result` with `isError: true`

The tool ran but encountered a business logic failure. The model **can** potentially recover by retrying with different parameters.

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: departure date '2024-01-01' is in the past. Please provide a future date."
      }
    ],
    "isError": true
  }
}
```

Use for: API errors, invalid business logic inputs, database errors, rate limit errors.

> **Rule of thumb:** If the model might retry with different inputs, use `isError: true` in the result. If the request structure itself is wrong, use the `error` field.

---

## Common Error Scenarios and Codes

| Scenario | Expected Code | Notes |
|----------|--------------|-------|
| Server doesn't support `resources/list` | `-32601` | Client checked `tools` capability only |
| Tool name doesn't exist | `-32602` | "Unknown tool" — params were structurally valid |
| Missing required tool argument | `-32602` | Invalid parameters |
| Request has no `method` field | `-32600` | Not a valid request object |
| Malformed JSON body | `-32700` | Parse error before method is known |
| `Mcp-Session-Id` is expired | `-32020` or HTTP 404 | Session expired; client must re-initialize |
| Tool threw an exception | `isError: true` in result | Not a protocol error |
| Database connection failed | `isError: true` in result | Actionable: model may retry |
| Invalid auth token | HTTP 401 before JSON-RPC | Caught at transport layer, not JSON-RPC layer |
| Rate limit exceeded | HTTP 429 before JSON-RPC OR `isError: true` | Caught at transport or returned as tool result |

---

## Error Handling in FastMCP

```python
from fastmcp import FastMCP
from fastmcp.exceptions import ToolError, ResourceError

mcp = FastMCP("my-server", mask_error_details=False)

@mcp.tool
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ToolError("Division by zero — adjust denominator")  # isError: true in result
    return a / b

@mcp.resource("data://items/{id}")
def get_item(id: str) -> str:
    if id not in database:
        raise ResourceError(f"Item {id} not found")  # → -32602 Invalid Params
    return database[id]
```

## Error Handling in TypeScript SDK

```typescript
import { McpError, ErrorCode } from "@modelcontextprotocol/sdk/types.js";

server.registerTool("divide", {
  description: "Divide two numbers",
  inputSchema: { a: z.number(), b: z.number() },
}, async ({ a, b }) => {
  if (b === 0) {
    // Tool execution error — model can recover
    return {
      content: [{ type: "text", text: "Error: b cannot be zero" }],
      isError: true,
    };
  }
  return { content: [{ type: "text", text: String(a / b) }] };
});

// Protocol error — something structurally wrong
throw new McpError(ErrorCode.InvalidParams, "Parameter 'a' must be positive");
```

---

## Debugging with Error Codes

```
Received error code...
├── -32700 → Check JSON syntax; look for embedded newlines (stdio) or encoding issues
├── -32600 → Check for missing jsonrpc/method fields; confirm it's not a notification
├── -32601 → Confirm server declared capability; check method name spelling exactly
├── -32602 → Check tool/resource/prompt name exists; validate parameter types
├── -32603 → Server-side bug; check server logs
├── -32002 → Legacy server: resource URI not found (treat as 404)
├── -32020 → Check HTTP headers: Mcp-Session-Id, MCP-Protocol-Version, Origin
├── -32021 → Client didn't declare required capability during initialize
├── -32022 → Protocol version mismatch; check both sides' supported versions
└── -32042 → Server requires user interaction; handle elicitation flow
```

---

*Sources: [MCP C# SDK Error Codes](https://csharp.sdk.modelcontextprotocol.io/api/ModelContextProtocol.McpErrorCode.html) | [JSON-RPC 2.0 Spec](https://www.jsonrpc.org/specification) | [MCP Spec 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) | Verified: 2026-06-30*
