# FastMCP API Reference

> FastMCP 3.0 (Python) — the framework behind ~70% of all MCP servers. This reference covers the complete public API. For the raw MCP Python SDK, see [07-python-sdk-api.md](./07-python-sdk-api.md).

> **Note:** Verified against FastMCP 3.0 as of June 2026. FastMCP evolves rapidly — confirm against [gofastmcp.com](https://gofastmcp.com) for latest API details.

---

## Installation

```bash
uv add fastmcp          # Python 3.10+
# or
pip install fastmcp
```

---

## FastMCP — Server Class

```python
from fastmcp import FastMCP

mcp = FastMCP(
    name: str,                              # Server name (required)
    instructions: str | None = None,        # System-level LLM guidance
    on_duplicate_tools: str = "warn",       # "warn" | "error" | "replace" | "ignore"
    on_duplicate_resources: str = "warn",   # Same options as above
    on_duplicate_prompts: str = "warn",     # Same options as above
    mask_error_details: bool = False,       # Hide internal error details from clients
    dereference_schemas: bool = True,       # Inline JSON schema $refs
    strict_input_validation: bool = False,  # Strict Pydantic validation
)
```

### Run the server

```python
# stdio (default) — for Claude Code, Claude Desktop, Cursor
mcp.run()
mcp.run(transport="stdio")

# Streamable HTTP — for remote/shared servers
mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
# Available at http://localhost:8000/mcp

# SSE (deprecated — legacy support only)
mcp.run(transport="sse", host="0.0.0.0", port=8000)

# Dev mode with auto-reload
# uv run mcp dev server.py --reload
```

### Lifespan hooks

```python
from contextlib import asynccontextmanager
from fastmcp import FastMCP
import asyncpg

@asynccontextmanager
async def lifespan(server: FastMCP):
    # Startup
    server.state.db_pool = await asyncpg.create_pool(dsn=DATABASE_URL)
    yield
    # Shutdown
    await server.state.db_pool.close()

mcp = FastMCP("my-server", lifespan=lifespan)
```

### Component visibility control

```python
mcp.disable(keys={"tool:admin_action"})    # Disable by key
mcp.disable(tags={"admin"})                # Disable all tagged "admin"
mcp.enable(tags={"public"}, only=True)     # Enable only "public" tagged components
```

---

## @mcp.tool — Tool Decorator

```python
@mcp.tool(
    name: str | None = None,               # Defaults to function name
    description: str | None = None,        # Defaults to docstring
    tags: set[str] | None = None,          # Categorization labels
    annotations: ToolAnnotations | dict | None = None,  # MCP annotations
    timeout: float | None = None,          # Execution limit in seconds
    version: str | int | None = None,      # Tool version
    output_schema: dict | None = None,     # Custom JSON Schema for structured output
    run_in_thread: bool = True,            # False: run sync functions on event loop
    meta: dict | None = None,              # Custom metadata passed to clients
)
def tool_function(...) -> ReturnType: ...
```

### Parameter types — what FastMCP accepts

```python
# Primitives
def tool(a: int, b: str, c: float, d: bool) -> str: ...

# Optional and defaults
def tool(opt: str | None = None, count: int = 10) -> str: ...

# Pydantic constraints
from typing import Annotated
from pydantic import Field

def tool(
    age: Annotated[int, Field(ge=0, le=150)],
    name: Annotated[str, "User's full name"],
) -> str: ...

# Pydantic models as input
from pydantic import BaseModel

class SearchQuery(BaseModel):
    query: str
    max_results: int = 10

def tool(search: SearchQuery) -> list[str]: ...

# Dependency injection (hidden from LLM)
from fastmcp.dependencies import Depends

def get_user_id() -> str: return "user_123"

def tool(user_id: str = Depends(get_user_id)) -> str: ...
```

### Return types — what FastMCP converts

```python
# String → TextContent
def tool() -> str: return "text"

# Dict / dataclass / Pydantic model → structured output
def tool() -> dict: return {"key": "value"}

# Image / Audio / File helpers
from fastmcp.utilities.types import Image, Audio, File

def tool() -> Image: return Image(path="chart.png")
def tool() -> Audio: return Audio(data=bytes_data, format="wav")
def tool() -> File: return File(path="report.pdf")

# List of content
def tool() -> list[Image]: return [Image(path="1.png"), Image(path="2.png")]

# Full control
from fastmcp.tools.tool import ToolResult
from mcp.types import TextContent

def tool() -> ToolResult:
    return ToolResult(
        content=[TextContent(type="text", text="Summary")],
        structured_content={"data": "value"},
        meta={"execution_time_ms": 145},
    )
```

### Tool annotations

```python
from mcp.types import ToolAnnotations

@mcp.tool(annotations=ToolAnnotations(
    title="Human Display Name",
    readOnlyHint=True,      # Does not modify any state
    destructiveHint=False,  # Not irreversible
    idempotentHint=True,    # Calling N times = calling once
    openWorldHint=False,    # No external system calls
))
def my_tool() -> str: ...
```

### Error handling

```python
from fastmcp.exceptions import ToolError

@mcp.tool
def safe_divide(a: float, b: float) -> float:
    if b == 0:
        raise ToolError("Cannot divide by zero")  # → isError: true in result
    return a / b
```

---

## @mcp.resource — Resource Decorator

```python
@mcp.resource(
    uri: str,                               # URI or URI template (required)
    name: str | None = None,               # Defaults to function name
    description: str | None = None,        # Defaults to docstring
    mime_type: str | None = None,          # e.g., "application/json"
    tags: set[str] | None = None,
    annotations: dict | None = None,
    version: str | int | None = None,
    meta: dict | None = None,
)
def resource_function(...) -> str | bytes: ...
```

### Static resource

```python
@mcp.resource("config://app/settings")
def get_settings() -> str:
    return json.dumps({"debug": False, "version": "1.0.0"})
```

### URI template resource

```python
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    user = db.get_user(user_id)
    return json.dumps(user.dict())
```

### Template with query parameters

```python
@mcp.resource("items://{category}{?limit,offset}")
def list_items(category: str, limit: int = 20, offset: int = 0) -> str:
    items = db.query(category=category, limit=limit, offset=offset)
    return json.dumps(items)
```

### Binary resource

```python
@mcp.resource("chart://{chart_id}", mime_type="image/png")
def get_chart(chart_id: str) -> bytes:
    return generate_chart(chart_id)
```

### Error handling

```python
from fastmcp.exceptions import ResourceError

@mcp.resource("db://{table}/schema")
def get_schema(table: str) -> str:
    if table not in ALLOWED_TABLES:
        raise ResourceError(f"Table '{table}' not found or not accessible")
    return json.dumps(get_table_schema(table))
```

---

## @mcp.prompt — Prompt Decorator

```python
@mcp.prompt(
    name: str | None = None,
    title: str | None = None,              # Display name
    description: str | None = None,
    tags: set[str] | None = None,
    meta: dict | None = None,
    version: str | int | None = None,
)
def prompt_function(...) -> str | list | PromptResult: ...
```

### Single message prompt

```python
@mcp.prompt
def code_review(language: str, code: str) -> str:
    return f"Review this {language} code for bugs and style issues:\n\n```{language}\n{code}\n```"
```

### Multi-message prompt

```python
from fastmcp.prompts import Message

@mcp.prompt
def analysis_conversation(data: str, focus: str = "general") -> list[Message]:
    return [
        Message(f"You are a data analyst focused on {focus} insights."),
        Message("Here is the data to analyze:\n\n" + data),
        Message("Begin with a brief summary, then dive into key findings.", role="user"),
    ]
```

### Prompt with full control

```python
from fastmcp.prompts import PromptResult, Message

@mcp.prompt
def structured_prompt(task: str) -> PromptResult:
    return PromptResult(
        messages=[Message(f"Complete this task: {task}")],
        description="A structured task completion prompt",
        meta={"version": "2.0"},
    )
```

---

## Context Object (ctx: Context)

Inject `ctx: Context` as a parameter to access MCP protocol features:

```python
from fastmcp import Context

@mcp.tool
async def process(data: str, ctx: Context) -> dict:
    # Logging
    await ctx.debug("Starting processing")
    await ctx.info(f"Processing {len(data)} bytes")
    await ctx.warning("Large payload detected")
    await ctx.error("Processing failed")

    # Progress reporting
    await ctx.report_progress(progress=50, total=100)

    # Read a resource from the server
    resource = await ctx.read_resource("config://app/settings")

    # LLM sampling (server asks client to run inference)
    # Note: deprecated in 2026-07 RC
    response = await ctx.sample(
        prompt="Summarize this in one sentence: " + data,
        max_tokens=100,
    )

    # Request metadata
    request_id = ctx.request_id      # Current request ID
    client_id = ctx.client_id        # Connected client identifier

    return {"processed": True}
```

---

## FastMCP Client

```python
from fastmcp import Client

# Connect to a server object directly (in-memory, for testing)
async with Client(mcp_server) as client:
    result = await client.call_tool("tool_name", {"param": "value"})

# Connect to a remote HTTP server
async with Client("http://localhost:8000/mcp") as client:
    tools = await client.list_tools()
    result = await client.call_tool("search", {"query": "python"})

# Connection options
client = Client(
    server_source,           # FastMCP instance | file path | HTTP URL
    env=None,                # Environment variables for subprocess
    auto_initialize=True,    # Auto-handshake on connect
    log_handler=None,        # Callback for server log messages
    progress_handler=None,   # Callback for progress notifications
    sampling_handler=None,   # Callback for sampling requests
    timeout=None,            # Request timeout in seconds
)
```

### Client methods

```python
async with Client(mcp) as client:
    # Tools
    tools = await client.list_tools()           # → list[Tool]
    result = await client.call_tool(            # → CallToolResult
        name="tool_name",
        arguments={"key": "value"},
    )

    # Resources
    resources = await client.list_resources()   # → list[Resource]
    content = await client.read_resource(       # → ReadResourceResult
        uri="resource://path",
    )

    # Prompts
    prompts = await client.list_prompts()       # → list[Prompt]
    prompt = await client.get_prompt(           # → GetPromptResult
        name="prompt_name",
        arguments={"param": "value"},
    )

    # Utilities
    await client.ping()                         # Keepalive check
    info = client.initialize_result            # Server capabilities + info
    connected = client.is_connected()           # Connection status
```

---

## FastMCP CLI

```bash
# Run server with stdio (default)
fastmcp run server.py

# Run server with Streamable HTTP
fastmcp run server.py --transport streamable-http --port 8000

# Dev mode: auto-reload on file changes
fastmcp dev server.py

# Inspect a server with MCP Inspector
fastmcp inspect server.py

# Install into Claude Desktop
fastmcp install server.py --name "My Server"
```

---

## Complete Minimal Server

```python
from fastmcp import FastMCP, Context

mcp = FastMCP("demo-server", instructions="You are a helpful calculator server.")

@mcp.tool(annotations={"readOnlyHint": True, "idempotentHint": True})
async def add(a: float, b: float, ctx: Context) -> float:
    """Add two numbers together."""
    await ctx.info(f"Adding {a} + {b}")
    return a + b

@mcp.resource("math://constants")
def math_constants() -> str:
    """Common mathematical constants."""
    import json
    return json.dumps({"pi": 3.14159265, "e": 2.71828182})

@mcp.prompt
def math_tutor(topic: str) -> str:
    """Generate a math tutoring prompt."""
    return f"Explain {topic} step by step with examples."

if __name__ == "__main__":
    mcp.run()
```

---

*Source: [gofastmcp.com](https://gofastmcp.com) | Verified: 2026-06-30*
