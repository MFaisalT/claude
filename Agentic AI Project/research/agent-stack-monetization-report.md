# Monetizing the Paperclip × OpenClaw × Hermes Agent Stack

**A practical research report for solo founders and small teams, October 2026**

> Produced by applying the saved instructions in [`../prompts/agent-stack-monetization.md`](../prompts/agent-stack-monetization.md).
> **Evidence labels:** `[Verified]` means confirmed in the project's primary README or docs. `[Reported]` means from secondary sources; check before relying on it. `[Assumption]` means a modelling input I chose and explain.
> Model prices are Anthropic first-party API list prices as of 2026-10-06. Re-check before you quote clients.

---

## 0. Executive summary

**Thesis.** These three projects fit together cleanly as **governance (Paperclip) → channels (OpenClaw) → cognition (Hermes)**. The money does not come from the agents. It comes from picking a **narrow vertical in which an agent finishes a unit of work you can bill for**, such as a resolved ticket, a booked appointment, a published article, or a delivered brief, at a **gross margin above 80%**. The stack's real edge is control: Paperclip's budgets, approvals, and per-company isolation let one operator run many client "companies" without margins or risk getting out of hand.

**Ranked business models for a solo founder:**

| Rank | Model | Why |
|---|---|---|
| 1 | **Vertical support and booking network** (WhatsApp/Telegram for clinics, salons, trades, real estate) | Fastest to first revenue, billable per resolution or booking, low capital at risk, model cost per conversation measured in cents |
| 2 | **Niche intelligence briefs** (one vertical, paid daily or weekly) | Token cost is fixed per brief, not per subscriber, so margins grow with the list. The moat is data access, not prompts |
| 3 | **Content studio**, sold as an add-on to #1 or #2 | Cheap to produce, but output is commoditized and search engines penalize scaled low-value content |
| 4 | **"Zero-human" ads agency** | Highest revenue per client, but you are spending other people's money through APIs with gated access. Start with humans approving every change |

**Recommended first business:** #1, aimed at a single vertical. It has the clearest ROI story ("we answered 1,140 messages and booked 86 appointments this month"), a pricing anchor buyers already understand (per-resolution AI support), and it is the cleanest test of all three layers working together.

### Reality check: where the original brief needs correcting

1. **Budget thresholds are configurable, not a fixed 80%/100%.** Paperclip describes "scoped budget policies with warning thresholds and hard stops". It also warns that enforcement "uses recorded spend", so usage reporting and work already in flight can delay a stop `[Verified]`. Treat Paperclip as the *first* line of defense and set provider-side caps as the second (§3.1).
2. **Hermes `MEMORY.md` cannot grow without limit.** It is capped at **2,200 characters** (about 800 tokens), and `USER.md` at **1,375 characters**. A write that would exceed the cap returns an error, and the agent has to consolidate. Both files are loaded as a frozen snapshot at session start `[Reported, from mirrors of the official docs]`. The things that *do* grow are the session-search database, the skill library, OpenClaw workspace memory files, and Paperclip task threads (§4.3).
3. **`SOUL.md` and `MEMORY.md` are not unique to Hermes.** OpenClaw also uses workspace persona and memory files, and Hermes's `hermes claw migrate` imports OpenClaw's `SOUL.md`, memories, and skills `[Verified]`. The "brain" role belongs to Hermes because of its **learning loop** (it writes and refines skills after complex tasks) and its **sandboxed execution backends**, not because of the file names.
4. **OpenClaw and Hermes overlap heavily.** Both ship multi-channel gateways, cron, and skills `[Verified]`. If you don't give each capability a single owner, you get two bots answering the same customer. §1.3 assigns ownership.
5. **"Zero-human" is a sales line, not an operating model.** Each business below names its required human touchpoints. For anything that spends money, publishes publicly, or messages customers in bulk, the human is the **Paperclip board**.
6. **Platform policy limits these businesses as much as the technology does.** Since 15 January 2026, Meta's WhatsApp Business terms ban *general-purpose* AI assistants. Business-specific bots for support, booking, and sales are still allowed `[Reported]`. Similar constraints apply to ads APIs, search spam policies, and financial publishing (§2).

---

## 1. Architectural synergy and integration matrix

### 1.1 Role matrix

| Concern | **Paperclip**: the company | **OpenClaw**: the front desk | **Hermes Agent**: the back office |
|---|---|---|---|
| Core job | Goals, org chart, issues/tickets, scheduling, budgets, approvals, audit | Customer conversations across channels in real time; light tools; browser actions | Heavy multi-step work, research, code and data jobs, learning new skills |
| Key primitives `[Verified]` | Companies → goals → projects → issues (atomic checkout, execution locks, blockers); **routines** triggered by cron, webhook, or API; heartbeats through a DB-backed wakeup queue; Skill Studio with versioning and rollback; connection actions set to *Allowed / Ask first / Off*; board approvals; revisioned config | Gateway control plane; 20+ channels (WhatsApp, Telegram, Slack, Discord, iMessage, Teams, Signal, Google Chat, …); DM pairing by default; optional Docker sandbox (`off` / `non-main` / `all`); plugins and skills via ClawHub | Bounded curated memory (`MEMORY.md`, `USER.md`), `SOUL.md` persona, full-text session search; autonomous skill creation and patching (agentskills.io-compatible); isolated parallel subagents; 7 terminal backends (local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox); built-in cron; MCP; providers including Nous Portal, OpenRouter, OpenAI, and custom endpoints |
| How it's invoked | Human board through the React UI; webhooks or API into routines | Inbound channel message; Paperclip HTTP/webhook adapter `[Verified: OpenClaw listed as a Paperclip adapter]` | Paperclip's **Hermes adapter** (Hermes and Hermes Gateway are listed adapters) `[Verified]`; cron |
| Must **not** do | Talk to end customers directly | Run long or expensive jobs; hold host shell privileges; install unreviewed skills | Own a public-facing channel in the three-layer setup; spend money without a Paperclip approval |
| State it owns | Ledger of work, spend, approvals, and artifacts (PostgreSQL) | Per-customer conversation sessions and short-term context | Long-term operating knowledge (skills) and a compact client and user profile |

### 1.2 End-to-end request flow (support and booking example)

```
 End user (WhatsApp / Telegram)
        │ 1. message
        ▼
 ┌──────────────────────────┐   fast path (≈90% of traffic)
 │ OpenClaw gateway         │── 2. answer FAQ / check slots / take booking ──► reply in < 5 s
 │ (per-client container,   │
 │  sandbox mode "all")     │── 3. needs heavy work or policy decision
 └──────────┬───────────────┘        │ POST signed webhook → Paperclip routine
            │                        ▼
            │            ┌──────────────────────────────┐
            │            │ Paperclip (company = client) │ 4. routine creates issue,
            │            │  goal link · budget check    │    links goal, checks budget
            │            └──────────┬───────────────────┘
            │                       │ 5. heartbeat wakes assignee (Hermes adapter)
            │                       ▼
            │            ┌──────────────────────────────┐
            │            │ Hermes worker (Docker/Modal) │ 6. subagents research / compute,
            │            │  skills · memory · MCP       │    write work product, refine skill
            │            └──────────┬───────────────────┘
            │                       │ 7. task review / board approval if gated
            │                       ▼   (refund, discount, bulk message, spend)
            │            ┌──────────────────────────────┐
            └────────────│ Paperclip → OpenClaw adapter │ 8. deliver result to customer thread
                         └──────────────────────────────┘
                                    │ 9. payment link / booking confirmation / upsell offer
                                    ▼
                         Revenue event (Stripe / booking system) → logged against issue
```

**The same flow as steps:**

1. A customer message arrives at that client's own OpenClaw gateway. Unknown senders go through DM pairing or an allowlist. For a public business line, use an allowlist policy combined with rate limiting.
2. On the fast path, a small model (§3.3) answers using skills scoped to the client: the FAQ, opening hours, a booking API, and a catalog. Most traffic should end here.
3. Escalation triggers send the work to Paperclip through a webhook-triggered routine. Each run of a routine creates a tracked task `[Verified]`. Typical triggers are low confidence, a refund or discount request, a complaint, an order above a set value, a multi-step research task, or anything that changes money or data.
4. Paperclip creates an issue under that client's company, links it to a goal (for example "CSAT ≥ 4.6, response < 1 min"), and runs a **budget check before invocation** `[Verified: wakeup queue → budget checks → workspace resolution → secret injection → skill loading → adapter invocation]`.
5. The heartbeat wakes the assigned Hermes agent.
6. Hermes runs in a sandboxed backend, fans out subagents if needed, and writes a work product. If the job was new, it saves or patches a skill so the next one is cheaper.
7. If the action is gated, Paperclip holds it for **task review or board approval**. Gated actions include refunds, discounts above a threshold, outbound campaigns, ad-spend changes, and publishing.
8. Paperclip returns the result to OpenClaw, which replies in the original thread.
9. Revenue is recorded against the issue: a payment link, a booking confirmation, or an accepted upsell. This gives you **revenue per issue**, the input for token ROI in §3.2.

### 1.3 Resolving the overlap

| Capability | Owner | Rationale |
|---|---|---|
| Customer-facing channels | **OpenClaw** | Widest channel coverage, including iMessage, Teams, and Google Chat. Keep the Hermes gateway disabled for customer channels |
| Operator alerts (budget warnings, approvals waiting) | **Hermes gateway** *or* Paperclip connectors, sent to *your* private Telegram | Keeps operator traffic separate from customer traffic |
| Scheduling | **Paperclip routines** | Every run becomes an auditable, budgeted issue. Use Hermes's built-in cron only for maintenance inside a single worker |
| Skills | Hermes **authors** them; Paperclip Skill Studio **versions, tests, and rolls back** the ones you promote | Self-written skills are untrusted until reviewed. ClawHub skills are not installed in production (§4.1) |
| Long-term memory | Hermes for each client's operating knowledge; OpenClaw for each customer's conversation context | Each piece of memory has one writer |

**Two-layer alternative (recommended for v0):** run **Paperclip + Hermes** only, and let the Hermes gateway handle Telegram, WhatsApp, Slack, Signal, or email. This gives you a smaller attack surface, one less runtime, and a migration path (`hermes claw migrate`). Add OpenClaw once a client needs a channel that only it supports, or wants its richer device features.

### 1.4 Hosting blueprint (24/7, sandboxed)

**Topology** (sizes are `[Assumption]`, because model inference runs through APIs and CPU load is light; headless browsers use the most RAM, roughly 0.5–1 GB each):

| Host | Runs | Size to start | Exposure |
|---|---|---|---|
| **A: Control plane** | Paperclip, PostgreSQL (or managed Postgres), Caddy | 4 vCPU / 8 GB | Admin UI reachable **only over a tailnet** (`--bind tailnet`, authenticated mode) `[Verified]`. Caddy exposes only the webhook paths, protected by HMAC |
| **B: Edge** | One **OpenClaw container per client**, each with its own Unix user, volume, and channel credentials | 4 vCPU / 8 GB for about 10–20 low-volume clients | No public ports where the channel allows it (Telegram long polling). Webhook channels go through Caddy only. Gateway bound to loopback |
| **C: Execution** | Hermes workers on the **Docker** backend, or **Modal / Daytona** serverless backends that hibernate when idle `[Verified]` | 4 vCPU / 16 GB, or pay per use | No inbound ports. Outbound traffic restricted by an egress allowlist |

**Bootstrap commands** (verified against the projects' READMEs; pin versions in production):

```bash
# Host A: Paperclip
npx paperclipai@latest onboard --yes          # quickstart (embedded Postgres for local)
# production: point at your own Postgres; bind to the tailnet only (--bind tailnet)

# Host B: OpenClaw (inside each client container)
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon
openclaw gateway status
openclaw pairing approve <channel> <code>      # approve a paired sender

# Host C: Hermes
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes setup            # wizard; select Docker/Modal/Daytona terminal backend
hermes model            # choose provider:model
hermes claw migrate --dry-run   # only if consolidating from an OpenClaw install
```

**OpenClaw sandbox configuration.** Use these keys from the official sandboxing docs `[Verified]`. For customer-facing bots, use `"all"`, not the documented example value `"non-main"`:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",            // every session sandboxed; "non-main" leaves the main session on the host
        backend: "docker",
        scope: "agent",
        workspaceAccess: "none",
        workspaceRoot: "~/.openclaw/sandboxes"
      }
    }
  }
}
```

Two caveats from the docs `[Verified]`: the **gateway itself is never sandboxed**, and **elevated exec runs on the host and bypasses the sandbox**. Turn elevated exec off for customer agents.

**Baseline hardening** (standard Linux practice):

- Default-deny firewall: `ufw default deny incoming`. Allow 443 on host A only.
- Tailscale for all admin access. SSH by key only, from the tailnet only.
- Unattended security upgrades. Pin the versions of all three agent projects and upgrade on a schedule, after reading the release notes and security advisories.
- Secrets live in Paperclip secret injection or a vault, **never** in skills or memory files. Use one provider API key per client (see §3.1).
- Backups: nightly `pg_dump` to encrypted object storage, plus encrypted snapshots of the Hermes and OpenClaw data directories. They contain customer PII.
- Observability: Paperclip supports OpenTelemetry and Sentry `[Verified]`. Add an external uptime check per client gateway and route alerts to your phone.

---

## 2. Monetization blueprints

Unit-economics inputs used throughout come from §3.2. All prices are `[Assumption]` unless marked otherwise.

### 2.1 Vertical support and upsell network (rank #1)

**Offer:** "An always-on WhatsApp and Telegram front desk for <vertical> that answers, books, follows up, and upsells, and hands anything tricky to the owner's phone."

| Layer | Responsibilities |
|---|---|
| **Paperclip** | One company per client. Goals: response time, booking rate, CSAT. Routines for nightly summaries, weekly reports, and reactivation campaigns (approval-gated). The escalation queue, where the client owner is an approver for refunds and exceptions. Budget policy per client |
| **OpenClaw** | Client-specific gateway on the official WhatsApp Business number and/or a Telegram bot. Fast-path skills for FAQ, availability lookup, booking create/modify, and payment links. Per-customer conversation context |
| **Hermes** | Back-office jobs: synthesizing the FAQ from the client's website and documents, analysing complaints, drafting reactivation offers, and turning resolved escalations into new FAQ skills (the learning loop). Keeps a short `USER.md`-style client profile (tone, policies, prices) |
| **Third parties** | Booking system (Calendly, Cal.com, or a vertical PMS), Stripe, and the WhatsApp Cloud API through Meta or a BSP such as Twilio |

**Delivery loop:** Day 0 onboarding takes about two hours of your time: ingest documents, generate the FAQ skill, connect the calendar, and set up a test number. From day 1 the bot runs, unresolved items reach the owner through Paperclip and Telegram, and there is a nightly digest. Each week you send a report covering conversations, resolutions, bookings, revenue influenced, and the top unanswered questions. Those unanswered questions become new skills once you approve them.

**Pricing (recommended):** a base subscription plus an outcome fee.

- A base of $149–$399 per month per location covers the number, channels, and up to N conversations `[Assumption]`.
- Add **$0.50–$1.00 per AI-resolved conversation** above N, *or* **$2–$5 per booked appointment**. Market anchor: Intercom prices its Fin AI agent at **$0.99 per resolution** `[Reported]`.
- Avoid pure revenue share for SMBs. Attribution disputes cost more than the extra fee is worth.

**Unit economics** for one location with 1,000 conversations a month, 85% resolved by AI, and 60 bookings:

| Line | Monthly |
|---|---|
| Revenue: $249 base + 60 × $3 bookings | **$429** |
| Model cost: 1,000 conversations on the fast path with Claude Haiku 5.5 (about $0.01 each), plus 150 escalations on Claude Sonnet 5.5 (about $0.90 each) | ≈ $10 + $135 = **$145** (cut this by escalating less; see §3.3) |
| WhatsApp messaging (service replies inside the 24-hour window; templates billed per message) | $0–$30 `[Reported: Meta per-message pricing; check your market]` |
| Infrastructure share | ≈ $15 |
| Your time: 1 hour a month at $100 an hour | $100 |
| **Contribution** | **≈ $140–$170 (32–39%)**. Pushing the escalation rate from 15% to 5% lifts this to roughly 55–60% |

The lesson: **the escalation rate drives the margin, not the model price.** Every escalation that Hermes turns into a fast-path skill makes every later conversation cheaper. Track the escalation rate per client every week.

**Human touchpoints:** the client owner approves refunds, discounts, and complaints. You approve every new skill before it is promoted, and every outbound campaign.

**Policy and legal:** follow WhatsApp's business-bot rules and the 24-hour customer-service window, and get opt-in before marketing templates. Under TCPA (US) and GDPR/ePrivacy (EU), get consent for proactive messages. Disclose that the user is talking to an AI; several jurisdictions require it. **Do not run commercial WhatsApp through an unofficial WhatsApp Web session.** Use the official Cloud API, because number bans destroy the client relationship `[Assumption: confirm which WhatsApp transport your OpenClaw version uses]`.

**Moat:** a growing per-client skill library, integrations, and vertical templates you can clone into a new Paperclip company in minutes.
**Kill criteria:** escalation rate still above 25% after 60 days, or churn above 5% a month across the first 10 clients.

### 2.2 Niche intelligence and "alpha" briefs (rank #2)

**Offer:** "Every morning at 7:00, the 5 things that matter in <niche> and what to do about them." Example niches: commercial real-estate listings in one metro, municipal tenders, SaaS pricing changes, regulatory filings in one sector.

| Layer | Responsibilities |
|---|---|
| **Paperclip** | A company per product. Routines handle ingestion (hourly), synthesis (daily), and the weekly deep dive. An editorial-review gate controls publishing, and the budget is per product |
| **Hermes** | Scrapers and API pullers in the Docker backend; subagents for extraction and classification on a cheap model; synthesis on a frontier model; skill growth (new source parsers). Memory holds the house style and the subscriber segments |
| **OpenClaw** | Distribution and access control: a paid Telegram channel or bot, a WhatsApp broadcast to opted-in subscribers, a Slack Connect channel for B2B, and Q&A against the archive ("ask the brief") for paying members only |
| **Third parties** | Stripe for subscriptions, a newsletter platform for email, and licensed data feeds where needed |

**Pricing:** B2C/prosumer at $15–$49 per month. B2B team seats at $199–$999 per month with "ask the brief" Q&A and custom alerts `[Assumption]`.

**Unit economics:** the cost per brief is about **$0.50 a day** (§3.2, example D), so roughly $15 a month *per product*, **not per subscriber**. With 100 subscribers at $29, MRR is $2,900 and gross margin is about 95% before data licensing. **Data access is the real cost and the real moat.**

**Policy and legal:**
- **Scraping:** respect each site's terms of service and robots.txt. Prefer official APIs and licensed feeds. US case law (*hiQ v. LinkedIn*) narrowed CFAA exposure for public data, but contract claims still apply. MLS and other real-estate data is usually licensed, not scrapable.
- **Financial content:** stay impersonal, general, and published on a regular schedule (the *Lowe v. SEC* publisher's exclusion). Never give personalized buy or sell advice; that is investment-adviser territory. Add disclaimers and get legal review before launching any markets product.

**Kill criteria:** fewer than 30 paid subscribers after 90 days of daily publishing, or a free-to-paid conversion rate below 2%.

### 2.3 Autonomous content studio (rank #3, best as an add-on)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | A company per brand. Goals tied to traffic and leads. Content calendar as recurring issues. **Every publish goes through an approval gate** (client or editor). Budget per brand |
| **Hermes** | Trend research subagents, briefs, drafts, image and video prompts through MCP tools, and repurposing one pillar into ten posts. The learning loop turns winning formats into skills ("thread hook formula v3") |
| **OpenClaw** | Posting and engagement on chat-native channels (Telegram channels, Discord, Slack communities), DM follow-ups with leads, and browser automation only where no API exists, used sparingly |
| **Third parties** | CMS (WordPress, Webflow), social schedulers or native APIs, image and video generation APIs, analytics |

**Pricing:** a monthly retainer per brand, $500–$2,500 by volume, or a $99–$299 add-on for clients of the support or brief products `[Assumption]`.

**Economics:** one SEO article costs about $0.36 in model fees plus images (§3.2, example C). The model cost is close to zero. **Editorial QA time is the cost that matters.**

**Policy risk (the reason this ranks #3):** Google's spam policies target *scaled content abuse*, meaning mass-produced pages that add little value `[Reported]`. YouTube, TikTok, and Meta require labels on realistic synthetic media. Build for quality per piece (original data, expert quotes, the client's own case studies), not for volume.

**Kill criteria:** client-reported leads per month don't cover the retainer by month 3.

### 2.4 "Zero-human" ads agency (rank #4, highest ceiling, highest risk)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | A company per advertiser. **Every connection action that changes spend, budgets, bids, or campaign status is set to "Ask first"** `[Verified: Allowed / Ask first / Off]`. Daily optimization routine, weekly report routine. Agent budget for model spend, kept separate from the client's media budget |
| **Hermes** | Keyword research, search-term mining, negative-keyword lists, creative variants, landing-page audits, anomaly detection, and proposed bid and budget changes written as a diff for approval. Skills for each platform's playbook |
| **OpenClaw** | Client communication: a weekly summary in Slack or WhatsApp, approval requests the client answers with one tap ("Approve +20% on Campaign X?"), and alerts |
| **Third parties** | Google Ads API (requires a developer token and access-level approval), Meta Marketing API (app review), TikTok Marketing API, and the client's analytics and CRM for conversion data |

**Pricing:** a monthly management fee or a percentage of ad spend. Agency norms run roughly 10–20% of spend with a minimum `[Assumption]`. A performance share (a percentage of incremental ROAS) only makes sense once you have reliable server-side conversion tracking. Otherwise you are underwriting their tracking problems.

**Why autonomy has to stay gated:** a bad bid or budget change loses real money within hours. Ads APIs need approvals and audits. Most agents' "optimizations" are noise unless they work from statistically meaningful conversion data.

**Recommended path:** human approves every change for the first 90 days. After that, auto-approve only changes inside a **change envelope**: bid changes within ±15%, daily budget changes within ±10%, negatives only. Everything outside the envelope still needs approval.

**Kill criteria:** no measurable CPA or ROAS improvement against the client's baseline after 60 days on two or more accounts.

### 2.5 Comparison matrix

Scores run from 1 (worst) to 5 (best) for a solo founder `[Assumption: author's judgment]`.

| Model | Time to first $ | Capital at risk | Regulatory exposure | Automation ceiling | Defensibility | **Solo fit** |
|---|---|---|---|---|---|---|
| Support and booking network | 5 | 5 | 4 | 4 | 3 | **5** |
| Intelligence briefs | 3 | 4 | 3 (5 if not financial) | 5 | 4 (data) | **4** |
| Content studio | 4 | 5 | 3 | 4 | 2 | **3** |
| Ads agency | 3 | 2 | 3 | 2 (gated) | 3 | **2** |

---

## 3. Financial mechanics, budgeting, and token ROI

### 3.1 Making budgets follow revenue

Paperclip supports scoped budget policies with **warning thresholds and hard stops**, and agents pause when they hit a stop `[Verified]`. Use them as a **revenue-linked control system**, not as arbitrary limits:

1. **Set the client company budget from revenue:** `Budget_client = MRR_client × target model-cost ratio`. Use 15–25% for support, 5% for briefs, and 10% for content `[Assumption]`.
2. **Split the budget by agent:** for example, the escalation worker gets 60%, report and digest routines 15%, skill development 15%, and 10% is reserve.
3. **Set the thresholds:** a **warning at 70%** sends an alert to your Telegram and switches to frugal mode (lower effort, cheaper model for non-critical routines). A **hard stop at 90%**, not 100%, absorbs the documented enforcement lag from recorded spend and in-flight work.
4. **Add a second, provider-side line of defense.** Give each client its own provider key or workspace with its own spend limit, for example an Anthropic Console workspace or an OpenRouter key credit limit, set at 110% of the Paperclip budget. If Paperclip's accounting lags or a process bypasses it, the provider still stops the spend.
5. **Degrade instead of going dark.** A Paperclip hard stop should pause **back-office** work only. The OpenClaw fast path keeps running on its cheap model under its own capped key, and escalations queue up for a human. A support bot that stops answering is a churn event.
6. **Upsell from overages.** Clients who hit warnings two months in a row are on the wrong plan. Have the monthly report routine draft the upgrade proposal automatically.

### 3.2 Token ROI

**Definitions:**

```
LLM_cost(task) = Σ_calls [ in_uncached × p_in + in_cache_read × p_cache + out × p_out ]   (thinking tokens bill as output)
Contribution(task) = Revenue(task) − LLM_cost − tool/API cost − infra allocation − human_minutes × hourly_rate
Token ROI = Revenue(task) / LLM_cost(task)          Gross margin = Contribution / Revenue
```

**Prices used** (Anthropic first-party, per million tokens, as of 2026-10-06):

| Model | Input | Output | Cache read | Use in this stack |
|---|---|---|---|---|
| Claude Haiku 5.5 (`claude-haiku-5-5`) | $0.10 | $0.50 | n/a | Fast path, extraction, classification (prices apply to prompts ≤ 100K tokens) |
| Claude Sonnet 5.5 (`claude-sonnet-5-5`) | $2.00 | $10.00 | $0.20 | Escalations, drafting, most Hermes work |
| Claude Opus 5.5 (`claude-opus-5-5`) | $4.00 | $20.00 | $0.20 | Supervisor decisions, final QA, hard synthesis |
| Claude Fable 5.1 (`claude-fable-5-1`) | $10.00 | $50.00 | $0.25 | Rarely justified for these businesses |

The Batch API runs at **50% off** for work that isn't latency-sensitive: nightly synthesis, bulk extraction, content drafts.

**Worked examples.** Token counts are `[Assumption]`; the arithmetic is exact.

| # | Task | Token profile | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 |
|---|---|---|---|---|---|
| A | 8-turn support conversation | 70K input (6K reusable prefix + history), 6K output including thinking | **$0.010** | $0.20 uncached / **≈$0.11** with ~75% cache reads | $0.40 / ≈$0.20 |
| B | Hermes escalation job (research + action) | 300K input, 30K output | $0.045 | **$0.90** | $1.80 |
| C | SEO article pipeline | research 150K/8K (Haiku) + draft 30K/12K (Sonnet) + QA 20K/4K (Opus) | **≈$0.36 total** (≈$0.26 with Batch for research and draft) | | |
| D | Daily niche brief | extract 1M/50K (Haiku) + synthesize 60K/6K (Opus) | **≈$0.49 per day** (≈$0.42 with Batch for extraction) | | |

**What follows from these numbers:**
- At $0.99 per resolution, example A on Haiku returns about **99×** its model cost, and B on Sonnet about **1.1×**. **Support margins depend on keeping escalations rare**, so every repeat escalation should become a fast-path skill.
- Briefs (D) cost the same whether they have 10 subscribers or 10,000.
- The extra cost of the frontier model in C (≈$0.16) is small next to the price of the article. Spend tokens where quality is what the customer is paying for.

### 3.3 Model routing and cost optimization

**Routing ladder:**

| Tier | Handles | Engine |
|---|---|---|
| 0 | Opening hours, booking lookups, status checks, templated confirmations | **Deterministic code or skills, no LLM** |
| 1 | Intent detection, FAQ answers, extraction, classification | Claude Haiku 5.5, or a local open-weight model behind an OpenAI-compatible endpoint (Hermes supports custom endpoints `[Verified]`) |
| 2 | Escalations, drafting, multi-step tool use | Claude Sonnet 5.5 |
| 3 | Supervisor decisions: approving a refund recommendation, final editorial QA, ad change proposals | Claude Opus 5.5 at `high` effort |

**Escalation triggers** between tiers: low classifier confidence, money or a policy involved, negative sentiment, a third back-and-forth turn without resolution, or the customer asking for a human.

**Before you build a cascade, measure the simpler option.** Run one strong model at **lower effort** on the same tasks. Newer models at low effort often match older ones at high effort, and **prompt caches are model-scoped**, so a three-model cascade gives up cache reuse. Judge on **cost per completed task**, not cost per call. A cheaper call that needs a retry or a human escalation is not cheaper.

**Other levers, in order of payoff:**
1. **Prompt caching:** keep the system prompt, tool list, and skill descriptions byte-stable and put volatile data last. Hermes already freezes the memory snapshot per session for this reason `[Reported]`.
2. **Batch API** for everything that isn't real-time (50% off).
3. **Progressive skill loading:** Hermes keeps only short skill descriptions in the prompt and loads the full body when needed `[Reported]`.
4. **Output discipline:** structured outputs and short replies. Thinking tokens bill as output, so use low effort for routine routes.

**When self-hosted open-weight models pay off:**

```
Break-even monthly tokens ≈ GPU_cost_per_month / blended_API_price_per_token
e.g. $1,000/mo rented GPU ÷ ~$0.20 per 1M blended (Haiku 5.5, 4:1 in:out) ≈ 5 billion tokens/month
```

At Haiku-class API prices, **self-hosting rarely pays on cost alone** for a solo operator. Self-host for **data residency or confidentiality**, or when the volume is genuinely in the billions of tokens. Do not self-host to save money at small scale. `[Assumption: GPU price; plug in your quote]`

---

## 4. Guardrails, security, and scaling risks

### 4.1 Threat model

| # | Threat | Where | Evidence | Mitigation |
|---|---|---|---|---|
| 1 | **Gateway RCE or exposed control plane** | OpenClaw gateway | CVE-2026-25253 (gateway RCE, CVSS reported as 8.8–9.8). Tens of thousands of internet-exposed instances, many without authentication `[Reported]` | Never expose the gateway publicly. Bind to loopback or the tailnet. Patch quickly. Run `openclaw doctor`-style security checks where available. One gateway per tenant container |
| 2 | **Malicious skills** (supply chain) | ClawHub and other public skill registries | The "ClawHavoc" campaign: hundreds of malicious skills, with estimates from roughly 340 to more than 1,400. About 37% of scanned skills had flaws `[Reported: Koi, Bitdefender, Snyk]` | **No public-registry installs in production.** Build skills in-house or vendor-review them, pin them by hash, and promote them only through Paperclip Skill Studio with tests |
| 3 | **Sandbox escape or host execution** | OpenClaw tools, Hermes terminal | Tools run on the host for the main session unless sandboxing is configured. Elevated exec bypasses the sandbox `[Verified]`. A Docker escape has been reported `[Reported]` | `sandbox.mode: "all"`, elevated exec off, rootless Docker or gVisor, Hermes on the Docker, Modal, or Daytona backend (never `local` in production), read-only root filesystem, no host mounts |
| 4 | **Prompt injection** from customer messages, scraped pages, or emails | All layers | Inherent to LLM agents | Avoid the "lethal trifecta": don't let one agent combine private data, untrusted input, and an outbound channel. The fast-path agent has no secrets and no arbitrary HTTP. Scraping subagents can't message customers. Every money or bulk-message action goes through a Paperclip approval |
| 5 | **Credential exfiltration** | Skills, memory files, logs | Malicious skills targeted keychains, SSH keys, and wallets `[Reported]` | Secrets injected at runtime, scoped per client and rotated. Never write them to `SOUL.md`, `MEMORY.md`, or skills. Egress allowlists per container |
| 6 | **Runaway spend** | Loops, retries, self-triggering routines | Paperclip enforcement lags recorded spend `[Verified]` | §3.1 two-layer caps, per-routine concurrency of 1, a maximum number of turns per issue, and a circuit breaker on repeated failures |
| 7 | **Unreviewed self-modification** | Hermes skill creation and patching | By design `[Verified]` | Skills Hermes writes go to a staging namespace. Promotion requires a Skill Studio test run and your approval, and you can roll back |

### 4.2 Human-in-the-loop gates

Map each irreversible action to a Paperclip control:

| Action | Control |
|---|---|
| Hiring a new agent or changing an agent's model or tier | **Board approval** `[Verified: approve hires]` |
| Spend changes (ads budgets, bids), refunds, discounts above $X | Connection action set to **"Ask first"**, with the approver being you or the client owner |
| Publishing content, or bulk or outbound messaging | **Task review** before the issue can complete |
| New or modified production skill | Skill Studio test plus approval, with versioned rollback |
| Configuration changes | Revisioned and can be rolled back `[Verified]`. Review the diff weekly |

**Earn autonomy gradually:** widen an auto-approve envelope only after N consecutive approvals with zero reversals, and log the change as a board decision.

### 4.3 Memory hygiene

| Store | Does it grow? | Policy |
|---|---|---|
| Hermes `MEMORY.md` / `USER.md` | **No.** Capped at 2,200 and 1,375 characters `[Reported]` | Weekly routine: consolidate entries, move procedures into skills, keep only durable facts |
| Hermes session history and search index | Yes | Retention of 90 days for raw transcripts; keep summaries. Honor customer deletion requests (GDPR) |
| Skill library | Yes ("skill sprawl") | Monthly dedupe-and-merge routine. Retire skills unused for 60 days. Measure skill hit rate |
| OpenClaw per-customer context and workspace memory | Yes | Per-customer retention window. Summarize, then truncate. PII redaction before long-term storage |
| Paperclip issues, threads, and artifacts | Yes (this is your audit trail) | Keep it. Archive to cold storage after 12 months. Index for reporting |

### 4.4 Throttling and resilience

- **Provider rate limits (429):** honor `retry-after`, use exponential backoff with jitter, and cap concurrency per tenant. Paperclip's wakeup queue already merges duplicate wakeups `[Verified]`.
- **Fallbacks:** if Tier 2 fails or refuses, retry once on a sibling model. Otherwise queue the work for a human. Never loop silently.
- **Degraded mode:** if the LLM provider is down, the OpenClaw fast path switches to deterministic replies ("We've received your message; a team member will reply by 10:00") and opens a Paperclip issue.
- **Idempotency:** Paperclip's atomic issue checkout and execution locks prevent two workers handling the same job `[Verified]`. Make outbound actions (bookings, payments) idempotent with keys.

### 4.5 Multi-company isolation on one server

| Boundary | Paperclip provides | You must add |
|---|---|---|
| Data and permissions | Company-scoped tasks, agents, permissions, activity, and access checks in one deployment `[Verified]` | Separate Postgres schemas or databases if a client contract requires physical separation |
| Runtime | (not its scope) | **One OpenClaw container and one Hermes home or volume per client**, with a separate Unix UID, no shared volumes, and CPU and memory limits |
| Credentials | Secret injection per invocation `[Verified]` | A separate provider key, channel token, and OAuth app grant for each client |
| Network | (not its scope) | Per-container egress allowlist. No container-to-container traffic. Gateways on loopback or the tailnet only |
| Blast radius | Pause or terminate a company's agents `[Verified]` | A tested runbook: rotate that client's keys, wipe and redeploy its containers from pinned images |

---

## 5. Execution plan for a solo founder

### Days 1–30: build and land the design partner
1. Pick **one vertical and one city** (for example dental clinics or physiotherapy) and line up three design-partner prospects before writing code.
2. Stand up hosts A and C. Use the **two-layer stack** (Paperclip + Hermes gateway on Telegram) so you can iterate quickly.
3. Build the vertical template: FAQ ingestion skill, booking skill (Cal.com or the vertical's PMS), escalation routine, nightly digest, and weekly report.
4. Set budget policy (warning 70%, stop 90%) plus a capped provider key per client.
5. Launch one design partner for free or at cost, in return for a case study and weekly feedback.

**Gate:** escalation rate below 30% and the owner says "I'd pay for this".

### Days 31–60: production hardening and the first paying customers
1. Move customer channels to **OpenClaw on the official WhatsApp Business API**, with `sandbox.mode: "all"`, per-client containers, and a loopback gateway.
2. Promote reviewed skills through Skill Studio. Run the weekly memory and skill hygiene routines.
3. Turn on Stripe billing (base plus per-booking or per-resolution) and the outcome-based weekly report.
4. Sign 3–5 paying clients.

**Gate:** contribution margin above 40% per client and zero security findings from your own red-team pass (prompt injection, pairing bypass, secret leakage).

### Days 61–90: scale and the second product
1. Template the client company (Paperclip company package plus OpenClaw/Hermes container images) so onboarding takes under two hours.
2. Push the escalation rate under 10% using the learning loop. Target contribution margin above 60%.
3. Launch the **niche brief** for the same vertical (for example a "local healthcare operator weekly"), cross-sold to your support clients and sold on its own.

**Gate:** MRR covers your infrastructure, model, and tool costs at least 5×, and churn is under 5% a month.

### Launch checklist
- [ ] Gateway not publicly reachable. Admin UI on the tailnet only. Webhooks HMAC-signed
- [ ] Sandbox `all`. Elevated exec off. Hermes on Docker, Modal, or Daytona. Containers rootless with no host mounts
- [ ] No public-registry skills. All skills versioned and tested in Skill Studio
- [ ] Per-client provider keys with spend limits. Paperclip budgets set to 70/90
- [ ] Approval gates on refunds, discounts, bulk messages, publishing, and ad spend
- [ ] Official channel APIs. Opt-in and consent records. AI disclosure in the first message
- [ ] DPA and privacy policy. Retention and deletion routines
- [ ] Encrypted nightly backups, with a restore tested once
- [ ] Alerts: budget warnings, gateway down, error-rate spikes, escalation backlog
- [ ] Incident runbook per client: rotate, wipe, redeploy

---

## 6. Sources

**Primary (project docs and repos):**
1. Paperclip README: features, budgets, governance, adapters, heartbeat pipeline, multi-company, deployment. <https://github.com/paperclipai/paperclip>
2. Hermes Agent README: memory files, skills and agentskills.io, subagents, gateway, terminal backends, cron, providers, `hermes claw migrate`, install. <https://github.com/NousResearch/hermes-agent>
3. Hermes Agent docs: persistent memory (limits, frozen snapshot). <https://hermes-agent.nousresearch.com/docs/user-guide/features/memory> (doc mirror consulted: <https://www.mintlify.com/NousResearch/hermes-agent/user-guide/features/memory>)
4. OpenClaw README: channels, gateway, DM pairing, install, host-vs-sandbox default. <https://github.com/openclaw/openclaw>
5. OpenClaw sandboxing docs: `agents.defaults.sandbox` keys, gateway not sandboxed, elevated exec. <https://docs.openclaw.ai/gateway/sandboxing.md>
6. Anthropic model pricing (first-party API, cached 2026-10-06): Claude Developer Platform documentation.

**Secondary (`[Reported]`; verify before relying on them):**

7. OpenClaw security guide: CVE-2026-25253, malicious skills. <https://www.bitdoze.com/openclaw-security-guide/>
8. OpenClaw security vulnerabilities 2026. <https://blog.barrack.ai/openclaw-security-vulnerabilities-2026/>
9. OpenClaw Security Report, January 2026. <https://insiderllm.com/guides/openclaw-security-report-january-2026/>
10. Known vulnerabilities tracker. <https://clawdocs.org/security/known-vulnerabilities>
11. Meta's WhatsApp ban on general-purpose AI chatbots (effective 15 January 2026). <https://respond.io/blog/whatsapp-general-purpose-chatbots-ban>, <https://www.business-standard.com/technology/tech-news/meta-bans-ai-chatbots-from-whatsapp-business-api-chatgpt-to-shut-down-jan-2026-125102200335_1.html>
12. Hermes Agent technical overviews. <https://www.turingpost.com/p/hermes>, <https://mranand.substack.com/p/inside-hermes-agent-how-a-self-improving>
13. Paperclip deployment write-up. <https://zeabur.com/blogs/deploy-paperclip-ai-agent-orchestration>

**Not independently verified in this pass**, so check them before citing to clients: the default OpenClaw gateway port, the exact transport OpenClaw uses for WhatsApp, Intercom Fin's current per-resolution price, current WhatsApp per-message rates in your market, and Google, Meta, and TikTok ads-API access tiers.

*This report is not legal or investment advice. Get counsel before launching anything that touches financial content, regulated data, or automated consumer messaging.*
