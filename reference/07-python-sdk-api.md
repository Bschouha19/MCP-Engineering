# MCP Python SDK API Reference (Raw SDK)

> The raw MCP Python SDK gives you full protocol control — no abstractions. Use this when you need behaviour that FastMCP doesn't expose, or when you want to understand what FastMCP is doing underneath.

> **FastMCP vs raw SDK:** For most servers, use FastMCP (see [06-fastmcp-api.md](./06-fastmcp-api.md)). Use the raw SDK when you need: custom capability negotiation, non-standard message handling, or maximum performance in high-throughput scenarios (~30-40% faster than FastMCP in benchmarks).

> **Note:** The raw Python SDK v1.x is current stable. v2 is pre-release (expected Q3 2026). Verify import paths against [py.sdk.modelcontextprotocol.io](https://py.sdk.modelcontextprotocol.io) if breaking changes have landed.

---

## Installation

```bash
uv add mcp          # Installs the raw MCP Python SDK
# or
pip install mcp
```

---

## Low-Level Server

```python
from mcp.server.lowlevel import Server
from mcp import types
import mcp.server.stdio as stdio_server_module

# Create server
server = Server("my-server")
```

### Register tool handlers

```python
@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_weather",
            description="Get current weather for a city",
            inputSchema={
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name"
                    }
                },
                "required": ["location"],
            },
        )
    ]

@server.call_tool()
async def call_tool(
    name: str,
    arguments: dict,
) -> list[types.TextContent | types.ImageContent | types.EmbeddedResource]:
    if name == "get_weather":
        location = arguments["location"]
        return [types.TextContent(type="text", text=f"Weather in {location}: 18°C")]
    raise ValueError(f"Unknown tool: {name}")
```

### Register resource handlers

```python
@server.list_resources()
async def list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="config://settings",
            name="Application Settings",
            description="Read-only application configuration",
            mimeType="application/json",
        )
    ]

@server.read_resource()
async def read_resource(uri: str) -> str | bytes:
    if uri == "config://settings":
        return json.dumps({"debug": False, "version": "1.0"})
    raise ValueError(f"Unknown resource: {uri}")
```

### Register prompt handlers

```python
@server.list_prompts()
async def list_prompts() -> list[types.Prompt]:
    return [
        types.Prompt(
            name="code_review",
            description="Review code for issues",
            arguments=[
                types.PromptArgument(
                    name="code",
                    description="The code to review",
                    required=True,
                ),
                types.PromptArgument(
                    name="language",
                    description="Programming language",
                    required=False,
                ),
            ],
        )
    ]

@server.get_prompt()
async def get_prompt(
    name: str,
    arguments: dict | None,
) -> types.GetPromptResult:
    if name == "code_review":
        code = (arguments or {}).get("code", "")
        lang = (arguments or {}).get("language", "")
        return types.GetPromptResult(
            messages=[
                types.PromptMessage(
                    role="user",
                    content=types.TextContent(
                        type="text",
                        text=f"Review this {lang} code:\n\n{code}",
                    ),
                )
            ]
        )
    raise ValueError(f"Unknown prompt: {name}")
```

---

## Running with stdio Transport

```python
import asyncio
from mcp.server.lowlevel import Server
from mcp.server.stdio import stdio_server

async def main():
    server = Server("my-server")
    # ... register handlers ...
    
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Running with Streamable HTTP (FastAPI integration)

```python
from fastapi import FastAPI, Request, Response
from mcp.server.lowlevel import Server
from mcp.server.streamable_http import StreamableHTTPServerTransport
import uvicorn

server = Server("my-server")
# ... register handlers ...

app = FastAPI()

@app.post("/mcp")
async def mcp_endpoint(request: Request):
    body = await request.json()
    transport = StreamableHTTPServerTransport(
        mcp_session_id=request.headers.get("Mcp-Session-Id"),
    )
    async with transport.connect() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )
    return transport.response

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## MCP Types Reference

```python
from mcp import types

# Tool input/output
types.Tool(name, description, inputSchema)
types.CallToolResult(content, isError=False)
types.TextContent(type="text", text)
types.ImageContent(type="image", data, mimeType)
types.EmbeddedResource(type="resource", resource)

# Resources
types.Resource(uri, name, description, mimeType)
types.ResourceContents(uri, mimeType, text)          # text resource
types.BlobResourceContents(uri, mimeType, blob)      # binary resource

# Prompts
types.Prompt(name, description, arguments)
types.PromptArgument(name, description, required)
types.PromptMessage(role, content)
types.GetPromptResult(messages, description)

# Errors
types.McpError(code, message, data)

# Capabilities
types.ServerCapabilities(tools, resources, prompts, logging)
types.ClientCapabilities(roots, sampling)
```

---

## Client API

```python
from mcp import ClientSession
from mcp.client.stdio import stdio_client
from mcp.client.streamable_http import streamablehttp_client

# Connect via stdio (spawn subprocess)
async with stdio_client(
    command="python",
    args=["server.py"],
    env={"API_KEY": "..."},
) as (read_stream, write_stream):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()
        
        # List tools
        tools_result = await session.list_tools()
        
        # Call a tool
        result = await session.call_tool("tool_name", {"key": "value"})
        
        # List resources
        resources_result = await session.list_resources()
        
        # Read a resource
        content = await session.read_resource("resource://uri")
        
        # Get a prompt
        prompt = await session.get_prompt("prompt_name", {"arg": "value"})

# Connect via Streamable HTTP
async with streamablehttp_client("http://localhost:8000/mcp") as (read, write, _):
    async with ClientSession(read, write) as session:
        await session.initialize()
        result = await session.call_tool("tool_name", {"key": "value"})
```

---

## Full Low-Level Server Example

```python
import asyncio
import json
from mcp.server.lowlevel import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("example-server")

DATABASE = {
    "users": [
        {"id": "1", "name": "Alice", "email": "alice@example.com"},
        {"id": "2", "name": "Bob",   "email": "bob@example.com"},
    ]
}

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_user",
            description="Get a user by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string", "description": "User ID"}
                },
                "required": ["user_id"],
            },
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_user":
        user_id = arguments["user_id"]
        users = [u for u in DATABASE["users"] if u["id"] == user_id]
        if not users:
            return [types.TextContent(type="text", text=f"User {user_id} not found")]
        return [types.TextContent(type="text", text=json.dumps(users[0]))]
    raise ValueError(f"Unknown tool: {name}")

@server.list_resources()
async def list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="db://users/schema",
            name="Users Schema",
            mimeType="application/json",
        )
    ]

@server.read_resource()
async def read_resource(uri: str) -> str:
    if uri == "db://users/schema":
        return json.dumps({
            "table": "users",
            "columns": ["id", "name", "email"],
        })
    raise ValueError(f"Unknown resource: {uri}")

async def main():
    async with stdio_server() as (read, write):
        await server.run(
            read, write,
            server.create_initialization_options(),
        )

if __name__ == "__main__":
    asyncio.run(main())
```

---

*Source: [py.sdk.modelcontextprotocol.io](https://py.sdk.modelcontextprotocol.io) | [GitHub](https://github.com/modelcontextprotocol/python-sdk) | Verified: 2026-06-30*
