# Deployra — Technical Specification

> Version 1.0 | March 2026 | Engineering Reference

## Table of Contents

1. [CLI Command Reference](#cli-command-reference)
2. [Configuration Specification](#configuration-specification)
3. [Framework Detection Algorithm](#framework-detection-algorithm)
4. [Container Build Pipeline](#container-build-pipeline)
5. [LLM Proxy Protocol](#llm-proxy-protocol)
6. [WebSocket Protocol](#websocket-protocol)
7. [Event Telemetry Schema](#event-telemetry-schema)
8. [Cost Calculation](#cost-calculation)
9. [Rate Limiting](#rate-limiting)
10. [Agent State Machine](#agent-state-machine)
11. [Deployment Rollback](#deployment-rollback)
12. [Zero-Downtime Deployments](#zero-downtime-deployments)
13. [Health Check Protocol](#health-check-protocol)

---

## CLI Command Reference

### Global Flags

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--help` | `-h` | Show help for any command | — |
| `--version` | `-v` | Print CLI version | — |
| `--config` | `-c` | Path to config file | `~/.deployra/config.json` |
| `--team` | `-t` | Team slug to operate on | Current team from config |
| `--json` | — | Output in JSON format | `false` |
| `--verbose` | — | Enable debug logging | `false` |
| `--no-color` | — | Disable colored output | `false` |

### `deployra init`

Initialize a new Deployra project in the current directory.

```
Usage: deployra init [flags]

Flags:
  --name <string>        Agent name (lowercase, alphanumeric, hyphens)
  --framework <string>   Framework: langchain, crewai, autogen, llamaindex, custom
  --runtime <string>     Runtime: python3.11, python3.12, node20, node22
  --template <string>    Start from template: basic, rag, multi-agent, chatbot
  --no-interactive       Skip interactive prompts, use defaults
```

**Behavior:**
1. If `--no-interactive`, uses provided flags or defaults.
2. Otherwise, prompts for each missing value with auto-detected suggestions.
3. Scans directory for framework indicators (imports, dependencies).
4. Generates `deployra.yaml` with detected/selected configuration.
5. Creates `.deployraignore` if not present (copies `.gitignore` + defaults).
6. Prints next steps.

**Example:**
```bash
$ deployra init --name sales-agent
Detected framework: langchain (from requirements.txt)
Detected runtime: python3.12 (from .python-version)

Created deployra.yaml
Created .deployraignore

Next steps:
  1. Review deployra.yaml
  2. Set secrets: deployra config set secrets.OPENAI_API_KEY sk-...
  3. Deploy: deployra deploy
```

---

### `deployra deploy`

Build and deploy the agent to Deployra.

```
Usage: deployra deploy [flags]

Flags:
  --env <string>        Target environment: staging, production (default: production)
  --wait                Wait for deployment to complete (default: true)
  --no-wait             Return immediately after triggering deployment
  --no-cache            Build without Docker layer cache
  --dry-run             Show what would be deployed without deploying
  --message <string>    Deployment message (like a commit message)
  --force               Skip confirmation prompts
  --timeout <duration>  Max wait time for deployment (default: 5m)
```

**Behavior:**
1. Reads `deployra.yaml` from current directory (or `--config` path).
2. Validates configuration (schema validation, resource limits check).
3. Runs framework detection if `framework` not set.
4. Generates optimized Dockerfile for detected framework.
5. Creates source tarball (respecting `.deployraignore`).
6. Uploads tarball via `POST /v1/agents/{id}/deployments` (multipart, S3 presigned URL).
7. Streams build logs in real-time.
8. Waits for deployment health check to pass (unless `--no-wait`).
9. Prints agent URL and status.

**Output:**
```
$ deployra deploy
▸ Uploading source (2.3 MB)... done (1.2s)
▸ Building image... done (18.4s)
  └─ Base: python:3.12-slim
  └─ Dependencies: 47 packages
  └─ Image size: 342 MB
▸ Deploying to production... done (12.1s)
  └─ Health check passed
  └─ Instance count: 1

✓ Agent deployed successfully in 31.7s

  URL:    https://sales-agent-abc123.deployra.run
  Status: running
  Deploy: v4
```

**Dry Run Output:**
```
$ deployra deploy --dry-run
Dry run — no changes will be made

Agent:      sales-agent
Framework:  langchain (auto-detected)
Runtime:    python3.12
Resources:  512 CPU / 1024 MB RAM
LLM:        openai/gpt-4o (fallback: anthropic/claude-sonnet-4-20250514)
Budget:     $25/day, $500/month

Generated Dockerfile:
  FROM python:3.12-slim AS deps
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  FROM python:3.12-slim
  COPY --from=deps /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
  COPY . /app
  WORKDIR /app
  CMD ["python", "agent.py"]

Estimated build time: ~20s
Estimated image size: ~340 MB
```

---

### `deployra logs`

View and stream agent logs.

```
Usage: deployra logs [agent-name] [flags]

Flags:
  --follow, -f          Stream logs in real-time
  --since <duration>    Show logs since duration (e.g., 1h, 30m, 7d)
  --until <timestamp>   Show logs until timestamp (ISO 8601)
  --tail <int>          Number of most recent lines to show (default: 100)
  --level <string>      Filter by level: debug, info, warn, error
  --source <string>     Filter by source: agent, system, llm_proxy
  --run <string>        Filter by run ID
  --format <string>     Output format: text, json (default: text)
  --no-timestamps       Hide timestamps in output
```

**Example:**
```bash
$ deployra logs sales-agent -f --level error
2026-03-10 14:30:01 [ERROR] agent: Failed to parse customer email: invalid format
2026-03-10 14:30:15 [ERROR] llm_proxy: OpenAI rate limited, retrying in 2s...
2026-03-10 14:32:44 [ERROR] agent: Database connection timeout after 5000ms
^C
```

---

### `deployra status`

Show agent status and metrics.

```
Usage: deployra status [agent-name] [flags]

Flags:
  --json          Output in JSON format
  --watch, -w     Refresh every 5 seconds
```

**Output:**
```
$ deployra status sales-agent

Agent: sales-agent
Status: running ●
Uptime: 4d 7h 23m
Deploy: v4 (deployed 2h ago by @developer42)

Resources:
  CPU:    512m (avg 34% utilization)
  Memory: 1024 MB (avg 412 MB used)
  Instances: 1/5

Today's Metrics:
  Runs:       142
  Success:    139 (97.9%)
  Errors:     3
  LLM Calls:  1,247
  Tokens:     2.4M (1.8M in / 0.6M out)
  Cost:       $12.47

Budget:
  Daily:   $12.47 / $25.00 (49.9%)
  Monthly: $187.32 / $500.00 (37.5%)

URL: https://sales-agent-abc123.deployra.run
```

---

### `deployra destroy`

Tear down a deployed agent.

```
Usage: deployra destroy [agent-name] [flags]

Flags:
  --force         Skip confirmation prompt
  --keep-data     Keep telemetry data and logs (delete only compute resources)
```

**Behavior:**
1. Shows agent summary (name, status, cost, data size).
2. Requires confirmation unless `--force` is passed.
3. Stops ECS Fargate task.
4. Removes DNS record.
5. Deletes ECR images (unless `--keep-data`).
6. Marks agent as `stopped` in database.
7. Optionally purges telemetry data.

---

### `deployra login`

Authenticate with Deployra.

```
Usage: deployra login [flags]

Flags:
  --token <string>    Authenticate with API key (for CI/CD)
```

**OAuth Flow:**
1. CLI generates a random state token.
2. Opens browser to `https://api.deployra.ai/auth/github?state={state}&cli=true`.
3. Starts local HTTP server on port 9876 to receive callback.
4. User authenticates on GitHub.
5. API redirects to `http://localhost:9876/callback?token={jwt}`.
6. CLI receives JWT, stores in `~/.deployra/config.json`.
7. Verifies token by calling `GET /v1/me`.

---

## Configuration Specification

### `deployra.yaml` Full Schema

```yaml
# Required fields
name: string                    # Agent name. Lowercase, alphanumeric, hyphens. Max 63 chars.
                                # Regex: ^[a-z][a-z0-9-]{0,62}$

# Optional fields (with defaults)
runtime: string                 # python3.11 | python3.12 | node20 | node22
                                # Default: python3.12

framework: string               # langchain | crewai | autogen | llamaindex | custom
                                # Default: auto-detected

entrypoint: string              # Path to main file
                                # Default: agent.py (Python) | agent.js (Node)

# Resource allocation
resources:
  cpu: integer                  # Millicores: 256 | 512 | 1024 | 2048 | 4096
                                # Default: 512
  memory: integer               # MB: 512 | 1024 | 2048 | 4096 | 8192
                                # Default: 1024
  storage: integer              # Ephemeral storage in GB: 5 | 10 | 20 | 50
                                # Default: 10

# LLM configuration
llm:
  provider: string              # openai | anthropic | google | together | groq
  model: string                 # Model identifier (e.g., gpt-4o, claude-sonnet-4-20250514)
  fallback:                     # Ordered list of fallback providers
    - provider: string
      model: string
  budget:
    daily_limit_usd: number     # Daily budget in USD. Default: 10.00
    monthly_limit_usd: number   # Monthly budget in USD. Default: 100.00
    alert_threshold: number     # Alert at this % of budget. Default: 0.80
    action: string              # pause | alert | kill. Default: pause
  cache:
    enabled: boolean            # Enable response caching. Default: true
    ttl_seconds: integer        # Cache TTL. Default: 3600
    max_temperature: number     # Don't cache above this temperature. Default: 0.0

# Scaling configuration
scaling:
  min_instances: integer        # Minimum running instances. Default: 1
  max_instances: integer        # Maximum instances. Default: 1
  target_cpu_percent: integer   # CPU target for scaling. Default: 70
  scale_up_cooldown: integer    # Seconds between scale-ups. Default: 60
  scale_down_cooldown: integer  # Seconds between scale-downs. Default: 300

# Health check
health_check:
  enabled: boolean              # Enable health checks. Default: true
  path: string                  # Health check endpoint. Default: /health
  port: integer                 # Port to check. Default: 8080
  interval_seconds: integer     # Check interval. Default: 30 (range: 10-300)
  timeout_seconds: integer      # Check timeout. Default: 5 (range: 1-30)
  healthy_threshold: integer    # Consecutive successes to mark healthy. Default: 2
  unhealthy_threshold: integer  # Consecutive failures to mark unhealthy. Default: 3

# Trigger configuration
triggers:
  - type: string                # http | schedule | webhook
    path: string                # HTTP/webhook path. Default: /run
    method: string              # HTTP method. Default: POST
    cron: string                # Cron expression (schedule type only)
    timezone: string            # Timezone for cron. Default: UTC

# Environment variables
env:
  KEY: string                   # Static value
  KEY: ${secrets.SECRET_NAME}   # Reference to stored secret

# Build configuration
build:
  dockerfile: string            # Custom Dockerfile path (overrides auto-generation)
  context: string               # Build context directory. Default: .
  args:                         # Build arguments
    KEY: string
  cache: boolean                # Enable build cache. Default: true

# Networking
networking:
  port: integer                 # Container port. Default: 8080
  public: boolean               # Expose to internet. Default: true
  cors:
    origins:                    # Allowed CORS origins
      - string
    methods:                    # Allowed methods. Default: [GET, POST]
      - string
```

### Validation Rules

| Field | Rule |
|-------|------|
| `name` | Required. Match `^[a-z][a-z0-9-]{0,62}$`. Unique per team. |
| `resources.cpu` | Must be one of: 256, 512, 1024, 2048, 4096. Must not exceed plan limit. |
| `resources.memory` | Must be one of: 512, 1024, 2048, 4096, 8192. Must be ≥ 2× CPU (Fargate requirement). |
| `llm.budget.daily_limit_usd` | Must be > 0 and ≤ 10,000. |
| `scaling.min_instances` | Must be ≥ 0 and ≤ max_instances. |
| `scaling.max_instances` | Must be ≤ plan limit (free: 2, pro: 20, team: 100). |
| `triggers[].cron` | Must be valid cron expression. Minimum interval: 1 minute. |
| `env` | Keys must match `^[A-Z_][A-Z0-9_]*$`. Values max 10KB. |

---

## Framework Detection Algorithm

```
function detectFramework(projectDir):
    // Step 1: Explicit config
    if deployra.yaml has "framework" field:
        return config.framework

    // Step 2: Python import scanning
    pythonFiles = glob(projectDir, "**/*.py")
    for file in pythonFiles:
        imports = extractImports(file)
        if "langchain" in imports or "langchain_core" in imports:
            candidates.add("langchain", confidence=0.9)
        if "crewai" in imports:
            candidates.add("crewai", confidence=0.9)
        if "autogen" in imports or "pyautogen" in imports:
            candidates.add("autogen", confidence=0.9)
        if "llama_index" in imports:
            candidates.add("llamaindex", confidence=0.9)

    // Step 3: Dependency file scanning
    if exists("requirements.txt"):
        deps = parse(requirements.txt)
        if "langchain" in deps: candidates.add("langchain", confidence=0.8)
        if "crewai" in deps: candidates.add("crewai", confidence=0.8)
        if "pyautogen" in deps: candidates.add("autogen", confidence=0.8)

    if exists("pyproject.toml"):
        deps = parse(pyproject.toml, section="dependencies")
        // Same logic as requirements.txt

    if exists("package.json"):
        deps = parse(package.json, field="dependencies")
        if "langchain" in deps: candidates.add("langchain", confidence=0.8)
        if "@langchain/core" in deps: candidates.add("langchain", confidence=0.9)

    // Step 4: Resolution
    if candidates.empty:
        return "custom"

    // If multiple frameworks detected, pick highest confidence
    // If tie, prefer: langchain > crewai > autogen > llamaindex
    return candidates.highest().framework
```

---

## Container Build Pipeline

### Stage 1: Source Preparation (CLI-side)

```
1. Read deployra.yaml
2. Detect framework (if not explicit)
3. Generate Dockerfile:
   a. Select base image based on runtime:
      python3.12 → python:3.12-slim (127MB)
      python3.11 → python:3.11-slim (125MB)
      node20     → node:20-slim (180MB)
      node22     → node:22-slim (185MB)
   b. Generate multi-stage build:
      Stage 1 (deps): Install dependencies only (cached between builds)
      Stage 2 (app): Copy deps + source code
   c. Inject Deployra sidecar:
      - Fluentd log collector
      - Health check agent
      - Prometheus metrics exporter
   d. Set LLM proxy environment variables
   e. Set entrypoint

4. Create .deployraignore if not exists:
   .git/
   .env
   .env.*
   __pycache__/
   *.pyc
   node_modules/
   .venv/
   venv/
   .mypy_cache/
   .pytest_cache/
   *.egg-info/
   dist/
   build/

5. Create tarball:
   tar czf source.tar.gz --exclude-from=.deployraignore .
   Validate: size ≤ 500MB
```

### Stage 2: Upload

```
1. CLI requests presigned S3 URL:
   POST /v1/agents/{id}/deployments
   Response: { upload_url: "https://s3.../presigned", deployment_id: "dep_xxx" }

2. CLI uploads tarball to S3:
   PUT {upload_url} with source.tar.gz
   Content-Type: application/gzip

3. CLI notifies API:
   POST /v1/deployments/{id}/build
```

### Stage 3: Build (Server-side, Kaniko)

```
1. Build Service receives build trigger from API
2. Launches ECS Fargate task with Kaniko executor:
   - Image: gcr.io/kaniko-project/executor:latest
   - CPU: 2048m, Memory: 4096MB
   - Timeout: 300 seconds
   - Mount: S3 source tarball
   - Cache: S3 bucket for layer caching

3. Kaniko builds Docker image:
   a. Pull base image (cached)
   b. Install dependencies (cached if requirements.txt unchanged)
   c. Copy source code
   d. Tag image: {ecr_registry}/{agent_id}:v{version}

4. Push image to ECR

5. Report build result to API:
   - Success: image URI, build duration, image size
   - Failure: error message, build logs
```

### Stage 4: Deploy

```
1. Orchestrator receives deploy trigger
2. Creates ECS Fargate task definition:
   - Main container: agent image from ECR
   - Sidecar container: deployra-sidecar (log collection, health checks)
   - Environment: LLM proxy URL, agent config, secrets references
   - Resources: per deployra.yaml config
   - Networking: assigned security group, public IP (if public: true)

3. Creates/updates ECS service:
   - If new agent: create service with desired count = min_instances
   - If existing agent: update service with new task definition
   - Deployment config: minimum healthy percent = 100%, maximum percent = 200%

4. Waits for task to reach RUNNING state
5. Waits for health check to pass (unhealthy_threshold × interval)
6. Updates DNS (Route 53): {agent-slug}-{short-id}.deployra.run → ALB
7. Marks deployment as "active" in database
8. Marks previous deployment as "rolled_back"
```

### Generated Dockerfile Templates

**LangChain (Python):**
```dockerfile
# Stage 1: Dependencies
FROM python:3.12-slim AS deps
WORKDIR /deps
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/deps/packages -r requirements.txt

# Stage 2: Application
FROM python:3.12-slim
WORKDIR /app

# Copy dependencies
COPY --from=deps /deps/packages /usr/local/lib/python3.12/site-packages

# Copy application source
COPY . /app

# Deployra sidecar env vars (injected at deploy time)
ENV OPENAI_API_BASE=http://llm-proxy.internal:8443/openai/v1
ENV ANTHROPIC_API_BASE=http://llm-proxy.internal:8443/anthropic/v1
ENV PORT=8080

EXPOSE 8080
CMD ["python", "agent.py"]
```

**CrewAI (Python):**
```dockerfile
FROM python:3.12-slim AS deps
WORKDIR /deps
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/deps/packages -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=deps /deps/packages /usr/local/lib/python3.12/site-packages
COPY . /app
ENV OPENAI_API_BASE=http://llm-proxy.internal:8443/openai/v1
ENV PORT=8080
EXPOSE 8080
CMD ["python", "-m", "crewai", "run"]
```

**Custom Node.js:**
```dockerfile
FROM node:20-slim AS deps
WORKDIR /deps
COPY package*.json .
RUN npm ci --production

FROM node:20-slim
WORKDIR /app
COPY --from=deps /deps/node_modules ./node_modules
COPY . /app
ENV OPENAI_API_BASE=http://llm-proxy.internal:8443/openai/v1
ENV PORT=8080
EXPOSE 8080
CMD ["node", "agent.js"]
```

---

## LLM Proxy Protocol

### Request Flow

The proxy operates as a transparent HTTP reverse proxy. It accepts requests in the format of any supported LLM provider and forwards them upstream.

### Supported Endpoints

| Provider | Proxy Path | Upstream URL |
|----------|-----------|-------------|
| OpenAI | `http://llm-proxy.internal:8443/openai/v1/*` | `https://api.openai.com/v1/*` |
| Anthropic | `http://llm-proxy.internal:8443/anthropic/v1/*` | `https://api.anthropic.com/v1/*` |
| Google | `http://llm-proxy.internal:8443/google/v1/*` | `https://generativelanguage.googleapis.com/v1/*` |
| Together | `http://llm-proxy.internal:8443/together/v1/*` | `https://api.together.xyz/v1/*` |
| Groq | `http://llm-proxy.internal:8443/groq/v1/*` | `https://api.groq.com/openai/v1/*` |

### Request Headers

The proxy injects/modifies these headers:

```
X-Deployra-Agent-Id: agt_abc123
X-Deployra-Team-Id: team_xyz
X-Deployra-Run-Id: run_789
Authorization: Bearer <user's actual API key from Secrets Manager>
```

The agent's original `Authorization` header (which contains a Deployra-issued proxy token) is stripped and replaced with the user's actual provider API key.

### Streaming Support

For streaming requests (`stream: true`), the proxy:

1. Forwards the request upstream with streaming enabled.
2. Reads each SSE chunk from the upstream response.
3. Forwards each chunk to the agent immediately (no buffering).
4. Accumulates token counts from delta chunks.
5. On the final `[DONE]` message, emits a `llm_call` event to Kafka with total usage.

### Format Translation

When a fallback triggers cross-provider routing, the proxy translates request/response formats:

**OpenAI → Anthropic Translation:**

```json
// OpenAI format (input)
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello"}
  ],
  "temperature": 0.7,
  "max_tokens": 1000
}

// Anthropic format (translated)
{
  "model": "claude-sonnet-4-20250514",
  "system": "You are a helpful assistant.",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "temperature": 0.7,
  "max_tokens": 1000
}
```

**Anthropic → OpenAI Response Translation:**

```json
// Anthropic response
{
  "content": [{"type": "text", "text": "Hello!"}],
  "usage": {"input_tokens": 12, "output_tokens": 5}
}

// Translated to OpenAI format (returned to agent)
{
  "choices": [{"message": {"role": "assistant", "content": "Hello!"}}],
  "usage": {"prompt_tokens": 12, "completion_tokens": 5, "total_tokens": 17}
}
```

### Error Handling

| Upstream Error | Proxy Behavior |
|---------------|----------------|
| 429 Rate Limited | Retry with backoff (respect `Retry-After` header) |
| 500-504 Server Error | Retry up to 3 times, then fallback |
| 401 Unauthorized | Return 401 to agent with message "Invalid LLM API key" |
| 400 Bad Request | Return 400 to agent (no retry) |
| Timeout (30s) | Retry once, then fallback |
| Connection Error | Retry with backoff, then fallback |

---

## WebSocket Protocol

### Connection

```
URL: wss://api.deployra.ai/v1/agents/{agent_id}/logs/stream
Auth: ?token=<jwt> query parameter
Protocol: Standard WebSocket (RFC 6455)
Compression: permessage-deflate
```

### Message Types (Server → Client)

**Log Event:**
```json
{
  "type": "log",
  "id": "evt_abc123",
  "timestamp": "2026-03-10T14:30:00.123456Z",
  "level": "info",
  "source": "agent",
  "message": "Processing customer query: 'What is my order status?'",
  "metadata": {
    "run_id": "run_789",
    "step": 3,
    "function": "process_query"
  }
}
```

**Status Event:**
```json
{
  "type": "status",
  "timestamp": "2026-03-10T14:30:00Z",
  "agent_status": "running",
  "instances": 2,
  "cpu_percent": 45.2,
  "memory_mb": 412
}
```

**Cost Event:**
```json
{
  "type": "cost",
  "timestamp": "2026-03-10T14:30:00Z",
  "provider": "openai",
  "model": "gpt-4o",
  "input_tokens": 1250,
  "output_tokens": 340,
  "cost_usd": 0.0065
}
```

**Error Event:**
```json
{
  "type": "error",
  "timestamp": "2026-03-10T14:30:00Z",
  "code": "budget_exceeded",
  "message": "Daily budget of $25.00 exceeded. Agent paused.",
  "action_required": true
}
```

### Message Types (Client → Server)

**Ping (keepalive):**
```json
{ "type": "ping" }
```

**Subscribe with filters:**
```json
{
  "type": "subscribe",
  "filters": {
    "level": ["error", "warn"],
    "source": ["agent"],
    "run_id": "run_789"
  }
}
```

**Unsubscribe (reset to all events):**
```json
{ "type": "unsubscribe" }
```

### Connection Lifecycle

```
1. Client connects with JWT token
2. Server validates JWT, extracts user_id and team_id
3. Server verifies user has access to agent_id
4. Server subscribes to Kafka consumer group for agent's events
5. Server sends initial status event
6. Server forwards matching events to client
7. Client sends ping every 30s
8. Server responds with pong
9. If no ping received in 90s, server closes connection
10. On disconnect, server unsubscribes from Kafka
```

---

## Event Telemetry Schema

### Common Envelope

Every event follows this structure:

```json
{
  "event_id": "string",           // Unique event ID (UUID v7 for time-ordering)
  "event_type": "string",         // Event type identifier
  "version": 1,                   // Schema version
  "timestamp": "string",          // ISO 8601 with microsecond precision
  "agent_id": "string",           // Agent that generated the event
  "team_id": "string",            // Team that owns the agent
  "deployment_id": "string",      // Active deployment ID
  "run_id": "string | null",      // Run ID if within a run context
  "environment": "string",        // staging | production
  "data": {}                      // Event-type-specific payload
}
```

### Event Type Catalog

#### `agent.started`
```json
{
  "data": {
    "image_uri": "123456.dkr.ecr.us-east-1.amazonaws.com/agt_abc:v4",
    "cpu_millicores": 512,
    "memory_mb": 1024,
    "framework": "langchain",
    "runtime": "python3.12",
    "instance_id": "i-abc123"
  }
}
```

#### `agent.stopped`
```json
{
  "data": {
    "reason": "user_requested | budget_exceeded | health_check_failed | error | scale_down",
    "exit_code": 0,
    "uptime_seconds": 86400,
    "total_runs": 142,
    "total_cost_usd": 12.47
  }
}
```

#### `llm.call`
```json
{
  "data": {
    "request_id": "req_abc123",
    "provider": "openai",
    "model": "gpt-4o",
    "endpoint": "/v1/chat/completions",
    "input_tokens": 1250,
    "output_tokens": 340,
    "total_tokens": 1590,
    "cost_usd": 0.006525,
    "latency_ms": 1820,
    "status_code": 200,
    "cached": false,
    "retries": 0,
    "fallback_used": false,
    "streaming": true,
    "temperature": 0.7,
    "max_tokens": 1000
  }
}
```

#### `llm.error`
```json
{
  "data": {
    "request_id": "req_abc123",
    "provider": "openai",
    "model": "gpt-4o",
    "error_type": "rate_limited | server_error | auth_error | timeout | connection_error",
    "error_code": 429,
    "error_message": "Rate limit exceeded",
    "retry_count": 3,
    "fallback_triggered": true,
    "fallback_provider": "anthropic",
    "fallback_model": "claude-sonnet-4-20250514"
  }
}
```

#### `agent.run.started`
```json
{
  "data": {
    "trigger_type": "http | schedule | webhook | manual",
    "trigger_path": "/run",
    "input_size_bytes": 2048,
    "input_summary": "Customer query about order #12345"
  }
}
```

#### `agent.run.completed`
```json
{
  "data": {
    "status": "completed | failed | timeout",
    "duration_ms": 47000,
    "steps": 5,
    "llm_calls": 8,
    "total_tokens": 12450,
    "total_cost_usd": 0.0847,
    "output_size_bytes": 1024,
    "output_summary": "Generated response with 3 action items",
    "error_message": null
  }
}
```

#### `budget.alert`
```json
{
  "data": {
    "alert_type": "threshold | exceeded",
    "budget_type": "daily | monthly",
    "threshold_percent": 0.80,
    "current_spend_usd": 20.00,
    "limit_usd": 25.00,
    "action_taken": "alert | pause | kill"
  }
}
```

#### `deployment.started`
```json
{
  "data": {
    "version": 4,
    "source_hash": "a1b2c3d4e5f6...",
    "framework": "langchain",
    "triggered_by": "cli | dashboard | api | rollback",
    "user_id": "usr_abc123"
  }
}
```

#### `deployment.completed`
```json
{
  "data": {
    "version": 4,
    "status": "active | failed",
    "build_duration_ms": 23450,
    "deploy_duration_ms": 18200,
    "total_duration_ms": 41650,
    "image_size_mb": 342,
    "error_message": null
  }
}
```

---

## Cost Calculation

### Formula

```
cost_usd = (input_tokens × input_price_per_million / 1,000,000) +
           (output_tokens × output_price_per_million / 1,000,000)
```

### Multi-Model Pricing Table

Prices per 1 million tokens. Updated monthly from provider pricing pages.

| Provider | Model | Input $/M | Output $/M | Context Window | Notes |
|----------|-------|-----------|------------|----------------|-------|
| OpenAI | gpt-4o | $2.50 | $10.00 | 128K | Default recommended |
| OpenAI | gpt-4o-mini | $0.15 | $0.60 | 128K | Best value |
| OpenAI | gpt-4.1 | $2.00 | $8.00 | 1M | Long context |
| OpenAI | gpt-4.1-mini | $0.40 | $1.60 | 1M | Long context value |
| OpenAI | gpt-4.1-nano | $0.10 | $0.40 | 1M | Ultra-cheap |
| OpenAI | o3 | $10.00 | $40.00 | 200K | Reasoning |
| OpenAI | o3-mini | $1.10 | $4.40 | 200K | Reasoning value |
| OpenAI | o4-mini | $1.10 | $4.40 | 200K | Latest reasoning |
| Anthropic | claude-opus-4-20250918 | $15.00 | $75.00 | 200K | Most capable |
| Anthropic | claude-sonnet-4-20250514 | $3.00 | $15.00 | 200K | Best balance |
| Anthropic | claude-haiku-4-5-20251001 | $0.80 | $4.00 | 200K | Fast + cheap |
| Google | gemini-2.5-pro | $1.25 | $10.00 | 1M | Long context |
| Google | gemini-2.0-flash | $0.10 | $0.40 | 1M | Ultra-fast |
| Google | gemini-2.0-flash-lite | $0.075 | $0.30 | 1M | Cheapest |
| Meta | llama-3.3-70b (Together) | $0.88 | $0.88 | 128K | Open source |
| Meta | llama-3.1-405b (Together) | $3.50 | $3.50 | 128K | Open source large |
| Mistral | mistral-large (Together) | $2.00 | $6.00 | 128K | EU-hosted option |
| Groq | llama-3.3-70b (Groq) | $0.59 | $0.79 | 128K | Ultra-fast inference |

### Example Calculations

**Typical agent run (8 LLM calls, GPT-4o):**
```
Total input: 10,000 tokens
Total output: 3,000 tokens
Cost = (10,000 × $2.50 / 1,000,000) + (3,000 × $10.00 / 1,000,000)
     = $0.025 + $0.030
     = $0.055 per run
```

**Heavy agent run (20 LLM calls, Claude Sonnet):**
```
Total input: 50,000 tokens
Total output: 15,000 tokens
Cost = (50,000 × $3.00 / 1,000,000) + (15,000 × $15.00 / 1,000,000)
     = $0.15 + $0.225
     = $0.375 per run
```

---

## Rate Limiting

### Algorithm: Sliding Window Counter

Implementation uses Redis sorted sets for precise rate limiting:

```
function isRateLimited(apiKey, limit, windowSeconds):
    now = currentTimeMs()
    windowStart = now - (windowSeconds × 1000)
    key = "ratelimit:{apiKey}"

    // Remove expired entries
    ZREMRANGEBYSCORE(key, 0, windowStart)

    // Count current window
    count = ZCARD(key)

    if count >= limit:
        retryAfter = ZRANGE(key, 0, 0)[0] + (windowSeconds × 1000) - now
        return { limited: true, retryAfterMs: retryAfter }

    // Add current request
    ZADD(key, now, "{now}:{random}")
    EXPIRE(key, windowSeconds + 1)

    return { limited: false, remaining: limit - count - 1 }
```

### Rate Limit Headers

Every response includes:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1710086400
```

When rate limited:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 23
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710086400

{
  "error": {
    "code": "rate_limited",
    "message": "Rate limit exceeded. Retry after 23 seconds.",
    "status": 429
  }
}
```

---

## Agent State Machine

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │
┌─────────┐   ┌──────────┐   ┌───────┐   ┌───────────┐   ┌───────────┐
│ created  │──▶│ building │──▶│ built │──▶│ deploying │──▶│  running  │
└─────────┘   └──────────┘   └───────┘   └───────────┘   └───────────┘
                    │                          │               │  │  │
                    │                          │               │  │  │
                    ▼                          ▼               │  │  ▼
               ┌─────────┐              ┌─────────┐           │  │ ┌──────────┐
               │  error   │◀─────────────│  error  │◀──────────┘  │ │  paused  │
               └─────────┘              └─────────┘               │ └──────────┘
                    │                        │                    │      │
                    │                        │                    │      │
                    ▼                        ▼                    ▼      │
               ┌─────────┐            ┌─────────┐          ┌─────────┐  │
               │ stopped │            │ stopped │          │ stopping│  │
               └─────────┘            └─────────┘          └─────────┘  │
                                                                │       │
                                                                ▼       │
                                                           ┌─────────┐  │
                                                           │ stopped │◀─┘
                                                           └─────────┘
```

### State Transitions

| From | To | Trigger | Side Effects |
|------|----|---------|-------------|
| `created` | `building` | Deploy triggered | Build job started |
| `building` | `built` | Build succeeds | Image pushed to ECR |
| `building` | `error` | Build fails or times out (300s) | Error event emitted, user notified |
| `built` | `deploying` | Orchestrator starts ECS task | Fargate task created |
| `deploying` | `running` | Health check passes | DNS updated, deployment marked active |
| `deploying` | `error` | Task fails to start or health check never passes (120s) | Error event, user notified |
| `running` | `paused` | Budget exceeded (action=pause) or manual pause | Fargate task stopped, keep task definition |
| `running` | `stopping` | Destroy command or scale-to-zero | SIGTERM sent to agent |
| `running` | `error` | 3 consecutive health check failures | Auto-restart triggered |
| `running` | `building` | New deploy triggered | Previous deployment preserved until new one is active |
| `paused` | `running` | Resume command or budget reset | Fargate task restarted |
| `paused` | `stopped` | Destroy command | Task definition deleted |
| `stopping` | `stopped` | Graceful shutdown complete or 30s timeout | Resources cleaned up |
| `error` | `running` | Auto-restart succeeds (max 5 attempts) | Restart event logged |
| `error` | `stopped` | Max restart attempts exceeded or manual stop | User notified, cleanup |

---

## Deployment Rollback

### Mechanism

Every deployment creates a new version number (monotonically increasing per agent). The previous deployment's image URI and config are preserved in the `deployments` table.

### Rollback Process

```
1. User triggers rollback:
   CLI: deployra rollback [--to-version N]
   API: POST /v1/agents/{id}/deployments/{dep_id}/rollback

2. System creates a NEW deployment record:
   - version: current_version + 1
   - image_uri: copied from target rollback deployment
   - config_snapshot: copied from target rollback deployment
   - status: "deploying"
   - metadata: { rollback_from: current_dep_id, rollback_to: target_dep_id }

3. ECS task updated with previous image
4. Health check passes → deployment active
5. Previous deployment marked as "rolled_back"

Duration: ~15 seconds (no build step, image already in ECR)
```

### Automatic Rollback

If a new deployment's health check fails after 120 seconds:
1. New deployment marked as "failed"
2. Previous deployment's task definition re-activated
3. DNS remains pointed to previous deployment
4. User notified via webhook/email

---

## Zero-Downtime Deployments

### Strategy: Rolling Update (ECS Default)

```
Configuration:
  deployment_configuration:
    minimum_healthy_percent: 100    # Never go below current capacity
    maximum_percent: 200            # Temporarily double capacity

Timeline:
  T+0s:   New task definition registered
  T+1s:   ECS launches new task(s) alongside old task(s)
  T+15s:  New task reaches RUNNING state
  T+45s:  New task passes health check (healthy_threshold × interval)
  T+46s:  ALB starts routing traffic to new task
  T+47s:  ECS begins draining old task (deregistration_delay: 30s)
  T+77s:  Old task stopped
  T+78s:  Deployment complete

Total time: ~78 seconds
Zero dropped requests: Guaranteed by ALB connection draining
```

### Connection Draining

When the old task is being replaced:
1. ALB stops sending new requests to old task.
2. Old task finishes in-flight requests (30s drain period).
3. If agent has active LLM calls, they complete before shutdown.
4. After drain period, SIGTERM sent to old task.
5. 30s graceful shutdown period for cleanup.
6. SIGKILL if still running.

---

## Health Check Protocol

### Agent-Side Implementation

Agents should expose a health endpoint. If they don't, Deployra's sidecar provides a basic TCP check.

**Recommended endpoint:**
```
GET /health HTTP/1.1
Host: localhost:8080
```

**Expected response (HTTP 200):**
```json
{
  "status": "healthy",
  "uptime_seconds": 3847,
  "version": "v4",
  "last_run_at": "2026-03-10T14:25:00Z",
  "active_tasks": 2,
  "memory_mb": 412,
  "checks": {
    "database": "ok",
    "llm_provider": "ok"
  }
}
```

**Unhealthy response (HTTP 503):**
```json
{
  "status": "unhealthy",
  "reason": "database connection pool exhausted",
  "checks": {
    "database": "error: connection pool exhausted (50/50)",
    "llm_provider": "ok"
  }
}
```

### Sidecar Health Checker

The Deployra sidecar runs alongside every agent and performs health checks:

```
Health Check Loop:
  every {interval_seconds}:
    1. Send HTTP GET to {agent_host}:{port}{path}
    2. Wait up to {timeout_seconds} for response

    if response.status == 200:
        consecutive_failures = 0
        if consecutive_successes >= healthy_threshold:
            mark agent as "healthy"

    if response.status != 200 OR timeout:
        consecutive_successes = 0
        consecutive_failures += 1
        emit health_check_failed event

        if consecutive_failures >= unhealthy_threshold:
            mark agent as "unhealthy"
            trigger restart (if auto_restart enabled)
```

### ALB Health Check (Platform-Level)

In addition to agent health checks, the ALB performs its own health checks:

```
Target: ECS task on port 8080
Path: /health
Interval: 10 seconds
Timeout: 5 seconds
Healthy threshold: 2
Unhealthy threshold: 3
Success codes: 200
```

---

*This specification is the definitive technical reference for Deployra implementation. All implementation must conform to this spec.*
