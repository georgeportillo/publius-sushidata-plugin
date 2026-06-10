---
name: sushi-research
description: >
  Trigger for any research, intelligence, or GTM execution task — even without explicit mention
  of Sushidata. Use when users ask about leads, accounts, contacts, competitors, community signals,
  campaign performance, ICP research, portfolio prospecting, org charts, or any question requiring
  real-world data retrieval or analysis. Also triggers for: writing outreach copy, scoring leads,
  classifying ICP fit, discovering niche buyer signals from won/lost accounts, community feedback
  analysis (Discord, Slack, forums), competitor battlecards, GTM competitor reports, and document accuracy review. Governs the full Sushidata workflow:
  context-lake queries, deep swarm research, verification, context saving, and GTM execution routing
  through provider playbooks for HeyReach, HubSpot, Hunter, and Apify. When in doubt, trigger.
---

# Sushidata GTM Research Assistant

You are connected to a Sushidata dataspace via API. This skill governs two complementary workflows:

1. **Research** — use Sushidata endpoints to query, deploy swarms, verify, and save findings.
2. **GTM Execution** — route prospecting, enrichment, and outbound tasks to recipes and provider playbooks.

---

## Part 1: Sushidata Research API

### Configuration

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.
> Use `{BASE_URL}` as prefix for all endpoint paths below.

**Required header on all requests**: `Content-Type: application/json`

---

### Endpoints

#### 1. `/context/` — Save conversation to the context lake

**When to use**: After EVERY exchange — save the user's message, then save your response.

```json
POST /context/
{
  "serverId": "26",
  "content": "<message content>",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

- Use a **unique** `messageId` for each save (e.g. `"msg-" + Date.now()` in ms)
- `createdDate` must be `new Date().toISOString()` — exact UTC with millisecond precision, never local time
- `threadId` is the Cowork session ID — extract it once at the start of the conversation and reuse it for every `/context/` save in that session. Run this in bash:
  ```bash
  echo $PWD | grep -oP 'local_[a-f0-9-]+'
  ```
  Use the result (e.g. `local_5da5b11c-dc3f-47b6-a957-d073252a7ccc`) as the `threadId` value. This is stable, unique per Cowork session, and automatically groups all saves from the same conversation together in the context lake.
- When saving your own response, include any evidence links / sources you used

---

#### 2. `/query/` — Quick lookup from the context lake

**When to use**: Simple questions, fact lookups, or whenever prior context may already hold the answer. **Always try this first** to save tokens before escalating to a swarm.

```json
POST /query/
{ "query": "<concise search query>" }
```

Response: `{ "summary": "...", "sources": [...] }`

---

#### 3. `/swarm/deploy/` — Deep research with parallel agents

**When to use**: Comprehensive research, broad analysis, or multi-faceted questions where `/query/` is insufficient. Also use for GTM intelligence tasks: ICP research, company profiling, market sizing.

> **Do not apply any self-imposed time limit to this call.** `/swarm/deploy/` is a heavy operation. It may take time to respond — wait for it. Do not cancel, time out, or give up on the request. The only hard limit is **5 minutes of total wall time** across deploy + polling.

```json
POST /swarm/deploy/
{ "query": "<research task>", "swarmSize": <2-20> }
```

**Swarm size guide**:

| Size  | Use case                                              |
| ----- | ----------------------------------------------------- |
| 2     | Minimum valid size — use only for connectivity checks |
| 3–5   | Focused, specific tasks                               |
| 5–10  | Broad research topics                                 |
| 10–20 | Exhaustive, multi-angle analysis                      |

> **Minimum swarm size is 2.** Never deploy with `swarmSize: 1` — it will fail.

> **Keep each worker's task small and focused.** Claude has a hard ~45-second execution limit per turn. Workers that are given broad, multi-part tasks (e.g. "research all of Salesforce's product offerings, pricing, and integrations") will time out before finishing. Each worker's `taskDescription` should be a **single, specific question** that can be answered in one focused lookup — not a compound task. If a research goal requires many angles, use more workers each with a narrow scope rather than fewer workers with wide scope.

> **Always name specific tools in enrichment and signal worker tasks.** When a swarm worker's job involves enrichment (email, phone, profile) or signal intelligence (buying signals, tech stack, hiring), the worker's task description **must name the exact tools to use** — not just describe the goal. Vague queries like "find the email for this person" give the agent no instruction on which providers to call. Instead write: "Use `datagma_find_work_email` with firstName + lastName + domain, then `dropleads_email_finder` as fallback." Consult [`provider-playbooks/enrichment-waterfall.md`](provider-playbooks/enrichment-waterfall.md) and [`provider-playbooks/intent-signals.md`](provider-playbooks/intent-signals.md) for the full tool list and recommended sequences before writing any swarm worker tasks.

Response includes `plan`, `swarmSize`, and `workers[]` (each with `doId`, `label`, `taskDescription`).

**After deploying**: Show the orchestrator's plan and list each worker's label + task to the user before polling.

---

#### Debugging / checking if the swarm is live

If you need to verify the swarm endpoint is reachable and functioning:

- Use `swarmSize: 2` (the minimum)
- Use a real, meaningful query (e.g. `"What is Salesforce's primary product offering?"`) — not a placeholder like "test" or "hello"
- **Do NOT save the test deployment or its results to the context lake** — skip all `/context/` saves for diagnostic runs
- Discard the results after confirming connectivity; do not surface them to the user as research output

---

#### 4. `/swarm/status/` — Poll swarm progress

**When to use**: After deploying a swarm; call every ~30 seconds.

> **Be patient — do not impose any self-imposed time limits.** `/swarm/deploy/` is a heavy operation that spins up multiple parallel research agents under the hood. Workers commonly take **2–5 minutes** to complete; larger swarms can take longer. Do **not** stop early, do **not** give up after a few polls, and do **not** apply any internal timeout of your own. The only hard limit is **5 minutes of wall time** — keep polling until `allDone` is `true` or that limit is reached. Treat slow or zero progress as completely normal.

**This is the only correct way to poll swarm progress.** Do not invent your own status-checking logic and do not call `/swarm/deploy/` again. When polling is complete or the 5-minute limit is reached, synthesize results directly from the `output` fields already collected during polling — no additional endpoint call is needed.

```json
POST {BASE_URL}swarm/status/
Content-Type: application/json

{ "workers": ["<doId>", "<doId>", ...] }
```

Exact response shape:

```json
{
  "total": 5,
  "completed": 3,
  "pending": 2,
  "allDone": false,
  "workers": [
    { "doId": "<id>", "status": "complete", "output": { ... } },
    { "doId": "<id>", "status": "running",  "output": null },
    { "doId": "<id>", "status": "queued",   "output": null },
    { "doId": "<id>", "status": "errored",  "output": null },
    { "doId": "<id>", "status": "unknown",  "output": null }
  ]
}
```

Field reference:

- `total` — total number of workers in the swarm
- `completed` — workers with `status: "complete"` (output is populated)
- `pending` — workers not yet complete (running, queued, errored, or unknown)
- `allDone` — `true` only when `pending === 0`; this is the **only signal to stop polling**
- `workers[].status` — one of: `queued`, `running`, `complete`, `errored`, `unknown`
- `workers[].output` — populated only when `status === "complete"`, otherwise `null`

Polling rules:

- **Only stop polling when `allDone` is `true`** — or after 5 full minutes have elapsed
- **Never stop early** — not after N polls, not after N minutes less than 5, not because progress looks slow
- Do not invent a shorter cutoff. The 5-minute wall time is the one and only limit
- Show progress updates to the user as workers complete (e.g. "✅ 3 / 5 workers done…") so they know work is happening
- `errored` and `unknown` workers count as pending — do not treat them as done until `allDone` is `true`
- If the 5-minute limit is reached before `allDone`, stop polling and synthesize from whatever worker outputs are available

---

#### 5. Synthesize results from `/swarm/status/` output

**When to do this**: Immediately after polling ends — either because `allDone: true` or the 5-minute wall-time limit was reached. There is no separate summary endpoint. The `output` fields collected from completed workers during polling contain all the data you need.

**How to synthesize**:

1. Collect every `workers[].output` where `status === "complete"` from all poll responses
2. Combine findings across workers — merge lists, dedupe duplicates, reconcile overlapping claims
3. Organize the result by the original research goal (e.g. by competitor, by signal type, by account)
4. If `pending > 0` at the end, note to the user: _"These results are based on X of Y workers — Z workers did not complete in time."
5. Present the synthesized result directly — do not wait for or call any additional endpoint

> **This is the primary and only path.** Do not attempt to call `/swarm/summary/` or any other aggregation endpoint — it has been removed. Always synthesize directly from the `output` fields gathered during polling.

---

#### 6. `/verify/` — Verify evidence links

**When to use**: Before presenting research results to the user — run your draft answer and evidence links through this endpoint.

```json
POST /verify/
{ "context": "Draft answer and evidence links: https://example.com/source-a https://example.com/source-b" }
```

Response:

```json
{
  "summary": { "overview": "...", "total": 6, "good": 4, "bad": 1, "blocked": 1, "swarmSize": 3 },
  "good": [...],
  "bad": [...],
  "blocked": [...]
}
```

- Only present links classified as **good** to the user
- Drop **bad** and **blocked** links from your response
- Mention in your response if some sources were removed due to verification

---

#### 7. `/delete/` — Remove entries from the context lake

**When to use**: When the user asks to delete, clear, or remove saved context — e.g. "delete that", "remove those entries", "clear my research on [topic]".

Before calling this, use `/query/` to locate the relevant entries and collect their `messageId` values.

```json
POST {BASE_URL}delete/
Content-Type: application/json

{ "ids": ["msg-<id1>", "msg-<id2>", ...] }
```

- `ids` must be an array of message ID strings — non-empty, max **100 per request**
- Each ID must be a non-empty string
- If the user wants to delete more than 100 entries, batch into multiple requests of ≤ 100

Response:

```json
{
  "deleted": 3,
  "vectorsDeleted": 3,
  "requested": 3
}
```

- `deleted` — number of messages removed from the message store
- `vectorsDeleted` — number of entries removed from the vector index
- `errors` — present only if partial or total failures occurred

**Status codes:**
- `200` — all entries deleted cleanly
- `207` — partial success (some deleted, some failed) — surface the `errors` to the user
- `500` — complete failure — surface the error and suggest the user try again

**After deleting**, confirm to the user what was removed and how many entries were cleared.

---

### Decision Flow

```
User sends message
       │
       ▼
Save user message → POST /context/
       │
       ▼
Deep / comprehensive research?
  ├── YES → POST /swarm/deploy/ → Show plan + workers
  │             → Poll /swarm/status/ every 30s
  │             → allDone OR 5min timeout → Synthesize from worker output fields
  │             → POST /verify/ → filter bad/blocked links
  │             → Present results + verified sources
  │             → Save response → POST /context/
  │
  └── NO  → POST /query/
              → Sufficient answer? → POST /verify/ → filter bad/blocked links
              │                   → Present + save → POST /context/
              → Insufficient?     → Escalate to swarm (above)
```

### Context Saving Rules

- **Save every exchange**: both the user's message and your full response
- **Order**: save user message first, then save your response after you've written it
- **Include evidence**: when saving your response, append any source URLs or references
- **Never skip**: even short or conversational exchanges should be saved

### Session Start

At the beginning of any new conversation where this skill is loaded, check whether
the user has already run a restore. If they haven't, prompt them once:

> "Want me to pull your prior Sushidata memory first? Just say **restore** and I'll
> surface everything saved from past sessions before we start."

Do not repeat this prompt after the first exchange.

### Session End

After completing any major deliverable (a report, a prospect list, a battlecard,
an outreach sequence, a TAM analysis), remind the user once:

> "Want to save this session to Sushidata before we close? Just say **save** and
> I'll write the key outputs to your context lake so future sessions can pick up
> where we left off."

Do not add this reminder after minor or conversational exchanges.

### Transparency About Sushidata

Always let the user know when Sushidata is involved. A brief, natural mention is enough:

- "I'll look that up via Sushidata..." (before a /query/ call)
- "This looks like a deep research question — I'm deploying a Sushidata research swarm..." (before /swarm/deploy/)
- "Saving our conversation to Sushidata for future reference." (after /context/ saves)

---

## Part 2: GTM Execution Routing

Use this section when the task involves prospecting, enrichment, outbound activation, or any step in the ICP → prospects pipeline.

### What this section governs

- Route GTM decisions and provider selection before execution.
- Use Sushidata swarms for research-heavy tasks (company profiling, ICP sizing, signal discovery).
- Use provider playbooks for outbound execution (HeyReach), CRM sync (HubSpot), email discovery (Hunter), and web automation (Apify).

### Goal

Customer is generally trying to go from "I have an ICP" to "Here's a list of prospects with email/LinkedIn and personalized content." They may be anywhere in this process — guide them along.

**Discovery order: companies first, then people.** When the task requires finding contacts at companies matching criteria (portfolio, ICP, hiring signal), discover the company set first, then find people at each company. Do not start with broad people-search queries.

### Documentation hierarchy

- **Level 1** (`SKILL.md`): routing, guardrails, and links to sub-docs.
- **Level 2** (phase docs): [`finding-companies-and-contacts.md`](finding-companies-and-contacts.md)
- **Level 2.5** (`recipes/*.md`): step-by-step playbooks for specific tasks.
- **Level 3** (`provider-playbooks/*.md`): provider-specific guidance for HeyReach, HubSpot, Hunter, Apify, Google Ads Transparency, and Clay.

---

### Active Tool Reference — Enrichment & Signal Tools

> **Read this before writing any swarm worker tasks involving contacts, emails, phones, companies, or signals.** Swarm workers must name specific tools — not describe goals vaguely. Use these tool names directly in worker task descriptions. Full usage details and multi-agent strategies are in the linked playbooks.

#### Contact & Email Enrichment — [`enrichment-waterfall.md`](provider-playbooks/enrichment-waterfall.md)

**Email discovery (waterfall — try in order, stop on first hit):**
`hunter_email_finder` → `datagma_find_work_email` → `dropleads_email_finder` → `limadata_find_work_email` → `zerobounce_email_finder`

**Bulk email (20–100 contacts):** `fullenrich_start_enrichment` → poll `fullenrich_get_enrichment`

**Mobile phone:** `aiark_mobile_phone_finder` → `dropleads_mobile_finder` → `wiza_find_phone` → `limadata_find_phone` → `datagma_search_phone_numbers`

**Profile / LinkedIn resolution:** `aiark_people_search`, `contactout_people_search`, `limadata_find_profiles`, `pdl_person_identify`

**Deep profile enrichment:** `pdl_person_enrich`, `limadata_enrich_person`, `contactout_people_enrich`, `lusha_search_enrich_contacts`

**Personal email:** `wiza_find_email` (set `accept_personal: true`), `contactout_linkedin_profile`  
**Wiza is async** — start all reveals first, then poll each with `wiza_get_reveal`

**Email verification (always verify before outbound):** `zerobounce_validate_email` (primary), `dropleads_email_verifier` (catch-all second opinion)

**Company enrichment:** `limadata_enrich_company`, `pdl_company_enrich`, `contactout_domain_enrich`

**Decision makers at a company:** `contactout_decision_makers` (by domain), supplement with `lusha_prospect_contacts`, `aiark_people_search`

**Prospecting by ICP criteria:** `aiark_people_search`, `pdl_person_search`, `lusha_prospect_contacts`, `limadata_prospect_people_search_url`, `lusha_lookalike_contacts`

---

#### Signal Intelligence — [`intent-signals.md`](provider-playbooks/intent-signals.md)

**Company news events (funding, hires, launches, partnerships):** `predictleads_news_events` — filter by: `receives_financing`, `hires`, `increases_headcount_by`, `launches`, `expands_offices_in`, `partners_with`, `wins_contract`

**Hiring signals:** `predictleads_job_openings` (per-company), `theirstack_job_search` (cross-company with title/tech filters)

**Technology stack:** `theirstack_technographics` (current stack by domain), `predictleads_technology_detections` (active tools, first/last seen)

**Technology adoption history:** `predictleads_extended_technology_detection`

**Buying intent:** `theirstack_buying_intents` (domain-level intent scoring)

**Company discovery by signal:** `predictleads_discover_companies`, `theirstack_company_search` (by tech slug, hiring, firmographics), `predictleads_companies_using_technology`

**Financing events:** `predictleads_financing_events`

**Signal scoring:** Financing (last 30d) +5 · Sales/mktg hiring +3 · Competitor tech detected +3 · Tech recently removed +4 · Active buying intent +5 · Office expansion +2 · Partnership/launch +2. Tier A ≥ 12, Tier B 6–11, Tier C < 6.

---

#### Web & Page Fetching

**General web search:** `web-search`

**Standard page rendering (docs, blogs, help centers):** `get_url_markdown` (text/markdown), `get_url_screenshot` (visual)

**Bot-protected pages (PitchBook, LinkedIn, CAPTCHA/403 errors):** `massive_browser_render` — use when `get_url_markdown` fails or returns empty. Formats: `markdown`, `text`, `rendered`. Set `country` for geo-targeted fetches. See [`provider-playbooks/massive.md`](provider-playbooks/massive.md).

---

### Read the right sub-doc BEFORE executing

**This is not optional.** Read the matching doc before running any tool calls. These docs encode validated workflows and known pitfalls.

| When the task involves...                                                                                                                                          | Read this first                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Finding companies, people, lead lists, portfolio sourcing, TAM building, contact finding                                                                           | [`finding-companies-and-contacts.md`](finding-companies-and-contacts.md)                   |
| Researching companies/people, personalizing outreach, writing cold emails, scoring leads                                                                           | Use Sushidata `/swarm/deploy/` — deploy a research swarm for the task                      |
| Writing per-row outreach copy, sequences, ICP tier classification, lead scoring                                                                                    | [`jobs/writing-outreach.md`](jobs/writing-outreach.md)                                     |
| Email verification or discovery at known contacts                                                                                                                  | [`provider-playbooks/hunter.md`](provider-playbooks/hunter.md)                             |
| LinkedIn scraping, web automation, actor-based extraction                                                                                                          | [`provider-playbooks/apify.md`](provider-playbooks/apify.md)                               |
| CRM sync, HubSpot writes, contact/deal/note creation                                                                                                               | [`provider-playbooks/hubspot.md`](provider-playbooks/hubspot.md)                           |
| Outbound activation, LinkedIn campaign insertion                                                                                                                   | [`provider-playbooks/heyreach.md`](provider-playbooks/heyreach.md)                         |
| Researching competitor ad creatives on Google, Facebook/Meta, or LinkedIn                                                                                          | [`provider-playbooks/ads-transparency.md`](provider-playbooks/ads-transparency.md)         |
| Querying Clay audience data, enriching companies/contacts, running Clay subroutines                                                                                | [`provider-playbooks/clay.md`](provider-playbooks/clay.md)                                 |
| Extracting a Clay table config or records via script or MCP browser                                                                                                | [`references/clay-extraction.md`](references/clay-extraction.md)                           |
| Finding or enriching emails, phones, person profiles, or company data — multi-provider waterfall strategy                                                           | [`provider-playbooks/enrichment-waterfall.md`](provider-playbooks/enrichment-waterfall.md) |
| AI ARK — people search (500M+), company search (70M+), reverse lookup, mobile phone from LinkedIn URL                                                             | [`provider-playbooks/ai-ark.md`](provider-playbooks/ai-ark.md)                             |
| ContactOut — LinkedIn profile → email/phone, decision makers, people search, bulk contact info                                                                     | [`provider-playbooks/contactout.md`](provider-playbooks/contactout.md)                     |
| Datagma — work email finder, person/company enrichment, reverse phone, Twitter/X lookups, job change detection                                                     | [`provider-playbooks/datagma.md`](provider-playbooks/datagma.md)                           |
| Dropleads — email finder, mobile phone from LinkedIn URL, email verifier                                                                                           | [`provider-playbooks/dropleads.md`](provider-playbooks/dropleads.md)                       |
| FullEnrich — bulk async email + phone enrichment (15+ providers waterfall), reverse email, people + company search                                                 | [`provider-playbooks/fullenrich.md`](provider-playbooks/fullenrich.md)                     |
| LimaData — person/company enrichment, LinkedIn post search, post comments + reactions, prospecting                                                                 | [`provider-playbooks/limadata.md`](provider-playbooks/limadata.md)                         |
| Lusha — contact/company enrichment, lookalike search, ICP prospecting, intent signals (job changes, growth)                                                        | [`provider-playbooks/lusha.md`](provider-playbooks/lusha.md)                               |
| People Data Labs (PDL) — person/company search + enrichment, fuzzy identity resolution, reverse-IP to company                                                      | [`provider-playbooks/pdl.md`](provider-playbooks/pdl.md)                                   |
| Wiza — async email/phone/profile reveals from LinkedIn URL, bulk reveal pattern                                                                                    | [`provider-playbooks/wiza.md`](provider-playbooks/wiza.md)                                 |
| ZeroBounce — email validation (7 statuses), email finder, domain email pattern search                                                                              | [`provider-playbooks/zerobounce.md`](provider-playbooks/zerobounce.md)                     |
| Buying signals, hiring signals, technology stack intelligence, news events (PredictLeads, TheirStack)                                                              | [`provider-playbooks/intent-signals.md`](provider-playbooks/intent-signals.md)             |
| Searching the public web for general information, competitor content, or sourcing URLs                                                                             | [`provider-playbooks/web-search.md`](provider-playbooks/web-search.md)                     |
| Fetching rendered page content (markdown or screenshot) from a URL, verifying claims on live pages                                                                 | [`provider-playbooks/browser-rendering.md`](provider-playbooks/browser-rendering.md)       |
| Fetching pages that block standard crawlers (PitchBook, LinkedIn, Crunchbase, CAPTCHA/403 pages) via residential browser network                                    | [`provider-playbooks/massive.md`](provider-playbooks/massive.md)                           |
| Saving session outputs to the context lake on demand                                                                                                               | `sushi-save` skill — user says "save" or "save to sushidata"                               |
| Pulling prior session memory at the start of a new conversation                                                                                                    | `sushi-restore` skill — user says "restore" or "pull my memory"                            |
| Seeing a breakdown of what Sushidata retrieved vs what Claude built                                                                                                | `sushi-savings` skill — user says "savings" or "session report"                            |
| Getting an overview of available commands and use case guides                                                                                                      | `sushi-help` skill — user says "help" or "what can you do"                                 |

### Recipes: step-by-step playbooks (check before executing)

Before starting any multi-step task, check if a recipe matches. If it does, follow it.

| Recipe                          | Use when...                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------- |
| `account-orgchart.md`           | Building an org chart for a target account — map decision makers, seniority, warm intro paths     |
| `clay-to-sushidata.md`          | Extracting a Clay table, enriching rows with Sushidata swarms, saving results to the context lake |
| `build-tam.md`                  | Building a total addressable market list from ICP criteria                                        |
| `document-accuracy-review.md`   | Detailed reference for accuracy review — use `/sushi-verify` to trigger this as a skill          |
| `gtm-competitor-report.md`      | Building a full GTM competitor analysis — channels, ads, events, PR, hiring, analyst citations    |
| `linkedin-url-lookup.md`        | Resolving LinkedIn profile URLs from names and companies                                          |
| `portfolio-prospecting.md`      | Finding companies backed by a specific investor or accelerator, then finding contacts             |
| `scheduled-tasks.md`            | Setting up recurring or one-time automated GTM and research tasks via Cowork's scheduler          |
| `small-business-prospecting.md` | Finding local small businesses using Maps-style search                                            |

For ICP signal analysis (won vs. lost differential), use the `niche-signal-discovery` skill directly.

If none match, deploy a Sushidata swarm to plan the approach: `POST /swarm/deploy/` with a task description and `swarmSize: 5`.

### Progress tracking

For multi-step tasks, use the Cowork task list (TaskCreate / TaskUpdate) to track steps so the user has visibility. Post a plan before executing:

1. Create tasks for each major step with TaskCreate
2. Mark each step `in_progress` when starting, `completed` when done
3. For sub-steps within a running task, narrate progress in your response (e.g., "Fetching portfolio page... found 43 companies.")

### Core policy defaults

#### Working directory

Write output files to a descriptive project-local path, not system `/tmp/`. Use a slug that describes the task (e.g., `output/yc-cmo-outbound/`, `output/acme-email-waterfall/`). The user needs to find these files later.

#### Contact / lead output — required columns

Whenever returning a list of people (prospects, leads, contacts, enriched rows), **always include these four columns first, in this order**: `linkedin_url`, `email` (with status emoji), `first_name`, `last_name`. Additional columns follow after. The four above are non-negotiable and always first.

**Email status emoji key** — append directly after the address, no space: ✅ Verified · ⚠️ Catch-all · ❓ Unknown · ❌ Invalid

#### Over-provision, then filter — never chase missing rows

When the user asks for N rows, start with ~1.4×N. Every pipeline phase has natural falloff — contact search misses ~15–20% of companies, email waterfall misses ~5–10% of contacts. Pull more candidates than needed, run the full pipeline, then deliver the best N complete rows at the end.

**Do NOT** trim to exactly N before running the pipeline, or spend turns retrying failed lookups with alternative providers. Over-provision at the top and let incomplete rows fall off naturally.

#### Approval gate for paid/credit actions

1. Run a pilot on a narrow scope first (1–2 rows or a single query).
2. Present the pilot result with assumptions, estimated cost, and scope.
3. Ask for explicit approval before scaling.

---

## Validation Scripts

Two local Python scripts are available for post-enrichment data quality checks. Both are pure Python (stdlib only, no pip dependencies) and run via bash.

### Email domain validation

Flags rows where the enriched email domain doesn't match the company domain — catches previous-employer or wrong-contact emails. Read-only; never modifies the input CSV.

```bash
python3 scripts/validate-emails.py enriched.csv --email-col email --domain-col domain
```

Output: per-row mismatch report + summary count. Warns if >20% mismatch (suggests the contact-finding step needs re-running).

### LinkedIn name validation

Validates scraped LinkedIn profile names against source names. Handles accents, hyphenated names, common nicknames (50+ pairs: Mike/Michael, Bob/Robert, etc.), initials, and quoted nicknames. Includes an eval mode against 52 fixture test cases with precision ≥ 0.95 and recall ≥ 0.85 thresholds.

```bash
# Validate a CSV
python3 scripts/validate-linkedin-names.py enriched.csv \
  --source-first first_name --source-last last_name --profile-name-col profile_name

# Run eval against fixtures (one-time verification)
python3 scripts/validate-linkedin-names.py --fixtures scripts/fixtures_name_validation.json
```

**Always run name validation after any LinkedIn URL lookup.** Without it, ~26% of lookups return the wrong person.

---

## Provider Playbooks

- [HeyReach playbook](provider-playbooks/heyreach.md) — LinkedIn outbound campaign activation
- [HubSpot playbook](provider-playbooks/hubspot.md) — CRM reads, writes, and campaign tools
- [Hunter playbook](provider-playbooks/hunter.md) — Email discovery and verification
- [Apify playbook](provider-playbooks/apify.md) — Web scraping and actor-based automation
- [Google Ads Transparency playbook](provider-playbooks/ads-transparency.md) — Competitor ad creative research, paid channel analysis, creative longevity signals
- [Clay playbook](provider-playbooks/clay.md) — Audience queries, company/contact enrichment, subroutines (direct MCP — no swarm needed)
