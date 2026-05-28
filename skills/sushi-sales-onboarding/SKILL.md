---
name: sushi-sales-onboarding
description: >
  Customer onboarding and daily lead generation workflow. Triggers: "onboard me",
  "set up lead gen", "daily leads", "configure my ICP", "set up signals",
  "run my daily leads", "generate leads", "morning leads", "lead gen setup",
  "sales onboarding", "configure prospecting", or any request to set up automated
  account discovery and daily lead generation. Phase 1 (onboarding) runs once;
  Phase 2 (daily generation) runs on each subsequent invocation once onboarding is complete.
---

# Sushidata Sales Onboarding & Daily Lead Generation

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

This skill has two phases:

1. **Onboarding** — one-time setup that captures business context, ICP, signals, win/loss patterns, disqualification criteria, and scoring model.
2. **Daily Lead Generation** — runs Monday–Friday, discovers 40 net-new qualified accounts with buyer persona contacts.

---

## Phase Detection

On every invocation:

1. `POST {BASE_URL}query/` with `{ "query": "context.company_description context.icp context.buyer_personas context.signals context.disqualification_criteria" }`
2. If the context lake returns valid values for all five keys → skip to **Phase 2**.
3. If any are missing → run **Phase 1** from the first missing step.

---

# PHASE 1: ONBOARDING

---

## Step 1 — Website Scrape & Business Context Extraction

Ask:

> "What is your company's website URL? I'll scrape it to auto-generate your business context so you don't have to fill everything out manually."

Once provided, use WebSearch and WebFetch to scrape:

- Homepage
- `/about` or `/about-us`
- `/product`, `/solutions`, or `/platform`
- `/customers` or `/case-studies` (if present)
- `/pricing` (if present)

If WebFetch returns a JavaScript shell with no content, escalate to Browser Rendering.

From the scraped content, extract and present three fields:

### `COMPANY_DESCRIPTION`
2–4 sentences: who they are, what they sell, how they help. Value proposition, not fluff.

### `IDEAL_CUSTOMER_PROFILE (ICP)`
Structured description including:
- Company size (headcount, revenue tiers)
- Industry verticals
- Business model (B2B, B2C, PLG, enterprise)
- Geographic focus
- Technology environment (from case studies, integrations)

### `BUYER_PERSONAS`
List of likely buyers with:
- Job title(s) and seniority
- Department / function
- Relevant responsibilities
- Why they'd care about this solution

Present all three and ask:

> "Here's what I extracted. Please:
> - ✅ Confirm if accurate
> - ✏️ Edit anything that's off
> - ➕ Add anything missing
>
> Once confirmed, this powers all future lead generation."

After confirmation, save to the context lake:

```json
POST {BASE_URL}context/
{
  "serverId": "26",
  "content": "ONBOARDING — company_description: [value] | icp: [value] | buyer_personas: [value]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<ISO 8601>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Step 2 — Signal Configuration

Present:

> "Signals are real-world events that indicate a company is worth targeting right now. Review each one — tell me if it's relevant and give clarifying detail."

### Signal: `fundraising`
Fires on funding rounds (TechCrunch, press, Crunchbase).

Ask:
- Which round stages? (Seed / A / B / C+)
- Minimum funding amount?
- Do specific investors matter?

### Signal: `hiring_signal`
Fires on job postings matching keywords (LinkedIn Jobs, Bumble intent).

Ask:
- What job titles signal investment in your problem space?
- What keywords in a posting indicate fit? (e.g., "revenue operations", "voice AI", "data pipeline")

### Signal: `brand_move`
Fires on trade press appearances (rebrand, agency hire, campaign launch, product drop, exec announcement). Sources: Marketing Dive, LBB, Ad Age, vertical trade pubs.

Ask:
- Which brand activities signal a company is in motion?
- Specific trade publications to monitor?

### Signal: `li_reactor`
Fires on LinkedIn post activity matching configured types:

| post_type | Description |
|---|---|
| `Competitor Post` | Engaging with or posting about a competitor |
| `AI Art Gen Post` | Exploring AI-generated content |
| `Sentiment Post` | Expressing frustration relevant to your category |
| `Custom Post` | Configure based on your use case |

Ask:
- Which types are useful?
- For Competitor Post: which competitors?
- For Sentiment Post: what pain keywords?
- Any custom LinkedIn activity to watch?

### Custom Signals

Ask:

> "Any signals not listed above? (G2/Capterra reviews, GitHub issues, Reddit threads, tech stack detection, conference attendance, executive changes)"

Save all confirmed signals:

```json
POST {BASE_URL}context/
{
  "serverId": "26",
  "content": "ONBOARDING — signals: [JSON array of signal configs with signal_trigger, is_active, configuration, relevance_notes]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<ISO 8601>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Step 3 — Win/Loss Intelligence

First, check if the `niche-signal-discovery` skill can be leveraged. If the customer has Closed Won / Closed Lost domain lists, invoke `niche-signal-discovery` to surface differential signals automatically.

If no automated data is available, ask:

> "To score accounts more accurately, I need to understand your best and worst deals."

**WIN PATTERNS:**
1. What did closed-won companies have in common?
2. What does a champion at a won account look like?
3. Was there a trigger event that accelerated deals?

**LOSS PATTERNS:**
4. Most common reasons deals die?
5. Company types that waste time but rarely convert?
6. Personas that lack buying authority?

Save to context lake as `context.win_patterns` and `context.loss_patterns`.

---

## Step 4 — Disqualification Criteria

Ask:

> "Define hard disqualifiers — clear reasons a company is automatically excluded before outreach."

| Category | Question |
|---|---|
| Industry Exclusions | Industries that are definitively not a fit? |
| Size Exclusions | Headcount/revenue thresholds (too small or too large)? |
| Business Model Exclusions | Consumer-only, D2C, non-commercial excluded? |
| Persona Exclusions | Right-size company but buyer persona doesn't exist? |
| Pain Signal Absence | Exclude companies showing no evidence of core pain? |
| Geographic Exclusions | Countries/regions you don't sell into? |
| Competitive Exclusions | Companies on an incompatible/competitive solution? |
| Current Customer Exclusions | Auto-excluded via dedup against context lake + CRM |

Save as `context.disqualification_criteria[]`:

```json
{
  "category": "",
  "rule": "",
  "rationale": "",
  "hard_block": true
}
```

---

## Step 5 — Account Scoring & Prioritization Model

Using confirmed ICP, signals, win patterns, and disqualifiers, define the scoring tiers:

| Tier | Label | Criteria |
|---|---|---|
| **1** | Strong Fit — Priority Outreach | ICP match on industry + size + model AND ≥1 active signal AND passes all disqualifiers AND persona confirmed |
| **2** | Moderate Fit — Secondary Outreach | ICP match on 2/3 dimensions AND passes all disqualifiers AND ≥1 persona present |
| **3** | Weak Fit — Hold/Nurture | Partial ICP match OR no signal detected; passes disqualifiers |

Disqualified accounts are excluded from all tiers and logged with the triggering rule.

Save scoring model to context lake as `context.scoring_model`.

Confirm with the customer:

> "Here's the scoring framework I'll use to rank every account. Does this match your prioritization?"

---

## Step 6 — Persona Discovery Template

Confirm the persona discovery order with the customer:

1. **Economic buyer** — final decision-maker
2. **Champion** — day-to-day user / internal advocate
3. **Technical evaluator** — gatekeeper

For each persona match, the daily run will extract:
- Full name, current title, LinkedIn URL
- Business email (via Hunter — see `provider-playbooks/hunter.md`)
- Time in role (tenure signal)
- Recent activity signals

Save persona discovery template to context lake.

Tell the customer:

> "Onboarding complete. Your context lake now has everything needed to run daily lead generation. On each run I'll find 40 net-new qualified accounts with buyer contacts, scored and ready for outreach."

---

# PHASE 2: DAILY LEAD GENERATION

> Runs on each invocation after onboarding is complete.
> Target: 40 net-new qualified accounts with buyer persona contacts.

---

## Step 1 — Context Restore & Deduplication

Load from context lake:
- `context.icp`, `context.buyer_personas`, `context.signals`, `context.disqualification_criteria`, `context.scoring_model`
- `context.delivered_accounts[]` — all previously delivered domains
- `context.disqualified_accounts[]` — all previously excluded domains
- `context.heyreach_events[]` — contacts touched via HeyReach in last 90 days

Build a combined exclusion list of:
- All previously delivered company domains
- All disqualified company domains
- All LinkedIn URLs with HeyReach activity in the configured lookback window (default 90 days)

Use `scripts/dedupe_utils.py` for apex-domain matching against the exclusion list. Log exclusion list size and last run date.

---

## Step 2 — Swarm Search for Net-New Accounts

Deploy a Sushidata research swarm:

```json
POST {BASE_URL}swarm/deploy/
{
  "query": "Find 60 companies matching this ICP: [load from context.icp]. Active signals to look for: [load from context.signals where is_active=true]. Exclude these domains: [exclusion list]. For each company return: name, domain, LinkedIn URL, industry, headcount, location, and which signals were detected with evidence URLs.",
  "swarmSize": 10
}
```

Over-provision at 60 to account for disqualification falloff (~33% buffer). Follow the discovery order from `finding-companies-and-contacts.md`: companies first, then people.

Poll `{BASE_URL}swarm/status/` every 30s. Show progress. Get results via `{BASE_URL}swarm/summary/` when complete or at 5-minute timeout.

---

## Step 3 — Disqualification Filtering

Run every discovered account against `context.disqualification_criteria[]`. Exclude any account triggering a `hard_block: true` rule. Log excluded accounts:

```json
POST {BASE_URL}context/
{
  "serverId": "26",
  "content": "DISQUALIFIED: [domain] — rule: [rule] — date: [today]",
  "messageId": "msg-<timestamp>",
  ...
}
```

---

## Step 4 — Scoring & Segmentation

Score each passing account using the Tier 1/2/3 model from `context.scoring_model`. Rank Tier 1 first, then Tier 2, until 40 accounts are confirmed.

If fewer than 40 pass:
- Loosen one non-critical ICP dimension and re-search
- Log the shortfall and reason
- Do NOT pad with Tier 3 without customer authorization

---

## Step 5 — Persona Discovery

For each of the 40 confirmed accounts, find buyer personas using the provider escalation order from `finding-companies-and-contacts.md`:

1. `hunter_domain_search domain={domain}` — structured contact discovery
2. Filter by titles/departments from `context.buyer_personas`
3. `hunter_email_verify email={email}` — mandatory before any outbound
4. If Hunter returns <2 contacts, supplement with WebSearch + Sushidata swarm for role discovery

Prioritize: economic buyer → champion → technical evaluator.

Run `scripts/validate-emails.py` on the final enriched set to flag domain mismatches.

---

## Step 6 — Context Lake Write-Back

Save the full run results:

```json
POST {BASE_URL}context/
{
  "serverId": "26",
  "content": "DAILY RUN [date] — accounts_discovered: X, disqualified: Y, delivered: 40, signals_triggered: [...], dedup_exclusions: Z. Delivered accounts: [JSON array with domain, name, tier, segment, signals, personas]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<ISO 8601>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Step 7 — Output Delivery

Present the 40 accounts in a structured table. For each:

| Field | Value |
|---|---|
| Company | Name + domain + LinkedIn URL |
| Industry / Size / Location | From swarm results |
| Tier | 1 or 2 |
| ICP Segment | Sub-segment classification |
| Signals Detected | `[ { signal_trigger, date, summary, source_url } ]` |
| Disqualification Check | PASS |
| Buyer Personas | `[ { name, title, email, linkedin, tenure, recent_activity } ]` |
| Recommended First Contact | The highest-priority persona |

After delivery, offer:

> **Next steps:**
> - **Activate in HeyReach** — push these contacts into a LinkedIn campaign (see `provider-playbooks/heyreach.md`)
> - **Write personalized outreach** — I'll draft per-account sequences using signals as hooks
> - **Sync to HubSpot** — create contacts and deals in your CRM (see `provider-playbooks/hubspot.md`)
> - **Deep-dive any account** — run a full research swarm on a specific company
> - **Save** — write this session to the context lake for future reference

---

## Context Lake Schema Reference

```json
{
  "context": {
    "company_description": "",
    "icp": {
      "industry": [],
      "size": {},
      "model": "",
      "geo": [],
      "tech": []
    },
    "buyer_personas": [
      { "title": "", "department": "", "seniority": "", "why_they_care": "" }
    ],
    "signals": [
      {
        "signal_trigger": "fundraising | hiring_signal | brand_move | li_reactor",
        "is_active": true,
        "configuration": {
          "relevant_rounds": [],
          "min_amount_usd": null,
          "relevant_investors": [],
          "matched_job_titles": [],
          "matched_keywords": [],
          "relevant_event_types": [],
          "monitored_publications": [],
          "active_post_types": [],
          "competitor_names": [],
          "pain_keywords": []
        },
        "relevance_notes": ""
      }
    ],
    "win_patterns": {
      "company_traits": [],
      "champion_traits": [],
      "trigger_events": []
    },
    "loss_patterns": {
      "common_reasons": [],
      "time_wasting_profiles": [],
      "wrong_personas": []
    },
    "disqualification_criteria": [
      { "category": "", "rule": "", "rationale": "", "hard_block": true }
    ],
    "scoring_model": {
      "tier_1": "ICP match (industry + size + model) AND ≥1 signal AND all disqualifiers pass AND persona confirmed",
      "tier_2": "ICP match (2/3) AND all disqualifiers pass AND ≥1 persona",
      "tier_3": "Partial ICP OR no signal; passes disqualifiers"
    },
    "delivered_accounts": [
      {
        "company_domain": "",
        "company_name": "",
        "company_linkedin": "",
        "tier": "",
        "segment": "",
        "signals_detected": [],
        "personas": [],
        "delivered_date": ""
      }
    ],
    "disqualified_accounts": [
      { "domain": "", "rule_triggered": "", "date": "" }
    ],
    "heyreach_events": [
      {
        "event_trigger": "message_sent | message_response | connection_request_sent | connection_request_accepted",
        "event_date": "",
        "campaign": { "id": "", "name": "" },
        "contact": { "name": "", "linkedin": "", "title": "", "company": "" },
        "source": { "platform": "HeyReach" }
      }
    ],
    "run_log": [
      {
        "date": "",
        "accounts_discovered": 0,
        "accounts_disqualified": 0,
        "accounts_delivered": 0,
        "signals_triggered": [],
        "shortfall_flag": false,
        "dedup_exclusions_count": 0
      }
    ]
  }
}
```

---

## Rules

- **Phase detection is automatic.** Never ask the user which phase to run — check the context lake.
- **Companies first, then people.** Follow `finding-companies-and-contacts.md` discovery order.
- **Over-provision, then filter.** Search for ~1.5× target count to absorb disqualification falloff.
- **Never skip deduplication.** Zero overlap with previous runs is mandatory.
- **Approval gate for paid actions.** Before scaling Hunter/HeyReach calls beyond the first batch, show cost estimate and get explicit approval (per `sushi-research` policy).
- **Save every run.** Both the request and results go to the context lake. Future sessions must be able to retrieve what was delivered without re-running.
- **Use existing playbooks.** Route to `provider-playbooks/hunter.md` for email discovery, `provider-playbooks/heyreach.md` for activation, `provider-playbooks/hubspot.md` for CRM sync. Do not re-implement provider logic.
- **Leverage niche-signal-discovery.** If the customer provides won/lost domain lists during Step 3, invoke the `niche-signal-discovery` skill for automated differential analysis.
