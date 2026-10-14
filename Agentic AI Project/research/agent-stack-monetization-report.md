# Monetizing the Paperclip × OpenClaw × Hermes Agent Stack

**A practical research report for solo founders and small teams, October 2026, second research pass**

> Produced by applying the saved instructions in [`../prompts/agent-stack-monetization.md`](../prompts/agent-stack-monetization.md).
>
> **Method.** Pass 1 used the project READMEs and web sources. Pass 2, this version, reads the **source code and in-repo docs** of all three projects, shallow-cloned on 2026-10-14:
>
> | Project | Snapshot used |
> |---|---|
> | `paperclipai/paperclip` | `7df55f3`, committed 2026-10-14 |
> | `NousResearch/hermes-agent` | HEAD, committed 2026-10-13 |
> | `openclaw/openclaw` | HEAD, committed 2026-10-14 |
>
> Market, policy, and legal claims come from web sources, listed in §6. Appendix A lists what changed from pass 1 and why.
>
> **Evidence labels:** `[Verified]` means confirmed in the project's source code or in-repo docs at the snapshot above. `[Reported]` means from secondary sources; check before relying on it. `[Assumption]` means a modelling input I chose and explain.
> Model prices are Anthropic first-party API list prices as of 2026-10-06.

---

## 0. Executive summary

**Thesis.** The three projects do fit together as **governance (Paperclip) → channels (OpenClaw *and* Hermes) → cognition (Hermes)**. The code changes three important parts of the original plan:

1. **Hermes, not OpenClaw, should own commercial WhatsApp.** OpenClaw's WhatsApp channel runs only over WhatsApp Web through Baileys. Its own docs say "there is no separate Twilio WhatsApp channel". Hermes ships an official **WhatsApp Business Cloud API** adapter, which its docs describe as the production path with "no account-ban risk" `[Verified]`.
2. **The cheap model must not hold the tools.** OpenClaw's security docs warn against running small or cheap models on tool-enabled agents that read untrusted input, such as public customer chats `[Verified]`. The profitable pattern is: a small model classifies and drafts, deterministic code performs the action, and a stronger model handles escalations (§3.3).
3. **OpenClaw is not a multi-tenant boundary.** Its docs say: "One trust boundary per gateway … not a hostile multi-tenant security boundary" `[Verified]`. Each client needs its own gateway, credentials, and ideally its own OS user or host.

The money still comes from a **narrow vertical where an agent finishes a unit of work you can bill for**. The stack's edge is control: Paperclip's budgets reserve spend before a run starts and fail closed, it has approval gates, and it isolates each company's data.

**No independently verified revenue case study** of Paperclip-run "zero-human companies" exists yet. Coverage is mostly promotional, and one secondhand report says Paperclip's creator acknowledged the project had not yet generated revenue `[Reported]`. Treat every number below as a model to test, not a benchmark.

**Ranked business models for a solo founder (unchanged from pass 1):**

| Rank | Model | Why |
|---|---|---|
| 1 | **Vertical support and booking network** (WhatsApp Cloud API + Telegram for clinics, salons, trades, real estate) | Fastest to first revenue. Buyers already understand per-resolution pricing (Intercom Fin: about $0.99 per resolution `[Reported]`). Modelled contribution of 55–67% per client (§2.1) |
| 2 | **Niche intelligence briefs** | Model cost is about $0.42–$0.49 per brief *per product*, not per subscriber. The moat is data access |
| 3 | **Content studio** (add-on) | Model cost is negligible, but Google's spam updates (June and August 2026) keep targeting scaled, low-value content `[Reported]` |
| 4 | **"Zero-human" ads agency** | Spends client money. Google Ads API production access starts at 2,880 operations a day (Explorer tier) `[Reported: Google developer docs]` |

**Recommended first build:** **Paperclip + Hermes** (one Hermes *profile* per client, with the official WhatsApp Cloud API and Telegram). Add **OpenClaw** only for channels Hermes doesn't serve well for a given client, such as iMessage through `imsg` on a Mac, or Teams. Keep it to one gateway per client.

### Reality check: the brief versus the code

| # | Claim in the brief | What the code and docs say |
|---|---|---|
| 1 | "80% alerts, 100% auto-pause" | **Correct as defaults.** Policies default to `warnPercent: 80` (configurable from 1 to 99) and `hardStopEnabled: true`, which pauses at 100% of `amount`. Scopes are `company`, `agent`, or `project`. Windows are `calendar_month_utc` or `lifetime` `[Verified: packages/db/src/schema/budget_policies.ts, packages/shared/src/constants.ts]`. Spend is **reserved before dispatch**, but the code comment warns: *"Estimates constrain admission; they cannot cap a provider's bill"* `[Verified: server/src/services/budget-reservations.ts]`. Pass 1 wrongly said the thresholds were not fixed at 80/100. |
| 2 | "Ever-expanding MEMORY.md" | **Wrong.** `MEMORY.md` is capped at 2,200 characters and `USER.md` at 1,375. A write that would exceed the cap returns an error and does not auto-compact. Both are loaded as a frozen snapshot per session `[Verified: website/docs/user-guide/features/memory.md]`. What *does* grow is **gateway session history**: "Gateway conversations do not reset after inactivity or at a daily boundary" `[Verified: sessions.md]` (§4.3). |
| 3 | "OpenClaw = the WhatsApp arm" | **Risky for commercial use.** OpenClaw's WhatsApp channel uses Baileys (WhatsApp Web) only. Hermes has `whatsapp-cloud` (official API) `[Verified]`. |
| 4 | "Paperclip delegates down to OpenClaw" | **Yes, as an employee.** OpenClaw joins a company through an invite prompt, then board approval, then a one-time API key claim. It connects over the gateway WebSocket (`openclaw_gateway`, default `ws://127.0.0.1:18789`) and installs a Paperclip skill, which lets it **create Paperclip tasks itself** `[Verified: packages/adapters/openclaw-gateway/doc/ONBOARDING_AND_TEST_PLAN.md]`. The legacy webhook adapter was removed. |
| 5 | "OpenClaw uses Hermes for heavy cognition" | **Not directly.** There is no OpenClaw→Hermes adapter. The connection runs **through Paperclip issues**. Hermes is invoked by Paperclip's built-in `hermes_local` (a CLI child process) or `hermes_gateway` (HTTP/SSE to a Hermes API server) adapters `[Verified: packages/adapters/hermes/README.md]`. |
| 6 | "Hermes = SOUL.md / MEMORY.md brain" | Both projects use persona and memory files. Hermes's real differentiators are its **skill learning loop** with a **curator** that marks unused skills stale after 14 days and archives them after 30, **profiles** (isolated homes), and **hardened Docker sandboxes** `[Verified]`. |
| 7 | "Zero-human" | Everything that spends money, publishes, or messages in bulk should pass a Paperclip approval or an "Ask first" gate. Under EU AI Act Art. 50, chatbots must disclose that they are AI from **2 August 2026** `[Reported]`. |

---

## 1. Architectural synergy and integration matrix

### 1.1 Role matrix (verified against code)

| Concern | **Paperclip**: the company | **Hermes Agent**: back office + official channels | **OpenClaw**: extra channels + device reach |
|---|---|---|---|
| Core job | Goals, issues, routines, budgets, approvals, audit, multi-company | Heavy multi-step work; skill learning; **WhatsApp Cloud API**, Telegram, Slack, Email and more through its gateway | Channels Hermes lacks or does worse at (iMessage via `imsg` on macOS, Teams, Google Chat, 20+ others); browser and device nodes |
| Integration with Paperclip | — | `hermes_local` or `hermes_gateway` adapter. The gateway adapter blocks non-loopback plain HTTP unless you set a "dangerous" override `[Verified]` | `openclaw_gateway` over WebSocket with a device key and token header; joins through board approval `[Verified]` |
| Wakes on | — | Assignment, issue comments ("comment-driven wakes"), routines `[Verified]` | Assignment, routines |
| Key safety features | Budget reservations, fail-closed hard stop, `hire_agent` / `budget_override_required` / `request_board_approval` approvals, tool connections set to `allowed` / `ask_first` / `off`, revisioned routines `[Verified]` | `approvals.mode: smart` (default) or `manual`; `unattended_mode: deny`; Docker backend with `--cap-drop ALL`, `no-new-privileges`, and a PID limit; `docker_forward_env: []`; egress-isolation guide `[Verified]` | `sandbox.mode` (`off` / `non-main` / `all`); DM pairing by default; `openclaw security audit --deep --fix` `[Verified]` |
| Must **not** do | Talk to end customers | Share a profile between two agent processes ("Never point two agent processes at the same profile") `[Verified]` | Serve several mutually untrusting clients from one gateway `[Verified]` |

### 1.2 End-to-end request flow (support and booking, recommended v1)

```
 Customer ── WhatsApp (Cloud API webhook) ──► Hermes gateway, profile "client-acme"
                                               │  (front-desk agent: drafts + classifies;
                                               │   NO shell, NO web, NO arbitrary HTTP)
             fast path ≈85–95% ◄───────────────┤ 1. intent + slot lookup via deterministic booking service
                                               │ 2. booking executed by code after customer confirms
                                               │
                                               │ 3. escalate: create Paperclip issue
                                               ▼    (signed routine webhook, hmac_sha256, 300 s replay window)
                               ┌───────────────────────────────────┐
                               │ Paperclip: company "Acme Clinic"  │ 4. budget reservation → invocation block check
                               │ goal · project · issue · approvals│
                               └──────────────┬────────────────────┘
                                              │ 5. wake assignee (hermes_local / hermes_gateway)
                                              ▼
                               ┌───────────────────────────────────┐
                               │ Hermes back-office profile        │ 6. Docker backend, subagents, skills;
                               │ "acme-ops" (separate from front)  │    proposes refund/discount/reply
                               └──────────────┬────────────────────┘
                                              │ 7. ask_first / request_board_approval for money-moving actions
                                              ▼
                               Owner taps approve (Paperclip UI or operator Telegram)
                                              │ 8. issue comment → front-desk agent wakes, replies in thread
                                              ▼
                               Payment link / booking confirmation → revenue logged on the issue
```

**Why two Hermes profiles per client** (front desk and back office): profiles are separate homes with their own memory, skills, `.env`, and gateway state `[Verified]`. The **front-desk** profile reads untrusted customer text, so it gets no dangerous tools. The **back-office** profile has tools but **never reads raw customer text directly**. It works from a Paperclip issue that the front desk wrote. This splits the "lethal trifecta" (private data, untrusted input, and an outbound channel in one agent) across a governed boundary.

**Paperclip routine webhook details** `[Verified: docs/api/routines.md]`:
- Trigger kinds are `schedule` (cron plus timezone), `webhook`, and API.
- Webhook signing modes are `bearer`, `hmac_sha256`, `github_hmac`, and `none`. The replay window is 30–86,400 seconds (default 300) for HMAC.
- Concurrency policies are `coalesce_if_active` (default), `skip_if_active`, and `always_enqueue`.
- The webhook secret is shown **once**.

### 1.3 Channel and capability ownership

| Capability | Owner | Why |
|---|---|---|
| WhatsApp, commercial | **Hermes `whatsapp-cloud`** | Official Meta Cloud API. Needs a Meta Business account, a public HTTPS webhook, and approved templates outside the 24-hour window `[Verified]` |
| WhatsApp, personal or demo | Either (Baileys) | "Unofficial and ban-prone" (Hermes docs). Never use for client numbers |
| Telegram, Slack, Discord, Email, Signal | Hermes (default) or OpenClaw | Pick one per client; never both on the same bot account |
| iMessage | **OpenClaw** | `imsg` on a signed-in macOS host. BlueBubbles support was removed `[Verified]` |
| SMS | Either. OpenClaw's Twilio plugin validates Twilio signatures by default `[Verified]` | |
| Scheduling | **Paperclip routines** | Each run is an auditable, budgeted issue. Hermes cron is only for in-profile maintenance |
| Task board | **Paperclip** | Hermes also ships a Kanban board (`~/.hermes/kanban.db`). Don't run two boards of record |
| Skills | Hermes authors them; the Hermes **curator** prunes them; Paperclip Skill Studio versions the ones you promote | ClawHub and other public registries are not installed in production (§4.1) |

### 1.4 Hosting blueprint

| Host | Runs | Start size `[Assumption]` | Exposure |
|---|---|---|---|
| **A: Control** | Paperclip and PostgreSQL | 4 vCPU / 8 GB | UI on the tailnet only (`--bind tailnet`). Public paths only for routine webhooks |
| **B: Front desk** | One Hermes **front-desk profile per client** (gateway), plus OpenClaw gateways where needed (one per client, each under its own Unix user) | 4 vCPU / 8 GB for about 15–25 SMB clients | WhatsApp Cloud API needs a **public HTTPS webhook** per profile (reverse proxy or tunnel). OpenClaw gateways bind to loopback on port 18789 by default; change the port per instance |
| **C: Back office** | Hermes back-office profiles on the **Docker** backend, or Modal, Daytona, or Vercel Sandbox | 4 vCPU / 16 GB, or serverless | No inbound traffic. Egress allowlist following Hermes's network-isolation guide |

**Bootstrap commands** (verified in the repos; pin versions in production):

```bash
# Paperclip (host A)
npx paperclipai@latest onboard --yes
pnpm dev --bind tailnet        # source install alternative; health: GET /api/health

# Hermes (hosts B/C): one profile per client role
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes profile create acme-front && acme-front setup     # WhatsApp Cloud API wizard + model
hermes profile create acme-ops   && acme-ops setup       # terminal backend: docker
hermes curator run --dry-run                             # preview skill pruning

# OpenClaw (only if needed), one gateway per client
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon
openclaw security audit --deep && openclaw security audit --fix
```

**Hermes config for the back-office profile** (keys from `security.md`):

```yaml
terminal:
  backend: docker
  docker_forward_env: []        # keep secrets out of the sandbox
  container_cpu: 1
  container_memory: 2048
  container_persistent: false   # ephemeral workspace per task
approvals:
  mode: smart                   # docker backend skips dangerous-cmd checks (container is the boundary)
  unattended_mode: deny         # API/webhook-invoked sessions never auto-approve
  cron_mode: deny
curator:
  enabled: true                 # stale after 14 days, archive after 30 (defaults)
```

**OpenClaw sandbox for any customer-facing gateway:**

```json5
{ agents: { defaults: { sandbox: { mode: "all", backend: "docker", scope: "agent", workspaceAccess: "none" } } } }
```

The gateway itself is not sandboxed, and elevated exec bypasses the sandbox `[Verified]`.

**When OpenClaw joins Paperclip**, the adapter docs recommend `scopes: ["operator.admin"]` `[Verified]`. For production, grant the narrowest operator scope that still lets the agent receive tasks and post replies. Don't run with admin scope. Keep `disableDeviceAuth: false`, which is the default.

---

## 2. Monetization blueprints

### 2.1 Vertical support and booking network (rank #1)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | One company per client. Goals: response time, booking rate, CSAT. Routines: nightly digest, weekly report, reactivation campaign (approval-gated). Approvals for refunds, discounts, and complaints. Budget policies at company and agent level |
| **Hermes front desk** (`<client>-front`) | WhatsApp Cloud API and Telegram. Classifies, answers from the FAQ, drafts replies. Calls a **deterministic booking service** with a strict schema. No shell, web, or browser |
| **Hermes back office** (`<client>-ops`) | Escalations, complaint analysis, offer drafting, turning resolved escalations into FAQ entries and skills (promoted only after approval) |
| **OpenClaw** (optional) | iMessage for premium clients, run on a dedicated Mac with a dedicated bot macOS user `[Verified: deployment patterns doc]` |
| **Third parties** | Cal.com or the vertical's practice-management system, Stripe, Meta WhatsApp Business account, Twilio if SMS is needed |

**Pricing:** base plus outcome.
- A base of $149–$399 per location per month `[Assumption]`.
- Plus $2–$5 per booked appointment, *or* $0.50–$1.00 per AI-resolved conversation above an included allowance.
- Anchor: Intercom Fin is about $0.99 per resolution, charged only when no escalation is needed, and billed on top of helpdesk seats `[Reported]`.

**Unit economics** for one location with 1,000 conversations a month, 15% escalated, and 60 bookings. Token figures come from §3.2.

| Line | Pattern A: small model drafts, code acts | Pattern B: mid-tier model holds the tools |
|---|---|---|
| Revenue: $249 base + 60 × $3 | **$429** | **$429** |
| Front-desk model cost (1,000 × example A) | Haiku 5.5: **$10** | Sonnet 5.5, cached: **$110** |
| Escalations (150 × example B, $0.30) | $45 | $45 |
| WhatsApp messaging: about 4,000 business messages. Service-window messages may have become billable on **1 October 2026**; sources conflict on any free allowance | $0–$25 `[Reported]` | $0–$25 |
| Infrastructure share | $15 | $15 |
| Your time (1 hour at $100) | $100 | $100 |
| **Contribution** | **≈ $234–$259 (55–60%)** | **≈ $134–$159 (31–37%)** |
| …at a 5% escalation rate | ≈ $264–$289 (62–67%) | ≈ $164–$189 (38–44%) |

**What follows:**
1. **Your own time is the largest cost line.** Make onboarding a template (Paperclip company package plus Hermes profile distribution) before you have five clients.
2. The **safe** architecture is also the **cheap** one. Keeping tools away from the model that reads untrusted text lets that model be small.
3. The escalation rate is the second lever. Review the "top unanswered" list weekly and turn each recurring item into an FAQ entry.

**Policy and legal:**
- Meta's WhatsApp terms allow business-specific AI (support, bookings, order tracking) and ban general-purpose assistants (since 15 January 2026; Meta decides what counts as the "primary" use) `[Reported]`. Keep the bot scoped to the client's business.
- Templates are required outside the 24-hour window. Get opt-in before marketing messages.
- **AI disclosure in the first message.** This is mandatory in the EU from 2 August 2026 (fines up to €15M or 3% of turnover `[Reported]`), and good practice everywhere.
- TCPA (US) and GDPR/ePrivacy (EU) consent rules apply to proactive messages.
- Sign a data processing agreement (DPA) with each client. Medical verticals add HIPAA or the national equivalent, so avoid storing clinical details in agent memory.

**Kill criteria:** escalation rate above 25% at day 60, or monthly churn above 5% across the first 10 clients.

### 2.2 Niche intelligence briefs (rank #2)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | A company per product. Routines: hourly ingestion, daily synthesis, weekly deep dive. A task-review gate before publishing. Project budgets (`lifetime` windows suit one-off backfills) |
| **Hermes** | Ingestion and extraction subagents on Haiku 5.5, through the **Batch API** where latency doesn't matter; synthesis on Opus 5.5; skills for each new source parser; delivery to email, Telegram, or Slack through its gateway |
| **OpenClaw** | Optional extra channels for B2B subscribers (Teams, Google Chat) |

**Pricing:** prosumer $15–$49 a month; B2B $199–$999 a month with "ask the archive" Q&A `[Assumption]`.
**Economics:** about $0.42–$0.49 a day per product (example E). With 100 subscribers at $29, MRR is $2,900 and model cost is about $15 a month, so the real costs are data licensing and your editorial time.
**Legal:**
- Respect each site's terms of service and robots.txt, and prefer licensed feeds. *hiQ v. LinkedIn* narrowed CFAA exposure for scraping public data, but contract claims still apply.
- Real-estate MLS data is usually licensed.
- Financial briefs should stay impersonal and published on a regular schedule (the *Lowe v. SEC* publisher's exclusion). No personalized advice.

**Kill criteria:** fewer than 30 paid subscribers at day 90, or free-to-paid conversion below 2%.

### 2.3 Content studio (rank #3, best as an add-on)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | A company per brand; the content calendar as routines; **every publish behind a task review**; budget per brand |
| **Hermes** | Trend research, briefs, drafts, image and video generation through MCP tools, repurposing. Winning formats become skills, and the curator retires formats that stop working |
| **OpenClaw** | Posting and engagement on chat-native communities |

**Economics:** about $0.36 per article in model cost, or about $0.26 with Batch (example D). **Editorial QA is the real cost.**
**Policy:** Google's spam policy (revised 28 August 2026) targets pages made mainly to manipulate rankings, "regardless of production method". Spam updates in June and August 2026 come with no Search Console notice and no reconsideration process `[Reported]`. EU Art. 50 and platform rules (YouTube, TikTok, Meta) require labels on synthetic media.
**Kill criteria:** client-attributed leads don't cover the retainer by month 3.

### 2.4 Ads agency (rank #4)

| Layer | Responsibilities |
|---|---|
| **Paperclip** | Every tool connection that changes spend, bids, budgets, or campaign status is set to **`ask_first`**. In app definitions, set `askFirstRiskLevels` to `["write", "destructive"]` `[Verified: tool-access types]`. Daily optimization and weekly report routines |
| **Hermes** | Search-term mining, negative keywords, creative variants, anomaly detection. Proposes changes as a diff for approval |
| **OpenClaw or the Hermes gateway** | One-tap approval requests and the weekly summary to the client in Slack or WhatsApp |

**API access:** Google Ads tokens start at Test or **Explorer** access (2,880 production operations a day). **Basic** allows 15,000 a day after review, which takes about 2–5 business days. **Standard** is unlimited and is meant for large or multi-user tools `[Reported: Google developer docs]`. Meta and TikTok marketing APIs require app review.
**Pricing:** a management fee or a percentage of spend (10–20% is typical) `[Assumption]`. Use performance fees only once server-side conversion tracking is reliable.
**Path:** human approval of every change for 90 days. After that, auto-approve only inside an envelope (bids ±15%, budgets ±10%, negatives only).
**Kill criteria:** no CPA or ROAS gain against baseline at day 60 on two or more accounts.

### 2.5 Comparison matrix

Scores run from 1 (worst) to 5 (best) `[Assumption]`.

| Model | Time to first $ | Capital at risk | Regulatory exposure | Automation ceiling | Defensibility | **Solo fit** |
|---|---|---|---|---|---|---|
| Support and booking | 5 | 5 | 3 (WhatsApp policy, AI Act, health data) | 4 | 3 | **5** |
| Intelligence briefs | 3 | 4 | 3 (5 if not financial) | 5 | 4 | **4** |
| Content studio | 4 | 5 | 2 (spam policy) | 4 | 2 | **3** |
| Ads agency | 3 | 2 | 3 | 2 | 3 | **2** |

---

## 3. Financial mechanics, budgeting, and token ROI

### 3.1 How Paperclip budgets actually behave (from the source code)

| Mechanism | Behavior `[Verified]` |
|---|---|
| Policy fields | `scopeType` (company / agent / project), `amount` (cents), `windowKind` (`calendar_month_utc` / `lifetime`), `warnPercent` (default 80, range 1–99), `hardStopEnabled` (default true), `notifyEnabled`, `reservationCents`, `unpricedUsagePolicy` (default `block`) |
| Cost reporting | Adapters report a cost event (provider, model, input and output tokens, cost in cents) after each heartbeat (`POST /api/companies/{id}/cost-events`) |
| **Pre-dispatch reservation** | Before provider work starts, the run reserves `reservationCents` under a company lock. If observed spend plus held reservations would exceed the limit, the run is refused ("Available budget is reserved by unfinished runs") |
| **Fail-closed hard stop** | A hard-stopped scope stays blocked when spend ≥ amount, **or** when there are unaccounted runs, **or** when there is **unpriced usage** (unless `unpricedUsagePolicy: allow`) |
| Resolving | `keep_paused` or `raise_budget_and_resume`. The new amount must exceed observed spend. Also resets automatically on the first of the month (UTC) |
| Known limit | *"Estimates constrain admission; they cannot cap a provider's bill."* A single long run can overshoot |

**Configuration for one client** (support example, $429 MRR):

1. **Company budget** = MRR × target model-cost ratio. Use 20% here, which is about $85. Set `warnPercent: 70`.
2. **Agent budgets:** back office $50, front desk $20, routines $10. The rest is headroom.
3. **Set `reservationCents` per agent** to the 95th-percentile cost of one run (for example $0.60 for back-office runs) so reservations stop runaway concurrency.
4. **Price local models before enabling them.** With the default `unpricedUsagePolicy: block`, an unpriced local-model call holds a hard-stopped scope paused. Either set a notional price or switch to `allow` deliberately.
5. **Provider-side backstop.** Give each client its own Anthropic workspace with a monthly spend limit (calls error until the next period once it's hit `[Reported]`), or an OpenRouter key with a credit limit. Set it at about 110% of the Paperclip company budget.
6. **Degrade, don't go dark.** The front-desk profile uses its own capped key, so a back-office hard stop never silences customer replies. Escalations queue up for a human instead.

### 3.2 Token ROI

```
LLM_cost = Σ [ uncached_in × p_in + cache_read_in × p_cache + out × p_out ]   (thinking bills as output)
Contribution = Revenue − LLM_cost − tool/API − infra − human_minutes × rate
Token ROI = Revenue / LLM_cost
```

| Model | Input $/M | Output $/M | Cache read $/M |
|---|---|---|---|
| Claude Haiku 5.5 (`claude-haiku-5-5`) | 0.10 | 0.50 | — (prompts ≤ 100K) |
| Claude Sonnet 5.5 (`claude-sonnet-5-5`) | 2.00 | 10.00 | 0.20 |
| Claude Opus 5.5 (`claude-opus-5-5`) | 4.00 | 20.00 | 0.20 |
| Claude Fable 5.1 (`claude-fable-5-1`) | 10.00 | 50.00 | 0.25 |

The Batch API is 50% off.

Token counts below are `[Assumption]`; the arithmetic is exact.

| # | Task | Tokens | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 |
|---|---|---|---|---|---|
| A | 8-turn front-desk conversation | 70K in / 6K out | **$0.010** | $0.20, or ≈$0.11 with 75% cache reads | $0.40 / ≈$0.20 |
| B | Support escalation (back office) | 100K / 10K | $0.015 | **$0.30** | $0.60 |
| C | Heavy research job | 300K / 30K | $0.045 | $0.90 | $1.80 |
| D | SEO article: research (Haiku) + draft (Sonnet) + QA (Opus) | 150K/8K + 30K/12K + 20K/4K | **≈$0.36** (≈$0.26 with Batch for research and draft) | | |
| E | Daily brief: extract (Haiku) + synthesize (Opus) | 1M/50K + 60K/6K | **≈$0.49** (≈$0.42 with Batch for extraction) | | |

At $0.99 per resolution, A on Haiku returns about **99×** its model cost and B on Sonnet about **3.3×**.

**Hidden cost to fix first: Hermes gateway sessions never reset on their own.** A customer chat "can run for weeks as a single session … it grows expensive (compaction runs repeatedly over an ever-longer history)" `[Verified: memory.md, sessions.md]`. Time-based resets require the `hermes-session-reset-policy` plugin `[Verified]`. For support bots, install it and reset on idle (for example 24 hours) so example A stays near 70K tokens instead of growing without limit.

### 3.3 Model routing and the security constraint

| Tier | Handles | Engine | Tools allowed |
|---|---|---|---|
| 0 | Opening hours, availability, booking create/modify after explicit confirmation | **Deterministic service** | n/a (code) |
| 1 | Intent, FAQ answers, drafting replies to untrusted customers | Haiku 5.5 or a local model | **Read-only, strict-schema lookups only.** No shell, web, browser, or cross-chat messaging |
| 2 | Escalations, multi-step work | Sonnet 5.5 (low to medium effort) | Tools inside the Docker sandbox with an egress allowlist |
| 3 | Approving recommendations, final QA, ad change proposals | Opus 5.5 (high effort) | Proposes only. Execution goes through `ask_first` |

This follows OpenClaw's own guidance: *"Do not use older/weaker/smaller tiers for tool-enabled agents or untrusted inboxes … If you must use a smaller model, limit potential damage: read-only tools, strong sandboxing"* `[Verified: docs/gateway/security/prompt-injection.md]`.

**Before building a cascade**, measure one strong model at lower effort on the same tasks. Prompt caches are model-scoped, and the metric that matters is cost per *completed* task.

**Other cost levers, in order:**
1. Byte-stable system prompts. Hermes already freezes the memory snapshot for cache stability `[Verified]`.
2. The Batch API.
3. Progressive skill loading.
4. Low effort on routine routes.
5. Hermes's `/reload-mcp` invalidates the prompt cache, so avoid reloading MCP tools mid-session `[Verified: security.md]`.

**Self-hosting break-even:** `GPU $/month ÷ blended API $/token`. For example, $1,000 a month divided by about $0.18 per million tokens (Haiku 5.5 at a 4:1 input-to-output ratio) is about **5.5 billion tokens a month**. Self-host for data residency, not to save money at small scale `[Assumption: GPU price]`.

---

## 4. Guardrails, security, and scaling risks

### 4.1 Threat model

| # | Threat | Evidence | Mitigation |
|---|---|---|---|
| 1 | **OpenClaw gateway token theft leading to RCE** | **CVE-2026-25253** (GHSA-g8p2-7wf7-98mq), CVSS 8.8: the Control UI took a gateway URL from a query parameter and sent the stored token; missing WebSocket origin checks allowed cross-site hijacking. **Localhost-only instances were affected.** Fixed in **2026.1.29** `[Reported: OSV, The Hacker News]` | Stay on current releases, run `openclaw security audit --deep`, keep device auth on, and don't browse untrusted sites from the operator browser profile |
| 2 | Exposed gateways | Shodan-based counts from tens of thousands to six figures `[Reported]` | Loopback or tailnet only. Gateway HTTP bearer auth is "effectively all-or-nothing operator access" `[Verified: network-exposure.md]` |
| 3 | **Malicious skills** | ClawHub "ClawHavoc": roughly 340 to 1,400+ malicious skills; about 37% of scanned skills had flaws `[Reported]` | No public-registry installs. The Hermes curator never touches hub skills, so you have to review those yourself. Promote skills only through Skill Studio |
| 4 | Sandbox bypass | OpenClaw: tools run on the host unless sandboxing is on, and elevated exec bypasses it. Hermes: the `local` backend has no isolation `[Verified]` | OpenClaw `mode: "all"`, elevated exec off. Hermes on the Docker, Modal, Daytona, or Vercel Sandbox backend |
| 5 | **Prompt injection** | Applies to any untrusted content, not just public DMs `[Verified]` | The tier rules in §3.3, the front-desk/back-office split, `unattended_mode: deny`, and `ask_first` on money-moving actions |
| 6 | Secret exfiltration | Variables in `docker_forward_env` "can be read and exfiltrated" `[Verified]` | Keep the forward list empty. Use Hermes's credential vault (passwords injected into the page, never seen by the model) or secret providers (1Password, Bitwarden) |
| 7 | Runaway spend | Reservations can't cap a provider's bill `[Verified]` | `reservationCents`, per-agent budgets, provider workspace caps, routine `coalesce_if_active` |
| 8 | Cross-client leakage | OpenClaw: one trust boundary per gateway. Hermes: never share a profile `[Verified]` | Per-client gateway, profile, credentials, and Unix user. Paperclip company scoping for data |

### 4.2 Human-in-the-loop gates

| Action | Paperclip control `[Verified]` |
|---|---|
| New agent or role | `hire_agent` approval (`POST /api/companies/{id}/agent-hires` creates a draft plus the approval) |
| Strategy change | `approve_ceo_strategy` |
| Budget exceeded | `budget_override_required` incident, resolved with `keep_paused` or `raise_budget_and_resume` |
| Refunds, discounts, ad spend, publishing, bulk sends | Tool connection set to `ask_first` (risk levels `write` / `destructive`) **or** `request_board_approval` |
| Routine edits | Append-only revisions; `baseRevisionId` concurrency check; restore |

Approvals follow the lifecycle `pending → approved | rejected | revision_requested → resubmitted`, with comments and linked issues, all available through the API. You can therefore build a small Telegram approve/reject bot for clients on top of `POST /api/approvals/{id}/approve`.

### 4.3 Memory and state hygiene

| Store | Grows? | Policy |
|---|---|---|
| Hermes `MEMORY.md` / `USER.md` | No (2,200 / 1,375 characters; errors when full) | The agent consolidates in batched operations. Keep procedures in skills |
| **Hermes gateway sessions** | **Yes, indefinitely** unless reset | `hermes-session-reset-policy` plugin with an idle reset. Pruning deletes only *ended* sessions |
| Hermes skills | Yes | Curator: stale at 14 days, archived at 30, never deleted. Pin critical skills. Turn on `consolidate: true` only if you accept its cost (about 50–100 API calls per sweep) |
| OpenClaw transcripts | Yes. Stored per agent in SQLite, readable by anyone with filesystem access `[Verified]` | Disk encryption, retention windows, a separate OS user per client |
| Paperclip issues and activity | Yes (this is your audit trail) | Keep it. Archive after 12 months |

### 4.4 Throttling and resilience

- **Provider 429s:** backoff with jitter, plus per-tenant concurrency limits. Paperclip routines merge overlapping runs by default `[Verified]`.
- **Model fallback:** Hermes supports fallback providers `[Verified: fallback-providers.md]`.
- **Idempotency:** Paperclip's atomic issue checkout and duplicate-dispatch fencing in reservations `[Verified]`. Booking and payment calls need idempotency keys.
- **Degraded mode:** if the LLM is down, the front desk sends a deterministic holding reply and opens a Paperclip issue.

### 4.5 Multi-company isolation on one server

| Boundary | Paperclip | You add |
|---|---|---|
| Data and permissions | Company-scoped access checks `[Verified]` | Separate databases if a contract requires it |
| Runtime | — | One Hermes profile per client role, one OpenClaw gateway per client, a separate Unix user for each |
| Credentials | Secret injection per run | Per-client provider workspace, Meta app/WABA, and channel tokens |
| Network | — | Hermes egress isolation; no east-west traffic between containers |
| Incident | Pause or terminate the company's agents | Runbook: rotate keys, wipe the profile or gateway, redeploy from a pinned version |

---

## 5. Execution plan

### Days 1–30: build and land a design partner
1. Pick one vertical and one city. Line up three prospects before writing code.
2. Build: Paperclip (host A) plus Hermes front-desk and back-office profiles. Use **Telegram first** for the fastest iteration, and start Meta Business verification for the WhatsApp Cloud API in parallel.
3. Write the deterministic booking service (Cal.com or the vertical's system) with strict JSON schemas.
4. Set budgets (`warnPercent: 70`, per-agent `reservationCents`) and a provider workspace cap. Install the session-reset plugin.
5. Put the AI disclosure in the greeting.

**Gate:** escalation rate below 30%, and the owner says they would pay.

### Days 31–60: production
1. Switch to the WhatsApp Cloud API (templates approved, public HTTPS webhook).
2. Back office on the Docker backend with an empty `docker_forward_env`, egress isolation, `unattended_mode: deny`, and `ask_first` on money actions.
3. Stripe billing; the weekly outcome report routine.
4. Red-team it yourself: prompt injection through customer messages, cross-chat messaging, pairing bypass, secret leakage.
5. Sign 3–5 paying clients.

**Gate:** contribution above 50% per client and no open security findings.

### Days 61–90: scale
1. Package a Paperclip company template and Hermes profile distributions so onboarding takes under 2 hours.
2. Get escalations below 10%.
3. Launch the niche brief for the same vertical.
4. Add OpenClaw only if a client pays for iMessage or Teams.

**Gate:** MRR at least 5× total infrastructure, model, and tool costs, and monthly churn under 5%.

### Launch checklist
- [ ] Paperclip on the tailnet. Routine webhooks use `hmac_sha256`
- [ ] Hermes: Docker/serverless backend, `docker_forward_env: []`, `unattended_mode: deny`, session-reset plugin, curator on
- [ ] OpenClaw (if used): current release, `sandbox.mode: "all"`, elevated exec off, `openclaw security audit --deep` clean, minimal operator scopes, one gateway per client
- [ ] No public-registry skills
- [ ] Per-client budgets with reservations, plus a provider workspace cap
- [ ] `ask_first` on refunds, discounts, spend, publishing, and bulk sends
- [ ] WhatsApp Cloud API (not Baileys) for client numbers; templates and opt-in recorded
- [ ] AI disclosure (EU Art. 50); DPA; retention and deletion routines
- [ ] Encrypted backups, with a restore tested; alerts on budget incidents, gateway down, and escalation backlog

---

## 6. Sources

**Primary: in-repo code and docs** (snapshots in the header):
1. Paperclip budgets: `packages/db/src/schema/budget_policies.ts`, `packages/shared/src/validators/budget.ts`, `packages/shared/src/constants.ts`, `server/src/services/budgets.ts`, `server/src/services/budget-reservations.ts`, `docs/guides/board-operator/costs-and-budgets.md`, `docs/api/costs.md`. <https://github.com/paperclipai/paperclip>
2. Paperclip approvals and routines: `docs/api/approvals.md`, `docs/api/routines.md`; tool access `packages/shared/src/types/tool-access.ts`.
3. Paperclip adapters: `packages/adapters/openclaw-gateway/doc/ONBOARDING_AND_TEST_PLAN.md`, `packages/adapters/hermes/README.md`, `packages/adapters/hermes-gateway/README.md`, `docs/guides/openclaw-docker-setup.md`.
4. Hermes: `website/docs/user-guide/features/memory.md`, `features/curator.md`, `profiles.md`, `sessions.md`, `security.md`, `egress/network-isolation.md`, `features/credential-vault.md`, `messaging/whatsapp-cloud.md`, `features/kanban.md`. <https://github.com/NousResearch/hermes-agent>
5. OpenClaw: `docs/channels/whatsapp.md`, `docs/channels/imessage.md`, `docs/channels/sms.md`, `docs/gateway/security/index.md`, `security/prompt-injection.md`, `security/access-control.md`, `security/network-exposure.md`, `security/running-the-audit.md`, `docs/gateway/config-gateway.md`. <https://github.com/openclaw/openclaw>
6. Anthropic API pricing (first-party, cached 2026-10-06).

**Secondary (`[Reported]`):**

7. CVE-2026-25253: <https://osv.dev/vulnerability/CVE-2026-25253>, <https://thehackernews.com/2026/02/openclaw-bug-enables-one-click-remote.html>, <https://vulnerability.circl.lu/vuln/CVE-2026-25253>
8. ClawHub malicious skills and exposure: <https://www.bitdoze.com/openclaw-security-guide/>, <https://blog.barrack.ai/openclaw-security-vulnerabilities-2026/>, <https://clawdocs.org/security/known-vulnerabilities>
9. WhatsApp AI-provider policy: <https://respond.io/blog/whatsapp-general-purpose-chatbots-ban>, <https://respond.io/blog/whatsapp-chatbot-policy-2026>, <https://mobileecosystemforum.com/2025/12/01/metas-whatsapp-ai-chatbot-ban/>
10. WhatsApp pricing change on 1 October 2026 (sources conflict): <https://www.engagelab.com/blog/whatsapp-pricing-2026-service-message-cost>, <https://yournotify.com/blog/whatsapp-pricing-update-2026/>, <https://help.zoho.com/portal/en/community/topic/whatsapp-message-pricing-changes-effective-october-1-2026-21-9-2026>
11. EU AI Act Art. 50: <https://labs.cloudsecurityalliance.org/research/csa-research-note-eu-ai-act-article-50-transparency-20260729/>, <https://www.joneswalker.com/en/insights/blogs/ai-law-blog/yes-august-2-still-matters-the-eu-approved-a-high-risk-ai-delay-but-most-trans.html>
12. Google Ads API access levels: <https://developers.google.com/google-ads/api/docs/access-levels>
13. Google spam policies and 2026 updates: <https://ppc.land/scaled-content-abuse/>, <https://seosherpa.com/googles-june-2026-spam-update-has-finished-rolling-out/>, <https://gsqi.com/marketing-blog/august-2026-google-spam-update-case-studies/>
14. Intercom Fin pricing: <https://www.gleap.io/blog/intercom-fin-ai-pricing-2026>, <https://www.ringly.io/blog/intercom-pricing>
15. Anthropic workspace spend limits: <https://claude.com/resources/articles/workspaces>, <https://www.xray.tech/post/how-to-set-spend-limits-in-claude-console>
16. "Zero-human company" evidence: <https://videohighlight.com/v/Yp5KiHBSpZo>, <https://trends.vc/zero-human-companies-300000-mo-maximizer-mode-liability-gap/>, <https://fast.io/resources/paperclip-ai-review-2026.md>

**Still unverified** (official pages unreachable from the research sandbox): Meta's own rate card after 1 October 2026; Meta's official WhatsApp Business Solution terms text; the Official Journal citation for the AI Act Omnibus.

*Not legal or investment advice.*

---

## Appendix A: what changed from pass 1

| Topic | Pass 1 | Pass 2 (verified) | Impact |
|---|---|---|---|
| Budget defaults | "Configurable, not a fixed 80/100" | Defaults **are** 80% warning and 100% stop; both configurable; reservations; fails closed on unpriced usage | Budget section rewritten |
| WhatsApp ownership | OpenClaw (with "verify transport") | OpenClaw = Baileys only; Hermes = official Cloud API | Channel ownership and recommended stack changed |
| Fast-path model | Haiku with tools | Small models only for read-only work; code performs actions | Routing ladder plus Pattern A/B economics |
| Paperclip↔OpenClaw | "HTTP/webhook adapter" | `openclaw_gateway` over WebSocket; join through board approval; legacy webhook adapter removed | Flow redrawn |
| OpenClaw→Hermes | Implied direct handoff | Only through Paperclip issues | Flow redrawn |
| Multi-tenancy | Container per client | Officially "one trust boundary per gateway"; Hermes profiles must not be shared | Isolation table |
| Session cost | Not covered | Hermes gateway sessions never reset without a plugin | New cost lever |
| CVE detail | "Exposed port 18789" | Cross-site WebSocket hijacking; localhost also vulnerable; fixed in 2026.1.29 | Threat model |
| Support economics | 32–39% contribution | 55–60% with Pattern A, 31–37% with Pattern B (escalation recosted at $0.30) | Unit economics |
| New regulation | — | EU AI Act Art. 50 (from 2 Aug 2026); WhatsApp pricing change (1 Oct 2026) | Policy sections |
