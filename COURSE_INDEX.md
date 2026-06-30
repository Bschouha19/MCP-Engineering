# MCP Engineering Course Index
### Build Production MCP Servers, Clients, and Ecosystems — Volume 2

---

> **Prerequisites:** [Volume 1 — AI Engineering](https://github.com/Bschouha19/AI-Engineering-Handbook) — all 20 chapters
>
> **Assumed knowledge:** Python, Node.js, Anthropic SDK, AI agents (Vol 1 Ch 10–11), RAG (Vol 1 Ch 09), Docker

> **What you will build by the end:** DevAssist — a complete production MCP ecosystem with four coordinated MCP servers, an OAuth 2.1 gateway, a multi-server Claude agent, OpenTelemetry tracing, and a Smithery registry listing.

---

## Progress

**4 of 15 chapters complete**

---

## Course Structure

### MODULE 1 — FOUNDATIONS
*Build the complete mental model before writing any server code.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 01 | [What Is MCP and Why It Exists](chapters/chapter-01-what-is-mcp.md) | chapters/chapter-01-what-is-mcp.md | ✅ Complete | N×M problem, JSON-RPC 2.0, three-actor model, ecosystem map |
| 02 | [MCP Protocol Architecture — How It Actually Works](chapters/chapter-02-protocol-architecture.md) | chapters/chapter-02-protocol-architecture.md | ✅ Complete | Capability negotiation, message lifecycle, error codes, spec versions |
| 03 | [Your First MCP Server](chapters/chapter-03-first-server.md) | chapters/chapter-03-first-server.md | ✅ Complete | FastMCP, raw SDK, wiring to Claude Code + Cursor + Claude Desktop |

**Module 1 Learning Goal:** Understand what MCP is, why the protocol is designed the way it is, and have a working server connected to real clients before moving to Module 2.

**Module 1 Project:** A working MCP server with one tool, one resource, and one prompt — connected to Claude Code and Cursor.

---

### MODULE 2 — THE THREE PRIMITIVES
*Master what a server can expose: tools, resources, and prompts.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 04 | [MCP Tools — The Primary Primitive](chapters/chapter-04-tools.md) | chapters/chapter-04-tools.md | ✅ Complete | Tool schema design, annotations, return types, security, versioning |
| 05 | MCP Resources and Prompts | chapters/chapter-05-resources-prompts.md | 🔜 | URI schemes, resource templates, subscriptions, prompt arguments |
| 06 | Advanced Protocol Features | chapters/chapter-06-advanced-protocol.md | 🔜 | Sampling, Roots, Elicitation, progress tracking, cancellation, logging |

**Module 2 Learning Goal:** Know when to use a tool vs a resource vs a prompt, design tool schemas that LLMs interpret correctly, and use all protocol-level utilities.

**Module 2 Project:** A multi-primitive MCP server — tools for writing, resources for reading, prompts for reusable workflows — serving a real knowledge domain.

---

### MODULE 3 — TRANSPORT AND CONNECTIVITY
*Understand the plumbing: how clients and servers connect.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 07 | MCP Transports — How Clients and Servers Connect | chapters/chapter-07-transports.md | 🔜 | stdio internals, Streamable HTTP, legacy SSE migration, remote deployment |
| 08 | Building MCP Clients and Multi-Server Architectures | chapters/chapter-08-clients.md | 🔜 | Python MCP client, multi-server agents, tool routing, A2A protocol |

**Module 3 Learning Goal:** Deploy a remote MCP server reachable over HTTPS, and build a Python agent that connects to multiple MCP servers simultaneously.

**Module 3 Project:** A Claude agent connected to three MCP servers over Streamable HTTP, with tool namespace management and A2A-style delegation.

---

### MODULE 4 — REAL-WORLD SERVERS
*Build complete, deployable, production-grade servers in specific domains.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 09 | Building a Code and Version Control MCP Server | chapters/chapter-09-code-git-server.md | 🔜 | GitHub API tools, file system tools, git operations, path traversal prevention |
| 10 | Building a Database MCP Server | chapters/chapter-10-database-server.md | 🔜 | Schema resources, query tools, connection pooling, row-level security |
| 11 | Building API Wrapper MCP Servers | chapters/chapter-11-api-wrapper-server.md | 🔜 | REST → MCP pattern, OpenAPI auto-generation, rate limiting, Stripe example |

**Module 4 Learning Goal:** Build and deploy three different categories of production MCP server: code/VCS, structured data, and API wrapper.

**Module 4 Project:** A self-contained MCP server for a domain of your choice — GitHub, a PostgreSQL database, or a third-party API — with auth, tests, and Docker container.

---

### MODULE 5 — PRODUCTION ENGINEERING
*Security, observability, and deployment for servers that run in the real world.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 12 | Authentication, Authorization, and Security | chapters/chapter-12-auth-security.md | 🔜 | OAuth 2.1 + PKCE, gateway pattern, OWASP MCP Top 10, audit logging |
| 13 | Testing, Debugging, and Observability | chapters/chapter-13-testing-debugging.md | 🔜 | MCP Inspector, pytest for MCP, OpenTelemetry, CI/CD gates |
| 14 | Deploying MCP Servers at Scale | chapters/chapter-14-deployment.md | 🔜 | Docker, Cloud Run, Kubernetes, stateless scaling, Smithery publishing |

**Module 5 Learning Goal:** Harden an MCP server against the OWASP MCP Top 10, instrument it with OpenTelemetry, and deploy it to production with zero-downtime deploys and a registry listing.

**Module 5 Project:** Take a Module 4 server through the full production lifecycle: OAuth 2.1 auth, MCP Inspector test suite, OpenTelemetry traces, Cloud Run deployment, Smithery listing.

---

### MODULE 6 — CAPSTONE
*Build a complete production MCP ecosystem from scratch.*

| # | Chapter | File | Status | Key Skills |
|---|---------|------|--------|-----------|
| 15 | Capstone — Build a Production MCP Ecosystem | chapters/chapter-15-capstone.md | 🔜 | All of Volume 2: four servers, OAuth gateway, multi-server agent, OTel, VPS deploy |

**Capstone System — DevAssist:**
- GitHub MCP Server — repos, issues, PRs, code search
- PostgreSQL MCP Server — project data, deployment history
- Web Search MCP Server — documentation lookup
- RAG Knowledge Base MCP Server — internal docs (connects back to Vol 1 Ch 09)
- OAuth 2.1 Gateway — one gateway protecting all four servers
- Claude Agent — multi-server consumer with tool routing
- OpenTelemetry — one trace_id flowing across all tool calls and Claude API calls
- Docker Compose dev stack
- VPS deployment with Let's Encrypt TLS
- Smithery registry listing

---

## Chapter Dependency Map

```
Ch 01 (What Is MCP)
  └─► Ch 02 (Protocol Architecture)
        └─► Ch 03 (First Server)
              ├─► Ch 04 (Tools)
              │     └─► Ch 05 (Resources + Prompts)
              │           └─► Ch 06 (Advanced Protocol)
              └─► Ch 07 (Transports)
                    └─► Ch 08 (Clients)
                          ├─► Ch 09 (Code/Git Server)
                          ├─► Ch 10 (Database Server)
                          └─► Ch 11 (API Wrapper)
                                ├─► Ch 12 (Auth + Security)
                                ├─► Ch 13 (Testing + Debugging)
                                └─► Ch 14 (Deployment)
                                      └─► Ch 15 (Capstone)
```

---

## Learning Path Options

### Standard Path (recommended)
Read all 15 chapters in order. Takes approximately 6–8 weeks part-time.

### Server-Author Fast Track
Ch 01 → Ch 02 → Ch 03 → Ch 04 → Ch 05 → Ch 07 → Ch 09 or Ch 10 or Ch 11 → Ch 12 → Ch 14
Skips: Advanced protocol features (Ch 06), client building (Ch 08), and observability depth (Ch 13)
Takes approximately 3–4 weeks part-time.

### Security-Focused Path
Ch 01 → Ch 02 → Ch 03 → Ch 04 → Ch 07 → Ch 12 → Ch 13
Focus: Protocol understanding + tool design + transport + security + testing
Takes approximately 2–3 weeks part-time.

---

## MCP Specification Coverage

| Spec Feature | Chapter | Status |
|-------------|---------|--------|
| JSON-RPC 2.0 message format | Ch 02 | 🔜 |
| Capability negotiation | Ch 02 | 🔜 |
| stdio transport | Ch 07 | 🔜 |
| Streamable HTTP transport | Ch 07 | 🔜 |
| Legacy SSE (deprecated) | Ch 07 | 🔜 |
| Tools primitive | Ch 04 | ✅ |
| Resources primitive | Ch 05 | 🔜 |
| Prompts primitive | Ch 05 | 🔜 |
| Sampling (client feature, deprecated in 2026-07 RC) | Ch 06 | 🔜 |
| Roots (client feature, deprecated in 2026-07 RC) | Ch 06 | 🔜 |
| Elicitation (client feature) | Ch 06 | 🔜 |
| Progress notifications | Ch 06 | 🔜 |
| Cancellation | Ch 06 | 🔜 |
| Logging | Ch 06 | 🔜 |
| Tool annotations | Ch 04 | ✅ |
| Resource subscriptions | Ch 05 | 🔜 |
| Resource templates | Ch 05 | 🔜 |
| Embedded resources | Ch 05 | 🔜 |
| OAuth 2.1 + PKCE | Ch 12 | 🔜 |
| Dynamic client registration | Ch 12 | 🔜 |
| .well-known/mcp.json | Ch 14 | 🔜 |
| W3C Trace Context (2026-07 RC) | Ch 13 | 🔜 |
| Tasks primitive (2026-07 RC) | Ch 06 | 🔜 |
| Multi-Round-Trip Requests (2026-07 RC) | Ch 06 | 🔜 |

---

*Last updated: 2026-06-30 — 4 of 15 chapters complete*
