# MCP Security Checklist

> Pre-deployment security review for MCP servers. Work through this list before any remote server goes live. For the auth implementation, see [05-oauth-flow.md](./05-oauth-flow.md).

---

## OWASP MCP Top 10 (2026)

These are the ten most critical security risks in MCP deployments. Each item below maps to a checklist action.

| # | Risk | One-Line Description |
|---|------|---------------------|
| 1 | Tool Poisoning | Malicious server descriptions manipulate the LLM |
| 2 | Prompt Injection via Tool Results | Tool output contains instructions that hijack the conversation |
| 3 | Rug Pull Attacks | Server description changes after the user approved it |
| 4 | Excessive Tool Permissions | Tools can do far more than they need to |
| 5 | Cross-Server Context Contamination | Data from one server leaks into another server's context |
| 6 | Ambient Authority Misuse | Tools operate on behalf of the user without explicit consent |
| 7 | Static API Key Exposure | Secrets embedded in server code or config |
| 8 | Unvalidated Tool Output | Tool results passed to downstream systems without sanitization |
| 9 | Server Impersonation | Malicious server pretends to be a trusted server |
| 10 | Sensitive Data in Logs | User data, tokens, or PII written to log files |

---

## Transport Security

- [ ] **TLS on all remote connections** — all Streamable HTTP traffic uses HTTPS; never HTTP in production
- [ ] **Certificate from a trusted CA** — Let's Encrypt, or a commercial CA; self-signed only for local dev
- [ ] **Validate Origin header** — prevent DNS rebinding attacks; reject requests with unexpected Origin
- [ ] **Bind locally if local-only** — local servers bind to `127.0.0.1`, not `0.0.0.0`
- [ ] **CORS configured** — Streamable HTTP servers declare allowed origins explicitly

```python
# FastMCP: validate Origin in middleware
from fastapi import Request, HTTPException

ALLOWED_ORIGINS = {"https://claude.ai", "https://cursor.sh"}

@app.middleware("http")
async def validate_origin(request: Request, call_next):
    origin = request.headers.get("origin", "")
    if request.method != "OPTIONS" and origin and origin not in ALLOWED_ORIGINS:
        raise HTTPException(403, "Origin not allowed")
    return await call_next(request)
```

---

## Authentication

- [ ] **OAuth 2.1 + PKCE implemented** — required for all remote servers per MCP spec
- [ ] **No API keys in remote servers** — OAuth tokens only; API keys only for local stdio servers where appropriate
- [ ] **Token validation on every request** — verify signature, expiry, audience, and issuer
- [ ] **user_id from token only** — NEVER accept user_id from request body or tool arguments; always extract from the validated JWT
- [ ] **Minimum required scopes** — define granular scopes; clients request only what they need
- [ ] **Refresh token rotation** — issue new refresh tokens on every use; invalidate old ones
- [ ] **Token revocation endpoint** — users can revoke access immediately
- [ ] **Dynamic client registration** — or a curated allowlist of approved client_ids

```python
# CORRECT: user_id from validated token
async def call_tool_handler(
    request: ToolCallRequest,
    token_payload: dict = Depends(verify_token),  # Validated JWT
) -> ToolCallResult:
    user_id = token_payload["sub"]  # From token — not from request
    ...

# WRONG: user_id from request — attacker can supply any user_id
async def bad_handler(request: ToolCallRequest) -> ToolCallResult:
    user_id = request.arguments["user_id"]  # Never do this
    ...
```

---

## Tool Design Security

- [ ] **Least-privilege tool design** — tools do exactly what they say and nothing more
- [ ] **Separate read and write tools** — never combine into one tool with a `write=True` flag
- [ ] **Mark destructive tools** — set `destructiveHint=True`; clients will show confirmation dialogs
- [ ] **No shell command execution** — if required, allowlist specific commands; never `exec(user_input)`
- [ ] **Path traversal prevention** — validate file paths; resolve against allowed base directories

```python
import os

ALLOWED_BASE = "/home/user/projects"

@mcp.tool
def read_file(path: str) -> str:
    # Resolve and verify the path stays within the allowed directory
    resolved = os.path.realpath(os.path.join(ALLOWED_BASE, path))
    if not resolved.startswith(ALLOWED_BASE + os.sep):
        raise ToolError(f"Access denied: path outside allowed directory")
    with open(resolved) as f:
        return f.read()
```

---

## Input Validation

- [ ] **Validate all tool inputs** — type, range, length, format; reject unexpected fields
- [ ] **Schema `additionalProperties: false`** — don't silently ignore unknown parameters
- [ ] **Sanitize before downstream systems** — never pass raw user/LLM input to SQL, shell, or template engines
- [ ] **Parameterized queries** — SQL injection prevention; never f-string SQL

```python
# WRONG: SQL injection risk
query = f"SELECT * FROM users WHERE name = '{name}'"

# CORRECT: parameterized
query = "SELECT * FROM users WHERE name = $1"
await conn.fetch(query, name)
```

---

## Tool Output Validation (Risk #8)

- [ ] **Scan tool results before returning** — detect and neutralize prompt injection in tool outputs
- [ ] **Label untrusted content** — wrap external/user data so the LLM knows it's untrusted

```python
@mcp.tool
def search_documents(query: str) -> str:
    results = search_engine.query(query)
    # Label results as untrusted before returning to LLM
    safe_results = []
    for doc in results:
        safe_results.append({
            "source": doc["url"],
            "content": f"[UNTRUSTED DOCUMENT CONTENT]\n{doc['text']}\n[END UNTRUSTED]"
        })
    return json.dumps(safe_results)
```

---

## Server Description and Tool Descriptions (Risks #1, #3)

- [ ] **Keep descriptions concise and neutral** — avoid instructions that could be interpreted as commands
- [ ] **Hash your tool schemas** — detect rug pull attacks by comparing current schema hashes to the approved-at-onboarding hashes
- [ ] **Immutable descriptions in production** — tool/resource descriptions should not change post-deployment without a version bump

---

## Secrets Management

- [ ] **No secrets in source code** — use environment variables or a secrets manager
- [ ] **No secrets in Docker images** — use Docker secrets or inject at runtime via env
- [ ] **Rotate API keys regularly** — at least every 90 days; immediately if compromised
- [ ] **Separate keys per environment** — dev, staging, and prod each have different credentials

```python
# WRONG: hardcoded secret
DATABASE_URL = "postgresql://admin:password123@db.example.com/prod"

# CORRECT: from environment
import os
DATABASE_URL = os.environ["DATABASE_URL"]
```

---

## Logging and Data Handling (Risk #10)

- [ ] **Never log tokens or secrets** — mask Authorization headers in access logs
- [ ] **Never log full request bodies** — tool arguments may contain PII
- [ ] **Structured logging** — use JSON logs; include trace_id, tool_name, user_id (hashed if PII), duration
- [ ] **Log audit trail** — who called which tool, when, and what the result was (without the full content)
- [ ] **PII data minimization** — don't return more user data than the caller needs

```python
# Safe audit log — no PII in log values
logger.info("tool_call", extra={
    "tool": tool_name,
    "user_id_hash": hashlib.sha256(user_id.encode()).hexdigest()[:8],
    "duration_ms": duration,
    "success": not result.isError,
    "trace_id": ctx.request_id,
})
```

---

## Rate Limiting

- [ ] **Per-user rate limits** — prevent a single user from overwhelming the server
- [ ] **Per-tool rate limits** — expensive tools (LLM calls, DB queries) get tighter limits
- [ ] **Global rate limits** — protect the server as a whole
- [ ] **Rate limit response** — return HTTP 429 with `Retry-After` header; not a 500

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/mcp")
@limiter.limit("60/minute")
async def mcp_endpoint(request: Request): ...
```

---

## Authorization (Row-Level Security)

- [ ] **Check authorization before calling tools** — don't let the LLM decide if a user can access something
- [ ] **User can only see their own data** — scope every DB query by `user_id` from token
- [ ] **Resource URI includes user scope** — e.g., `db://user_id/data`, not `db://all_data`

```python
@mcp.tool
async def get_my_orders(ctx: Context) -> str:
    user_id = ctx.client_id  # From auth context, not from args
    orders = await db.fetch("SELECT * FROM orders WHERE user_id = $1", user_id)
    return json.dumps(orders)
```

---

## Session Security (Streamable HTTP)

- [ ] **Cryptographically random session IDs** — use `secrets.token_urlsafe(32)` or UUID4
- [ ] **Session expiry** — sessions expire after inactivity (15–60 minutes)
- [ ] **Session invalidation on auth change** — if token is revoked, invalidate the session
- [ ] **Session ID in header only** — never in URL query parameters (appears in logs)

---

## Multi-Server Security (Clients)

- [ ] **Isolate server contexts** — results from Server A should not automatically flow to Server B
- [ ] **Curated server list** — only connect to servers from approved sources
- [ ] **Tool namespace isolation** — prefix tool names by server to prevent collision attacks
- [ ] **Verify server identity** — check `.well-known/mcp.json` matches the expected server

---

## Pre-Deployment Final Check

```
Before go-live, verify:
□ All endpoints use HTTPS
□ Origin header validation active
□ OAuth 2.1 + PKCE implemented and tested
□ user_id extracted from token, not from request
□ All SQL queries use parameterized statements
□ No secrets in code or Docker images
□ Rate limiting active on all endpoints
□ Audit logging writes tool name, user_id hash, trace_id — not PII
□ Input schemas have additionalProperties: false
□ Destructive tools annotated with destructiveHint: true
□ MCP Inspector security scan run (v0.14.1+)
□ OWASP Top 10 items reviewed against this server's tools
```

---

*Sources: [OWASP MCP Top 10](https://owasp.org) | [MCP Security Spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) | Verified: 2026-06-30*
