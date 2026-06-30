# MCP Engineering Handbook

### Build Production MCP Servers, Clients, and Ecosystems — Volume 2

A complete, practical MCP Engineering course for software developers who have completed Volume 1 (AI Engineering). Not a protocol reference — a handbook that produces engineers capable of designing, building, debugging, deploying, and operating production MCP systems.

---

## Prerequisites

This is Volume 2 of the AI Engineering Handbook series. Before starting this volume, complete:

**[Volume 1 — AI Engineering](https://github.com/Bschouha19/AI-Engineering)**

Volume 1 covers: LLMs, AI APIs, Prompt Engineering, Structured Outputs, Embeddings, Vector Databases, RAG, AI Agents, Multi-Agent Systems, Local AI, Fine-Tuning, Multi-Modal AI, Production Architecture, Testing, Observability, Security, Cost Engineering, and a complete Capstone project.

Volume 2 assumes familiarity with: Python, Node.js, the Anthropic SDK, AI Agents, RAG pipelines, and Docker.

---

## What This Is

The Model Context Protocol (MCP) is an open standard by Anthropic that solves the N×M integration problem: instead of every AI application connecting to every tool with custom code, MCP provides a universal protocol so tools are written once and work everywhere.

This handbook teaches you to build on both sides of that protocol — the servers that expose capabilities, and the clients that consume them.

**By the end of this volume you will be able to:**

- Design and build production-grade MCP servers in Python and TypeScript
- Engineer all three server primitives: Tools, Resources, and Prompts
- Configure and operate both stdio and Streamable HTTP transports
- Implement OAuth 2.1 authentication for remote MCP servers
- Build MCP clients that consume multiple servers in a single agent
- Test and debug MCP servers with MCP Inspector and automated test suites
- Deploy MCP servers to production with Docker, Cloud Run, and Kubernetes
- Publish servers to the Smithery, Glama, and mcp.so registries
- Understand and mitigate the OWASP MCP Top 10 security risks
- Build complete multi-server MCP ecosystems with OpenTelemetry tracing

---

## Who It's For

| Reader | Background | What They Get |
|--------|-----------|--------------|
| **AI Engineer** | Completed Volume 1 | Full MCP Engineering capability — server author and client builder |
| **Backend Developer** | Python / Node.js, new to MCP | Protocol mastery plus production deployment patterns |
| **Technical Architect** | Designing AI systems | Decision frameworks for when and how to use MCP at scale |

---

## Progress

**0 of 15 chapters complete** — Volume 2 in progress.

| Module | Chapters | Status |
|--------|----------|--------|
| 1 — Foundations | Ch 01–03 | 🔜 Next |
| 2 — The Three Primitives | Ch 04–06 | 🔜 |
| 3 — Transport and Connectivity | Ch 07–08 | 🔜 |
| 4 — Real-World Servers | Ch 09–11 | 🔜 |
| 5 — Production Engineering | Ch 12–14 | 🔜 |
| 6 — Capstone | Ch 15 | 🔜 |

---

## Course Structure

### Module 1 — Foundations

| # | Chapter | Status |
|---|---------|--------|
| 01 | What Is MCP and Why It Exists | 🔜 Next |
| 02 | MCP Protocol Architecture — How It Actually Works | 🔜 |
| 03 | Your First MCP Server | 🔜 |

### Module 2 — The Three Primitives

| # | Chapter | Status |
|---|---------|--------|
| 04 | MCP Tools — The Primary Primitive | 🔜 |
| 05 | MCP Resources and Prompts | 🔜 |
| 06 | Advanced Protocol Features | 🔜 |

### Module 3 — Transport and Connectivity

| # | Chapter | Status |
|---|---------|--------|
| 07 | MCP Transports — How Clients and Servers Connect | 🔜 |
| 08 | Building MCP Clients and Multi-Server Architectures | 🔜 |

### Module 4 — Real-World Servers

| # | Chapter | Status |
|---|---------|--------|
| 09 | Building a Code and Version Control MCP Server | 🔜 |
| 10 | Building a Database MCP Server | 🔜 |
| 11 | Building API Wrapper MCP Servers | 🔜 |

### Module 5 — Production Engineering

| # | Chapter | Status |
|---|---------|--------|
| 12 | Authentication, Authorization, and Security | 🔜 |
| 13 | Testing, Debugging, and Observability | 🔜 |
| 14 | Deploying MCP Servers at Scale | 🔜 |

### Module 6 — Capstone

| # | Chapter | Status |
|---|---------|--------|
| 15 | Capstone — Build a Production MCP Ecosystem | 🔜 |

---

## What Each Chapter Contains

Every chapter includes all of the following:

- **Learning objectives** — what you will be able to do after reading
- **Why this topic exists** — the engineering problem it solves
- **Real-world analogy** — something a developer immediately recognises
- **Core concepts** — every term defined: technical + plain English + analogy
- **Architecture diagrams** — Mermaid diagrams before any code
- **Three implementation levels** — learning example → production example → enterprise example
- **Technology comparisons** — never "use this tool" without comparing alternatives
- **Decision frameworks** — how to choose, not just what to choose
- **Production issues** — realistic failures with symptoms, root cause, diagnosis, fix, and prevention
- **Best practices** — numbered, actionable, with code
- **Security considerations** — threats specific to the topic
- **Cost breakdown** — free vs paid, development vs production
- **Common mistakes** — wrong/right code pairs
- **Debugging guide** — diagnostic flowchart + error reference table
- **Exercises** — 5 practical exercises with time estimates
- **Quiz** — 10 questions with full answers
- **Mini project** — 2–3 hours, builds something useful
- **Production project** — 1–2 days, realistic system
- **Key takeaways, glossary, and further reading**

---

## Tools and Technologies Covered

| Category | Tools |
|----------|-------|
| **MCP Frameworks** | FastMCP 3.0 (Python), MCP Python SDK, MCP TypeScript SDK |
| **AI Providers** | Anthropic Claude (Haiku 4.5, Sonnet 4.6, Opus 4.8, Fable 5) |
| **MCP Clients** | Claude Code CLI, Claude Desktop, Cursor, VS Code |
| **Authentication** | OAuth 2.1 + PKCE, dynamic client registration |
| **Gateways** | Kong AI MCP Proxy, Azure API Management |
| **Debugging** | MCP Inspector (v0.14.1+) |
| **Observability** | OpenTelemetry, W3C Trace Context |
| **Registries** | Smithery, Glama, mcp.so |
| **Infrastructure** | Docker, Cloud Run, Kubernetes, VPS |
| **Databases** | PostgreSQL, SQLite |

---

## How to Use This Handbook

1. **Complete Volume 1 first** — this volume builds directly on agents, tools, and RAG
2. **Read chapters in order** — each chapter builds on previous ones
3. **Run every code example** — reading code is not learning code
4. **Complete the exercises** before moving to the next chapter
5. **Build the mini project** — this converts reading into doing
6. **Build the production project** at the end of each module

---

## Prerequisites — Tools to Install Before Chapter 1

- Python 3.12+ with `uv` package manager
- Node.js 22+ with `npm`
- Docker Desktop
- Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
- Claude Desktop (for testing MCP servers)
- Anthropic API key

All installation instructions are covered in [Volume 1, Chapter 3](https://github.com/Bschouha19/AI-Engineering/blob/main/chapters/chapter-03-dev-environment.md).

---

## The AI Engineering Handbook Series

| Volume | Title | Status | Repository |
|--------|-------|--------|-----------|
| 1 | AI Engineering — From Zero to Production | ✅ Complete (20 chapters) | [AI-Engineering](https://github.com/Bschouha19/AI-Engineering) |
| 2 | MCP Engineering | 🔄 In Progress | [MCP-Engineering](https://github.com/Bschouha19/MCP-Engineering) |
| 3 | AI Agent Engineering | 🔜 Planned | — |
| 4 | n8n AI Workflow Automation | 🔜 Planned | — |
| 5 | RAG Deep Dive | 🔜 Planned | — |
| 6 | Vector Database Engineering | 🔜 Planned | — |
| 7 | Coding Agents | 🔜 Planned | — |
| 8 | DevOps AI | 🔜 Planned | — |
| 9 | Technical PM AI | 🔜 Planned | — |
| 10 | Enterprise AI Systems | 🔜 Planned | — |
| 11 | AI Architecture Patterns | 🔜 Planned | — |
| 12 | Real Production Case Studies | 🔜 Planned | — |

---

*Volume 2 started June 2026. Target: 15 chapters.*
