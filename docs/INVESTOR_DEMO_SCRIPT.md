# Deployra — Live Demo Script for Investor Meetings

> Version 1.0 | March 2026 | Investor Meetings Playbook

## Table of Contents

1. [Pre-Demo Setup Checklist](#pre-demo-setup-checklist)
2. [5-Minute Live Demo Script](#5-minute-live-demo-script)
3. [Wow Moments](#wow-moments)
4. [Handling Demo Failures](#handling-demo-failures)
5. [Follow-Up Materials](#follow-up-materials)
6. [Q&A Preparation](#qa-preparation)

---

## Pre-Demo Setup Checklist

Complete these steps 30 minutes before every investor meeting.

### Environment Prep

- [ ] **Internet connection:** Verify stable connection. Have phone hotspot as backup.
- [ ] **Terminal:** Open iTerm2/Warp with font size 18+ (readable on projector/screen share).
- [ ] **Terminal theme:** Light background for readability (or dark with high contrast).
- [ ] **Browser:** Chrome with dashboard open in a tab. Clear history (no embarrassing autocompletes).
- [ ] **Dashboard:** Pre-authenticated at `app.deployra.ai`. Agent list page loaded.
- [ ] **Notifications:** Disable all desktop notifications. Enable Do Not Disturb.
- [ ] **Docker Desktop:** Closed (to avoid confusion — builds happen server-side).

### Demo Agent Prep

- [ ] **Demo project directory:** `~/demo/sales-agent/` — a real LangChain sales outreach agent.
- [ ] **deployra.yaml:** Pre-configured with sensible defaults.
- [ ] **Secrets set:** OpenAI API key already configured (`deployra config set secrets.OPENAI_API_KEY`).
- [ ] **Clean state:** Delete any previous deployments of the demo agent. Run `deployra destroy sales-agent --force` if needed.
- [ ] **Verify CLI:** Run `deployra --version` to ensure it's working.
- [ ] **Warm ECR cache:** Deploy the agent once beforehand so Docker layers are cached. Then destroy it. This ensures the live demo build takes ~20s instead of ~60s.

### Existing Agent for Metrics

- [ ] **"research-agent":** A second agent that's been running for 7+ days with real traffic. This shows dashboard metrics, cost charts, and log streaming.
- [ ] **Verify research-agent status:** `deployra status research-agent` — should be `running` with meaningful metrics.
- [ ] **Verify dashboard data:** Check that cost charts and run history have 7+ days of data.

### Backup Plan

- [ ] **Pre-recorded demo video:** 3-minute screencast of the full demo flow. Store on laptop (not in the cloud).
- [ ] **Screenshots:** 5 key screenshots of the dashboard saved locally. If dashboard is down, show screenshots.
- [ ] **Offline pitch deck:** PDF version of pitch deck on laptop (not just Google Slides).

---

## 5-Minute Live Demo Script

### Minute 0:00–0:30 — Setup the Context

**[Show the demo project in your editor/terminal]**

> "Let me show you the product. I have a real AI agent here — a LangChain-based sales outreach agent. It takes a company name, researches them, and drafts personalized cold emails. About 200 lines of Python."

**[Terminal: Show the project directory]**

```bash
ls -la ~/demo/sales-agent/
```

> "Standard Python project. `agent.py`, `requirements.txt`, a few modules. No Dockerfile, no infrastructure config — just the agent code."

---

### Minute 0:30–1:00 — Initialize

**[Terminal: Run deployra init]**

```bash
cd ~/demo/sales-agent
deployra init --name sales-agent
```

> "Watch this — the CLI scans the project and auto-detects the framework."

**[CLI output shows: "Detected framework: langchain (from requirements.txt)"]**

> "It found LangChain in the requirements, detected Python 3.12, and generated a `deployra.yaml` config file. One file, 15 lines. That's it."

**[Quickly show the generated deployra.yaml]**

```bash
cat deployra.yaml
```

> "Notice the budget controls — $25 per day, $500 per month. If the agent goes haywire, it pauses automatically. No $5,000 surprise bills."

---

### Minute 1:00–2:00 — Deploy (The Money Shot)

**[Terminal: Run deployra deploy]**

```bash
deployra deploy
```

> "One command. Watch the timer."

**[CLI shows progress in real-time]**

```
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
  Deploy: v1
```

> "31 seconds. From zero to production. No Dockerfile. No ECS task definition. No load balancer configuration. No Kubernetes cluster. Just `deployra deploy`."

**[Pause for effect. This is Wow Moment #1.]**

> "That URL is live right now. Let me hit it."

---

### Minute 2:00–2:30 — Trigger the Agent

**[Terminal: curl the agent endpoint]**

```bash
curl -s -X POST https://sales-agent-abc123.deployra.run/run \
  -H "Content-Type: application/json" \
  -d '{"company": "Stripe"}' | jq .
```

> "I'm sending it a request — 'research Stripe and draft an outreach email.' This is a real LLM call hitting OpenAI right now."

**[Wait 5-10 seconds for response. Show the output.]**

> "There it is — a personalized cold email based on real research about Stripe. The agent made 4 LLM calls to generate that."

---

### Minute 2:30–3:30 — Dashboard (Wow Moment #2)

**[Switch to browser — Dashboard agent list]**

> "Now here's where it gets interesting. Let me show you the dashboard."

**[Click into the sales-agent detail page]**

> "This agent has been running for 32 seconds and I can already see: one run completed, 4 LLM calls, $0.055 cost, 1,590 total tokens."

**[Now switch to the "research-agent" that's been running for 7+ days]**

> "But let me show you a more interesting agent — this research agent has been running for 10 days."

**[Show the research-agent detail page with rich metrics]**

> "Over the last 10 days: 1,247 runs, $187 in LLM costs, 97.8% success rate, average run takes 34 seconds. Every single data point here comes from our LLM proxy — zero instrumentation code from the developer."

**[Click into the Costs tab — show the cost breakdown chart]**

> "Here's the cost breakdown. You can see exactly which model costs what. This agent uses GPT-4o for complex queries and GPT-4o-mini for simpler ones. 72% of cost is GPT-4o. If I wanted to optimize, I'd switch more calls to the mini model."

**[Point to the budget bar]**

> "And see this budget bar — $12 out of $25 today. If this hits $25, the agent pauses automatically. Not an alert that someone ignores at 3 AM — the agent actually stops making LLM calls."

---

### Minute 3:30–4:00 — Live Logs (Wow Moment #3)

**[Click into the Logs tab for the research-agent]**

> "Real-time logs. Watch — these are streaming live right now."

**[Point to a log line showing an LLM call]**

> "Every LLM call is logged with the model, token count, and cost. You can see the agent's reasoning in real-time. This is the kind of visibility that takes months to build yourself."

**[Back to terminal — show CLI logs]**

```bash
deployra logs research-agent -f --level info
```

> "Same stream in the terminal. Our developers live in this view."

---

### Minute 4:00–4:30 — Model Fallback (Wow Moment #4)

> "One more thing. Let me show you the fallback chain."

**[Show the deployra.yaml config with fallback configuration]**

> "This agent is configured with OpenAI as primary and Anthropic as fallback. If OpenAI goes down — rate limited, server error, timeout — our proxy automatically retries, and if that fails, switches to Claude. Zero code changes. The developer configured this in 3 lines of YAML."

> "This is the kind of production reliability that enterprise customers demand and that no developer wants to build themselves."

---

### Minute 4:30–5:00 — Close

> "So to recap: I took a Python agent, deployed it in 31 seconds with one command, and immediately got: production hosting, real-time LLM cost tracking, budget enforcement, model fallback, live logs, and a monitoring dashboard. All without writing a single line of infrastructure code."

> "That's Deployra. The deployment platform for AI agents."

**[Pause. Take questions.]**

---

## Wow Moments

Engineer these moments to land with investors:

| # | Moment | When | Emotional Response |
|---|--------|------|-------------------|
| 1 | **31-second deploy** | After `deployra deploy` completes | "That was... fast." Establishes product quality. |
| 2 | **Live cost data appears** | Seconds after first agent run | "It already shows $0.055." Real-time feels magical. |
| 3 | **7-day dashboard** | Showing research-agent metrics | "This is the visibility I wish I had." Enterprise value click. |
| 4 | **Budget bar** | Pointing at the budget enforcement UI | "Auto-pause at $25/day." Every founder/CTO has a cost horror story. |
| 5 | **Model fallback config** | Showing 3 lines of YAML | "Zero code changes." Elegance of the solution. |

**Tip:** Don't rush these moments. Pause after each one. Let the investor process what they just saw. Silence is powerful.

---

## Handling Demo Failures

### Scenario 1: Build Fails

**Symptom:** `deployra deploy` shows build error.

**Recovery:**
> "Looks like there's a build issue — let me show you the build logs."

Show the error, then:
> "This actually demonstrates our error handling. The build failed, but the previous deployment is still running. Zero downtime. Let me fix this and redeploy."

**If the fix isn't quick:** Switch to the pre-running research-agent and demo the dashboard/logs instead. The deploy demo becomes a 30-second story instead of a live demo.

### Scenario 2: Internet Goes Down

**Recovery:**
> "Looks like the wifi dropped — let me switch to my hotspot."

Connect phone hotspot. If that also fails:
> "Let me show you the recorded demo instead."

Play the pre-recorded 3-minute video.

### Scenario 3: Dashboard Loads Slowly

**Recovery:**
> "The dashboard takes a moment to load — while it's loading, let me explain what you'll see..."

Describe the agent list, metrics, and cost charts while waiting. If it doesn't load within 15 seconds, show the prepared screenshots.

### Scenario 4: Agent Returns Error

**Symptom:** The curl request to the agent returns a 500 error.

**Recovery:**
> "Interesting — the agent hit an error. Let me check the logs."

```bash
deployra logs sales-agent --tail 10 --level error
```

> "You can see exactly what went wrong — the LLM returned an unexpected format. In production, this is what debugging looks like on Deployra. Full error context, the LLM call that caused it, and the cost it incurred."

**Turn the failure into a feature showcase.**

### Scenario 5: Everything Fails

**Recovery:**
> "Technology! Let me talk you through the product with some screenshots while my engineer fixes things."

Show the 5 prepared screenshots. Narrate the same story. The narrative is more important than the live demo — if you tell the story well, investors will trust the tech.

---

## Follow-Up Materials

Send within 2 hours of the meeting:

### Email Template

```
Subject: Deployra Follow-Up — Demo + Materials

Hi [Partner Name],

Great meeting you today. As promised, here are the follow-up materials:

1. Product demo recording: [link to 3-min video]
2. Architecture overview: [link to ARCHITECTURE.md or PDF]
3. Financial model: [link to one-page summary]
4. Pitch deck: [link to PDF]

Key numbers from our conversation:
- Deploy time: 31.7 seconds (you saw it live)
- LLM cost tracking accuracy: 100% (extracted from provider responses)
- Budget enforcement: real-time, sub-second (<1ms overhead)
- Current waitlist: 2,000+ developers

Happy to do a deeper technical dive or connect you with beta users.
What does your timeline look like for follow-up?

Best,
[Name]
```

### Materials Checklist

- [ ] Pitch deck (PDF, 12 slides)
- [ ] Demo video (3-min recording, hosted on Loom or YouTube unlisted)
- [ ] Architecture overview (1-page summary, not the full doc)
- [ ] Financial model (1-page summary with key metrics)
- [ ] Product screenshots (5 high-quality screenshots)
- [ ] Team bio / LinkedIn profile link
- [ ] Company one-pager (for partner to share internally)

---

## Q&A Preparation

### Questions Likely During/After Demo

**During Demo:**

| Likely Question | When It Comes Up | Answer |
|-----------------|------------------|--------|
| "How fast is that without cache?" | After the 31s deploy | "About 50-60 seconds. The first build downloads dependencies. Subsequent builds cache the dependency layer and take 20-30 seconds." |
| "What happens if the LLM call takes 5 minutes?" | After the agent run | "Our proxy has a 30-second timeout. If it's a streaming response, we support up to 5 minutes. The agent can also set custom timeouts." |
| "Where is this running?" | After deploy | "ECS Fargate on AWS. Each agent gets its own isolated container. No shared compute. Equivalent of a dedicated VM, but managed." |
| "How do you count tokens?" | After showing cost data | "We extract token counts from the provider's response. OpenAI, Anthropic, and Google all return token usage in their API responses. 100% accurate — we don't estimate." |
| "Can I see the Dockerfile?" | After showing auto-detection | "Absolutely. Run `deployra deploy --dry-run` and it shows the generated Dockerfile. You can also provide your own Dockerfile if you prefer." |

**After Demo:**

| Likely Question | Answer |
|-----------------|--------|
| "What frameworks do you support?" | "LangChain, CrewAI, AutoGen, and LlamaIndex out of the box. Any Python or Node.js agent works with the 'custom' framework option. We auto-detect the framework from imports and dependencies." |
| "How do you handle state/memory?" | "Agents can use any external state store — PostgreSQL, Redis, etc. We inject secrets for database URLs. For simple state, agents can use the ephemeral filesystem (5-20GB). We're building managed state persistence for Phase 2." |
| "What about GPU workloads?" | "Not yet — that's in our Year 2 roadmap. Most AI agents today use API-based LLMs (OpenAI, Anthropic), which don't need GPUs. When agents start running local models, we'll add GPU support on Fargate." |
| "Can agents talk to each other?" | "Currently, agents are isolated by design (security benefit). Multi-agent communication via shared databases or message queues works fine. Native agent-to-agent orchestration is Phase 2." |
| "What's your uptime been?" | "Our beta environment has been running for 8 weeks with 99.95% uptime. Two incidents: one Neon database cold start (3-minute delay, not downtime) and one ALB certificate rotation (30-second disruption). Both are now automated." |
| "How do enterprise customers feel about sending LLM traffic through your proxy?" | "This is the #1 question enterprises ask. Three facts that build trust: (1) we never log prompt content — only metadata (tokens, cost, latency). (2) All traffic is TLS 1.3 encrypted. (3) We can deploy the proxy in the customer's VPC for enterprise tier. We're building toward SOC 2 to provide auditable security guarantees." |

### Follow-Up Questions to Ask Investors

1. "What's your typical timeline from first meeting to term sheet?"
2. "Are there portfolio companies you think we should talk to?"
3. "What would you need to see to get conviction on this deal?"
4. "Would it be helpful if I connected you with one of our beta users?"
5. "Who else on your team should I meet?"

---

*Practice this demo at least 3 times before every investor meeting. Time yourself. The deploy must be under 40 seconds, the full demo under 5 minutes.*
