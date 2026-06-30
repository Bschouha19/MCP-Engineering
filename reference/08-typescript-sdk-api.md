# MCP TypeScript SDK API Reference

> The official TypeScript/Node.js SDK for building MCP servers and clients.

> **Note:** TypeScript SDK v2 is pre-release as of mid-2026, targeting stable release alongside the 2026-07-28 MCP spec. API below reflects the current stable v1.x. Verify against [ts.sdk.modelcontextprotocol.io](https://ts.sdk.modelcontextprotocol.io) if breaking changes have landed.

---

## Installation

```bash
npm install @modelcontextprotocol/sdk
# or
pnpm add @modelcontextprotocol/sdk
```

Optional transport packages:
```bash
npm install @modelcontextprotocol/node      # Node.js Streamable HTTP
npm install @modelcontextprotocol/express   # Express integration
npm install @modelcontextprotocol/hono      # Hono integration
```

---

## McpServer — High-Level Server Class

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer(
  {
    name: string,     // Server name
    version: string,  // Server version e.g. "1.0.0"
  },
  {
    capabilities: {   // Optional: declare specific capabilities
      logging: {},    // Enable structured logging
      tools: { listChanged: true },
      resources: { subscribe: true, listChanged: true },
      prompts: { listChanged: true },
    },
  }
);
```

---

## Tool Registration

```typescript
import { z } from "zod";

server.registerTool(
  "tool_name",          // Tool name (string)
  {
    description: "What this tool does",
    inputSchema: {      // Zod schemas for each parameter
      location: z.string().describe("City name"),
      units: z.enum(["celsius", "fahrenheit"]).optional().default("celsius"),
    },
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true,
    },
  },
  async (args, context) => {
    const { location, units } = args;  // Typed from Zod schema
    // ... implementation ...
    return {
      content: [{ type: "text", text: `Weather in ${location}: 18°C` }],
      isError: false,  // Optional; set to true for tool execution errors
    };
  }
);
```

### Tool handler context

```typescript
async (args, { server, log, reportProgress }) => {
  // Logging
  log.debug("debug message");
  log.info("info message");
  log.warning("warning");
  log.error("error");

  // Progress
  await reportProgress({ progress: 50, total: 100 });

  return { content: [{ type: "text", text: "done" }] };
}
```

### Tool with image result

```typescript
server.registerTool("screenshot", {
  description: "Take a screenshot",
  inputSchema: {},
}, async () => {
  const imageBase64 = await captureScreen();
  return {
    content: [{
      type: "image",
      data: imageBase64,
      mimeType: "image/png",
    }],
  };
});
```

### Tool execution error

```typescript
return {
  content: [{ type: "text", text: "Error: database connection failed" }],
  isError: true,  // LLM sees this as an error it may be able to recover from
};
```

### Protocol error (use McpError)

```typescript
import { McpError, ErrorCode } from "@modelcontextprotocol/sdk/types.js";

throw new McpError(
  ErrorCode.InvalidParams,
  "Parameter 'date' must be in ISO 8601 format"
);
// ErrorCode options:
// ErrorCode.ParseError       → -32700
// ErrorCode.InvalidRequest   → -32600
// ErrorCode.MethodNotFound   → -32601
// ErrorCode.InvalidParams    → -32602
// ErrorCode.InternalError    → -32603
```

---

## Resource Registration

### Static resource

```typescript
server.registerResource(
  "app-config",                    // Resource name
  "config://app/settings",         // URI
  {
    description: "Application configuration",
    mimeType: "application/json",
  },
  async (uri: URL) => ({
    contents: [{
      uri: uri.toString(),
      mimeType: "application/json",
      text: JSON.stringify({ debug: false, version: "1.0.0" }),
    }],
  })
);
```

### URI template resource

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

server.registerResource(
  "user-profile",
  new ResourceTemplate("users://{user_id}/profile", {
    list: async () => ({
      resources: [
        { uri: "users://1/profile", name: "Alice" },
        { uri: "users://2/profile", name: "Bob" },
      ],
    }),
  }),
  {
    description: "User profile data",
    mimeType: "application/json",
  },
  async (uri: URL, params: { user_id: string }) => ({
    contents: [{
      uri: uri.toString(),
      mimeType: "application/json",
      text: JSON.stringify(await getUserProfile(params.user_id)),
    }],
  })
);
```

---

## Prompt Registration

```typescript
server.registerPrompt(
  "code_review",
  {
    description: "Review code for bugs and style issues",
    argsSchema: {
      code: z.string().describe("The code to review"),
      language: z.string().optional().describe("Programming language"),
    },
  },
  async ({ code, language }) => ({
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: `Review this ${language ?? ""} code:\n\n${code}`,
        },
      },
    ],
  })
);
```

---

## Server Notifications

```typescript
// Notify clients that the tool list has changed
server.sendToolListChanged();

// Notify clients that the resource list has changed
server.sendResourceListChanged();

// Notify clients that the prompt list has changed
server.sendPromptListChanged();
```

---

## stdio Transport

```typescript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const transport = new StdioServerTransport();
await server.connect(transport);

// Write logs to stderr (NOT stdout — stdout is reserved for MCP messages)
console.error("Server started on stdio");
```

---

## Streamable HTTP Transport

### With Express

```typescript
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { randomUUID } from "crypto";

const app = express();
app.use(express.json());

// Map session IDs to transport instances (for stateful sessions)
const sessions = new Map<string, StreamableHTTPServerTransport>();

app.post("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  
  let transport: StreamableHTTPServerTransport;
  if (sessionId && sessions.has(sessionId)) {
    transport = sessions.get(sessionId)!;
  } else {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => sessions.set(id, transport),
    });
    await server.connect(transport);
  }
  
  await transport.handleRequest(req, res, req.body);
});

// Handle GET for server-initiated SSE stream
app.get("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string;
  const transport = sessions.get(sessionId);
  if (!transport) {
    res.status(404).json({ error: "Session not found" });
    return;
  }
  await transport.handleRequest(req, res);
});

// Session cleanup on DELETE
app.delete("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string;
  sessions.delete(sessionId);
  res.status(200).end();
});

app.listen(3000);
```

### Stateless mode (for serverless / Cloud Run)

```typescript
const transport = new StreamableHTTPServerTransport({
  sessionIdGenerator: undefined,  // null session = stateless
});
```

---

## Sampling (Server → Client LLM Inference)

> Deprecated in 2026-07 RC. Shown for understanding legacy servers.

```typescript
const result = await server.server.createMessage({
  messages: [
    { role: "user", content: { type: "text", text: "Summarize this in one sentence: " + data } }
  ],
  maxTokens: 150,
  modelPreferences: {
    hints: [{ name: "claude-haiku-4-5" }],
    costPriority: 0.9,
    speedPriority: 0.8,
    intelligencePriority: 0.2,
  },
});
const text = result.content.type === "text" ? result.content.text : "";
```

---

## Client API

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

// Connect via stdio
const transport = new StdioClientTransport({
  command: "python",
  args: ["server.py"],
  env: { API_KEY: "..." },
});

const client = new Client(
  { name: "my-client", version: "1.0.0" },
  { capabilities: { roots: { listChanged: true } } }
);

await client.connect(transport);

// List and call tools
const { tools } = await client.listTools();
const result = await client.callTool({
  name: "tool_name",
  arguments: { key: "value" },
});

// Read resources
const { resources } = await client.listResources();
const { contents } = await client.readResource({ uri: "resource://path" });

// Get prompts
const { prompts } = await client.listPrompts();
const { messages } = await client.getPrompt({
  name: "prompt_name",
  arguments: { key: "value" },
});

// Connect via Streamable HTTP
const httpTransport = new StreamableHTTPClientTransport(
  new URL("http://localhost:3000/mcp"),
  {
    requestInit: {
      headers: { Authorization: "Bearer token" },
    },
  }
);
await client.connect(httpTransport);
```

---

## Key Import Paths

```typescript
// Server
import { McpServer }                   from "@modelcontextprotocol/sdk/server/mcp.js";
import { Server }                       from "@modelcontextprotocol/sdk/server/index.js";  // Low-level
import { StdioServerTransport }         from "@modelcontextprotocol/sdk/server/stdio.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { ResourceTemplate }             from "@modelcontextprotocol/sdk/server/mcp.js";
import { completable }                  from "@modelcontextprotocol/sdk/server/completable.js";

// Client
import { Client }                       from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport }         from "@modelcontextprotocol/sdk/client/stdio.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

// Types and errors
import { McpError, ErrorCode }          from "@modelcontextprotocol/sdk/types.js";
import type { Tool, Resource, Prompt }  from "@modelcontextprotocol/sdk/types.js";
```

---

## Minimal Working Server

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "demo", version: "1.0.0" });

server.registerTool("greet", {
  description: "Greet a user by name",
  inputSchema: { name: z.string().describe("User's name") },
}, async ({ name }) => ({
  content: [{ type: "text", text: `Hello, ${name}!` }],
}));

const transport = new StdioServerTransport();
await server.connect(transport);
console.error("Demo server running");
```

---

*Source: [ts.sdk.modelcontextprotocol.io](https://ts.sdk.modelcontextprotocol.io) | [GitHub](https://github.com/modelcontextprotocol/typescript-sdk) | Verified: 2026-06-30*
