# OAuth 2.1 + PKCE Flow for MCP

> All remote MCP servers MUST implement OAuth 2.1 with PKCE. This reference covers the complete flow, the MCP-specific discovery mechanism, and ready-to-use code.

---

## Why OAuth 2.1, Not API Keys?

| Concern | API Keys | OAuth 2.1 + PKCE |
|---------|----------|-----------------|
| Token theft from storage | Catastrophic (keys don't expire) | Limited (tokens expire, refresh tokens rotate) |
| Granular permissions | No | Yes (scopes) |
| User consent | No | Yes |
| Revocation | Manual | Instant (`/revoke` endpoint) |
| Public clients (no client secret) | Insecure | PKCE solves this |
| Dynamic client registration | Not applicable | Supported |
| MCP spec compliance | No | Yes (mandatory for remote servers) |

---

## Complete OAuth 2.1 + PKCE Flow

```
MCP Client                Auth Server              MCP Server
    │                          │                        │
    │  1. Discover auth server │                        │
    ├── GET /.well-known/mcp.json ──────────────────────►│
    │◄── { authorization_servers: [...] } ──────────────┤
    │                          │                        │
    │  2. Discover auth endpoints                        │
    ├── GET /.well-known/oauth-authorization-server ─────►│
    │◄── { auth_endpoint, token_endpoint, ... } ─────────┤
    │                          │                        │
    │  3. Generate PKCE pair   │                        │
    │   code_verifier = random 43-128 chars             │
    │   code_challenge = BASE64URL(SHA256(code_verifier))│
    │                          │                        │
    │  4. Redirect user to authorize                     │
    ├── GET /authorize?client_id=...&code_challenge=... ─►│
    │                   user logs in & consents          │
    │◄── redirect to callback with ?code=auth_code ──────┤
    │                          │                        │
    │  5. Exchange code for tokens                       │
    ├── POST /token { code, code_verifier } ─────────────►│
    │◄── { access_token, refresh_token, expires_in } ─────┤
    │                          │                        │
    │  6. Call MCP server with token                     │
    ├── POST /mcp (Authorization: Bearer <access_token>) ─►│
    │◄── MCP response ───────────────────────────────────┤
```

---

## Step-by-Step Implementation

### Step 1: Server Discovery

```python
import httpx

async def discover_auth_server(mcp_server_url: str) -> dict:
    # MCP servers publish their auth server at /.well-known/mcp.json
    well_known_url = f"{mcp_server_url.rstrip('/')}/.well-known/mcp.json"
    
    async with httpx.AsyncClient() as client:
        response = await client.get(well_known_url)
        mcp_metadata = response.json()
    
    # metadata contains authorization_servers array
    auth_server_url = mcp_metadata["authorization_servers"][0]["issuer"]
    
    # Fetch OAuth metadata from the auth server
    oauth_url = f"{auth_server_url}/.well-known/oauth-authorization-server"
    async with httpx.AsyncClient() as client:
        response = await client.get(oauth_url)
        return response.json()
    # Returns: authorization_endpoint, token_endpoint, scopes_supported, etc.
```

### Step 2: Generate PKCE Pair

```python
import secrets
import hashlib
import base64

def generate_pkce_pair() -> tuple[str, str]:
    # code_verifier: cryptographically random string, 43-128 chars
    code_verifier = secrets.token_urlsafe(64)
    
    # code_challenge: BASE64URL(SHA256(code_verifier))
    digest = hashlib.sha256(code_verifier.encode()).digest()
    code_challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
    
    return code_verifier, code_challenge
```

```typescript
import { randomBytes, createHash } from "crypto";

function generatePkcePair(): { verifier: string; challenge: string } {
  const verifier = randomBytes(64).toString("base64url");
  const challenge = createHash("sha256")
    .update(verifier)
    .digest("base64url");
  return { verifier, challenge };
}
```

### Step 3: Build Authorization URL

```python
from urllib.parse import urlencode

def build_auth_url(
    auth_endpoint: str,
    client_id: str,
    redirect_uri: str,
    code_challenge: str,
    scopes: list[str],
    state: str,  # CSRF protection
) -> str:
    params = {
        "response_type": "code",
        "client_id": client_id,
        "redirect_uri": redirect_uri,
        "code_challenge": code_challenge,
        "code_challenge_method": "S256",
        "scope": " ".join(scopes),
        "state": state,
    }
    return f"{auth_endpoint}?{urlencode(params)}"
```

### Step 4: Exchange Code for Tokens

```python
async def exchange_code(
    token_endpoint: str,
    client_id: str,
    code: str,
    code_verifier: str,
    redirect_uri: str,
) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(token_endpoint, data={
            "grant_type": "authorization_code",
            "client_id": client_id,
            "code": code,
            "code_verifier": code_verifier,  # PKCE verification
            "redirect_uri": redirect_uri,
        })
        tokens = response.json()
    # tokens: { access_token, refresh_token, expires_in, token_type }
    return tokens
```

### Step 5: Use Token in MCP Requests

```python
async def call_mcp_with_auth(
    mcp_url: str,
    access_token: str,
    mcp_message: dict,
) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            mcp_url,
            json=mcp_message,
            headers={
                "Authorization": f"Bearer {access_token}",
                "Accept": "application/json, text/event-stream",
                "MCP-Protocol-Version": "2025-11-25",
            },
        )
        return response.json()
```

### Step 6: Refresh Token Rotation

```python
async def refresh_access_token(
    token_endpoint: str,
    client_id: str,
    refresh_token: str,
) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(token_endpoint, data={
            "grant_type": "refresh_token",
            "client_id": client_id,
            "refresh_token": refresh_token,
        })
        new_tokens = response.json()
    # OAuth 2.1 requires refresh token rotation:
    # new_tokens contains a NEW refresh_token — store it, discard the old one
    return new_tokens
```

---

## Server-Side: Validating Bearer Tokens

```python
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

security = HTTPBearer()

async def verify_token(
    credentials: HTTPAuthorizationCredentials = Security(security),
) -> dict:
    token = credentials.credentials
    try:
        # Verify signature, expiry, audience, issuer
        payload = jwt.decode(
            token,
            key=PUBLIC_KEY,
            algorithms=["RS256"],
            audience="my-mcp-server",
            issuer="https://auth.example.com",
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(401, f"Invalid token: {e}")

# Use in FastAPI route
@app.post("/mcp")
async def mcp_endpoint(
    request: Request,
    token_payload: dict = Depends(verify_token),
):
    user_id = token_payload["sub"]  # Always get user_id from token, never from request body
    ...
```

---

## Dynamic Client Registration

MCP supports OAuth 2.1 Dynamic Client Registration (RFC 7591), allowing clients to register themselves without manual setup:

```python
async def register_client(registration_endpoint: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(registration_endpoint, json={
            "client_name": "My MCP Client",
            "redirect_uris": ["http://localhost:8080/callback"],
            "grant_types": ["authorization_code", "refresh_token"],
            "response_types": ["code"],
            "token_endpoint_auth_method": "none",  # Public client (uses PKCE)
            "scope": "tools:read tools:write",
        })
        return response.json()
    # Returns: { client_id, client_secret (if confidential), ... }
```

---

## .well-known/mcp.json — Server Metadata

Remote MCP servers should publish metadata at this endpoint:

```json
{
  "mcp_version": "2025-11-25",
  "server_name": "My MCP Server",
  "server_version": "1.0.0",
  "authorization_servers": [
    {
      "issuer": "https://auth.example.com"
    }
  ],
  "scopes_supported": [
    "tools:read",
    "tools:write",
    "resources:read"
  ]
}
```

---

## Common OAuth Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `invalid_grant` | Auth code expired or already used | Re-start the auth flow |
| `invalid_client` | Client ID not recognized | Check client registration |
| `invalid_request` | Missing required parameter | Check PKCE fields are present |
| `unauthorized_client` | Client not authorized for this grant type | Check client registration |
| `access_denied` | User denied consent | Handle gracefully in UI |
| HTTP 401 on MCP call | Token expired or invalid | Refresh using refresh_token |
| HTTP 400 on MCP call | `Mcp-Session-Id` mismatch | Re-initialize session |

---

## PKCE Quick Reference

| Parameter | Value |
|-----------|-------|
| `code_challenge_method` | Always `S256` (SHA-256; plain is not allowed in OAuth 2.1) |
| `code_verifier` length | 43–128 characters |
| `code_verifier` charset | `A-Z a-z 0-9 - . _ ~` |
| `code_challenge` | `BASE64URL(SHA256(ASCII(code_verifier)))` — no padding |
| Token lifetime | Typically 1 hour for access tokens |
| Refresh token rotation | Mandatory in OAuth 2.1 |

---

*Sources: [OAuth 2.1 Draft](https://oauth.net/2.1/) | [MCP Auth Spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) | [RFC 7636 PKCE](https://www.rfc-editor.org/rfc/rfc7636) | Verified: 2026-06-30*
