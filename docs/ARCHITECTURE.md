# Deployra — System Architecture Document

> Version 1.0 | March 2026 | Confidential

## Table of Contents

1. [System Overview](#system-overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Core Components](#core-components)
4. [Database Schema](#database-schema)
5. [Infrastructure](#infrastructure)
6. [API Specification](#api-specification)
7. [Auth & Security](#auth--security)

---

## System Overview

Deployra is a managed deployment platform for AI agents. It abstracts away container orchestration, monitoring, cost tracking, and LLM routing — letting developers go from `git push` to production agent in under 60 seconds.

The platform follows a microservices-oriented architecture built on three principles:

1. **Agent Isolation** — Every deployed agent runs in its own container with dedicated resources, network namespace, and secrets scope.
2. **Transparent LLM Interception** — All LLM API calls are routed through Deployra's proxy layer for cost tracking, retry/fallback, and budget enforcement — with zero code changes required from the developer.
3. **Event-Driven Telemetry** — Every agent action, LLM call, and system event is captured as a structured event, streamed through Kafka, and stored in ClickHouse for real-time and historical analysis.

---

## High-Level Architecture

```mermaid
graph TB
    subgraph "Developer"
        CLI[Deployra CLI<br/>TypeScript]
        SDK[deployra.yaml<br/>Config]
    end

    subgraph "Edge Layer"
        CDN[CloudFront CDN]
        ALB[Application Load Balancer]
    end

    subgraph "API Layer"
        GW[API Gateway<br/>Go + Chi Router]
        WS[WebSocket Server<br/>Live Logs]
        AUTH[Auth Service<br/>JWT + GitHub OAuth]
    end

    subgraph "Control Plane"
        ORCH[Orchestrator<br/>Go]
        BUILD[Build Service<br/>Docker + Kaniko]
        SCHED[Scheduler<br/>Agent Lifecycle]
    end

    subgraph "Data Plane"
        RT1[Agent Runtime<br/>ECS Fargate]
        RT2[Agent Runtime<br/>ECS Fargate]
        RT3[Agent Runtime<br/>ECS Fargate]
        PROXY[LLM Proxy<br/>Go Reverse Proxy]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL<br/>Neon)]
        KF[Kafka<br/>MSK Serverless]
        CH[(ClickHouse<br/>ClickHouse Cloud)]
        S3[S3<br/>Artifacts + Logs]
        ECR[ECR<br/>Container Registry]
    end

    subgraph "Dashboard"
        DASH[React Dashboard<br/>Tailwind CSS]
    end

    CLI --> ALB
    DASH --> CDN --> ALB
    ALB --> GW
    ALB --> WS
    GW --> AUTH
    GW --> ORCH
    GW --> PG
    ORCH --> BUILD
    ORCH --> SCHED
    BUILD --> ECR
    SCHED --> RT1
    SCHED --> RT2
    SCHED --> RT3
    RT1 --> PROXY
    RT2 --> PROXY
    RT3 --> PROXY
    PROXY -->|OpenAI/Anthropic/Google| EXT[External LLM APIs]
    PROXY --> KF
    RT1 --> KF
    RT2 --> KF
    RT3 --> KF
    KF --> CH
    WS --> KF
    GW --> S3
```

### Component Interaction Flow — Deploy an Agent

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant CLI as Deployra CLI
    participant API as API Gateway
    participant Build as Build Service
    participant ECR as Container Registry
    participant Orch as Orchestrator
    participant ECS as ECS Fargate
    participant Proxy as LLM Proxy

    Dev->>CLI: deployra deploy
    CLI->>CLI: Read deployra.yaml
    CLI->>CLI: Detect framework (LangChain/CrewAI/AutoGen)
    CLI->>CLI: Generate Dockerfile
    CLI->>API: POST /v1/deployments (source tarball)
    API->>API: Validate JWT + quotas
    API->>Build: Trigger build job
    Build->>Build: docker build (Kaniko)
    Build->>ECR: Push image
    Build->>API: Build complete (image URI)
    API->>Orch: Schedule deployment
    Orch->>ECS: Create Fargate task
    ECS->>ECS: Pull image from ECR
    ECS->>ECS: Start container
    ECS->>Proxy: Register LLM proxy endpoint
    Orch->>API: Deployment live
    API->>CLI: 200 OK {url, agent_id, status}
    CLI->>Dev: ✓ Agent deployed at agent-xyz.deployra.run
```

### Data Flow Diagram

```mermaid
graph LR
    subgraph "Ingestion"
        A[Agent Container] -->|stdout/stderr| F[Fluentd Sidecar]
        A -->|LLM calls| P[LLM Proxy]
        P -->|Token events| K[Kafka]
        F -->|Log events| K
        A -->|Health pings| H[Health Checker]
        H -->|Status events| K
    end

    subgraph "Processing"
        K -->|Raw events| C1[ClickHouse<br/>Raw Events]
        K -->|Materialized views| C2[ClickHouse<br/>Aggregations]
        K -->|Cost events| PG[PostgreSQL<br/>Billing Updates]
    end

    subgraph "Serving"
        C1 --> API[API Gateway]
        C2 --> API
        PG --> API
        API --> DASH[Dashboard]
        API --> WS[WebSocket<br/>Live Stream]
        K --> WS
    end
```

---

## Core Components

### 1. CLI Layer (TypeScript)

The Deployra CLI is a single binary built with TypeScript (compiled via `pkg` for distribution). It handles local project analysis, configuration, and communication with the Deployra API.

#### Commands

| Command | Description | Key Flags |
|---------|-------------|-----------|
| `deployra init` | Initialize a new agent project | `--framework <langchain\|crewai\|autogen\|custom>`, `--name <name>`, `--runtime <python3.11\|python3.12\|node20>` |
| `deployra deploy` | Build and deploy the agent | `--env <staging\|production>`, `--wait`, `--no-cache`, `--dry-run` |
| `deployra logs` | Stream live logs from agent | `--follow`, `--since <duration>`, `--tail <lines>`, `--format <json\|text>` |
| `deployra status` | Show agent status and metrics | `--json`, `--watch` |
| `deployra destroy` | Tear down a deployed agent | `--force`, `--keep-data` |
| `deployra config` | Manage configuration | `set`, `get`, `list` subcommands |
| `deployra login` | Authenticate via GitHub OAuth | `--token <api-key>` for CI/CD |
| `deployra whoami` | Show current user/team | — |

#### Config File: `deployra.yaml`

```yaml
name: my-sales-agent
runtime: python3.12
framework: langchain  # auto-detected if omitted
entrypoint: agent.py

resources:
  cpu: 512        # millicores (256, 512, 1024, 2048, 4096)
  memory: 1024    # MB (512, 1024, 2048, 4096, 8192)

llm:
  provider: openai
  model: gpt-4o
  fallback:
    - provider: anthropic
      model: claude-sonnet-4-20250514
    - provider: google
      model: gemini-1.5-pro
  budget:
    daily_limit_usd: 25.00
    monthly_limit_usd: 500.00
    action: pause  # pause | alert | kill

env:
  OPENAI_API_KEY: ${secrets.OPENAI_API_KEY}
  DATABASE_URL: ${secrets.DATABASE_URL}

scaling:
  min_instances: 1
  max_instances: 5
  target_cpu_percent: 70

health_check:
  path: /health
  interval_seconds: 30
  timeout_seconds: 5
  unhealthy_threshold: 3

triggers:
  - type: http           # HTTP endpoint trigger
    path: /run
  - type: schedule        # Cron trigger
    cron: "0 */6 * * *"
  - type: webhook         # External webhook trigger
    path: /webhook
```

#### Framework Auto-Detection Logic

The CLI detects the agent framework by scanning the project in this order:

1. **Explicit config** — `framework` field in `deployra.yaml` takes priority.
2. **Import scanning** — Parse Python/JS files for framework imports:
   - `from langchain` or `import langchain` → LangChain
   - `from crewai` or `import crewai` → CrewAI
   - `from autogen` or `import autogen` → AutoGen
   - `from llama_index` → LlamaIndex
3. **Dependency file** — Scan `requirements.txt`, `pyproject.toml`, or `package.json` for framework packages.
4. **Fallback** — If no framework detected, treat as custom agent with generic Python/Node runtime.

#### Docker Image Building Pipeline

```
Source Code
    │
    ▼
[1] CLI reads deployra.yaml + detects framework
    │
    ▼
[2] CLI generates optimized Dockerfile
    │  - Base image: python:3.12-slim or node:20-slim
    │  - Multi-stage build (deps first, source second)
    │  - Injects Deployra sidecar (log collector + health reporter)
    │  - Configures LLM proxy env vars
    │
    ▼
[3] CLI creates source tarball (.tar.gz)
    │  - Respects .deployraignore (like .dockerignore)
    │  - Max upload size: 500MB
    │
    ▼
[4] Upload tarball to API → S3 presigned URL
    │
    ▼
[5] Build Service pulls tarball from S3
    │
    ▼
[6] Kaniko builds image (no Docker daemon needed)
    │  - Layer caching via S3
    │  - Build timeout: 300 seconds
    │
    ▼
[7] Push image to ECR (private registry per team)
    │
    ▼
[8] Return image URI to orchestrator
```

---

### 2. API Gateway (Go + Chi Router)

The API Gateway is a stateless Go service behind an Application Load Balancer. It handles all REST API requests, WebSocket connections for live log streaming, and webhook delivery.

#### Architecture

- **Framework:** Go 1.22 + Chi router
- **Auth middleware:** JWT validation (RS256) + API key lookup
- **Rate limiting:** Token bucket per API key (100 req/min free, 1000 req/min pro, 5000 req/min enterprise)
- **Request validation:** Go struct tags with `go-playground/validator`
- **Serialization:** `encoding/json` with custom marshalers
- **Logging:** Structured logging via `slog`
- **Metrics:** Prometheus metrics exported at `/metrics`

#### JWT Auth Flow

```mermaid
sequenceDiagram
    participant User as User/CLI
    participant API as API Gateway
    participant GH as GitHub OAuth
    participant DB as PostgreSQL

    User->>API: GET /auth/github
    API->>GH: Redirect to GitHub OAuth
    GH->>User: GitHub login page
    User->>GH: Authorize
    GH->>API: Callback with auth code
    API->>GH: Exchange code for access token
    GH->>API: Access token + user info
    API->>DB: Upsert user record
    API->>API: Generate JWT (RS256, 24h expiry)
    API->>User: Set-Cookie: token=<jwt>
    Note over User,API: Subsequent requests include JWT
    User->>API: GET /v1/agents (Authorization: Bearer <jwt>)
    API->>API: Validate JWT signature + expiry
    API->>DB: Load user context + team
    API->>User: 200 OK
```

#### WebSocket for Live Log Streaming

```
Client connects: wss://api.deployra.ai/v1/agents/{agent_id}/logs/stream

Auth: JWT token passed as query param ?token=<jwt>

Message format (server → client):
{
  "type": "log",
  "timestamp": "2026-03-10T14:30:00.123Z",
  "level": "info",          // debug, info, warn, error
  "source": "agent",        // agent, system, llm_proxy
  "message": "Processing customer query...",
  "metadata": {
    "run_id": "run_abc123",
    "step": 3
  }
}

Control messages:
{ "type": "ping" }           // Client keepalive (every 30s)
{ "type": "pong" }           // Server response
{ "type": "subscribe", "filters": { "level": "error" } }
{ "type": "unsubscribe" }
```

#### Rate Limiting Strategy

| Tier | Requests/min | Deployments/day | Concurrent Agents | WebSocket Connections |
|------|-------------|-----------------|-------------------|-----------------------|
| Free | 100 | 10 | 2 | 2 |
| Pro | 1,000 | 100 | 20 | 10 |
| Team | 5,000 | 500 | 100 | 50 |
| Enterprise | 20,000 | Unlimited | Unlimited | Unlimited |

Implementation: Redis-backed sliding window rate limiter. Each request decrements a counter keyed by `ratelimit:{api_key}:{window}`. Window size: 60 seconds. Refill: continuous.

---

### 3. Agent Runtime Engine (Go + Docker + ECS Fargate)

The runtime engine manages the lifecycle of deployed agents from container creation to graceful shutdown.

#### Container Lifecycle Management

```mermaid
stateDiagram-v2
    [*] --> Created: API receives deploy request
    Created --> Building: Build service starts
    Building --> Built: Image pushed to ECR
    Built --> Deploying: Fargate task created
    Deploying --> Running: Health check passes
    Running --> Paused: Budget exceeded / Manual pause
    Paused --> Running: Resume command / Budget reset
    Running --> Stopping: Destroy command / Scale down
    Stopping --> Stopped: Graceful shutdown complete
    Running --> Error: Health check fails 3x
    Error --> Running: Auto-restart (max 5 attempts)
    Error --> Stopped: Max restarts exceeded
    Building --> Error: Build timeout (300s)
    Deploying --> Error: Task fails to start
    Stopped --> [*]
```

#### Agent Isolation Model

Each agent runs in complete isolation:

| Isolation Layer | Implementation |
|----------------|----------------|
| **Compute** | Dedicated ECS Fargate task (no shared host) |
| **Network** | AWS VPC security group per agent; no inter-agent communication |
| **Storage** | Ephemeral storage only (20GB default); no shared volumes |
| **Secrets** | AWS Secrets Manager with per-agent IAM policy |
| **DNS** | Unique subdomain: `{agent-id}.deployra.run` |
| **LLM Access** | Isolated proxy path with per-agent budget tracking |
| **Logs** | Separate CloudWatch log group per agent |

#### Health Checking

```
Health Check Protocol:
1. Deployra sidecar sends HTTP GET to agent's health endpoint
   - Default: GET http://localhost:8080/health
   - Configurable in deployra.yaml
   - Interval: 30 seconds (configurable: 10-300s)
   - Timeout: 5 seconds (configurable: 1-30s)

2. Expected response:
   HTTP 200 OK
   {
     "status": "healthy",
     "uptime_seconds": 3847,
     "last_run": "2026-03-10T14:25:00Z",
     "active_tasks": 2
   }

3. Failure handling:
   - 1 failure: Log warning, no action
   - 2 consecutive failures: Alert via webhook
   - 3 consecutive failures: Mark agent as "error", trigger restart
   - 5 total restarts: Mark agent as "stopped", alert user
```

#### Auto-Scaling Strategy

```yaml
# Per-agent scaling (configured in deployra.yaml)
scaling:
  min_instances: 1
  max_instances: 5
  metrics:
    - type: cpu
      target: 70          # Scale up when CPU > 70%
    - type: queue_depth
      target: 10           # Scale up when pending tasks > 10
  scale_up_cooldown: 60    # seconds between scale-up events
  scale_down_cooldown: 300 # seconds between scale-down events
```

Implementation: Custom Go controller that polls CloudWatch metrics every 15 seconds and adjusts ECS desired count. Uses Application Auto Scaling API for Fargate.

#### Resource Limits and Quotas

| Resource | Free Tier | Pro | Team | Enterprise |
|----------|-----------|-----|------|------------|
| CPU per agent | 256 millicores | 2048 | 4096 | Custom |
| Memory per agent | 512 MB | 4096 MB | 8192 MB | Custom |
| Ephemeral storage | 5 GB | 20 GB | 50 GB | Custom |
| Max concurrent agents | 2 | 20 | 100 | Unlimited |
| Max agent uptime | 1 hour | Unlimited | Unlimited | Unlimited |
| Network egress/month | 1 GB | 50 GB | 500 GB | Custom |

---

### 4. LLM Proxy Layer (Go Reverse Proxy)

The LLM Proxy is Deployra's core differentiator. It transparently intercepts all LLM API calls from deployed agents, enabling cost tracking, budget enforcement, retry/fallback, and caching — with zero code changes.

#### How It Works

```mermaid
sequenceDiagram
    participant Agent as Agent Container
    participant Proxy as LLM Proxy
    participant Cache as Redis Cache
    participant Primary as OpenAI API
    participant Fallback as Anthropic API
    participant Kafka as Kafka

    Agent->>Proxy: POST /v1/chat/completions
    Proxy->>Proxy: Validate budget (daily + monthly)
    alt Budget exceeded
        Proxy->>Agent: 429 Budget Exceeded
        Proxy->>Kafka: Emit budget_exceeded event
    end
    Proxy->>Cache: Check prompt cache (SHA-256 hash)
    alt Cache hit
        Cache->>Proxy: Cached response
        Proxy->>Agent: Return cached response
        Proxy->>Kafka: Emit cache_hit event
    end
    Proxy->>Primary: Forward request to OpenAI
    alt Primary fails (5xx or timeout)
        Proxy->>Proxy: Wait 1s (exponential backoff)
        Proxy->>Primary: Retry #1
        alt Still fails
            Proxy->>Fallback: Forward to Anthropic (translate request format)
            Fallback->>Proxy: Response
        end
    end
    Primary->>Proxy: Response with usage data
    Proxy->>Proxy: Count tokens (input + output)
    Proxy->>Proxy: Calculate cost ($)
    Proxy->>Kafka: Emit llm_call event
    Proxy->>Agent: Return response (unmodified)
```

#### Interception Mechanism

The proxy works by injecting environment variables into the agent container at deploy time:

```bash
# Injected into agent container
OPENAI_API_BASE=http://llm-proxy.internal:8443/openai/v1
ANTHROPIC_API_BASE=http://llm-proxy.internal:8443/anthropic/v1
GOOGLE_API_BASE=http://llm-proxy.internal:8443/google/v1

# Original API keys are stored in Secrets Manager
# Proxy injects them when forwarding to upstream providers
```

Most LLM SDKs (OpenAI Python, Anthropic Python, LangChain, etc.) respect `*_API_BASE` environment variables, so this works with zero code changes.

#### Token Counting

```go
// Token counting per request
type TokenUsage struct {
    InputTokens  int     `json:"input_tokens"`
    OutputTokens int     `json:"output_tokens"`
    TotalTokens  int     `json:"total_tokens"`
    CostUSD      float64 `json:"cost_usd"`
    Model        string  `json:"model"`
    Provider     string  `json:"provider"`
    CachedInput  bool    `json:"cached_input"`
}
```

Tokens are extracted from provider response payloads:
- **OpenAI:** `response.usage.prompt_tokens` + `response.usage.completion_tokens`
- **Anthropic:** `response.usage.input_tokens` + `response.usage.output_tokens`
- **Google:** `response.usageMetadata.promptTokenCount` + `response.usageMetadata.candidatesTokenCount`

For streaming responses, the proxy counts tokens from the final `[DONE]` message or sums delta chunks.

#### Cost Calculation Engine

```
Cost = (input_tokens × input_price_per_million / 1,000,000) +
       (output_tokens × output_price_per_million / 1,000,000)
```

**Pricing Table (per million tokens, as of March 2026):**

| Provider | Model | Input $/M | Output $/M |
|----------|-------|-----------|------------|
| OpenAI | gpt-4o | $2.50 | $10.00 |
| OpenAI | gpt-4o-mini | $0.15 | $0.60 |
| OpenAI | gpt-4.1 | $2.00 | $8.00 |
| OpenAI | o3 | $10.00 | $40.00 |
| Anthropic | claude-sonnet-4-20250514 | $3.00 | $15.00 |
| Anthropic | claude-haiku-4-5-20251001 | $0.80 | $4.00 |
| Anthropic | claude-opus-4-20250918 | $15.00 | $75.00 |
| Google | gemini-2.0-flash | $0.10 | $0.40 |
| Google | gemini-2.5-pro | $1.25 | $10.00 |
| Meta | llama-3.3-70b (via Together) | $0.88 | $0.88 |

Prices are stored in a configuration file and updated monthly. The proxy pulls the latest pricing on startup and caches it in memory.

#### Retry Logic with Exponential Backoff

```
Retry policy:
  max_retries: 3
  initial_delay: 1s
  max_delay: 30s
  backoff_multiplier: 2
  jitter: ±25%
  retryable_errors:
    - HTTP 429 (rate limited)
    - HTTP 500, 502, 503, 504 (server errors)
    - Connection timeout
    - Connection reset
  non_retryable_errors:
    - HTTP 400 (bad request)
    - HTTP 401 (auth failure)
    - HTTP 403 (forbidden)

Retry timeline:
  Attempt 1: Immediate
  Attempt 2: ~1s delay (0.75s - 1.25s with jitter)
  Attempt 3: ~2s delay (1.5s - 2.5s with jitter)
  Attempt 4: Fallback to secondary model
```

#### Fallback Chain

```yaml
# User configures in deployra.yaml
llm:
  provider: openai
  model: gpt-4o
  fallback:
    - provider: anthropic
      model: claude-sonnet-4-20250514
    - provider: google
      model: gemini-2.5-pro
```

Fallback triggers:
1. Primary model exhausts all retries (3 attempts)
2. Primary provider returns 401/403 (invalid API key)
3. Primary provider response latency exceeds 30s

When falling back, the proxy translates the request format between providers (e.g., OpenAI format → Anthropic format) using a standardized internal representation.

#### Budget Enforcement

```
Budget check flow (every LLM call):
1. Read agent's daily_spend and monthly_spend from Redis
2. Compare against configured limits in deployra.yaml
3. If limit exceeded:
   a. action: "pause"  → Return 429, pause agent, notify user
   b. action: "alert"  → Allow call but send webhook alert
   c. action: "kill"   → Return 429, stop agent, notify user
4. If within budget:
   a. Forward request to provider
   b. On response, atomically increment spend counters in Redis
   c. Persist to PostgreSQL every 60 seconds (batch writes)

Daily counters reset at midnight UTC.
Monthly counters reset on the 1st of each month at midnight UTC.
```

#### Caching Strategy

- **Key:** SHA-256 hash of `(model, messages, temperature, max_tokens, tools)`
- **Storage:** Redis with 1-hour TTL (configurable per agent)
- **Cache bypass:** Requests with `temperature > 0` are not cached by default (configurable)
- **Cache invalidation:** Manual via API or automatic on TTL expiry
- **Expected hit rate:** 15-30% for typical agent workloads (many agents repeat similar prompts)
- **Cost savings:** Cached responses incur zero LLM cost; only Deployra's cache serving cost applies

---

### 5. Telemetry Pipeline (Kafka → ClickHouse)

#### Event Schema

All events follow a common envelope:

```json
{
  "event_id": "evt_01H8X9K2M3",
  "event_type": "llm_call",
  "timestamp": "2026-03-10T14:30:00.123456Z",
  "agent_id": "agt_abc123",
  "team_id": "team_xyz",
  "deployment_id": "dep_456",
  "run_id": "run_789",
  "data": {
    // Event-specific payload
  }
}
```

**Event Types:**

| Event Type | Description | Data Fields |
|-----------|-------------|-------------|
| `agent_started` | Agent container started | `image_uri`, `resources`, `config` |
| `agent_stopped` | Agent container stopped | `reason`, `exit_code`, `uptime_seconds` |
| `agent_health` | Health check result | `status`, `latency_ms` |
| `llm_call` | LLM API call completed | `provider`, `model`, `input_tokens`, `output_tokens`, `cost_usd`, `latency_ms`, `cached` |
| `llm_error` | LLM API call failed | `provider`, `model`, `error_code`, `error_message`, `retry_count` |
| `budget_alert` | Budget threshold reached | `threshold_percent`, `current_spend`, `limit` |
| `agent_log` | Application log line | `level`, `message`, `source` |
| `deployment_started` | Deploy initiated | `source_hash`, `framework`, `trigger` |
| `deployment_completed` | Deploy finished | `duration_seconds`, `image_size_mb`, `status` |
| `agent_run_started` | Agent task/run began | `trigger_type`, `input_summary` |
| `agent_run_completed` | Agent task/run finished | `duration_seconds`, `steps`, `output_summary`, `total_cost_usd` |

#### Ingestion Pipeline

```
Agent Container / LLM Proxy / System
         │
         ▼
    Kafka Producer (Go)
    - Topic: deployra.events.{event_type}
    - Partitioned by agent_id (ordering guarantee per agent)
    - Compression: lz4
    - Batch size: 1000 messages or 100ms
         │
         ▼
    Kafka (MSK Serverless)
    - 6 partitions per topic
    - Retention: 7 days
    - Replication factor: 3
         │
         ▼
    ClickHouse Consumer
    - Kafka table engine for direct ingestion
    - Materialized views for aggregations
    - Buffer tables for micro-batching (1s flush)
```

#### ClickHouse Schema

```sql
-- Raw events table (ReplacingMergeTree for deduplication)
CREATE TABLE events (
    event_id     String,
    event_type   LowCardinality(String),
    timestamp    DateTime64(6, 'UTC'),
    agent_id     String,
    team_id      String,
    deployment_id String,
    run_id       Nullable(String),
    data         String,  -- JSON blob
    -- Extracted fields for fast queries
    provider     LowCardinality(Nullable(String)),
    model        LowCardinality(Nullable(String)),
    input_tokens Nullable(UInt32),
    output_tokens Nullable(UInt32),
    cost_usd     Nullable(Float64),
    latency_ms   Nullable(UInt32),
    log_level    LowCardinality(Nullable(String))
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (team_id, agent_id, timestamp, event_id)
TTL timestamp + INTERVAL 90 DAY;

-- Hourly cost aggregation (materialized view)
CREATE MATERIALIZED VIEW cost_hourly_mv TO cost_hourly AS
SELECT
    team_id,
    agent_id,
    toStartOfHour(timestamp) AS hour,
    provider,
    model,
    count() AS call_count,
    sum(input_tokens) AS total_input_tokens,
    sum(output_tokens) AS total_output_tokens,
    sum(cost_usd) AS total_cost_usd,
    avg(latency_ms) AS avg_latency_ms,
    quantile(0.95)(latency_ms) AS p95_latency_ms
FROM events
WHERE event_type = 'llm_call'
GROUP BY team_id, agent_id, hour, provider, model;

-- Agent run aggregation
CREATE MATERIALIZED VIEW agent_runs_mv TO agent_runs_agg AS
SELECT
    team_id,
    agent_id,
    toStartOfDay(timestamp) AS day,
    countIf(event_type = 'agent_run_completed') AS total_runs,
    avgIf(
        JSONExtractFloat(data, 'duration_seconds'),
        event_type = 'agent_run_completed'
    ) AS avg_duration_seconds,
    sumIf(cost_usd, event_type = 'llm_call') AS total_cost_usd,
    countIf(event_type = 'llm_error') AS total_errors
FROM events
GROUP BY team_id, agent_id, day;
```

#### Retention Policies

| Data Type | Retention | Storage |
|-----------|-----------|---------|
| Raw events | 90 days | ClickHouse (compressed) |
| Hourly aggregations | 2 years | ClickHouse |
| Daily aggregations | Indefinite | ClickHouse |
| Raw logs | 30 days | S3 (Glacier after 30 days) |
| Build artifacts | 90 days | S3 |

#### Query Patterns for Dashboard

```sql
-- Cost breakdown for last 7 days (agent detail page)
SELECT
    toStartOfDay(hour) AS day,
    provider,
    model,
    sum(total_cost_usd) AS cost
FROM cost_hourly
WHERE agent_id = 'agt_abc123'
  AND hour >= now() - INTERVAL 7 DAY
GROUP BY day, provider, model
ORDER BY day;

-- Error rate for last 24 hours (agent list page)
SELECT
    agent_id,
    countIf(event_type = 'llm_error') / countIf(event_type = 'llm_call') AS error_rate
FROM events
WHERE team_id = 'team_xyz'
  AND timestamp >= now() - INTERVAL 24 HOUR
  AND event_type IN ('llm_call', 'llm_error')
GROUP BY agent_id;
```

---

### 6. Dashboard (React + Tailwind)

#### Tech Stack

- **Framework:** React 19 + TypeScript
- **Styling:** Tailwind CSS v4
- **Bundler:** Vite
- **State management:** TanStack Query (server state) + Zustand (client state)
- **Charts:** Recharts
- **Real-time:** Native WebSocket
- **Routing:** React Router v7

#### Page Structure

| Page | Route | Description |
|------|-------|-------------|
| Login | `/login` | GitHub OAuth login |
| Agent List | `/agents` | All agents with status, cost, run count |
| Agent Detail | `/agents/:id` | Single agent overview with metrics |
| Agent Logs | `/agents/:id/logs` | Live log stream with filters |
| Agent Costs | `/agents/:id/costs` | Cost breakdown by model/day |
| Agent Runs | `/agents/:id/runs` | Run history with details |
| Agent Settings | `/agents/:id/settings` | Config, scaling, budget, env vars |
| Deployments | `/deployments` | Deployment history with status |
| Team Settings | `/settings/team` | Team members, roles, billing |
| API Keys | `/settings/api-keys` | API key management |
| Billing | `/settings/billing` | Plan, usage, invoices |

#### Key Visualizations

1. **Cost Over Time** — Stacked area chart showing daily LLM cost by model/provider. Includes budget line overlay.
2. **Runs Per Hour** — Bar chart showing agent run frequency. Color-coded by success/failure.
3. **Error Rate** — Line chart with error rate percentage. Threshold line at 5%.
4. **Latency Distribution** — Histogram of LLM call latencies with p50/p95/p99 markers.
5. **Token Usage** — Stacked bar chart showing input vs output tokens per model.
6. **Agent Status Grid** — Card grid showing all agents with real-time status indicators (green/yellow/red).

#### Real-Time Updates

```typescript
// WebSocket connection for live updates
const useAgentStream = (agentId: string) => {
  const [logs, setLogs] = useState<LogEntry[]>([]);

  useEffect(() => {
    const ws = new WebSocket(
      `wss://api.deployra.ai/v1/agents/${agentId}/logs/stream?token=${token}`
    );

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      if (msg.type === 'log') {
        setLogs(prev => [...prev.slice(-1000), msg]); // Keep last 1000 logs
      }
    };

    // Heartbeat every 30 seconds
    const heartbeat = setInterval(() => {
      ws.send(JSON.stringify({ type: 'ping' }));
    }, 30000);

    return () => {
      clearInterval(heartbeat);
      ws.close();
    };
  }, [agentId]);

  return logs;
};
```

---

### 7. Database Layer (PostgreSQL via Neon)

#### Complete Schema

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    github_id       BIGINT UNIQUE NOT NULL,
    github_username VARCHAR(255) NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    avatar_url      TEXT,
    display_name    VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at   TIMESTAMPTZ
);

-- Teams
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) UNIQUE NOT NULL,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'pro', 'team', 'enterprise')),
    stripe_customer_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_teams_slug ON teams(slug);

-- Team memberships
CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

-- Agents
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    framework       VARCHAR(50),
    runtime         VARCHAR(50) NOT NULL DEFAULT 'python3.12',
    status          VARCHAR(50) NOT NULL DEFAULT 'created'
                    CHECK (status IN ('created', 'building', 'deploying', 'running',
                                      'paused', 'stopped', 'error')),
    config          JSONB NOT NULL DEFAULT '{}',
    subdomain       VARCHAR(255) UNIQUE,
    current_deployment_id UUID,
    cpu_millicores  INT NOT NULL DEFAULT 256,
    memory_mb       INT NOT NULL DEFAULT 512,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, slug)
);

CREATE INDEX idx_agents_team_id ON agents(team_id);
CREATE INDEX idx_agents_status ON agents(status);

-- Deployments
CREATE TABLE deployments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    version         INT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'building', 'built', 'deploying',
                                      'active', 'rolled_back', 'failed')),
    image_uri       TEXT,
    source_hash     VARCHAR(64),
    build_duration_ms INT,
    deploy_duration_ms INT,
    config_snapshot JSONB NOT NULL,
    error_message   TEXT,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_deployments_agent_id ON deployments(agent_id);
CREATE INDEX idx_deployments_status ON deployments(status);
CREATE UNIQUE INDEX idx_deployments_agent_version ON deployments(agent_id, version);

-- API Keys
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(12) NOT NULL,  -- "dra_" + first 8 chars for identification
    scopes          TEXT[] NOT NULL DEFAULT ARRAY['read', 'write', 'deploy'],
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);

CREATE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_team_id ON api_keys(team_id);

-- Agent Runs
CREATE TABLE agent_runs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    deployment_id   UUID REFERENCES deployments(id),
    trigger_type    VARCHAR(50) NOT NULL
                    CHECK (trigger_type IN ('http', 'schedule', 'webhook', 'manual')),
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'completed', 'failed', 'timeout')),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    duration_ms     INT,
    total_llm_calls INT DEFAULT 0,
    total_tokens    INT DEFAULT 0,
    total_cost_usd  NUMERIC(10, 6) DEFAULT 0,
    input_summary   TEXT,
    output_summary  TEXT,
    error_message   TEXT
);

CREATE INDEX idx_agent_runs_agent_id ON agent_runs(agent_id);
CREATE INDEX idx_agent_runs_started_at ON agent_runs(started_at);

-- Cost Events (aggregated from ClickHouse, synced hourly for billing)
CREATE TABLE cost_events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    model           VARCHAR(100) NOT NULL,
    total_calls     INT NOT NULL DEFAULT 0,
    total_input_tokens BIGINT NOT NULL DEFAULT 0,
    total_output_tokens BIGINT NOT NULL DEFAULT 0,
    total_cost_usd  NUMERIC(10, 6) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_events_team_agent ON cost_events(team_id, agent_id, period_start);

-- Billing
CREATE TABLE billing (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    plan            VARCHAR(50) NOT NULL,
    base_amount_usd NUMERIC(10, 2) NOT NULL DEFAULT 0,
    usage_amount_usd NUMERIC(10, 2) NOT NULL DEFAULT 0,
    total_amount_usd NUMERIC(10, 2) NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'invoiced', 'paid', 'overdue', 'void')),
    stripe_invoice_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_billing_team_id ON billing(team_id, period_start);

-- Secrets (encrypted user LLM API keys)
CREATE TABLE secrets (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id        UUID REFERENCES agents(id) ON DELETE CASCADE,  -- NULL = team-wide
    name            VARCHAR(255) NOT NULL,
    encrypted_value BYTEA NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, agent_id, name)
);

CREATE INDEX idx_secrets_team_agent ON secrets(team_id, agent_id);
```

---

## Auth & Security

### GitHub OAuth Implementation

```
OAuth Flow:
1. CLI/Dashboard redirects to: https://github.com/login/oauth/authorize
   - client_id: DEPLOYRA_GITHUB_CLIENT_ID
   - redirect_uri: https://api.deployra.ai/auth/callback
   - scope: read:user, user:email
   - state: random 32-byte token (CSRF protection)

2. GitHub redirects back with authorization code

3. API server exchanges code for access token:
   POST https://github.com/login/oauth/access_token
   - client_id, client_secret, code

4. API server fetches user info:
   GET https://api.github.com/user (with access token)

5. Upsert user in PostgreSQL

6. Generate JWT:
   - Algorithm: RS256
   - Expiry: 24 hours
   - Payload: { sub: user_id, team_id, role, iat, exp }
   - Signed with platform private key

7. Return JWT to client (cookie for dashboard, JSON for CLI)
```

### API Key Generation and Scoping

```
API Key Format: dra_live_<32 random bytes base62>
Example: dra_live_7kX9mP2qR5vL8nJ3wF6yH1bT4cA0dE

Storage:
- Only the SHA-256 hash of the key is stored in PostgreSQL
- The key prefix (dra_ + first 8 chars) is stored for identification
- The full key is shown once at creation time, never again

Scopes:
- read:agents      - List and view agents
- write:agents     - Create, update, delete agents
- deploy           - Trigger deployments
- read:logs        - View agent logs
- read:billing     - View billing information
- admin            - Full access (team management, API key management)
```

### Multi-Tenant Data Isolation

| Layer | Strategy |
|-------|----------|
| **Database** | All queries include `team_id` in WHERE clause. Row-level security (RLS) enforced via PostgreSQL policies. |
| **API** | Middleware extracts `team_id` from JWT and injects into request context. Every handler receives team-scoped data. |
| **Compute** | Each agent runs in an isolated ECS Fargate task with a unique IAM role. No shared compute. |
| **Network** | Security groups restrict inter-agent communication. Agents can only reach the LLM proxy and external internet. |
| **Secrets** | AWS Secrets Manager with IAM policies scoped to `team_id/agent_id`. |
| **Telemetry** | ClickHouse queries always filter by `team_id`. No cross-team data leakage. |
| **Logs** | Separate CloudWatch log groups per agent: `/deployra/agents/{team_id}/{agent_id}`. |

### Encryption

| Data | At Rest | In Transit |
|------|---------|------------|
| User LLM API keys | AES-256-GCM (application-level encryption) | TLS 1.3 |
| Database | AWS RDS encryption (AES-256) | TLS 1.3 to Neon |
| Agent source code | S3 SSE-S3 | TLS 1.3 |
| Container images | ECR encryption (AES-256) | TLS 1.3 to ECR |
| Kafka messages | MSK encryption at rest | TLS 1.3 in transit |
| ClickHouse data | ClickHouse Cloud encryption | TLS 1.3 |

---

## Infrastructure

### AWS Service Mapping

| Component | AWS Service | Configuration |
|-----------|------------|---------------|
| API Gateway | ECS Fargate + ALB | 2 tasks, 512 CPU / 1GB RAM |
| Build Service | ECS Fargate (on-demand tasks) | 2048 CPU / 4GB RAM per build |
| Agent Runtime | ECS Fargate | Per-agent resource allocation |
| LLM Proxy | ECS Fargate + NLB | 4 tasks, 1024 CPU / 2GB RAM |
| Container Registry | ECR | Private registry per team |
| Database | Neon (PostgreSQL) | Pro plan, auto-scaling |
| Event Streaming | MSK Serverless | Auto-scaling, pay-per-use |
| Telemetry DB | ClickHouse Cloud | Serverless, auto-scaling |
| Cache | ElastiCache Redis | r7g.medium (single node) |
| Object Storage | S3 | Standard + Glacier lifecycle |
| CDN | CloudFront | Dashboard static assets |
| DNS | Route 53 | `*.deployra.ai` + `*.deployra.run` |
| Secrets | Secrets Manager | Per-team/agent secrets |
| Monitoring | CloudWatch | Logs, metrics, alarms |
| CI/CD | GitHub Actions | Build, test, deploy pipelines |
| SSL Certificates | ACM | Wildcard certs, auto-renewal |

### Networking & VPC Design

```
VPC: 10.0.0.0/16

Public Subnets (3 AZs):
  10.0.1.0/24 (us-east-1a) - ALB, NAT Gateway
  10.0.2.0/24 (us-east-1b) - ALB
  10.0.3.0/24 (us-east-1c) - ALB

Private Subnets (3 AZs):
  10.0.10.0/24 (us-east-1a) - API, Build, LLM Proxy
  10.0.11.0/24 (us-east-1b) - API, Build, LLM Proxy
  10.0.12.0/24 (us-east-1c) - API, Build, LLM Proxy

Isolated Subnets (3 AZs):
  10.0.20.0/22 (us-east-1a) - Agent containers (1024 IPs)
  10.0.24.0/22 (us-east-1b) - Agent containers (1024 IPs)
  10.0.28.0/22 (us-east-1c) - Agent containers (1024 IPs)

Security Groups:
  sg-alb:       Inbound 443 from 0.0.0.0/0
  sg-api:       Inbound 8080 from sg-alb
  sg-proxy:     Inbound 8443 from sg-agents
  sg-agents:    Outbound 8443 to sg-proxy, Outbound 443 to 0.0.0.0/0
  sg-redis:     Inbound 6379 from sg-api, sg-proxy
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Deployra Platform
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./... -race -cover
      - run: go vet ./...
      - run: golangci-lint run

  build-api:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
      - uses: aws-actions/amazon-ecr-login@v2
      - run: |
          docker build -t $ECR_REGISTRY/deployra-api:$GITHUB_SHA -f cmd/api/Dockerfile .
          docker push $ECR_REGISTRY/deployra-api:$GITHUB_SHA

  deploy-staging:
    needs: build-api
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: |
          aws ecs update-service --cluster deployra-staging \
            --service api --force-new-deployment

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: |
          aws ecs update-service --cluster deployra-prod \
            --service api --force-new-deployment
```

### Staging vs Production

| Aspect | Staging | Production |
|--------|---------|------------|
| URL | `staging.deployra.ai` | `api.deployra.ai` |
| ECS Cluster | `deployra-staging` | `deployra-prod` |
| Database | Neon staging branch | Neon main branch |
| Agent domain | `*.staging.deployra.run` | `*.deployra.run` |
| Scale | 1 task per service | 2+ tasks per service |
| Monitoring | Basic CloudWatch | CloudWatch + PagerDuty |
| Deploy | Auto on `main` push | Manual approval gate |

### Disaster Recovery

| Scenario | Strategy | RTO | RPO |
|----------|----------|-----|-----|
| Single AZ failure | Multi-AZ deployment, ALB reroutes | 0 min | 0 |
| Database failure | Neon auto-failover + point-in-time recovery | 2 min | 0 |
| Redis failure | ElastiCache auto-failover | 5 min | ~30s |
| Full region failure | DNS failover to us-west-2 (cold standby) | 30 min | 5 min |
| Data corruption | Neon branching for instant rollback | 5 min | 0 |
| Build service failure | Retry with exponential backoff, queue builds | 2 min | 0 |

### Cost Estimation (Monthly)

| Component | 100 Agents | 1,000 Agents | 10,000 Agents |
|-----------|-----------|--------------|---------------|
| ECS Fargate (agents) | $180 | $1,800 | $18,000 |
| ECS Fargate (platform) | $120 | $200 | $500 |
| Neon PostgreSQL | $19 | $69 | $249 |
| ClickHouse Cloud | $50 | $200 | $1,200 |
| MSK Serverless | $30 | $150 | $800 |
| ElastiCache Redis | $50 | $100 | $300 |
| ECR Storage | $5 | $50 | $500 |
| S3 | $5 | $30 | $200 |
| CloudFront | $10 | $30 | $100 |
| Route 53 | $5 | $10 | $20 |
| NAT Gateway | $35 | $100 | $400 |
| Secrets Manager | $5 | $40 | $400 |
| **Total** | **$514** | **$2,779** | **$22,669** |

---

## API Specification

### Base URL

```
Production: https://api.deployra.ai/v1
Staging:    https://staging.deployra.ai/v1
```

### Authentication

All API requests require authentication via one of:

```
# JWT Token (Dashboard, CLI)
Authorization: Bearer <jwt-token>

# API Key (Programmatic access)
Authorization: Bearer dra_live_7kX9mP2qR5vL8nJ3wF6yH1bT4cA0dE
```

### Pagination

All list endpoints use cursor-based pagination:

```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6ImFndF8xMjMifQ==",
    "has_more": true,
    "total": 47
  }
}
```

Query parameters: `?cursor=<cursor>&limit=20` (max limit: 100)

### Error Format

```json
{
  "error": {
    "code": "agent_not_found",
    "message": "Agent with ID 'agt_abc123' not found",
    "status": 404,
    "request_id": "req_xyz789"
  }
}
```

### Error Codes

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `invalid_request` | Malformed request body or parameters |
| 401 | `unauthorized` | Missing or invalid authentication |
| 403 | `forbidden` | Insufficient permissions or scope |
| 404 | `not_found` | Resource does not exist |
| 409 | `conflict` | Resource already exists or state conflict |
| 422 | `validation_error` | Request validation failed |
| 429 | `rate_limited` | Rate limit exceeded |
| 500 | `internal_error` | Unexpected server error |
| 503 | `service_unavailable` | Service temporarily unavailable |

---

### Endpoints

#### Auth

**`GET /auth/github`** — Initiate GitHub OAuth flow

Response: `302 Redirect` to GitHub OAuth page

---

**`GET /auth/callback`** — GitHub OAuth callback

Query params: `code`, `state`

Response:
```json
{
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "user": {
    "id": "usr_abc123",
    "github_username": "developer42",
    "email": "dev@example.com",
    "avatar_url": "https://avatars.githubusercontent.com/u/12345"
  },
  "team": {
    "id": "team_xyz",
    "name": "My Team",
    "slug": "my-team",
    "plan": "pro"
  }
}
```

---

#### Agents

**`POST /v1/agents`** — Create a new agent

Request:
```json
{
  "name": "sales-outreach-agent",
  "framework": "langchain",
  "runtime": "python3.12",
  "config": {
    "cpu_millicores": 512,
    "memory_mb": 1024,
    "llm": {
      "provider": "openai",
      "model": "gpt-4o",
      "budget": {
        "daily_limit_usd": 25.00,
        "monthly_limit_usd": 500.00,
        "action": "pause"
      }
    },
    "scaling": {
      "min_instances": 1,
      "max_instances": 3
    }
  }
}
```

Response: `201 Created`
```json
{
  "id": "agt_abc123",
  "name": "sales-outreach-agent",
  "slug": "sales-outreach-agent",
  "framework": "langchain",
  "runtime": "python3.12",
  "status": "created",
  "subdomain": "sales-outreach-agent-abc123.deployra.run",
  "config": { ... },
  "created_at": "2026-03-10T14:30:00Z",
  "updated_at": "2026-03-10T14:30:00Z"
}
```

---

**`GET /v1/agents`** — List all agents

Query params: `?status=running&cursor=<cursor>&limit=20`

Response: `200 OK`
```json
{
  "data": [
    {
      "id": "agt_abc123",
      "name": "sales-outreach-agent",
      "status": "running",
      "framework": "langchain",
      "current_deployment": {
        "id": "dep_456",
        "version": 3,
        "status": "active",
        "created_at": "2026-03-10T12:00:00Z"
      },
      "metrics": {
        "total_runs_24h": 142,
        "total_cost_24h_usd": 12.47,
        "error_rate_24h": 0.02,
        "avg_latency_ms": 2340
      },
      "created_at": "2026-03-01T10:00:00Z"
    }
  ],
  "pagination": {
    "cursor": "eyJpZCI6ImFndF9hYmMxMjMifQ==",
    "has_more": false,
    "total": 5
  }
}
```

---

**`GET /v1/agents/:id`** — Get agent details

Response: `200 OK` (same schema as create response, with additional `metrics` and `current_deployment` fields)

---

**`PATCH /v1/agents/:id`** — Update agent configuration

Request:
```json
{
  "config": {
    "llm": {
      "budget": {
        "daily_limit_usd": 50.00
      }
    },
    "scaling": {
      "max_instances": 5
    }
  }
}
```

Response: `200 OK` (updated agent object)

---

**`DELETE /v1/agents/:id`** — Delete an agent

Response: `204 No Content`

---

**`POST /v1/agents/:id/pause`** — Pause a running agent

Response: `200 OK` `{ "status": "paused" }`

---

**`POST /v1/agents/:id/resume`** — Resume a paused agent

Response: `200 OK` `{ "status": "running" }`

---

#### Deployments

**`POST /v1/agents/:id/deployments`** — Create a new deployment

Request: `multipart/form-data`
- `source` — tarball of agent source code (max 500MB)
- `config` — JSON string of deployment config overrides

Response: `201 Created`
```json
{
  "id": "dep_789",
  "agent_id": "agt_abc123",
  "version": 4,
  "status": "building",
  "created_at": "2026-03-10T14:35:00Z"
}
```

---

**`GET /v1/agents/:id/deployments`** — List deployment history

Response: `200 OK` (paginated list of deployments)

---

**`GET /v1/agents/:id/deployments/:dep_id`** — Get deployment details

Response: `200 OK`
```json
{
  "id": "dep_789",
  "agent_id": "agt_abc123",
  "version": 4,
  "status": "active",
  "image_uri": "123456789.dkr.ecr.us-east-1.amazonaws.com/agt_abc123:v4",
  "build_duration_ms": 23450,
  "deploy_duration_ms": 18200,
  "source_hash": "a1b2c3d4e5f6...",
  "config_snapshot": { ... },
  "created_at": "2026-03-10T14:35:00Z",
  "completed_at": "2026-03-10T14:35:42Z"
}
```

---

**`POST /v1/agents/:id/deployments/:dep_id/rollback`** — Rollback to a previous deployment

Response: `200 OK`
```json
{
  "id": "dep_790",
  "agent_id": "agt_abc123",
  "version": 5,
  "status": "deploying",
  "rollback_from": "dep_789",
  "rollback_to": "dep_788"
}
```

---

#### Logs

**`GET /v1/agents/:id/logs`** — Get historical logs

Query params: `?since=2026-03-10T00:00:00Z&until=2026-03-10T14:00:00Z&level=error&limit=100`

Response: `200 OK`
```json
{
  "data": [
    {
      "timestamp": "2026-03-10T14:30:00.123Z",
      "level": "error",
      "source": "agent",
      "message": "Failed to connect to database: connection refused",
      "run_id": "run_abc123"
    }
  ],
  "pagination": { ... }
}
```

---

**`WebSocket /v1/agents/:id/logs/stream`** — Live log stream

See WebSocket section above for protocol details.

---

#### Runs

**`GET /v1/agents/:id/runs`** — List agent runs

Response: `200 OK`
```json
{
  "data": [
    {
      "id": "run_abc123",
      "agent_id": "agt_abc123",
      "trigger_type": "http",
      "status": "completed",
      "started_at": "2026-03-10T14:30:00Z",
      "completed_at": "2026-03-10T14:30:47Z",
      "duration_ms": 47000,
      "total_llm_calls": 8,
      "total_tokens": 12450,
      "total_cost_usd": 0.0847
    }
  ]
}
```

---

**`GET /v1/agents/:id/runs/:run_id`** — Get run details with LLM call breakdown

Response: `200 OK`
```json
{
  "id": "run_abc123",
  "status": "completed",
  "duration_ms": 47000,
  "total_cost_usd": 0.0847,
  "llm_calls": [
    {
      "timestamp": "2026-03-10T14:30:02Z",
      "provider": "openai",
      "model": "gpt-4o",
      "input_tokens": 1250,
      "output_tokens": 340,
      "cost_usd": 0.0065,
      "latency_ms": 1820,
      "cached": false
    }
  ],
  "steps": [
    {
      "step": 1,
      "name": "analyze_query",
      "started_at": "2026-03-10T14:30:01Z",
      "duration_ms": 3200
    }
  ]
}
```

---

#### Costs

**`GET /v1/agents/:id/costs`** — Get cost breakdown

Query params: `?period=7d&granularity=hourly`

Response: `200 OK`
```json
{
  "total_cost_usd": 87.34,
  "period": {
    "start": "2026-03-03T00:00:00Z",
    "end": "2026-03-10T00:00:00Z"
  },
  "by_model": [
    { "provider": "openai", "model": "gpt-4o", "cost_usd": 72.10, "calls": 1420 },
    { "provider": "anthropic", "model": "claude-sonnet-4-20250514", "cost_usd": 15.24, "calls": 89 }
  ],
  "timeseries": [
    { "timestamp": "2026-03-03T00:00:00Z", "cost_usd": 12.50 },
    { "timestamp": "2026-03-04T00:00:00Z", "cost_usd": 11.20 }
  ]
}
```

---

#### Secrets

**`POST /v1/agents/:id/secrets`** — Create/update a secret

Request:
```json
{
  "name": "OPENAI_API_KEY",
  "value": "sk-..."
}
```

Response: `201 Created`
```json
{
  "name": "OPENAI_API_KEY",
  "created_at": "2026-03-10T14:30:00Z",
  "updated_at": "2026-03-10T14:30:00Z"
}
```

---

**`GET /v1/agents/:id/secrets`** — List secret names (values never returned)

Response: `200 OK`
```json
{
  "data": [
    { "name": "OPENAI_API_KEY", "updated_at": "2026-03-10T14:30:00Z" },
    { "name": "DATABASE_URL", "updated_at": "2026-03-09T10:00:00Z" }
  ]
}
```

---

**`DELETE /v1/agents/:id/secrets/:name`** — Delete a secret

Response: `204 No Content`

---

#### API Keys

**`POST /v1/api-keys`** — Create an API key

Request:
```json
{
  "name": "CI/CD Pipeline Key",
  "scopes": ["deploy", "read:agents"],
  "expires_in_days": 90
}
```

Response: `201 Created`
```json
{
  "id": "key_abc123",
  "name": "CI/CD Pipeline Key",
  "key": "dra_live_7kX9mP2qR5vL8nJ3wF6yH1bT4cA0dE",
  "key_prefix": "dra_7kX9mP2q",
  "scopes": ["deploy", "read:agents"],
  "expires_at": "2026-06-08T14:30:00Z",
  "created_at": "2026-03-10T14:30:00Z"
}
```

Note: The full `key` is only returned once at creation time.

---

**`GET /v1/api-keys`** — List API keys

**`DELETE /v1/api-keys/:id`** — Revoke an API key

---

#### Webhooks

**`POST /v1/webhooks`** — Create a webhook subscription

Request:
```json
{
  "url": "https://myapp.com/webhooks/deployra",
  "events": ["agent.started", "agent.stopped", "deployment.completed", "budget.exceeded"],
  "secret": "whsec_my_signing_secret"
}
```

Response: `201 Created`

Webhook payload format:
```json
{
  "id": "evt_abc123",
  "type": "deployment.completed",
  "timestamp": "2026-03-10T14:35:42Z",
  "data": {
    "agent_id": "agt_abc123",
    "deployment_id": "dep_789",
    "version": 4,
    "status": "active",
    "duration_seconds": 42
  }
}
```

Webhook delivery: HMAC-SHA256 signature in `X-Deployra-Signature` header. Retry with exponential backoff (3 attempts, 1m/5m/30m).

---

*This document is the single source of truth for Deployra's technical architecture. Updated March 2026.*
