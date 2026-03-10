# Deployra — MVP Development Roadmap

> Version 1.0 | March 2026 | 12-Week Sprint Plan

## Overview

This roadmap takes Deployra from zero to a functional beta platform in 12 weeks. The plan assumes a single full-stack engineer working full-time (50 productive hours/week). Total estimated effort: 540 hours.

**MVP Definition:** A developer can sign up, deploy a LangChain agent with one command, see live logs and LLM costs in a dashboard, and set budget limits — all in under 5 minutes.

---

## Phase 1: Foundation (Week 1-2)

**Sprint Goal:** Core infrastructure, database, and API skeleton deployed to staging.

**Estimated Hours:** 80h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 1.1 | Set up Go monorepo with `cmd/api`, `cmd/proxy`, `internal/`, `pkg/` structure | 3h | P0 | None | `go build ./...` succeeds, CI runs `go test` |
| 1.2 | Configure GitHub Actions CI pipeline (lint, test, build, Docker) | 4h | P0 | 1.1 | Every push triggers test + build. Docker image pushed to ECR on main merge |
| 1.3 | Set up AWS infrastructure with Terraform: VPC, subnets, security groups, ALB | 8h | P0 | None | `terraform apply` creates networking stack. ALB returns 200 on health endpoint |
| 1.4 | Provision Neon PostgreSQL, apply initial migration | 3h | P0 | None | Database accessible from private subnet. `users` table exists |
| 1.5 | Implement complete database schema (all tables from ARCHITECTURE.md) | 6h | P0 | 1.4 | All 10 tables created with indexes. Migration tool (golang-migrate) configured |
| 1.6 | Build API server skeleton: Chi router, middleware chain, health endpoint | 5h | P0 | 1.1 | `GET /health` returns 200. Structured logging via slog. Graceful shutdown |
| 1.7 | Implement GitHub OAuth flow (login, callback, JWT issuance) | 8h | P0 | 1.5, 1.6 | User can log in via GitHub. JWT issued with 24h expiry. User record created in DB |
| 1.8 | Build JWT validation middleware + API key auth middleware | 5h | P0 | 1.7 | Protected endpoints reject unauthorized requests. API keys validated against DB |
| 1.9 | Deploy API to ECS Fargate (staging) behind ALB | 6h | P0 | 1.3, 1.6 | API reachable at staging.deployra.ai. Auto-deploy on main merge |
| 1.10 | Set up ElastiCache Redis (single node) | 2h | P1 | 1.3 | Redis accessible from private subnet. Connection pool configured in API |
| 1.11 | Implement rate limiting middleware (Redis sliding window) | 4h | P1 | 1.8, 1.10 | Rate limits enforced per API key tier. 429 returned when exceeded |
| 1.12 | Set up ECR private registry with lifecycle policies | 2h | P0 | None | Registry created. Images older than 90 days auto-deleted |
| 1.13 | Configure Route 53 DNS: deployra.ai, api.deployra.ai, *.deployra.run | 3h | P0 | 1.3 | DNS resolves correctly. SSL certificates provisioned via ACM |
| 1.14 | Set up CloudWatch log groups, metrics, and basic alarms | 3h | P1 | 1.9 | API logs visible in CloudWatch. Alarm on 5xx error rate > 5% |
| 1.15 | Implement CRUD endpoints for agents (create, list, get, update, delete) | 8h | P0 | 1.5, 1.8 | All agent CRUD operations work. Proper validation, pagination, error handling |
| 1.16 | Implement CRUD endpoints for API keys (create, list, revoke) | 4h | P0 | 1.5, 1.8 | API keys created with SHA-256 hashing. Full key shown once. Revocation works |
| 1.17 | Write integration tests for auth flow and agent CRUD | 6h | P0 | 1.15, 1.16 | 90%+ test coverage on auth and agent packages. Tests run in CI |

### Risk Flags
- **AWS account limits:** ECS Fargate tasks have default limits. Request limit increase on day 1.
- **Neon cold starts:** Neon serverless can have cold start latency. Monitor and consider provisioned compute if P50 > 100ms.

### Deliverables
- [ ] Staging API live at `staging.deployra.ai`
- [ ] GitHub OAuth login working
- [ ] Agent CRUD API functional
- [ ] CI/CD pipeline running
- [ ] Database schema fully applied

---

## Phase 2: Core Runtime (Week 3-4)

**Sprint Goal:** Deploy a container to ECS Fargate from API call. LLM proxy intercepts and tracks costs.

**Estimated Hours:** 100h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 2.1 | Build Container Orchestrator service: create/update/delete ECS Fargate tasks | 12h | P0 | Phase 1 | API call creates a Fargate task. Task runs and is accessible via ALB |
| 2.2 | Implement agent container lifecycle state machine | 6h | P0 | 2.1 | State transitions (created→building→deploying→running→stopped) tracked in DB |
| 2.3 | Build health check system: sidecar container that pings agent health endpoint | 8h | P0 | 2.1 | Health checks run every 30s. 3 consecutive failures trigger restart |
| 2.4 | Implement agent auto-restart logic (max 5 restarts, exponential backoff) | 4h | P0 | 2.3 | Agent restarts on health check failure. Stops after 5 attempts |
| 2.5 | Build LLM Proxy: Go reverse proxy that intercepts OpenAI/Anthropic/Google calls | 14h | P0 | Phase 1 | Proxy forwards requests to upstream LLM providers. Response returned to agent |
| 2.6 | Implement token counting in LLM proxy (extract from provider responses) | 4h | P0 | 2.5 | Accurate token counts for OpenAI, Anthropic, Google response formats |
| 2.7 | Build cost calculation engine (token × price per model) | 4h | P0 | 2.6 | Cost calculated per request. Pricing table configurable. Matches manual calculation |
| 2.8 | Implement budget enforcement in proxy (daily + monthly limits) | 6h | P0 | 2.7, 1.10 | Agent paused when budget exceeded. 429 returned to agent. Budget counters in Redis |
| 2.9 | Build retry logic in proxy (exponential backoff, 3 attempts) | 4h | P0 | 2.5 | 5xx/429 errors retried. 4xx errors not retried. Backoff: 1s, 2s, 4s |
| 2.10 | Implement fallback chain (primary → secondary → tertiary provider) | 6h | P1 | 2.9 | When primary exhausts retries, request forwarded to fallback. Format translated |
| 2.11 | Build request format translator (OpenAI ↔ Anthropic ↔ Google) | 8h | P1 | 2.10 | LangChain agent sending OpenAI format can fallback to Anthropic seamlessly |
| 2.12 | Implement LLM response caching (Redis, SHA-256 key) | 4h | P2 | 2.5, 1.10 | Identical prompts (temp=0) return cached response. Cache hit rate tracked |
| 2.13 | Deploy LLM proxy to ECS Fargate behind NLB | 4h | P0 | 2.5 | Proxy reachable from agent containers on internal network |
| 2.14 | Build secrets management: encrypt/store/inject user LLM API keys | 6h | P0 | Phase 1 | User API keys stored encrypted (AES-256-GCM). Injected into agent container at runtime |
| 2.15 | Implement per-agent network isolation (security groups) | 4h | P0 | 2.1 | Agents cannot communicate with each other. Only proxy + internet egress allowed |
| 2.16 | Write integration tests: deploy agent → LLM call → cost tracked | 6h | P0 | All above | End-to-end test passes. Agent deployed, makes LLM call, cost appears in DB |

### Risk Flags
- **ECS task startup time:** Fargate cold start can be 30-60s. Monitor. Consider provisioned capacity for frequently-deployed agents.
- **LLM proxy latency:** Proxy adds latency to every LLM call. Target: <5ms overhead. Profile early.
- **Format translation bugs:** OpenAI → Anthropic format translation is complex. Test with real agent workloads.

### Deliverables
- [ ] Agent containers running on ECS Fargate
- [ ] LLM proxy intercepting and tracking all LLM calls
- [ ] Cost calculation working for OpenAI, Anthropic, Google
- [ ] Budget enforcement pausing agents when limits exceeded
- [ ] Retry + fallback chain working
- [ ] Secrets securely stored and injected

---

## Phase 3: CLI + Deploy Flow (Week 5-6)

**Sprint Goal:** Developer can run `deployra deploy` and have their agent running in production in under 60 seconds.

**Estimated Hours:** 90h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 3.1 | Set up TypeScript CLI project (Commander.js, tsup bundler, pkg for binaries) | 4h | P0 | None | `npm run build` produces CLI binary. `deployra --version` works |
| 3.2 | Implement `deployra login` (GitHub OAuth device flow for CLI) | 6h | P0 | 3.1, 1.7 | CLI opens browser, user authenticates, JWT stored in `~/.deployra/config.json` |
| 3.3 | Implement `deployra init` (interactive project setup) | 6h | P0 | 3.1 | Creates `deployra.yaml` with sensible defaults. Prompts for name, framework, runtime |
| 3.4 | Build framework auto-detection (scan imports, deps, infer framework) | 6h | P0 | 3.3 | Correctly detects LangChain, CrewAI, AutoGen from Python imports and requirements.txt |
| 3.5 | Build Dockerfile generator (per framework, multi-stage, optimized) | 8h | P0 | 3.4 | Generated Dockerfile builds successfully for LangChain, CrewAI, AutoGen, custom Python |
| 3.6 | Implement source tarball creation (.deployraignore, max 500MB) | 3h | P0 | 3.1 | Tarball excludes .git, node_modules, __pycache__, .env files. Size validated |
| 3.7 | Implement `deployra deploy` (upload source → trigger build → wait → show URL) | 10h | P0 | 3.5, 3.6, 2.1 | Full deploy flow works. Progress shown in CLI. URL returned on success |
| 3.8 | Build server-side build pipeline: receive tarball → Kaniko build → push to ECR | 12h | P0 | 3.7, 1.12 | Source code built into Docker image. Image pushed to ECR. Build logs streamed |
| 3.9 | Implement `deployra logs` (fetch historical + stream live via WebSocket) | 6h | P0 | 3.1, Phase 2 | Historical logs displayed. `--follow` flag streams live logs via WebSocket |
| 3.10 | Implement `deployra status` (show agent status, metrics summary) | 3h | P0 | 3.1, Phase 2 | Shows: status, uptime, run count, cost today, last error |
| 3.11 | Implement `deployra destroy` (tear down agent with confirmation) | 3h | P0 | 3.1, Phase 2 | Agent stopped, Fargate task deleted, DNS cleaned up. Requires `--force` or confirmation |
| 3.12 | Implement `deployra config set/get/list` | 2h | P1 | 3.1 | Config values stored in `~/.deployra/config.json` |
| 3.13 | Add deploy progress indicator (spinner, build stage, deploy stage) | 3h | P1 | 3.7 | CLI shows: Uploading → Building → Deploying → Live with timing |
| 3.14 | Implement `--dry-run` flag for deploy (show what would happen) | 2h | P1 | 3.7 | Shows: detected framework, generated Dockerfile, estimated resources |
| 3.15 | Test end-to-end: LangChain agent → deploy → running → logs → costs | 6h | P0 | All above | Complete flow works with a real LangChain agent. Under 60s deploy time |
| 3.16 | Test end-to-end: CrewAI agent deploy flow | 4h | P0 | 3.15 | CrewAI agent deploys and runs correctly |
| 3.17 | Test end-to-end: AutoGen agent deploy flow | 4h | P1 | 3.15 | AutoGen agent deploys and runs correctly |
| 3.18 | Build NPM/Homebrew distribution pipeline | 4h | P1 | 3.1 | `npm install -g @deployra/cli` and `brew install deployra` work |

### Risk Flags
- **Build time:** Kaniko builds can be slow without layer caching. Implement S3-based cache from day 1.
- **Large dependencies:** ML frameworks (torch, transformers) can make Docker images 5GB+. Use slim base images and multi-stage builds.
- **WebSocket reliability:** Live log streaming must handle reconnects gracefully. Implement auto-reconnect in CLI.

### Deliverables
- [ ] CLI installable via npm and Homebrew
- [ ] `deployra deploy` works end-to-end in under 60 seconds
- [ ] LangChain, CrewAI, and AutoGen frameworks supported
- [ ] Live log streaming working
- [ ] Framework auto-detection accurate

---

## Phase 4: Dashboard + Telemetry (Week 7-8)

**Sprint Goal:** Real-time dashboard showing agent status, logs, costs, and runs.

**Estimated Hours:** 100h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 4.1 | Set up React + Vite + TypeScript + Tailwind project | 3h | P0 | None | Dev server running. Tailwind configured. TypeScript strict mode |
| 4.2 | Build auth flow: GitHub OAuth login page → redirect → JWT cookie | 4h | P0 | 4.1, 1.7 | User logs in via GitHub. Session persisted. Redirect to /agents |
| 4.3 | Build shared layout: sidebar nav, header, content area | 4h | P0 | 4.1 | Responsive layout. Sidebar collapses on mobile. Dark mode support |
| 4.4 | Build Agent List page (/agents) | 8h | P0 | 4.3, 1.15 | Shows all agents with status badge, last deploy time, 24h cost, run count |
| 4.5 | Build Agent Detail page (/agents/:id) | 8h | P0 | 4.4 | Overview: status, config, metrics cards, recent runs, cost chart |
| 4.6 | Build Agent Logs page with live streaming (/agents/:id/logs) | 10h | P0 | 4.5 | Log viewer with level filters, search, auto-scroll. WebSocket live streaming |
| 4.7 | Build Agent Costs page (/agents/:id/costs) | 8h | P0 | 4.5 | Cost breakdown by model/provider. Daily/weekly/monthly chart. Budget line overlay |
| 4.8 | Build Agent Runs page (/agents/:id/runs) | 6h | P0 | 4.5 | Run history table. Click into run detail: LLM calls, steps, duration, cost |
| 4.9 | Build Agent Settings page (/agents/:id/settings) | 6h | P1 | 4.5 | Edit config, scaling, budget, env vars. Manage secrets (add/delete) |
| 4.10 | Set up Kafka (MSK Serverless) with event topics | 6h | P0 | Phase 2 | Topics created: events.llm_call, events.agent_log, events.agent_run, etc. |
| 4.11 | Implement Kafka producers in API, Proxy, and Runtime services | 6h | P0 | 4.10 | All events emitted to Kafka. lz4 compression. Batch producer |
| 4.12 | Set up ClickHouse Cloud with event tables and materialized views | 6h | P0 | 4.10 | Tables created per ARCHITECTURE.md schema. Kafka ingestion working |
| 4.13 | Build API endpoints for telemetry queries (costs, runs, timeseries) | 8h | P0 | 4.12 | Dashboard API endpoints return data from ClickHouse. <200ms latency |
| 4.14 | Implement WebSocket server for live log streaming to dashboard | 6h | P0 | 4.10 | Dashboard receives live logs via WebSocket. Kafka consumer → WebSocket fan-out |
| 4.15 | Build Recharts visualizations: cost chart, runs chart, latency histogram | 6h | P0 | 4.7, 4.8, 4.13 | Charts render correctly. Responsive. Tooltips with data |
| 4.16 | Deploy dashboard to S3 + CloudFront | 3h | P0 | 4.1 | Dashboard live at app.deployra.ai. Auto-deploy on main merge |
| 4.17 | Build Deployment History page (/deployments) | 4h | P1 | 4.3 | Lists all deployments across agents. Status, duration, triggered by |

### Risk Flags
- **ClickHouse query performance:** Complex aggregation queries can be slow. Pre-aggregate with materialized views.
- **WebSocket scaling:** Fan-out to many dashboard clients needs careful memory management. Use connection pooling.
- **Kafka consumer lag:** Monitor consumer lag. If ClickHouse ingestion falls behind, events will be delayed in dashboard.

### Deliverables
- [ ] Dashboard live at `app.deployra.ai`
- [ ] Real-time agent status, logs, costs visible
- [ ] Telemetry pipeline (Kafka → ClickHouse) processing events
- [ ] All 5 chart types rendering with real data
- [ ] Live log streaming in dashboard

---

## Phase 5: Auth + Multi-tenancy (Week 9-10)

**Sprint Goal:** Multi-team support with proper data isolation, role-based access, and billing integration.

**Estimated Hours:** 80h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 5.1 | Implement team creation and management (create, invite, remove members) | 8h | P0 | Phase 1 | Users can create teams, invite members by GitHub username, assign roles |
| 5.2 | Implement role-based access control (owner, admin, member, viewer) | 6h | P0 | 5.1 | Viewers cannot deploy. Members cannot manage team settings. Admins can do everything except transfer ownership |
| 5.3 | Add PostgreSQL Row Level Security (RLS) policies | 6h | P0 | 5.1 | All queries filtered by team_id. Cross-team data access impossible at DB level |
| 5.4 | Implement team switching in CLI and dashboard | 4h | P0 | 5.1 | Users with multiple teams can switch context. All operations scoped to active team |
| 5.5 | Build team settings page in dashboard (members, roles, billing) | 6h | P0 | 5.1, 4.3 | Team management UI. Invite flow. Role assignment. Remove member |
| 5.6 | Integrate Stripe: create customer, manage subscription, usage-based billing | 10h | P0 | 5.1 | Stripe customer created on team creation. Plan upgrades/downgrades work |
| 5.7 | Build billing page in dashboard (plan, usage, invoices) | 6h | P0 | 5.6 | Shows current plan, usage meters, invoice history, upgrade CTA |
| 5.8 | Implement plan-based quotas enforcement (agent count, CPU, memory) | 6h | P0 | 5.6 | Free tier limited to 2 agents. Pro to 20. Team to 100. Enforced at API level |
| 5.9 | Build usage metering: track compute hours + LLM costs per team | 6h | P0 | 5.6, 4.12 | Usage metrics sent to Stripe for usage-based billing. Accurate to the minute |
| 5.10 | Implement API key scoping per team | 4h | P0 | 5.1, 1.16 | API keys scoped to team. Cannot access other team's resources |
| 5.11 | Build onboarding flow (first-time user experience) | 6h | P1 | 5.1, 4.3 | New user guided through: create team → deploy first agent → view dashboard |
| 5.12 | Implement audit log (who did what, when) | 4h | P1 | 5.1 | All team actions logged: deploys, config changes, member changes |
| 5.13 | Build API Keys management page in dashboard | 4h | P0 | 5.10, 4.3 | Create, list, revoke API keys. Show last used date |
| 5.14 | Write security tests: cross-team access, privilege escalation attempts | 4h | P0 | 5.3 | No cross-team data leakage. Viewer cannot deploy. Member cannot delete team |

### Risk Flags
- **Stripe webhook reliability:** Use idempotency keys. Handle webhook retries. Verify signatures.
- **RLS performance:** Row Level Security can impact query performance. Benchmark with 1000+ teams.
- **Team migration:** If a user needs to move an agent between teams, this is complex. Defer to post-MVP.

### Deliverables
- [ ] Multi-team support with role-based access
- [ ] PostgreSQL RLS enforcing data isolation
- [ ] Stripe billing integration (subscription + usage-based)
- [ ] Onboarding flow for new users
- [ ] API key management scoped per team

---

## Phase 6: Polish + Beta (Week 11-12)

**Sprint Goal:** Harden error handling, polish UX, prepare for beta launch with 20 early users.

**Estimated Hours:** 90h

### Tasks

| # | Task | Hours | Priority | Dependencies | Acceptance Criteria |
|---|------|-------|----------|-------------|---------------------|
| 6.1 | Error handling audit: every API endpoint returns proper error codes | 6h | P0 | All phases | Consistent error format. No 500s for user errors. Actionable error messages |
| 6.2 | Implement zero-downtime deployments (blue-green via ECS) | 6h | P0 | Phase 2 | New deployment runs alongside old. Traffic shifted after health check passes |
| 6.3 | Build deployment rollback: `deployra rollback` to previous version | 4h | P0 | 6.2 | Rollback completes in under 30 seconds. Previous image re-deployed |
| 6.4 | Implement graceful shutdown: agents get SIGTERM, 30s to cleanup | 3h | P0 | Phase 2 | Agent receives SIGTERM. 30s grace period. Forced kill after timeout |
| 6.5 | Add CLI error messages with suggestions (e.g., "Did you mean...?") | 4h | P1 | Phase 3 | Common errors have helpful suggestions. Typo detection for commands |
| 6.6 | Build email notification system (deploy success/failure, budget alerts) | 6h | P1 | Phase 5 | Users receive email on deploy completion, budget threshold (80%, 100%) |
| 6.7 | Write user-facing documentation: Getting Started, CLI Reference, Config Reference | 10h | P0 | All phases | Docs site at docs.deployra.ai. 5-minute quickstart guide. Complete CLI reference |
| 6.8 | Build example agents: LangChain RAG agent, CrewAI research team, AutoGen coder | 8h | P0 | Phase 3 | 3 example agents in github.com/deployra/examples. README with deploy instructions |
| 6.9 | Performance testing: deploy 50 agents simultaneously | 6h | P0 | All phases | 50 agents deploy in under 5 minutes. No failures. System stable |
| 6.10 | Load testing: 1000 concurrent LLM calls through proxy | 4h | P0 | Phase 2 | Proxy handles 1000 req/s with <10ms overhead. No dropped requests |
| 6.11 | Security audit: check for injection, auth bypass, data leakage | 6h | P0 | All phases | No critical or high vulnerabilities. All user input sanitized |
| 6.12 | Implement webhook support for agent lifecycle events | 4h | P1 | Phase 4 | Webhooks delivered for: deploy, start, stop, error, budget events |
| 6.13 | Build beta waitlist page at deployra.ai with email capture | 3h | P0 | None | Landing page with waitlist form. Emails stored in Resend list |
| 6.14 | Set up beta invite system (invite codes, usage tracking) | 4h | P0 | Phase 5 | Generate invite codes. Track who signed up. Monitor usage |
| 6.15 | Production environment setup: separate ECS cluster, database, monitoring | 6h | P0 | Phase 1 | Production environment mirrors staging. PagerDuty alerts configured |
| 6.16 | Create demo video: 3-minute screencast of deploy flow | 4h | P1 | All phases | Video shows: init → deploy → logs → dashboard. Professional quality |
| 6.17 | Onboard first 5 beta users, collect feedback | 6h | P0 | All above | 5 users deploy agents successfully. Feedback documented. Top 3 issues identified |

### Risk Flags
- **Performance bottlenecks:** Load testing may reveal unexpected bottlenecks. Budget 2 days for fixes.
- **Beta user friction:** First users will find UX issues we missed. Reserve time for quick fixes.
- **Production parity:** Staging and production must be identical. Use Terraform modules to ensure consistency.

### Deliverables
- [ ] Zero-downtime deployments working
- [ ] Rollback functionality working
- [ ] Documentation site live
- [ ] 3 example agents published
- [ ] Performance tests passing (50 agents, 1000 req/s)
- [ ] Security audit completed
- [ ] Production environment live
- [ ] First 5 beta users onboarded
- [ ] Beta waitlist page live

---

## Summary: Week-by-Week Milestones

| Week | Milestone | Key Deliverable |
|------|-----------|-----------------|
| 1 | Infrastructure + DB | AWS VPC, Neon DB, CI/CD pipeline live |
| 2 | Auth + API | GitHub OAuth, Agent CRUD API, staging API live |
| 3 | Agent Runtime | ECS Fargate containers running, health checks working |
| 4 | LLM Proxy | Cost tracking, budget enforcement, retry/fallback |
| 5 | CLI Core | `deployra init`, `deploy`, `logs` commands working |
| 6 | Deploy Pipeline | Kaniko builds, end-to-end deploy in under 60s |
| 7 | Dashboard UI | Agent list, detail, logs, costs pages |
| 8 | Telemetry | Kafka → ClickHouse pipeline, real-time charts |
| 9 | Teams + Auth | Multi-team, RBAC, RLS, team management |
| 10 | Billing | Stripe integration, usage metering, plan enforcement |
| 11 | Hardening | Error handling, security audit, performance tests |
| 12 | Beta Launch | Docs, examples, first 5 beta users onboarded |

## Hour Allocation Summary

| Phase | Hours | % of Total |
|-------|-------|-----------|
| Phase 1: Foundation | 80h | 15% |
| Phase 2: Core Runtime | 100h | 19% |
| Phase 3: CLI + Deploy | 90h | 17% |
| Phase 4: Dashboard + Telemetry | 100h | 19% |
| Phase 5: Auth + Multi-tenancy | 80h | 15% |
| Phase 6: Polish + Beta | 90h | 17% |
| **Total** | **540h** | **100%** |

## Definition of Done (Global)

Every task is "done" when:
1. Code is written, reviewed (self-review for solo founder), and merged to `main`
2. Tests pass in CI (unit + integration where applicable)
3. Feature is deployed to staging and manually verified
4. No regressions in existing functionality
5. Task marked complete in this tracker

---

*This roadmap is a living document. Update weekly during Sunday planning sessions.*
