# MCP Server Deployment Checklist

> Pre-deployment review for taking an MCP server from development to production. Security items are in [09-security-checklist.md](./09-security-checklist.md). This checklist covers operational readiness.

---

## Deployment Targets — Quick Decision

```
Who needs to connect to this server?
├── Only me, on my machine → stdio (no deployment needed)
├── My team, on the same machine → stdio with shared config
├── Anyone with a URL → Streamable HTTP → pick a target:
    ├── No persistent state, burst traffic → Cloud Run / Lambda
    ├── Always-on, full control → VPS (DigitalOcean, Hetzner, Linode)
    └── Enterprise, Kubernetes cluster → Kubernetes + Ingress
```

---

## Pre-Deployment Checklist

### Code and Configuration

- [ ] **Environment variables for all secrets** — `DATABASE_URL`, `API_KEY`, etc. — no hardcoded values
- [ ] **`.env.example` committed** — template with all required variables; actual `.env` in `.gitignore`
- [ ] **Production vs development config** — debug mode OFF in production; log level set to `WARNING` or `ERROR`
- [ ] **Dependencies pinned** — `uv.lock` or `package-lock.json` committed; reproducible builds
- [ ] **Python version pinned** — `.python-version` or `pyproject.toml` `requires-python`

```toml
# pyproject.toml
[project]
name = "my-mcp-server"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "fastmcp>=3.0",
    "fastapi>=0.115",
    "uvicorn>=0.30",
]
```

### Docker Container

- [ ] **Non-root user** — container runs as unprivileged user, not `root`
- [ ] **Read-only filesystem** — if possible; mount only writable directories explicitly
- [ ] **Health check defined** — `HEALTHCHECK` in Dockerfile
- [ ] **No secrets in image layers** — secrets injected at runtime via env, not baked into image
- [ ] **Multi-stage build** — dev dependencies not in production image
- [ ] **Image tagged with version** — never deploy `latest`; use semantic versions or commit SHA

```dockerfile
# Production Dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
# Run as non-root
RUN addgroup --system mcp && adduser --system --group mcp
USER mcp
COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/
ENV PATH="/app/.venv/bin:$PATH"
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
    CMD curl -f http://localhost:8000/health || exit 1
EXPOSE 8000
CMD ["python", "-m", "src.main"]
```

### Health Check Endpoint

```python
# Every production MCP server needs a /health endpoint
@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "version": "1.0.0",
        "checks": {
            "database": await check_db(),
            "cache": await check_cache(),
        }
    }
```

---

## Transport Configuration

### stdio (local)

- [ ] **Server binary installed** — `uv`, `node`, or the script is on the PATH
- [ ] **Config file updated** — `~/.claude/settings.json`, `claude_desktop_config.json`, or Cursor config
- [ ] **Environment variables available** — secrets accessible to the spawned process
- [ ] **Server starts in under 2 seconds** — Claude Code has a startup timeout; slow starts fail silently

### Streamable HTTP (remote)

- [ ] **MCP endpoint at `/mcp`** — standard path clients expect
- [ ] **Supports both POST and GET on `/mcp`** — POST for client→server, GET for server→client SSE
- [ ] **DELETE `/mcp` for session teardown** — optional but recommended
- [ ] **`Mcp-Session-Id` header handled** — stateful sessions use session tracking; stateless servers ignore it
- [ ] **Origin header validated** — prevent DNS rebinding attacks
- [ ] **MCP-Protocol-Version header accepted** — validate and respond appropriately

---

## TLS and Networking

- [ ] **HTTPS everywhere** — no plain HTTP in production
- [ ] **Certificate auto-renewal** — Let's Encrypt with certbot or Caddy handles this automatically
- [ ] **HTTP → HTTPS redirect** — all port 80 traffic redirected to 443
- [ ] **TLS 1.2 minimum** — TLS 1.3 preferred; disable TLS 1.0 and 1.1
- [ ] **Custom domain** — not bare IP; needed for TLS and `.well-known` discovery

```nginx
# nginx TLS config
server {
    listen 443 ssl http2;
    server_name mcp.example.com;
    
    ssl_certificate /etc/letsencrypt/live/mcp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mcp.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    location /mcp {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Connection "";       # HTTP/1.1 keep-alive for SSE
        proxy_buffering off;                  # Required for SSE streaming
        proxy_cache off;
        proxy_read_timeout 300s;              # Long timeout for SSE streams
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /.well-known/ {
        proxy_pass http://127.0.0.1:8000;
    }
}

server {
    listen 80;
    server_name mcp.example.com;
    return 301 https://$host$request_uri;
}
```

---

## .well-known/mcp.json

Every remote MCP server should publish discovery metadata:

```json
{
  "mcp_version": "2025-11-25",
  "server_name": "My MCP Server",
  "server_version": "1.0.0",
  "authorization_servers": [
    { "issuer": "https://auth.example.com" }
  ],
  "scopes_supported": ["tools:read", "tools:write"],
  "documentation_url": "https://docs.example.com/mcp",
  "registry": {
    "smithery": "https://smithery.ai/server/my-server"
  }
}
```

---

## Observability

- [ ] **Structured logging** — JSON logs with trace_id, tool_name, duration_ms, user_id_hash
- [ ] **OpenTelemetry traces** — trace_id propagated across tool calls and downstream API calls
- [ ] **Metrics exported** — tool call count, p99 latency, error rate, active sessions
- [ ] **Alerting configured** — alert on error rate > 1%, p99 latency > 2s, health check failures
- [ ] **Log aggregation** — logs shipped to a central store (Loki, Datadog, CloudWatch)

```python
# FastMCP: OpenTelemetry auto-instrumentation
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
)
trace.set_tracer_provider(provider)
```

---

## Scaling

### Stateless horizontal scaling

```yaml
# docker-compose.yml for local multi-instance testing
services:
  mcp-server:
    image: my-mcp-server:1.0.0
    deploy:
      replicas: 3
    environment:
      - DATABASE_URL=${DATABASE_URL}
    # No session state in the container — state is in PostgreSQL/Redis
```

- [ ] **Session state in external store** — Redis for session tokens, PostgreSQL for user data; nothing in process memory
- [ ] **Load balancer configured** — nginx, AWS ALB, or Cloud Load Balancer
- [ ] **Health check used by load balancer** — unhealthy instances removed automatically
- [ ] **Graceful shutdown** — server handles `SIGTERM` and drains in-flight requests before exit

```python
import signal
import sys

async def graceful_shutdown(sig, loop):
    print(f"Received {sig}, shutting down...", file=sys.stderr)
    # Give in-flight requests 30 seconds to complete
    await asyncio.sleep(1)
    loop.stop()

loop = asyncio.get_event_loop()
loop.add_signal_handler(signal.SIGTERM, lambda: asyncio.create_task(graceful_shutdown("SIGTERM", loop)))
```

---

## Registry Publishing

### Smithery

```bash
# Install Smithery CLI
npm install -g @smithery/cli

# Publish your server
smithery publish \
  --name "my-server" \
  --description "What this server does" \
  --url "https://mcp.example.com" \
  --version "1.0.0" \
  --tags "database,postgresql,read-only"
```

Required for a quality Smithery listing:
- [ ] Clear one-line description
- [ ] Full documentation at `documentation_url`
- [ ] All required environment variables listed with descriptions
- [ ] Example tool calls in the README

---

## Zero-Downtime Deployment

For blue-green deploys (avoid breaking active client sessions):

```bash
# 1. Deploy new version alongside current
docker run -d --name mcp-server-v2 -p 8001:8000 my-mcp-server:1.1.0

# 2. Verify health of new version
curl http://localhost:8001/health

# 3. Test MCP functionality
npx @modelcontextprotocol/inspector http://localhost:8001/mcp

# 4. Update load balancer to route new traffic to v2
# (nginx upstream change, or AWS target group swap)

# 5. Drain v1 sessions (wait for active sessions to complete)
sleep 60

# 6. Stop v1
docker stop mcp-server-v1
```

---

## Version Management

- [ ] **Semantic versioning** — `MAJOR.MINOR.PATCH`; tool schema changes that break clients = MAJOR bump
- [ ] **Version in server info** — `serverInfo.version` returned in initialize response
- [ ] **Changelog maintained** — document breaking changes and new tools per version
- [ ] **Deprecation period** — don't remove tools immediately; mark deprecated for one release cycle

---

## Rollback Plan

Before every production deployment:
- [ ] **Previous Docker image tagged and available** — can redeploy in under 5 minutes
- [ ] **Database migration is reversible** — or no migration this release
- [ ] **Feature flag or environment variable** — can disable new tools without a redeploy
- [ ] **Runbook written** — step-by-step rollback procedure documented

---

## Cloud Run (Serverless) Deployment

```yaml
# cloud-run-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-mcp-server
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"   # Scale to zero when idle
        autoscaling.knative.dev/maxScale: "10"
    spec:
      containers:
        - image: gcr.io/project/my-mcp-server:1.0.0
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            limits:
              cpu: "1"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
```

```bash
gcloud run services replace cloud-run-service.yaml --region us-central1
```

---

## Post-Deployment Verification

Run these checks after every deployment:

```bash
# 1. Health check
curl https://mcp.example.com/health

# 2. Discovery metadata
curl https://mcp.example.com/.well-known/mcp.json

# 3. MCP Inspector end-to-end test
npx @modelcontextprotocol/inspector@latest https://mcp.example.com/mcp

# 4. List tools via curl (basic protocol check)
curl -X POST https://mcp.example.com/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

---

*Verified: 2026-06-30*
