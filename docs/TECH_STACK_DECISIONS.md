# Deployra — Technology Decision Records

> Version 1.0 | March 2026 | Engineering Decisions

Each decision follows the ADR (Architecture Decision Record) format: Context → Options → Decision → Consequences.

---

## ADR-001: Backend Language — Go

### Context

Deployra's backend includes: REST API, WebSocket server, LLM reverse proxy, container orchestrator, and telemetry pipeline. We need a language that's fast, concurrent, and easy to hire for.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Go** | Fast compile/runtime, goroutines for concurrency, excellent net/http stdlib, strong Docker/K8s ecosystem, easy to learn, good hiring pool | Less expressive type system, verbose error handling, no generics until recently |
| **Rust** | Maximum performance, memory safety, zero-cost abstractions, excellent for proxy workloads | Steep learning curve, slower development velocity, smaller hiring pool, longer compile times |
| **Node.js (TypeScript)** | Huge ecosystem, fast prototyping, same language as CLI/frontend, async I/O | Single-threaded (requires clustering), higher memory usage, GC pauses under load, weaker for CPU-bound proxy work |

### Decision

**Go.** The LLM proxy must add <5ms overhead per request while handling thousands of concurrent connections. Go's goroutine model handles this naturally. The standard library's `net/http` and `httputil.ReverseProxy` give us a production-quality proxy foundation on day one. Go also has the strongest ecosystem for container orchestration (Docker, Kubernetes, and the AWS SDK are all Go-native).

### Consequences

- **Positive:** Sub-millisecond proxy overhead achieved. Single binary deployment simplifies Docker images. Strong concurrency for WebSocket connections.
- **Negative:** Frontend team can't share code with backend. Error handling is verbose. Need discipline to avoid Go-specific pitfalls (goroutine leaks, context cancellation).
- **Revisit when:** If we need extensive ML integration on the server side (Python would be better). If compile times become an issue (unlikely at our codebase size).

---

## ADR-002: Database — PostgreSQL (via Neon)

### Context

We need a relational database for user data, agent configs, deployments, billing, and API keys. Multi-tenant data isolation via Row Level Security is a hard requirement.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL (Neon)** | RLS support, JSONB for flexible schemas, mature ecosystem, Neon offers serverless scaling + branching, strong Go drivers | No auto-sharding, Neon is newer (less battle-tested than RDS) |
| **PostgreSQL (AWS RDS)** | Battle-tested, Multi-AZ, automated backups, familiar | Not serverless (always-on cost), no branching, manual scaling |
| **MySQL (PlanetScale)** | Serverless, Vitess-backed sharding, fast | No RLS, weaker JSONB, PlanetScale removed free tier |
| **MongoDB** | Flexible schema, easy horizontal scaling | No ACID transactions for billing, no RLS, schema validation is weaker |

### Decision

**PostgreSQL on Neon.** RLS is non-negotiable for multi-tenant security — it guarantees data isolation at the database level, not just application level. Neon gives us serverless scaling (no paying for idle capacity at pre-launch), database branching (instant staging environments from production data), and a generous free tier for development.

### Consequences

- **Positive:** RLS policies enforce multi-tenancy even if application code has bugs. JSONB stores flexible agent configs without schema migrations. Neon branching enables instant staging/preview environments.
- **Negative:** Neon is less mature than RDS — potential cold-start latency issues. If Neon has problems, migration to RDS is straightforward (standard PostgreSQL).
- **Revisit when:** If Neon cold start P99 exceeds 500ms consistently. If we need to scale beyond Neon's limits (consider CockroachDB or Citus for horizontal sharding).

---

## ADR-003: Event Streaming — Kafka (MSK Serverless)

### Context

We need an event streaming system to pipe telemetry events (LLM calls, agent logs, health checks) from producers (API, proxy, runtime) to consumers (ClickHouse, WebSocket server, billing updater).

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Kafka (MSK Serverless)** | Industry standard, high throughput, durable, exactly-once semantics, ClickHouse has native Kafka consumer, serverless (no cluster management) | More complex than alternatives, MSK Serverless pricing can spike under high throughput |
| **Amazon SQS + SNS** | Simple, fully managed, cheap at low volume, no operational overhead | No streaming (poll-based), no ordering guarantees, ClickHouse doesn't have native SQS consumer, harder to replay events |
| **Redis Streams** | Simple, fast, already have Redis for caching, lightweight | Not durable by default, limited consumer group support, not designed for high-volume event streaming, risk of data loss |
| **Amazon Kinesis** | AWS-native, good ClickHouse integration via Lambda, pay-per-shard | More expensive than MSK Serverless, shard management overhead, less ecosystem tooling than Kafka |

### Decision

**Kafka via MSK Serverless.** Our telemetry pipeline processes thousands of events per second (every LLM call, every log line). Kafka's partitioning by `agent_id` gives us per-agent ordering, which is critical for correct aggregation. ClickHouse's native Kafka table engine enables zero-code ingestion — events flow from Kafka to ClickHouse without any consumer application. MSK Serverless eliminates cluster management while maintaining Kafka's guarantees.

### Consequences

- **Positive:** Native ClickHouse integration eliminates a consumer service. Kafka's durability means we never lose events. Replay capability enables re-processing if ClickHouse schema changes. MSK Serverless scales automatically.
- **Negative:** Kafka adds operational complexity (topic management, consumer lag monitoring). MSK Serverless pricing can be unpredictable at high throughput. Over-engineered for the first few months.
- **Revisit when:** If MSK costs exceed $500/month at pre-revenue stage (consider Redis Streams as interim solution). If we move multi-cloud (consider Confluent Cloud or Redpanda).

---

## ADR-004: Telemetry Database — ClickHouse

### Context

We need a database optimized for storing and querying billions of event records (LLM calls, agent runs, logs). Queries must return in under 200ms for dashboard responsiveness.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **ClickHouse (Cloud)** | Fastest for analytical queries (100x faster than Postgres for aggregations), columnar storage with excellent compression (10-20x), native Kafka consumer, materialized views for pre-aggregation, managed cloud option | Separate system to manage, eventual consistency, not great for point lookups |
| **TimescaleDB** | PostgreSQL-compatible (reuse existing Postgres), time-series optimized, good compression | Slower than ClickHouse for complex aggregations, limited managed cloud options, scaling requires manual partitioning |
| **InfluxDB** | Purpose-built for metrics, simple query language, good retention policies | Limited to time-series (not general analytics), poor for joining with other data, OSS version lacks clustering |
| **PostgreSQL (same DB)** | No new system, simpler architecture, can join with operational data | Terrible performance for analytical queries at scale, partitioning helps but doesn't solve 100M+ row queries, risk of query load impacting operational workloads |

### Decision

**ClickHouse Cloud.** Our dashboard needs sub-200ms responses for queries like "sum of LLM costs by model for the last 30 days across 1000 agents." At 10,000 agents generating 100 events/minute each, that's 1.4 billion events per day. ClickHouse handles this effortlessly with columnar storage and vectorized execution. The native Kafka table engine eliminates a consumer service.

### Consequences

- **Positive:** Dashboard queries return in 20-50ms even at scale. 10x compression means 1TB of events stores in 100GB. Materialized views pre-compute common aggregations. ClickHouse Cloud auto-scales.
- **Negative:** Two database systems to manage (Postgres + ClickHouse). Team needs to learn ClickHouse SQL (mostly compatible but with differences). Cannot join ClickHouse data with PostgreSQL data directly.
- **Revisit when:** If ClickHouse Cloud costs exceed budget (consider self-hosted ClickHouse on ECS). If event volume stays low (under 10M events/day), PostgreSQL with TimescaleDB extension might suffice.

---

## ADR-005: Container Orchestration — ECS Fargate

### Context

We need to run user agent containers in isolated, managed environments. Each agent needs its own container with configurable resources, health checks, and network isolation.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **ECS Fargate** | Serverless containers (no EC2 management), per-task isolation, integrates with ALB/NLB/CloudWatch/IAM, predictable pricing, good Go SDK | AWS-locked, cold start 30-60s, limited to AWS regions, more expensive than EC2 at scale |
| **EKS (Kubernetes)** | Industry standard, portable, massive ecosystem, fine-grained control, potentially cheaper at scale | Enormous operational overhead, control plane costs ($73/mo), requires K8s expertise, overkill for our scale |
| **Lambda** | Instant scaling, pay-per-invocation, zero ops | 15-minute timeout (agents need to run indefinitely), cold starts, limited customization, no long-running containers |
| **Fly.io** | Fast global deployment, good DX, competitive pricing, built-in networking | Less mature than AWS, smaller ecosystem, less enterprise trust, limited IAM/compliance features |
| **Google Cloud Run** | Serverless containers, fast scaling, good pricing, knative-based | Not in AWS ecosystem (adds multi-cloud complexity), less integration with our existing infra |

### Decision

**ECS Fargate.** It's the sweet spot between full Kubernetes and Lambda. Each agent gets an isolated Fargate task with its own CPU, memory, network namespace, and IAM role — without managing EC2 instances or a K8s control plane. The Go AWS SDK provides first-class ECS integration. Fargate's pricing is predictable: $0.04048/vCPU-hour + $0.004445/GB-hour.

### Consequences

- **Positive:** Zero server management. Per-agent isolation is automatic. Integrates seamlessly with ALB, CloudWatch, Secrets Manager. Predictable costs.
- **Negative:** AWS lock-in (our runtime only works on AWS). Cold starts (30-60s for new tasks). More expensive than EC2 at scale (consider switching at 1000+ agents). 4 vCPU / 30 GB max per task.
- **Revisit when:** At 1000+ agents, evaluate ECS on EC2 with Capacity Providers (20-40% cost savings). When we go multi-cloud, add GCP Cloud Run support. If cold starts are a user complaint, consider keeping warm pools.

---

## ADR-006: Frontend — React

### Context

The Deployra dashboard needs to display real-time data (live logs, streaming metrics), handle complex state (multiple agent views, filters), and support WebSocket connections.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **React (Vite + TanStack Query)** | Largest ecosystem, best real-time/WebSocket support, TanStack Query handles server state perfectly, excellent component libraries (Recharts, Tailwind), easiest hiring | Needs additional setup (routing, state management), JSX learning curve for new devs |
| **Next.js** | React + SSR, file-based routing, API routes, Vercel deployment | SSR is unnecessary for a dashboard (authenticated, not SEO-relevant), adds complexity, Vercel lock-in |
| **Svelte (SvelteKit)** | Smaller bundle, faster runtime, simpler reactivity model, enjoyable DX | Smaller ecosystem, fewer chart libraries, harder to hire, less mature for complex real-time apps |

### Decision

**React with Vite, TanStack Query, and Tailwind CSS.** The dashboard is a fully client-side application (no SSR needed — it's behind auth). React's ecosystem is unmatched for dashboards: Recharts for charts, TanStack Query for server state caching, and the WebSocket integration is well-understood. Next.js would add SSR complexity we don't need. Vite provides the fast build/dev experience.

### Consequences

- **Positive:** Massive library ecosystem. Easy to hire React developers. TanStack Query handles caching, revalidation, and optimistic updates. Recharts provides great charts out of the box.
- **Negative:** Client-side only means slower initial load (mitigated by CDN + code splitting). Bundle size requires monitoring. React 19's new features have a learning curve.
- **Revisit when:** If we build a public-facing marketing site (Next.js for SEO). If bundle size exceeds 500KB gzipped (consider lazy loading or framework switch).

---

## ADR-007: Authentication — Custom (GitHub OAuth + JWT)

### Context

We need authentication for the CLI, dashboard, and API. Developer-focused product means GitHub OAuth is the expected flow.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Custom (GitHub OAuth + JWT)** | Full control, simple implementation, no vendor dependency, GitHub OAuth is perfect for our audience, JWT is stateless for API scaling | Must handle session management, token refresh, security best practices ourselves |
| **Auth0** | Feature-rich, SSO/SAML support, social logins, compliance features | $23/mo per 1000 MAU ($2,300/mo at 100K users), vendor lock-in, adds latency to auth flow, complex for simple use case |
| **Clerk** | Beautiful pre-built components, good DX, React components | $25/mo per 10K MAU, less control over auth flow, dependency on their components, overkill for API/CLI auth |
| **Supabase Auth** | Free with Supabase, GoTrue-based, supports GitHub OAuth | Ties us to Supabase ecosystem, we chose Neon for Postgres, running separate Supabase just for auth is awkward |

### Decision

**Custom implementation with GitHub OAuth + JWT.** Our auth needs are straightforward: GitHub OAuth for login, JWT for session tokens, API keys for programmatic access. This is ~200 lines of Go code for the OAuth flow and ~100 lines for JWT middleware. At $0/month vs. $23-25/month per 1000 users (Auth0/Clerk), the savings at scale are significant. We add SAML/SSO via a library when enterprise customers demand it.

### Consequences

- **Positive:** $0 cost. Full control over the auth flow. No vendor dependency. Simple to implement and debug. JWT is stateless — scales horizontally without session stores.
- **Negative:** Must implement security best practices carefully (token rotation, CSRF protection, rate limiting on auth endpoints). SSO/SAML must be built later for enterprise.
- **Revisit when:** When enterprise customers require SAML/SSO (evaluate adding a library vs. switching to Auth0). If we have a security incident related to auth.

---

## ADR-008: LLM Proxy Approach — Transparent Reverse Proxy

### Context

We need to intercept, track, and control all LLM API calls from deployed agents. The approach must work with any LLM SDK without requiring code changes from the developer.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Transparent reverse proxy (env var injection)** | Zero code changes from developer, works with any SDK that respects `*_API_BASE`, intercepts at network level | Some SDKs may not respect base URL override, adds network hop, must handle streaming correctly |
| **SDK wrapper library** | Deep integration, can capture prompts/responses, more metadata | Requires developer to change import statements, framework-specific wrappers needed, maintenance burden per SDK |
| **Network-level interception (iptables/eBPF)** | Truly transparent, works with any SDK, no env vars needed | Complex to implement, hard to debug, security implications, platform-specific |
| **Sidecar proxy (Envoy/HAProxy)** | Battle-tested proxies, rich features, protocol-aware | Config complexity, resource overhead per container, harder to customize for LLM-specific logic |

### Decision

**Transparent reverse proxy via environment variable injection.** We inject `OPENAI_API_BASE`, `ANTHROPIC_API_BASE`, and `GOOGLE_API_BASE` into every agent container. All major LLM SDKs (openai-python, anthropic-python, langchain, crewai) respect these environment variables. Our Go reverse proxy receives the requests, extracts metadata, forwards to the real provider, counts tokens from the response, and emits telemetry events.

### Consequences

- **Positive:** Zero developer effort. Works with LangChain, CrewAI, AutoGen, and any custom code using standard SDKs. Simple to implement (Go's `httputil.ReverseProxy`). Easy to add new providers.
- **Negative:** SDKs that hardcode API URLs won't work (rare, but possible). Adds ~2ms network latency per LLM call. Must handle streaming SSE correctly. Must translate request/response formats for cross-provider fallback.
- **Revisit when:** If a major SDK stops respecting base URL override. If we need to capture prompt content for features like prompt versioning (would require SDK wrapper approach for opted-in users).

---

*Technology decisions are not permanent. Revisit quarterly based on scale, customer feedback, and market changes. Every decision above includes a "revisit when" trigger — use them.*
