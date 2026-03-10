# Deployra — Fundraising Playbook

> Version 1.0 | March 2026 | Confidential — Investor Use Only

## Table of Contents

1. [The Narrative](#the-narrative)
2. [Pitch Deck Outline](#pitch-deck-outline)
3. [Pre-Seed Strategy](#pre-seed-strategy)
4. [Investor Targeting](#investor-targeting)
5. [Due Diligence Prep](#due-diligence-prep)
6. [Investor FAQ](#investor-faq)

---

## The Narrative

### The 3-Minute Pitch Script

*Imagine you're at a dinner with a partner at a16z. They ask: "So what are you building?"*

---

"Every developer building AI agents hits the same wall. Building the agent takes a weekend. Deploying it to production takes three weeks.

Right now, if you want to run an AI agent — a LangChain agent, a CrewAI team, an AutoGen workflow — in production, you need to write a Dockerfile, configure an ECS task definition, set up a load balancer, build a monitoring stack, figure out how to track LLM costs, and pray that your agent doesn't enter an infinite loop and rack up a $5,000 OpenAI bill overnight.

I know this because I spent two years doing exactly that. I deployed 30+ AI agents across three companies, and every single time, I spent more time on infrastructure than on the agent itself.

**Deployra fixes this.** One command: `deployra deploy`. Your agent is live in 30 seconds with cost tracking, budget controls, auto-restart, and model fallback — all built in. Think Heroku, but purpose-built for AI agents.

Here's what makes this different from just running containers:

**First, the LLM proxy.** Every LLM call from your agent goes through our proxy. We count tokens, calculate costs, enforce budgets, cache repeated prompts, and auto-switch to a backup model if the primary goes down. The developer changes zero lines of code. This is the thing nobody else does.

**Second, the economics layer.** We show you exactly what each agent costs — per request, per run, per day. We enforce spending limits in real-time. An agent that hits its daily budget gets paused, not just flagged. This is the feature that sells to enterprise.

The market is massive. IDC projects $150 billion in AI infrastructure spend by 2028. The agent orchestration layer alone is $20 billion. And right now, there is no default platform for deploying AI agents. It's either AWS Bedrock — which is rigid and doesn't support most frameworks — or cobbling together Kubernetes, Prometheus, and custom scripts.

We're pre-revenue, launching our beta in Q2 2026, and already have 2,000 developers on the waitlist. I'm raising $1.5 million to hire two engineers, launch publicly, and get to $50K MRR within 12 months.

The AI agent wave is here. Thousands of developers are building agents. Zero of them have a good way to deploy them. We're building the deployment layer for the entire ecosystem."

---

## Pitch Deck Outline

### 12 Slides

**Slide 1: Title**
- Logo: Deployra
- Tagline: "The deployment platform for AI agents"
- Founder name, title
- Date, stage: Pre-Seed

---

**Slide 2: The Problem**
- Header: "Deploying AI agents to production is a nightmare"
- 3 pain points with icons:
  1. **Infrastructure overhead:** 3 weeks to deploy what took 3 days to build. Custom Docker, ECS, monitoring, logging.
  2. **Zero cost visibility:** No way to track per-agent LLM costs. Surprise $5K OpenAI bills.
  3. **No production safety net:** No budget controls, no auto-restart, no fallback when OpenAI goes down.
- Quote: "I spent more time deploying my agent than building it" — real quote from beta user

---

**Slide 3: The Solution**
- Header: "One command. Production-grade AI agents."
- Terminal screenshot: `deployra deploy` with output showing 31.7s deploy
- 4 value props in a row:
  - Deploy in 30s (not 3 weeks)
  - LLM cost tracking (per-token)
  - Budget controls (auto-pause)
  - Model fallback (auto-switch)

---

**Slide 4: Live Demo / Product Screenshots**
- Dashboard screenshot: agent list with status, costs, run counts
- Cost breakdown chart: daily cost by model
- Live logs view: streaming agent activity
- CLI output: deploy flow with progress indicator

---

**Slide 5: How It Works**
- 3-step flow:
  1. `deployra init` → Auto-detect framework, generate config
  2. `deployra deploy` → Build image, deploy container, assign URL
  3. Monitor → Dashboard shows status, costs, logs in real-time
- Architecture diagram (simplified): CLI → API → Container → LLM Proxy → Dashboard

---

**Slide 6: The LLM Proxy (Secret Weapon)**
- Header: "Every LLM call, tracked and controlled"
- Diagram: Agent → Proxy → OpenAI/Anthropic/Google
- Bullets:
  - Token counting and cost calculation per request
  - Budget enforcement (daily/monthly limits, auto-pause)
  - Retry with exponential backoff
  - Auto-fallback (OpenAI → Anthropic → Google)
  - Response caching (15-30% cost savings)
  - Zero code changes required from developer

---

**Slide 7: Market**
- Header: "$150B AI infrastructure market. $20B agent orchestration layer."
- Market size visualization:
  - TAM: $150B — AI infrastructure spend (IDC 2028 projection)
  - SAM: $20B — Agent orchestration and deployment
  - SOM: $500M — Developer-focused agent platforms (Year 5 target)
- Market tailwinds:
  - 84% of enterprises plan to deploy AI agents by 2027 (Gartner)
  - LangChain has 100K+ GitHub stars, CrewAI 25K+
  - Agent frameworks growing 10x YoY

---

**Slide 8: Traction**
- Header: "Pre-launch momentum"
- Metrics:
  - 2,000+ waitlist signups
  - 50+ beta applications from companies
  - MVP functional: end-to-end deploy in 31.7 seconds
  - 3 framework integrations (LangChain, CrewAI, AutoGen)
  - Website live at deployra.ai
- Social proof: logos of beta companies (if available), community size

---

**Slide 9: Business Model**
- Header: "SaaS + Usage-Based Revenue"
- Pricing tiers: Free / Pro ($29/mo) / Team ($99/seat/mo) / Enterprise (custom)
- Revenue breakdown pie chart:
  - 40% subscription revenue
  - 45% usage-based revenue (compute, proxy requests)
  - 15% enterprise contracts
- Unit economics targets:
  - CAC: $50 (organic/PLG)
  - LTV: $1,200 (Pro) / $8,500 (Team)
  - LTV/CAC: 24x (Pro) / 170x (Team)
  - Gross margin: 75-80%

---

**Slide 10: Competitive Landscape**
- Header: "No direct competitor. We own the deployment layer."
- 2x2 matrix:
  - X-axis: General purpose ← → Agent-specific
  - Y-axis: Infrastructure ← → Platform
  - Deployra: top-right (Agent-specific Platform)
  - AWS Bedrock: bottom-right (Agent-specific Infrastructure)
  - Heroku/Railway: top-left (General Platform)
  - Custom K8s: bottom-left (General Infrastructure)
  - LangSmith: middle-right (Agent-specific, but observability only)
- Key differentiators: LLM proxy, budget controls, framework auto-detection, one-command deploy

---

**Slide 11: Team**
- Header: "Solo founder. Shipping at startup speed."
- Photo + bio:
  - Full-stack engineer, [X] years experience
  - Previously: [relevant experience deploying AI agents]
  - Built and shipped the entire MVP in 12 weeks
  - Deep expertise: Go, TypeScript, AWS, container orchestration
- Hiring plan: 2 engineers with raise (1 backend/infra, 1 frontend/CLI)
- Advisory board (if applicable)

---

**Slide 12: The Ask**
- Header: "Raising $1.5M to become the default deployment platform for AI agents"
- Use of funds breakdown (pie chart):
  - Engineering (2 hires): 60% ($900K / 18 months)
  - Infrastructure (AWS, ClickHouse, etc.): 15% ($225K)
  - Marketing & community: 10% ($150K)
  - Operations & legal: 5% ($75K)
  - Buffer: 10% ($150K)
- Milestones the raise enables:
  - 1,000 weekly active agents
  - $50K MRR
  - 3-person engineering team
  - SOC 2 Type 1
  - Series A ready
- Timeline: Close by June 2026, 18-month runway

---

## Pre-Seed Strategy

### Target Raise

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Amount | $1.5M | 18-month runway for 3-person team + infrastructure |
| Instrument | SAFE | Standard for pre-seed, clean and fast |
| Valuation Cap | $10M post-money | Standard for strong solo founder with working MVP |
| Discount | 20% | Standard SAFE discount |
| Target close | June 2026 | 8 weeks from first meeting to close |

### Valuation Logic

$10M post-money cap is justified by:
1. **Working MVP:** Not a slide deck — the product is built and functional.
2. **Technical moat:** LLM proxy with format translation, token counting, and budget enforcement is non-trivial to replicate.
3. **Market timing:** AI agent deployment is a new category forming now. First-mover advantage.
4. **Waitlist traction:** 2,000+ signups indicates demand validation.
5. **Solo founder efficiency:** Built the entire platform in 12 weeks. Capital-efficient execution.
6. **Comparable valuations:** Similar pre-seed rounds in developer tools (Supabase, Railway, Vercel early days) at $8-15M caps.

### Use of Funds

| Category | Amount | % | Details |
|----------|--------|---|---------|
| Engineering | $900K | 60% | 2 senior engineers × $150K salary × 18 months + equity |
| Infrastructure | $225K | 15% | AWS ($100K Activate credits + $125K overage), ClickHouse, Neon, monitoring |
| Marketing | $150K | 10% | Content creator ($3K/mo), conference sponsorships, swag, design |
| Operations | $75K | 5% | Legal (incorporation, SAFE docs, IP), accounting, insurance |
| Buffer | $150K | 10% | Unexpected expenses, opportunistic hiring, runway extension |
| **Total** | **$1.5M** | **100%** | **18-month runway** |

### Milestones the Raise Enables

| Milestone | Timeline | Metric |
|-----------|----------|--------|
| Public launch (open beta) | Month 3 (Sep 2026) | 500+ signups in first week |
| 100 weekly active agents | Month 4 (Oct 2026) | Product-market fit signal |
| First paying customers | Month 4 (Oct 2026) | Revenue begins |
| 1,000 weekly active agents | Month 9 (Mar 2027) | Growth trajectory proven |
| $50K MRR | Month 12 (Jun 2027) | Series A ready |
| SOC 2 Type 1 | Month 12 (Jun 2027) | Enterprise readiness |
| Series A raise | Month 14 (Aug 2027) | $5-8M at $40-60M valuation |

### Fundraising Timeline

| Week | Activity |
|------|----------|
| Week 1 | Finalize pitch deck. Rehearse pitch 10 times. |
| Week 2 | Warm intro requests sent to Tier 1 investors. Apply to accelerators. |
| Week 3 | First partner meetings (4-5 meetings) |
| Week 4 | Follow-up meetings. Demo sessions. More Tier 1 meetings. |
| Week 5 | Deep dives with interested investors. Technical DD calls. |
| Week 6 | Term sheet negotiations. Reference checks. |
| Week 7 | Legal review. SAFE execution. |
| Week 8 | Wire transfers. Round closed. Announce. |

---

## Investor Targeting

### Tier 1 Investors (10 Firms)

| # | Firm | Why They Fit | Relevant Portfolio | Warm Intro Path |
|---|------|-------------|-------------------|-----------------|
| 1 | **Y Combinator** | Best brand for developer tools. Batch access + network. | Supabase, Railway, Vercel | Apply to W2027 batch. Also do standalone SAFE investments. |
| 2 | **Initialized Capital** | Garry Tan's fund. Deep developer tool expertise. Pre-seed focused. | Coinbase, Instacart (early) | Twitter DM to Garry (active on X). Through YC network. |
| 3 | **Boldstart Ventures** | Pre-seed/seed focused. Developer-first companies. | Snyk, BigPanda, Kustomer | Ed Sim active on LinkedIn. Cold email with strong product demo. |
| 4 | **Amplify Partners** | Deep infrastructure and developer tool thesis. | Datadog (early), HashiCorp (early) | Sunil Dhaliwal is accessible. Via mutual LinkedIn connections. |
| 5 | **Heavybit** | Developer tools accelerator + fund. Incredible GTM support. | Snyk, LaunchDarkly, PagerDuty | Apply to accelerator program. Dana Oshiro for intro. |
| 6 | **Essence VC** | AI-native fund. Technical partners who understand LLM infrastructure. | Multiple AI infra companies | Through AI Twitter community. Cold email with technical deep dive. |
| 7 | **Conviction Partners** | Sarah Guo's AI fund. Strong AI thesis and network. | Multiple AI companies | Through AI community events. Cold email with traction data. |
| 8 | **Sequoia Scout / Arc** | Arc program for early-stage. Sequoia brand opens all doors. | Countless | Through existing Sequoia portfolio founders. |
| 9 | **Haystack** | Semil Shah's fund. Pre-seed specialist. Developer tools experience. | Instacart, DoorDash (early) | Twitter DM (very accessible). Cold email works. |
| 10 | **South Park Commons** | Technical community + fund. Great for technical solo founders. | Technical founders community | Apply to SPC community. Through SF tech community. |

### Tier 2 Investors (15 Firms)

| # | Firm | Focus | Pre-Seed? |
|---|------|-------|-----------|
| 1 | Madrona Ventures | AI/ML infrastructure | Yes (Seed+) |
| 2 | Gradient Ventures (Google) | AI companies | Yes |
| 3 | Nexus Venture Partners | Developer tools, AI | Yes |
| 4 | Uncorrelated Ventures | Developer infrastructure | Yes |
| 5 | Work-Bench | Enterprise dev tools | Yes (NYC) |
| 6 | SignalFire | Data-driven, developer tools | Yes |
| 7 | First Round Capital | Developer tools generalist | Yes |
| 8 | Craft Ventures | SaaS, developer tools | Yes |
| 9 | Abstract Ventures | AI, developer tools | Yes |
| 10 | Basis Set Ventures | AI infrastructure | Yes |
| 11 | Operator Partners | AI infrastructure | Yes |
| 12 | General Catalyst | Generalist, AI thesis | Yes (Rough Draft for pre-seed) |
| 13 | Root Ventures | Deep tech, infrastructure | Yes |
| 14 | Bloomberg Beta | AI, future of work | Yes |
| 15 | Radical Ventures | AI-focused | Yes |

### Angel Investors to Target

| Name | Background | Why |
|------|-----------|-----|
| Guillermo Rauch | CEO, Vercel | Built the developer deployment platform. Understands the category. |
| Mitchell Hashimoto | Co-founder, HashiCorp | Infrastructure tooling legend. Endorsement = credibility. |
| Solomon Hykes | Founder, Docker | Container orchestration + developer tools. Perfect fit. |
| Harrison Chase | CEO, LangChain | Our platform deploys his framework. Massive strategic value. |
| Beyang Liu | Co-founder, Sourcegraph | Developer tools founder. Active angel. |
| Christina Cacioppo | CEO, Vanta | SOC 2 / compliance for startups. Relevant to enterprise motion. |
| Elad Gil | Author, serial angel | Prolific angel in developer tools and AI. |
| Zach Lloyd | CEO, Warp | Developer tools founder. Understands CLI-first products. |
| Jason Warner | Former CTO, GitHub | Developer platform expertise. Incredible network. |
| Julia Lipton | Awesome People Ventures | AI-focused angel fund. Active in AI community. |

### Accelerators

| Program | Application Deadline | Duration | Investment | Fit |
|---------|---------------------|----------|------------|-----|
| Y Combinator (W2027) | ~Sep 2026 | 3 months | $500K | Best for brand, network, follow-on |
| Techstars (NYC or SF) | Rolling | 3 months | $120K | Good mentorship, less brand value |
| South Park Commons | Rolling | 12 months | Community (no direct investment) | Best for technical solo founders |
| Heavybit | Rolling | 9 months | Developer tools specific | Best GTM support for dev tools |
| Neo | Rolling | Community + $100K | Young founder focused | Good community |

---

## Due Diligence Prep

### Technical DD Document Outline

1. **Architecture Overview** — System diagram, component descriptions, data flow
2. **Tech Stack Justification** — Why Go, why ClickHouse, why ECS Fargate (see TECH_STACK_DECISIONS.md)
3. **Code Quality** — Test coverage (target: 80%+), CI/CD pipeline, linting, code review process
4. **Security Posture** — Encryption, multi-tenancy isolation, secrets management (see SECURITY.md)
5. **Scalability Plan** — How we scale from 100 to 100,000 agents
6. **Infrastructure Costs** — AWS cost breakdown at current and projected scale
7. **Technical Risks** — Known risks and mitigation strategies
8. **IP** — What's proprietary, what's open source, no third-party IP risks

### Financial Model Summary (3-Year)

| Metric | Year 1 | Year 2 | Year 3 |
|--------|--------|--------|--------|
| Customers (paid) | 500 | 3,000 | 12,000 |
| MRR (end of year) | $65K | $450K | $2.5M |
| ARR (end of year) | $780K | $5.4M | $30M |
| Gross margin | 72% | 78% | 82% |
| Burn rate (monthly) | $85K | $180K | $350K |
| Team size | 3 | 10 | 25 |

*(See FINANCIAL_MODEL.md for detailed projections)*

### Cap Table Template

| Shareholder | Shares | % Ownership | Notes |
|-------------|--------|-------------|-------|
| Founder | 8,000,000 | 80.0% | 4-year vesting, 1-year cliff (self-imposed) |
| Option Pool | 1,000,000 | 10.0% | For first 2-3 hires |
| Pre-Seed Investors | 1,000,000 | 10.0% | Via SAFE, $10M post-money cap |
| **Total** | **10,000,000** | **100%** | Post pre-seed, pre-Series A |

After Series A ($5M at $50M post-money):
| Shareholder | % Ownership |
|-------------|-------------|
| Founder | 64.0% |
| Option Pool (expanded) | 12.0% |
| Pre-Seed Investors | 8.0% |
| Series A Investors | 10.0% |
| New Option Pool | 6.0% |

### SAFE Terms Recommendation

- **Instrument:** YC Standard SAFE (post-money)
- **Valuation Cap:** $10M post-money
- **Discount:** 20%
- **Pro-rata rights:** Yes (standard for lead investor)
- **MFN (Most Favored Nation):** Yes (standard)
- **No board seat** at pre-seed (maintain founder control)
- **Information rights:** Quarterly update email (informal)

---

## Investor FAQ

### The 20 Hardest Questions — With Answers

**1. "Why will you win as a solo founder?"**

I've built the entire MVP — backend, CLI, proxy, dashboard, infrastructure — in 12 weeks. That's the same output a 4-person team would produce in the same timeframe at a big company. I'm raising specifically to hire 2 engineers to accelerate, not to start building. The product is already functional. Solo founders built Vercel (Guillermo started alone), Heroku (James Lindenbaum was the primary builder), and Supabase (started with 2). Capital-efficient execution is a feature, not a bug.

**2. "What if AWS builds this?"**

AWS already tried with Bedrock Agents. It's rigid, framework-locked, and has terrible DX. AWS optimizes for enterprise procurement, not developer experience. They don't build great developer tools — they build infrastructure. Heroku existed for 10+ years alongside AWS. Railway, Render, Vercel all thrive alongside AWS. The opportunity is the developer experience layer that AWS will never prioritize.

**3. "What if LangChain builds their own deployment platform?"**

LangChain's business is the framework and LangSmith (observability). Building a deployment platform is a fundamentally different technical challenge — container orchestration, multi-cloud infrastructure, billing systems. It's like asking "what if React built Vercel?" They're complementary, not competitive. We'd love to be the recommended deployment platform in LangChain's docs. Harrison Chase is on our angel investor target list for exactly this reason.

**4. "How is this different from just running Docker containers?"**

Running Docker containers is step 1 of 50. You also need: a container registry, an orchestrator, health checking, auto-restart, log aggregation, a monitoring dashboard, LLM cost tracking, budget enforcement, model fallback, secrets management, DNS, SSL certificates, auto-scaling, zero-downtime deployments, and rollback. We do all 50 steps with one command. The LLM proxy alone — which intercepts, tracks, and controls every LLM call with zero code changes — is months of engineering.

**5. "Your market size seems aggressive. How did you get $20B for agent orchestration?"**

$20B comes from: 84% of enterprises plan to deploy AI agents by 2027 (Gartner). Average enterprise will run 50-200 agents. At $100-500/month per agent (our pricing), that's $60K-$1.2M per enterprise per year. With 500,000+ companies building AI products (per McKinsey), even capturing the mid-market segment at $10K/year average gives $5B. At maturity with platform expansion (marketplace, multi-cloud, GPU), $20B is conservative.

**6. "What's your moat? Why can't someone copy this in 3 months?"**

Three moats: (1) **Technical depth**: The LLM proxy with cross-provider format translation, real-time budget enforcement, and intelligent caching took 4 weeks of dedicated engineering. The telemetry pipeline took 3 weeks. A copy takes 6+ months. (2) **Ecosystem lock-in**: Once developers deploy 10+ agents on Deployra, migration cost is high — new URLs, new monitoring, re-configuration. (3) **Data advantage**: Every LLM call processed teaches us about agent workloads, cost patterns, and failure modes. This data improves our proxy, caching, and recommendations over time.

**7. "How do you acquire customers without a sales team?"**

Product-led growth. Developer tools sell themselves when the DX is exceptional. Our funnel: developer searches "how to deploy langchain agent" → finds our SEO-optimized tutorial → tries free tier → deploys agent in 5 minutes → realizes the cost tracking alone is worth $29/month → upgrades. Zero sales touch. Vercel, Supabase, and Railway all scaled to $10M+ ARR primarily through PLG before adding sales.

**8. "What's your gross margin at scale?"**

75-80%. Our primary cost is AWS compute (ECS Fargate) for running customer agents. Fargate cost per agent: ~$1.80/month for basic (256m CPU/512MB). We charge $29+/month. The LLM proxy and telemetry pipeline run on shared infrastructure that costs ~$0.001 per request to operate. As we scale, compute costs decrease (reserved instances, savings plans) while revenue per customer increases (more agents, more usage).

**9. "How do you handle the security of user LLM API keys?"**

User API keys are encrypted with AES-256-GCM at the application level before storage in our database. The encryption keys are stored in AWS Secrets Manager with IAM policies scoped to the specific service that needs them. Keys are only decrypted in-memory by the LLM proxy at request time. They're never logged, never stored in plaintext, and never accessible to any human — including our team. We're building toward SOC 2 Type 1 compliance by Month 12.

**10. "What's your path to $1M ARR?"**

$1M ARR = $83K MRR. At our pricing: ~1,800 Pro users ($29/mo = $52K MRR) + ~40 Team accounts averaging 4 seats ($99 × 4 = $16K MRR) + usage-based overage ($15K MRR). We reach this at ~1,500 paying customers. With 8% free-to-paid conversion and 40% month-1 retention, we need ~19,000 total signups. At 400 signups/month (our Month 6 target), we hit $1M ARR in Month 14-16.

**11. "Why Go instead of Rust or Node.js for the backend?"**

Go gives us: (1) Excellent performance for the LLM proxy (sub-5ms overhead per request). (2) First-class HTTP and WebSocket support in the standard library. (3) Goroutines for concurrent agent management. (4) Fast compile times for rapid iteration. (5) Strong hiring pool — more Go developers than Rust, and Go developers tend to be infrastructure-focused. Rust would be premature optimization. Node.js lacks the performance for proxy workloads. See TECH_STACK_DECISIONS.md for the full analysis.

**12. "What if the AI agent market doesn't materialize as expected?"**

The "AI agent" label might change, but the underlying need won't. Companies are automating workflows with LLMs. Whether they call them "agents," "workflows," "pipelines," or "autonomous systems," they need to deploy, monitor, and control LLM-powered software. Even if the agent hype cools, the core product — deploy LLM-powered applications with cost tracking and controls — remains valuable. We're not betting on a framework; we're betting on LLM-powered software being important.

**13. "How do you handle agent failures in production?"**

Three layers: (1) **Health checks**: Every 30 seconds, our sidecar pings the agent's health endpoint. Three consecutive failures trigger auto-restart (up to 5 restarts with exponential backoff). (2) **LLM fallback**: If OpenAI goes down, the proxy automatically routes to the configured fallback model (e.g., Anthropic) with request format translation. (3) **Budget safety**: If an agent enters an infinite loop, the budget enforcement catches it when the daily limit is hit and pauses the agent within seconds.

**14. "What's your pricing power? Can you raise prices?"**

Strong pricing power because: (1) Our pricing is 70-80% cheaper than the DIY alternative (AWS + custom monitoring). (2) The value proposition increases with usage — more agents = more cost savings from the proxy and caching. (3) Enterprise customers value the compliance, audit logs, and team controls far more than the price. (4) Usage-based pricing naturally grows with customer success. We can also add premium features (GPU support, multi-region, advanced analytics) at higher price points.

**15. "What's your churn going to look like?"**

We expect ~5% monthly churn for Pro users and ~2% for Team users. Developer tool churn is lower than average SaaS because of switching costs (re-deploying all agents, updating CI/CD, training team). Our best retention lever: the more agents a customer deploys, the harder it is to leave. Target: 120% net revenue retention (expansion from more agents and usage exceeds churn).

**16. "How do you plan to expand into enterprise?"**

Enterprise motion starts at Month 9 with a pilot program. We target companies that already have AI engineers deploying agents (find them by LangChain GitHub contributors, AI engineer job postings). The pitch: "Your team is already deploying agents on cobbled-together infrastructure. Deployra standardizes this with per-team budgets, RBAC, audit logs, and SOC 2 compliance." Enterprise ASP target: $30-50K/year. We add SSO (SAML), custom SLAs, and dedicated support for enterprise tier.

**17. "What are the biggest risks?"**

Three risks I think about daily: (1) **Platform risk from LLM providers**: If OpenAI or Anthropic changes their API or adds competing deployment features. Mitigation: we support all providers, so we're diversified. (2) **AWS dependency**: Our runtime runs on ECS Fargate. If AWS has major outages or raises prices significantly, it impacts us. Mitigation: architecture allows migration to GCP/Azure. (3) **Security breach**: We handle user API keys and agent traffic. A breach would be devastating. Mitigation: defense-in-depth security, SOC 2 process, application-level encryption.

**18. "How much have you spent so far?"**

About $3,000 total: $500/month AWS costs for 6 months (development and staging). Domain registration: $50. No paid marketing. No paid tools beyond AWS. Everything else was sweat equity. This is the most capital-efficient pre-seed company you'll see.

**19. "What does the team look like after the raise?"**

Three people: me (CEO/CTO, full-stack, focus on architecture and product), one senior backend engineer (Go, infrastructure, security — target: ex-AWS/GCP/Vercel), one senior full-stack engineer (TypeScript, React, CLI, DX — target: ex-Stripe/Vercel/GitHub). These two hires are force multipliers: they let me focus on product strategy and growth while they own core technical domains.

**20. "What does success look like in 18 months?"**

By December 2027: $50K+ MRR, 1,000+ weekly active agents, 3-person engineering team shipping weekly, SOC 2 Type 1 complete, 5+ enterprise pilots, Series A-ready with a clear path to $5M ARR. We'll have proven that AI agent deployment is a real category, and Deployra is the category leader.

---

*This fundraising playbook is a living document. Update after every investor meeting with new questions and refined answers.*
