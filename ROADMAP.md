# Volume 2 Roadmap — MCP Engineering

This document tracks planned improvements, known gaps, and future additions within Volume 2.

---

## Volume Status

**Current:** In Progress — 0 of 15 chapters written
**Target:** 15 chapters covering all major MCP Engineering topics
**Started:** June 2026

---

## Planned Chapters

| # | Chapter | Priority | Est. Completion |
|---|---------|----------|----------------|
| 01 | What Is MCP and Why It Exists | High | Session 1 |
| 02 | MCP Protocol Architecture | High | Session 2 |
| 03 | Your First MCP Server | High | Session 3 |
| 04 | MCP Tools — The Primary Primitive | High | Session 4 |
| 05 | MCP Resources and Prompts | High | Session 5 |
| 06 | Advanced Protocol Features | Medium | Session 6 |
| 07 | MCP Transports | High | Session 7 |
| 08 | Building MCP Clients | High | Session 8 |
| 09 | Code and Version Control Server | High | Session 9 |
| 10 | Database MCP Server | High | Session 10 |
| 11 | API Wrapper Servers | Medium | Session 11 |
| 12 | Authentication and Security | High | Session 12 |
| 13 | Testing, Debugging, Observability | High | Session 13 |
| 14 | Deploying at Scale | High | Session 14 |
| 15 | Capstone — DevAssist | High | Session 15 |

---

## Spec Monitoring

The MCP specification is actively evolving. These spec changes should be incorporated as they are finalised:

| Spec Change | RC Status | Impact | Target Chapter |
|-------------|-----------|--------|---------------|
| Multi-Round-Trip Requests (replaces Sampling/Roots) | 2026-07 RC | High — new auth pattern | Ch 06 |
| Tasks primitive (async long-running ops) | 2026-07 RC | Medium — new tool pattern | Ch 06 |
| MCP Apps (sandboxed HTML UI) | 2026-07 RC | Low — advanced feature | Ch 06 (note only) |
| W3C Trace Context propagation | 2026-07 RC | Medium — observability | Ch 13 |
| Stateless protocol core | 2026-07 RC | High — deployment model | Ch 14 |

When the 2026-07-28 RC is ratified as a stable spec, update affected chapters with a fact-check pass.

---

## Known Future Additions (Post Volume 2)

Topics not covered in Volume 2 that may appear in later volumes:

- **MCP in mobile applications** — React Native MCP client patterns (Volume consideration)
- **MCP fine-tuning** — training models specifically for tool selection (Volume 5 or 3)
- **A2A deep dive** — Google's Agent-to-Agent protocol warrants its own volume chapter (Volume 3)
- **MCP in the browser** — Streamable HTTP enables browser clients (niche, post V2)
- **Building a Host application** — implementing a full MCP host like Claude Desktop (advanced, post V2)

---

## AI Engineering Handbook — Full Series

| Volume | Title | Repository | Status |
|--------|-------|-----------|--------|
| 1 | AI Engineering | https://github.com/Bschouha19/AI-Engineering-Handbook | ✅ Complete |
| 2 | MCP Engineering | https://github.com/Bschouha19/MCP-Engineering | 🔄 In Progress |
| 3 | AI Agent Engineering | — | 🔜 Planned |
| 4 | n8n AI Workflow Automation | — | 🔜 Planned |
| 5 | RAG Deep Dive | — | 🔜 Planned |
| 6 | Vector Database Engineering | — | 🔜 Planned |
| 7 | Coding Agents | — | 🔜 Planned |
| 8 | DevOps AI | — | 🔜 Planned |
| 9 | Technical PM AI | — | 🔜 Planned |
| 10 | Enterprise AI Systems | — | 🔜 Planned |
| 11 | AI Architecture Patterns | — | 🔜 Planned |
| 12 | Real Production Case Studies | — | 🔜 Planned |

---

*Last updated: 2026-06-30*
