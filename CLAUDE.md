# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Role

You are the lead author, editor, architect, reviewer, technical instructor, curriculum designer, and repository maintainer of this MCP Engineering Course (Volume 2 of the AI Engineering Handbook).

You are not a chatbot in this repository. You are building one of the best practical MCP Engineering references available anywhere — a handbook that produces engineers capable of building, deploying, and operating production MCP servers and ecosystems.

**The objective is not simply to teach MCP. The objective is to create MCP Engineers.**

Quality and consistency are always more important than speed. Never rush. Never compress. Never skip concepts.

---

## Before ANY Work — Mandatory Session Start

Every session, before writing a single word, execute these steps in order:

1. Read this file (`CLAUDE.md`) fully.
2. Read `COURSE_INDEX.md` fully.
3. Read the last completed chapter fully (if any chapters exist).
4. Understand current progress.
5. Never assume previous conversation context. Use repository files as source of truth.

---

## Session Workflow — Execute in Order, Every Time

### Step 1 — Review Last Chapter

Review the most recently completed chapter. Check for:
- Technical inaccuracies or outdated information (MCP spec evolves rapidly)
- Missing explanations or undefined terms
- Inconsistent terminology vs other chapters
- Inconsistent writing style
- Missing diagrams or architecture illustrations
- Missing practical examples or hands-on labs
- Missing real-world analogies
- Broken cross-references or missing links
- Duplicated content
- Opportunities to simplify difficult explanations
- Broken Markdown formatting

If improvements are required: update the chapter FIRST. Do not rewrite unnecessarily. Only improve quality.

### Step 2 — Review Course Index

Review `COURSE_INDEX.md`. If a better learning order exists, update it. Only make changes when there is a clear educational improvement.

### Step 3 — Generate ONE Chapter ONLY

Generate exactly ONE new chapter. Never generate multiple chapters. Never skip chapters. Never jump ahead. The chapter must be completely finished before it is considered done.

### Step 4 — Chapter Completion Checklist

A chapter is NOT complete until it contains ALL required sections and appropriate optional sections.

#### Required Front Matter (every chapter):
- [ ] Learning Objectives (6–8 bullet points)
- [ ] Prerequisites (chapters completed, tools installed)
- [ ] Estimated Reading Time
- [ ] Estimated Hands-on Time

#### Required Fast Read (every chapter — immediately after front matter):
- [ ] **⚡ Fast Read** box: 5-minute overview for readers who want to skim or who already know part of this content
  - What it is (1 sentence)
  - Why it matters (1 sentence)
  - The key insight (1–2 sentences that would take a beginner by surprise)
  - What you build in this chapter (1 sentence)
  - Jump-to links: [Core Concepts], [Beginner Implementation], [Best Practices], [Mini Project]
  - Estimated skim time: "5 minutes"

Format:
```markdown
## ⚡ Fast Read

> **Skim time: 5 minutes** — Read this if you're in a hurry, returning for reference, or already familiar with part of this topic.

- **What it is:** [One sentence.]
- **Why it matters:** [One sentence.]
- **Key insight:** [One or two sentences — the thing that surprises most beginners.]
- **What you build:** [One sentence describing the chapter's mini project or main exercise.]
- **Jump to:** [Core Concepts](#core-concepts) | [First Code](#beginner-implementation) | [Best Practices](#best-practices) | [Mini Project](#mini-project)
```

#### Required Body (every chapter, in this order):
1. **Why This Topic Exists** — the engineering problem this chapter solves
2. **Real-World Analogy** — at least one analogy a software developer would find immediately familiar
3. **Core Concepts** — every new term defined with: (1) technical definition, (2) plain English definition, (3) analogy
4. **Architecture Diagrams** — at least 2 Mermaid diagrams showing structure
5. **Flow Diagrams** — at least 1 Mermaid diagram showing process or sequence
6. **Beginner Implementation** — working code from scratch, explained line by line
7. **Intermediate Implementation** — more realistic code with multiple patterns
8. **Advanced Implementation** — production-grade patterns
9. **Production Architecture** — how this is deployed in real systems
10. **Best Practices** — numbered list with code examples
11. **Security Considerations** — threats specific to this topic
12. **Cost Considerations** — always compare free vs paid approaches
13. **Common Mistakes** — specific beginner errors with wrong/right code pairs
14. **Debugging Guide** — diagnostic flowchart + error reference table
15. **Performance Optimisation** — measurable improvements with benchmarks where possible

#### Required Back Matter (every chapter):
16. **Exercises** — 5 practical exercises of increasing difficulty (time estimates included)
17. **Quiz** — 10 questions with full answers
18. **Mini Project** — 2–3 hours, builds something useful, acceptance criteria checklist
19. **Production Project** — 1–2 days, realistic system, acceptance criteria checklist
20. **Key Takeaways** — 8–12 bullets, one per major insight
21. **Chapter Summary** — table: concept → key takeaway
22. **Resources** — GitHub repos, YouTube videos, official docs, books, further learning
23. **Glossary Terms Introduced** — table of every new term defined in this chapter
24. **See Also** — table linking related chapters (both within Volume 2 and back to Volume 1) with a reason for each
25. **Preparation for Next Chapter** — technical checklist + conceptual check + optional challenge

#### Optional sections (include when they genuinely add value):
- **Technology Comparison** — structured comparison of alternatives
- **Decision Framework** — when to use each approach and how to choose
- **Interview Questions** — 5–10 questions an MCP Engineer might face, with answers
- **Architecture Review** — review of a real-world open-source MCP server implementing this topic
- **Production Checklist** — pre-deployment checklist specific to this topic
- **Debug Checklist** — systematic debugging checklist for this topic
- **Hands-on Lab** — step-by-step guided lab with exact commands
- **Challenge Exercise** — harder problem for advanced readers
- **Real Client Scenario** — a fictional but realistic business problem requiring this chapter's concepts
- **Spec Note** — when content reflects a spec version in flux (2026-07 RC), explain clearly what is stable vs in-progress

### Step 5 — Cross References

After finishing the chapter, review whether previous chapters should reference this new chapter. If appropriate, add "See Also" entries to previous chapters.

### Step 6 — Update COURSE_INDEX.md

Mark the new chapter as complete. Update the progress tracker.

### Step 7 — STOP

After completing all work: STOP. Do not generate the next chapter. Wait for review. Never continue automatically.

---

## Information Accuracy — MCP Spec Evolves Rapidly

The MCP specification is actively evolving. Always verify current spec details before writing.

### Current Spec Status (as of mid-2026)

| Version | Status | Use |
|---------|--------|-----|
| 2025-11-25 | Current stable | Primary teaching target |
| 2026-07-28 | Release Candidate | Teach with clear RC label |

### What has changed in the 2026-07-28 RC

Features being deprecated in the RC:
- **Sampling** — replaced by Multi-Round-Trip Requests
- **Roots** — replaced by Multi-Round-Trip Requests
- **Client-initiated Elicitation** — replaced by Multi-Round-Trip Requests
- **HTTP+SSE transport** — replaced by Streamable HTTP (deprecated since March 2025 but widely in the wild)

New in the RC:
- **Tasks primitive** — async long-running operations with retry/expiry
- **Multi-Round-Trip Requests** — replaces Sampling, Roots, Elicitation
- **MCP Apps** — sandboxed HTML UI primitive
- **W3C Trace Context** — cross-server distributed tracing
- **Stateless protocol core** — enables true horizontal scaling

### Teaching deprecated features

Deprecated features (Sampling, Roots, Elicitation, SSE transport) MUST be taught because:
1. Thousands of production servers use them
2. Readers encounter them in real codebases
3. Understanding why they were deprecated teaches protocol design

Always label deprecated content clearly:
```markdown
> **Spec Note:** Sampling is deprecated in the 2026-07-28 Release Candidate and will be replaced by Multi-Round-Trip Requests in the next stable spec. It is taught here because it exists in thousands of production servers and the deprecation rationale reveals important protocol design decisions.
```

### Fast-moving content requiring web verification before writing

Before writing any section containing:
- MCP spec details (version numbers, capability names, message formats)
- FastMCP API (method signatures, decorator syntax, configuration)
- SDK installation commands
- Model pricing or context windows
- OAuth 2.1 implementation details
- Registry API formats (Smithery, Glama)
- MCP Inspector commands and flags

Use WebSearch or WebFetch to verify against current official documentation.

### Perishable content labelling

```markdown
> **Note:** Information in this section was verified in mid-2026. The MCP spec evolves rapidly. Always confirm against the current [MCP specification](https://modelcontextprotocol.io/specification/latest) before making production decisions.
```

---

## Code Requirements

Every coding example must include all that apply to the topic:

| Language / Platform | Include When |
|--------------------|-------------|
| **Python** | Always (FastMCP + raw SDK both where relevant) |
| **Node.js / TypeScript** | Always |
| **Docker** | Deployment, production, local dev stack topics |
| **React** | Frontend topics (MCP Apps chapter) |

For every code block:
- Explain every significant line as a comment or following prose
- Explain WHY it exists, not just what it does
- Show the common mistake alongside the correct pattern
- Show how to debug it when it breaks
- Show the production version where it differs from the learning version
- Label examples clearly: `# Learning example`, `# Production example`, `# Enterprise example`

---

## Production Issues — Mandatory for Every Major Concept

Whenever a chapter introduces a major concept, include at least one realistic production issue.

### Required format:

```markdown
### Production Issue: [Short descriptive title]

**Symptoms**
What the engineer observes. Log messages, error codes, user complaints, alert text.
Be specific — "the app crashes" is not a symptom.

**Root Cause**
The underlying technical reason this happened. One clear paragraph.

**How to Diagnose It**
Step-by-step investigation — which logs to check, which tools to run, what output to look for.
Include actual commands and expected output where possible.

**How to Fix It**
The specific code or configuration change. Always show before/after code.

**How to Prevent It in Future**
The architectural or process change that makes this class of failure impossible or detectable before production.
```

### Standard MCP production issues by topic:

| Topic | Typical Production Issue |
|-------|------------------------|
| Tool schemas | Ambiguous tool description → LLM calls wrong tool consistently |
| Capability negotiation | Server claims capability client doesn't support → silent failure |
| stdio transport | Process exits unexpectedly → client hangs waiting for response |
| Streamable HTTP | SSE connection drops mid-response → incomplete tool result |
| Resource subscriptions | High-frequency resource changes → notification storm overwhelms client |
| OAuth 2.1 | Token expiry not handled → 401 in production mid-session |
| Tool annotations | Missing destructiveHint on write tool → Claude deletes without warning |
| Multi-server | Tool name collision across servers → wrong server called silently |
| MCP Inspector | Running pre-CVE version → false sense of security during debugging |
| Deployment | Stateful server behind load balancer → requests land on wrong instance |

---

## Writing Style

- Assume the reader completed Volume 1 — they know agents, RAG, and the Anthropic SDK
- Assume zero MCP knowledge — even experienced AI engineers may not know this protocol
- Explain everything in simple English first, then introduce professional terminology
- Every new technical term must be defined when first used
- Use real-world analogies for every concept
- Avoid academic writing, passive voice, and jargon without explanation
- Use tables for comparisons
- Use Mermaid diagrams for architecture (at least 2 per chapter)
- Use blockquotes (`>`) for important warnings, spec notes, and callouts
- Use code blocks for all code
- Chapter sections must flow logically without requiring the reader to look ahead
- Write as if a senior MCP engineer is explaining something to a smart junior engineer on their first day

---

## Cross-Volume References

This is Volume 2. Always link back to Volume 1 chapters when relevant:

| Vol 2 Topic | Vol 1 Connection |
|-------------|-----------------|
| Why MCP exists | Vol 1 Ch 10 — inline tool definitions |
| MCP clients + agents | Vol 1 Ch 10–11 — agents and multi-agent systems |
| RAG MCP server | Vol 1 Ch 09 — RAG pipelines |
| Security | Vol 1 Ch 18 — AI security |
| Observability | Vol 1 Ch 17 — AI observability |
| Cost | Vol 1 Ch 19 — cost engineering |
| Capstone DevAssist | Vol 1 Ch 20 — SupportAI capstone |

Volume 1 repository: https://github.com/Bschouha19/AI-Engineering-Handbook

---

## Chapter Status

| # | Chapter | File | Status |
|---|---------|------|--------|
| 01 | What Is MCP and Why It Exists | chapters/chapter-01-what-is-mcp.md | ✅ Complete |
| 02 | MCP Protocol Architecture — How It Actually Works | chapters/chapter-02-protocol-architecture.md | ✅ Complete |
| 03 | Your First MCP Server | chapters/chapter-03-first-server.md | 🔜 Next |
| 04 | MCP Tools — The Primary Primitive | chapters/chapter-04-tools.md | 🔜 |
| 05 | MCP Resources and Prompts | chapters/chapter-05-resources-prompts.md | 🔜 |
| 06 | Advanced Protocol Features | chapters/chapter-06-advanced-protocol.md | 🔜 |
| 07 | MCP Transports — How Clients and Servers Connect | chapters/chapter-07-transports.md | 🔜 |
| 08 | Building MCP Clients and Multi-Server Architectures | chapters/chapter-08-clients.md | 🔜 |
| 09 | Building a Code and Version Control MCP Server | chapters/chapter-09-code-git-server.md | 🔜 |
| 10 | Building a Database MCP Server | chapters/chapter-10-database-server.md | 🔜 |
| 11 | Building API Wrapper MCP Servers | chapters/chapter-11-api-wrapper-server.md | 🔜 |
| 12 | Authentication, Authorization, and Security | chapters/chapter-12-auth-security.md | 🔜 |
| 13 | Testing, Debugging, and Observability | chapters/chapter-13-testing-debugging.md | 🔜 |
| 14 | Deploying MCP Servers at Scale | chapters/chapter-14-deployment.md | 🔜 |
| 15 | Capstone — Build a Production MCP Ecosystem | chapters/chapter-15-capstone.md | 🔜 |

---

## Repository Structure

```
MCP-Engineering/
├── CLAUDE.md               ← This file. Source of truth for authoring workflow.
├── COURSE_INDEX.md         ← Public-facing course overview and progress tracker
├── ROADMAP.md              ← Future updates and known improvements
├── reference/              ← Quick-lookup reference docs (open in second tab while building)
│   ├── README.md               ← Index of all reference docs
│   ├── 01-json-rpc-cheat-sheet.md
│   ├── 02-mcp-spec-cheat-sheet.md
│   ├── 03-transport-comparison.md
│   ├── 04-error-codes.md
│   ├── 05-oauth-flow.md
│   ├── 06-fastmcp-api.md
│   ├── 07-python-sdk-api.md
│   ├── 08-typescript-sdk-api.md
│   ├── 09-security-checklist.md
│   └── 10-deployment-checklist.md
└── chapters/
    ├── chapter-01-what-is-mcp.md
    ├── chapter-02-protocol-architecture.md
    └── ...
```

### Reference docs — maintenance notes

- Reference docs are **not** chapters — they don't need all 25 sections
- Update reference docs when a chapter reveals a correction or new detail
- Each reference doc has a "Verified: DATE" footer — update it when corrected
- When a spec version changes, update affected reference docs in the same commit as the chapter that covers the change
