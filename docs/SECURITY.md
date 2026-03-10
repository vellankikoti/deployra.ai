# Deployra — Security Architecture

> Version 1.0 | March 2026 | Security Documentation

## Table of Contents

1. [Threat Model](#threat-model)
2. [Data Classification](#data-classification)
3. [Authentication & Authorization](#authentication--authorization)
4. [Multi-Tenant Isolation](#multi-tenant-isolation)
5. [Secrets Management](#secrets-management)
6. [Encryption Strategy](#encryption-strategy)
7. [SOC 2 Readiness](#soc-2-readiness)
8. [Incident Response Plan](#incident-response-plan)
9. [Compliance Considerations](#compliance-considerations)
10. [Penetration Testing Plan](#penetration-testing-plan)

---

## Threat Model

### System Boundaries

```
                    ┌─────────────────────────────────────────────┐
                    │              Deployra Platform               │
                    │                                             │
    Developer ─────►│  CLI → API → Build → Runtime → LLM Proxy  │──► LLM Providers
                    │                  ↕                          │   (OpenAI, etc.)
    Dashboard ─────►│             Dashboard                       │
                    │                  ↕                          │
                    │         PostgreSQL / ClickHouse / Kafka     │
                    └─────────────────────────────────────────────┘
```

### Threat Actors

| Actor | Motivation | Capability | Likelihood |
|-------|-----------|-----------|------------|
| **Malicious developer** | Abuse free tier, mine crypto, attack other tenants | Authenticated user with agent deployment capability | High |
| **External attacker** | Data theft, service disruption, ransomware | Unauthenticated, scanning for vulnerabilities | Medium |
| **Compromised agent** | Lateral movement, data exfiltration | Running code inside our infrastructure | Medium |
| **Insider threat** | Data access, credential theft | Admin access to infrastructure | Low |
| **Competitor** | Intelligence gathering, service disruption | Moderate technical capability | Low |

### STRIDE Analysis

| Threat | Category | Attack Vector | Impact | Mitigation |
|--------|----------|--------------|--------|------------|
| T1: Stolen JWT tokens | Spoofing | XSS, token leakage in logs | Account takeover | Short-lived JWTs (24h), HttpOnly cookies, no tokens in URLs |
| T2: Cross-tenant data access | Tampering | SQL injection, broken access control | Data breach | RLS policies, parameterized queries, team_id in all queries |
| T3: Agent container escape | Elevation | Container vulnerability exploitation | Host access | Fargate isolation (no shared host), non-root containers, read-only FS |
| T4: LLM API key exposure | Information Disclosure | Log leakage, database breach | Customer credential theft | AES-256-GCM encryption, never log keys, memory-only decryption |
| T5: DDoS on API/agents | Denial of Service | High-volume requests | Service unavailable | WAF (AWS Shield), rate limiting, CloudFront caching |
| T6: Agent-to-agent attack | Spoofing/Tampering | Network lateral movement | Cross-tenant compromise | Security groups block inter-agent traffic, no shared network |
| T7: Build pipeline injection | Tampering | Malicious Dockerfile, supply chain | Code execution in our infra | Kaniko (no Docker daemon), resource limits, network restrictions during build |
| T8: Malicious agent code | Elevation | Crypto mining, outbound attacks, resource abuse | Resource drain, reputation damage | CPU/memory limits, network egress limits, abuse monitoring |

### Top 5 Risks (Prioritized)

1. **Cross-tenant data leakage** — A bug in application code allows Team A to see Team B's data.
2. **LLM API key exposure** — Customer API keys leak through logs, errors, or database breach.
3. **Agent container escape** — Malicious code in an agent container gains access to infrastructure.
4. **Malicious agent abuse** — Free tier users run crypto miners or attack other systems from our IP.
5. **JWT/session hijacking** — Attacker gains access to a user's session.

---

## Data Classification

| Classification | Description | Examples | Storage | Access |
|---------------|-------------|---------|---------|--------|
| **Critical** | Exposure causes immediate financial harm | User LLM API keys, Stripe payment data, encryption keys | Encrypted (AES-256-GCM + at-rest encryption), Secrets Manager | LLM Proxy service only (API keys), Stripe service only (payment) |
| **Confidential** | Exposure causes competitive or privacy harm | Agent source code, deployment configs, user emails, team data | Encrypted at rest (S3 SSE, Neon encryption) | Authenticated team members only |
| **Internal** | Exposure causes operational risk | Infrastructure configs, API endpoints, internal metrics | Standard encryption at rest | Engineering team only |
| **Telemetry** | Aggregated operational data, no PII | Token counts, costs, latencies, error counts | ClickHouse encryption at rest | Authenticated team members (own team's data) |
| **Public** | No sensitivity | Documentation, marketing content, open-source code | No special protection | Anyone |

### What We Do NOT Store

- **Prompt/response content:** The LLM proxy does not log, store, or inspect the content of LLM requests or responses. We only capture metadata (tokens, model, cost, latency).
- **Passwords:** We use GitHub OAuth exclusively. No password storage.
- **Full API keys:** Only SHA-256 hashes and the first 8 characters (for identification) are stored. The full key is shown once at creation.
- **Customer end-user data:** We don't process or store data that flows through deployed agents. Agents handle their own data — we just run the container.

---

## Authentication & Authorization

### Authentication Mechanisms

| Mechanism | Used By | Token Type | Expiry | Storage |
|-----------|---------|-----------|--------|---------|
| GitHub OAuth + JWT | Dashboard, CLI | JWT (RS256) | 24 hours | HttpOnly cookie (dashboard), config file (CLI) |
| API Key | Programmatic access, CI/CD | Bearer token | 90 days (configurable) | Only SHA-256 hash stored in DB |
| Internal service tokens | Service-to-service | JWT (RS256) | 1 hour, auto-rotated | Environment variable, never persisted |

### JWT Structure

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-2026-03"
  },
  "payload": {
    "sub": "usr_abc123",
    "team_id": "team_xyz",
    "role": "admin",
    "scopes": ["read", "write", "deploy", "admin"],
    "iat": 1710086400,
    "exp": 1710172800,
    "iss": "deployra.ai"
  }
}
```

### Key Rotation

- JWT signing keys rotate every 90 days.
- Previous key remains valid for 7 days after rotation (graceful transition).
- Key ID (`kid`) in JWT header identifies which key to use for validation.
- Keys stored in AWS Secrets Manager with automated rotation.

### Authorization Model (RBAC)

| Action | Owner | Admin | Member | Viewer |
|--------|:-----:|:-----:|:------:|:------:|
| View agents, logs, costs | ✅ | ✅ | ✅ | ✅ |
| Deploy agents | ✅ | ✅ | ✅ | ❌ |
| Create/delete agents | ✅ | ✅ | ✅ | ❌ |
| Modify agent config | ✅ | ✅ | ✅ | ❌ |
| Manage secrets | ✅ | ✅ | ❌ | ❌ |
| Manage team members | ✅ | ✅ | ❌ | ❌ |
| Manage API keys | ✅ | ✅ | ❌ | ❌ |
| Manage billing | ✅ | ✅ | ❌ | ❌ |
| Transfer ownership | ✅ | ❌ | ❌ | ❌ |
| Delete team | ✅ | ❌ | ❌ | ❌ |

### API Key Scoping

```
API Key scopes (combinable):
  read:agents      - GET /v1/agents, /v1/agents/:id
  write:agents     - POST/PATCH/DELETE /v1/agents
  deploy           - POST /v1/agents/:id/deployments
  read:logs        - GET /v1/agents/:id/logs, WebSocket streams
  read:costs       - GET /v1/agents/:id/costs
  manage:secrets   - POST/DELETE /v1/agents/:id/secrets
  manage:keys      - POST/DELETE /v1/api-keys
  admin            - All permissions
```

---

## Multi-Tenant Isolation

### Isolation Layers

```
Layer 1: Database (PostgreSQL RLS)
├── Every table has team_id column
├── RLS policies enforce team_id = current_setting('app.team_id')
├── Application sets team_id from JWT before every query
└── Even SQL injection cannot access other teams' data

Layer 2: Application (API Middleware)
├── JWT middleware extracts team_id
├── Every handler receives team-scoped context
├── All queries include team_id filter
└── No admin endpoints bypass team scoping

Layer 3: Compute (ECS Fargate)
├── Each agent runs in separate Fargate task
├── Separate IAM role per agent
├── No shared compute (Fargate provides hypervisor-level isolation)
└── Separate security group per agent

Layer 4: Network
├── Agent containers in isolated subnets
├── Security groups block inter-agent traffic
├── Agents can only reach: LLM Proxy (port 8443), Internet (port 443)
└── No agent can access the API, database, or other infrastructure directly

Layer 5: Secrets
├── AWS Secrets Manager with IAM policies
├── Policy: agent IAM role can only access its own secrets
├── Secrets identified by path: /{team_id}/{agent_id}/{secret_name}
└── Cross-team access impossible at IAM level

Layer 6: Telemetry
├── ClickHouse queries always filter by team_id
├── Kafka topics are shared but events tagged with team_id
├── WebSocket connections verified for agent ownership
└── Dashboard only shows team's own data
```

### PostgreSQL RLS Implementation

```sql
-- Enable RLS on all multi-tenant tables
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;
ALTER TABLE deployments ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE secrets ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE cost_events ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their own team's data
CREATE POLICY agents_team_isolation ON agents
    USING (team_id = current_setting('app.team_id')::uuid);

CREATE POLICY deployments_team_isolation ON deployments
    USING (team_id = current_setting('app.team_id')::uuid);

-- Application sets team_id before every request
-- In Go API middleware:
-- db.Exec("SET LOCAL app.team_id = $1", teamIDFromJWT)
```

---

## Secrets Management

### User LLM API Keys

The most sensitive data we handle. An exposed API key can cost a customer thousands of dollars.

**Encryption Flow:**

```
User submits API key via CLI or Dashboard
    │
    ▼
API receives key over TLS 1.3
    │
    ▼
Application-level encryption:
    1. Generate random 96-bit nonce
    2. Encrypt key with AES-256-GCM using team-specific DEK
    3. Store: encrypted_value = nonce + ciphertext + auth_tag
    │
    ▼
Store in PostgreSQL (secrets table):
    encrypted_value BYTEA (encrypted blob)
    │
    ▼
When agent needs the key (deploy time / LLM proxy request):
    1. LLM Proxy service retrieves encrypted_value from DB
    2. Decrypts with team DEK from AWS Secrets Manager
    3. Key exists only in-memory, never written to disk/logs
    4. Injected into upstream request Authorization header
    5. Memory zeroed after request completes
```

**Key Encryption Key (KEK) Hierarchy:**

```
AWS KMS Master Key (CMK)
    └── Team Data Encryption Key (DEK) — one per team
        └── Encrypted secrets (API keys, DB URLs, etc.)

DEK rotation: Every 90 days (automatic)
KEK rotation: Every 365 days (AWS KMS managed)
```

### Platform Secrets

| Secret | Storage | Rotation |
|--------|---------|----------|
| JWT signing keys (RS256 private key) | AWS Secrets Manager | 90 days |
| Database connection string | AWS Secrets Manager | On password change |
| GitHub OAuth client secret | AWS Secrets Manager | On compromise |
| Stripe API keys | AWS Secrets Manager | 90 days |
| ClickHouse credentials | AWS Secrets Manager | 90 days |
| Internal service JWT signing key | AWS Secrets Manager | 30 days |
| Kafka SASL credentials | MSK managed | On rotation |

### What We Never Do

- Never log API keys, tokens, or secrets (even partially)
- Never store secrets in environment variables of running containers (use Secrets Manager references)
- Never include secrets in Docker images
- Never transmit secrets in URL query parameters
- Never return stored secrets via API (only names/metadata)
- Never store secrets in application config files

---

## Encryption Strategy

### Encryption at Rest

| Data | Encryption | Key Management |
|------|-----------|---------------|
| PostgreSQL (Neon) | AES-256, managed by Neon | Neon-managed keys |
| User LLM API keys | AES-256-GCM (application-level) | AWS KMS → per-team DEK |
| S3 (source code, artifacts) | AES-256, SSE-S3 | AWS-managed keys |
| ECR (container images) | AES-256, ECR default encryption | AWS-managed keys |
| ClickHouse Cloud | AES-256, managed by ClickHouse | ClickHouse-managed keys |
| ElastiCache Redis | AES-256, at-rest encryption | AWS-managed keys |
| Kafka (MSK) | AES-256, MSK encryption | AWS-managed keys |
| EBS (Fargate ephemeral storage) | AES-256, EBS encryption | AWS-managed keys |

### Encryption in Transit

| Connection | Protocol | Certificate |
|-----------|----------|-------------|
| Client → ALB | TLS 1.3 | ACM wildcard cert (*.deployra.ai) |
| ALB → API/Proxy | TLS 1.2+ | Internal ACM cert |
| API → Neon | TLS 1.3 | Neon-managed cert |
| API → ClickHouse | TLS 1.3 | ClickHouse Cloud cert |
| Proxy → LLM providers | TLS 1.3 | Provider certs (pinned) |
| API → Redis | TLS 1.2+ | ElastiCache cert |
| API → Kafka | TLS 1.2+ (SASL_SSL) | MSK cert |
| Agent → Proxy (internal) | TLS 1.2+ | Internal cert |
| WebSocket | WSS (TLS 1.3) | ACM wildcard cert |

### TLS Configuration

```
Minimum TLS version: 1.2 (external), 1.2 (internal)
Preferred: TLS 1.3
Cipher suites (TLS 1.3):
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256
  TLS_AES_128_GCM_SHA256

HSTS: max-age=31536000; includeSubDomains
Certificate transparency: Enabled
OCSP stapling: Enabled
```

---

## SOC 2 Readiness

### SOC 2 Type 1 Checklist (Target: Month 12)

#### Security (CC6)

| Control | Status | Implementation |
|---------|--------|---------------|
| CC6.1 Logical access security | 🟡 In Progress | GitHub OAuth, JWT, RBAC, API key scoping |
| CC6.2 Authentication mechanisms | 🟡 In Progress | OAuth 2.0, JWT (RS256), MFA via GitHub |
| CC6.3 Access provisioning | ⬜ Planned | Team invite flow, role assignment |
| CC6.4 Access removal | ⬜ Planned | Team member removal, API key revocation |
| CC6.5 User authentication | 🟡 In Progress | GitHub OAuth (inherits GitHub's MFA) |
| CC6.6 Encryption | 🟡 In Progress | TLS 1.3, AES-256-GCM for secrets |
| CC6.7 Transmission security | 🟡 In Progress | TLS on all connections |
| CC6.8 Malware prevention | ⬜ Planned | Container scanning, no shared compute |

#### Availability (A1)

| Control | Status | Implementation |
|---------|--------|---------------|
| A1.1 Infrastructure resilience | 🟡 In Progress | Multi-AZ deployment, auto-failover |
| A1.2 Environmental protections | ✅ Done | AWS data center controls (inherited) |
| A1.3 Backup and recovery | ⬜ Planned | Neon PITR, S3 versioning, ClickHouse backups |

#### Confidentiality (C1)

| Control | Status | Implementation |
|---------|--------|---------------|
| C1.1 Confidential information identification | 🟡 In Progress | Data classification (see above) |
| C1.2 Confidential information disposal | ⬜ Planned | Data deletion policy, S3 lifecycle |

#### Processing Integrity (PI1)

| Control | Status | Implementation |
|---------|--------|---------------|
| PI1.1 Accurate processing | 🟡 In Progress | Token counting validation, cost calculation auditing |
| PI1.2 Error handling | ⬜ Planned | Structured error codes, retry logic |

### SOC 2 Timeline

| Month | Activity |
|-------|----------|
| M6 | Select SOC 2 auditor (e.g., Vanta, Drata, or traditional firm) |
| M7 | Gap assessment — identify missing controls |
| M8-10 | Implement missing controls |
| M11 | Pre-audit readiness review |
| M12 | SOC 2 Type 1 audit (point-in-time assessment) |
| M18 | SOC 2 Type 2 audit (6-month observation period) |

---

## Incident Response Plan

### Severity Levels

| Level | Definition | Response Time | Examples |
|-------|-----------|--------------|---------|
| **SEV-1** | Customer data breach, full service outage | 15 minutes | API key exposure, database breach, platform-wide outage |
| **SEV-2** | Partial outage, data integrity issue | 30 minutes | Single service down, incorrect billing, agent deploy failures |
| **SEV-3** | Degraded performance, non-critical bug | 4 hours | Slow dashboard, delayed logs, build queue backup |
| **SEV-4** | Minor issue, cosmetic bug | 24 hours | UI glitch, documentation error, non-blocking bug |

### Incident Response Process

```
1. DETECT
   ├── CloudWatch alarm fires
   ├── Customer reports issue
   ├── PagerDuty notification
   └── Team member notices anomaly

2. TRIAGE (within 15 minutes)
   ├── Assign severity level
   ├── Identify affected systems and customers
   ├── Page additional responders if SEV-1/2
   └── Create incident channel (Slack: #incident-YYYY-MM-DD)

3. CONTAIN (within 30 minutes for SEV-1)
   ├── Isolate affected systems
   ├── Revoke compromised credentials
   ├── Block malicious traffic
   └── Preserve evidence (logs, snapshots)

4. COMMUNICATE
   ├── Internal: Incident channel updates every 15 minutes
   ├── Affected customers: Email within 1 hour (SEV-1/2)
   ├── Status page: Update at status.deployra.ai
   └── Investors: Brief within 24 hours (SEV-1 only)

5. RESOLVE
   ├── Implement fix
   ├── Verify fix in staging
   ├── Deploy to production
   ├── Verify resolution
   └── Monitor for regression (24 hours)

6. POST-MORTEM (within 48 hours)
   ├── Timeline of events
   ├── Root cause analysis (5 Whys)
   ├── Impact assessment (affected customers, data exposure)
   ├── Action items (prevent recurrence)
   └── Publish to team (SEV-1/2: share with investors)
```

### Data Breach Response (SEV-1)

If customer data (especially LLM API keys) is exposed:

1. **Immediate (0-30 min):**
   - Revoke all potentially compromised encryption keys
   - Rotate all platform secrets
   - Assess scope: which teams/agents are affected?

2. **Short-term (30 min - 4 hours):**
   - Notify affected customers directly (email + in-app)
   - Advise customers to rotate their LLM API keys
   - Engage security incident response firm (pre-selected)
   - Notify legal counsel

3. **Medium-term (4-72 hours):**
   - Complete forensic investigation
   - Implement additional security controls
   - Prepare public disclosure if required (GDPR: 72-hour notification)
   - Brief investors

4. **Long-term (1-4 weeks):**
   - Publish post-mortem
   - Implement all action items
   - Engage third-party penetration tester
   - Review and update security architecture

---

## Compliance Considerations

### GDPR (EU Users)

| Requirement | Implementation |
|-------------|---------------|
| Lawful basis | Legitimate interest (service delivery) + consent (marketing) |
| Data minimization | Only collect: GitHub username, email, avatar. No unnecessary data |
| Right to access | API endpoint: GET /v1/me/data-export |
| Right to deletion | API endpoint: DELETE /v1/me (deletes all user data, agents, secrets) |
| Data portability | Export in JSON format |
| Breach notification | 72-hour notification process (see Incident Response) |
| DPA with sub-processors | AWS DPA, Neon DPA, ClickHouse DPA, Stripe DPA |
| Data residency | US-East-1 (primary). EU region planned for Year 2 |

### SOC 2 (Enterprise Customers)

See [SOC 2 Readiness](#soc-2-readiness) section above.

### HIPAA (Healthcare Customers — Future)

**Current status:** Not HIPAA compliant. Not accepting PHI.

**Path to HIPAA (Year 3):**
1. Sign BAA with AWS (available for Fargate, S3, etc.)
2. Enable CloudTrail for all API calls
3. Implement audit logging for all PHI access
4. Add encryption controls beyond current baseline
5. Add access logging for all data access
6. Engage HIPAA compliance consultant
7. Estimated cost: $50K for compliance implementation + $20K/year for audits

**When to pursue:** When healthcare customers represent >10% of pipeline and are willing to sign annual contracts >$50K.

---

## Penetration Testing Plan

### Scope

| Area | In Scope | Out of Scope |
|------|----------|-------------|
| API Gateway | All REST endpoints, auth flows, input validation | Load testing (separate exercise) |
| Dashboard | XSS, CSRF, auth bypass, session management | Browser compatibility |
| CLI | Token storage, auth flow, command injection | Distribution pipeline |
| LLM Proxy | Request manipulation, bypass, data leakage | LLM provider APIs |
| Agent Runtime | Container escape, network isolation, privilege escalation | Customer agent code |
| Database | SQL injection, RLS bypass, data leakage | Database engine vulnerabilities |
| Infrastructure | AWS configuration, network security, IAM policies | Physical security (AWS responsibility) |

### Testing Schedule

| Test Type | Frequency | Methodology | Provider |
|-----------|-----------|-------------|----------|
| Automated vulnerability scan | Weekly | OWASP ZAP, Trivy (containers) | In-house |
| Manual penetration test | Quarterly | OWASP Testing Guide v4 | Third-party firm |
| Infrastructure security review | Quarterly | AWS Well-Architected, CIS Benchmarks | In-house + AWS review |
| Code security review | Per PR | Semgrep, CodeQL | Automated in CI |
| Dependency scanning | Daily | Dependabot, Snyk | Automated |
| Container image scanning | Per build | Trivy, ECR scanning | Automated in CI |

### Priority Test Cases

1. **Cross-tenant data access:** Attempt to access Team B's agents/data using Team A's credentials.
2. **API key privilege escalation:** Attempt to perform admin actions with a read-only API key.
3. **Container escape:** Attempt to break out of a Fargate task and access infrastructure.
4. **LLM API key extraction:** Attempt to extract stored API keys via API responses, error messages, or logs.
5. **SQL injection via agent config:** Inject SQL via JSONB config fields, agent names, or API parameters.
6. **WebSocket authorization bypass:** Connect to another team's agent log stream.
7. **Build pipeline injection:** Deploy an agent with a Dockerfile that attempts to access infrastructure.
8. **Rate limit bypass:** Attempt to circumvent rate limiting via multiple API keys, IP rotation.
9. **JWT manipulation:** Forge or tamper with JWT tokens to impersonate other users.
10. **Budget enforcement bypass:** Attempt to make LLM calls that don't count against budget.

### Remediation SLA

| Severity | Remediation Timeline | Verification |
|----------|---------------------|-------------|
| Critical | 24 hours | Retest within 48 hours |
| High | 7 days | Retest within 14 days |
| Medium | 30 days | Retest in next quarterly test |
| Low | 90 days | Retest in next quarterly test |
| Informational | Best effort | No retest required |

---

*Security is not a feature — it's a foundation. This document must be reviewed and updated quarterly, or after any security incident, whichever comes first.*
