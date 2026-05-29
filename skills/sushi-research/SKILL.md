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
  "createdDate": "<ISO 8601 timestamp>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

- Use a **unique** `messageId` for each save (e.g. `"msg-" + Date.now()` in ms)
- `createdDate` must be valid ISO 8601 (e.g. `"2026-05-07T14:30:00.000Z"`)
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

```json
POST /swarm/deploy/
{ "query": "<research task>", "swarmSize": <2-20> }
```

**Swarm size guide**:

| Size  | Use case                              |
| ----- | ------------------------------------- |
| 2     | Minimum valid size — use only for connectivity checks |
| 3–5   | Focused, specific tasks               |
| 5–10  | Broad research topics                 |
| 10–20 | Exhaustive, multi-angle analysis      |

> **Minimum swarm size is 2.** Never deploy with `swarmSize: 1` — it will fail.

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

```json
POST /swarm/status/
{ "workers": ["<doId>", ...] }
```

Response: `{ "total": N, "completed": N, "pending": N, "allDone": bool, "workers": [...] }`

- **Only stop polling when `allDone` is `true`** — or after 5 full minutes have elapsed
- **Never stop early** — not after N polls, not after N minutes less than 5, not because progress looks slow
- Do not invent a shorter cutoff. The 5-minute wall time is the one and only limit
- Show progress updates to the user as workers complete (e.g. "✅ 4 / 8 workers done…") so they know work is happening
- If the 5-minute limit is reached before `allDone`, proceed to `/swarm/summary/` with whatever is available

---

#### 5. `/swarm/summary/` — Get unified summary (partial or complete)

**When to use**: Once the swarm is complete (`allDone: true`), or to get partial results if the timeout is reached.

```json
POST /swarm/summary/
{ "workers": ["<doId>", ...], "query": "<original task>" }
```

Response: `{ "summary": "...", "completed": N, "total": N, "pending": N }`

If workers are still pending, clearly note: *"These results are based on X of Y workers — Z workers are still running."*

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
  │             → allDone OR 5min timeout → POST /swarm/summary/
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

### Read the right sub-doc BEFORE executing

**This is not optional.** Read the matching doc before running any tool calls. These docs encode validated workflows and known pitfalls.

| When the task involves...                                                                    | Read this first                                                        |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Finding companies, people, lead lists, portfolio sourcing, TAM building, contact finding     | [`finding-companies-and-contacts.md`](finding-companies-and-contacts.md) |
| Researching companies/people, personalizing outreach, writing cold emails, scoring leads     | Use Sushidata `/swarm/deploy/` — deploy a research swarm for the task  |
| Writing per-row outreach copy, sequences, ICP tier classification, lead scoring              | [`jobs/writing-outreach.md`](jobs/writing-outreach.md)                  |
| Email verification or discovery at known contacts                                            | [`provider-playbooks/hunter.md`](provider-playbooks/hunter.md)         |
| LinkedIn scraping, web automation, actor-based extraction                                    | [`provider-playbooks/apify.md`](provider-playbooks/apify.md)           |
| CRM sync, HubSpot writes, contact/deal/note creation                                        | [`provider-playbooks/hubspot.md`](provider-playbooks/hubspot.md)       |
| Outbound activation, LinkedIn campaign insertion                                             | [`provider-playbooks/heyreach.md`](provider-playbooks/heyreach.md)     |
| Researching competitor Google ad creatives, paid ad formats, creative longevity             | [`provider-playbooks/ads-transparency.md`](provider-playbooks/ads-transparency.md) |
| Querying Clay audience data, enriching companies/contacts, running Clay subroutines         | [`provider-playbooks/clay.md`](provider-playbooks/clay.md)                         |
| Extracting a Clay table config or records via script or MCP browser                        | [`references/clay-extraction.md`](references/clay-extraction.md)                   |
| Saving session outputs to the context lake on demand                                        | `sushi-save` skill — user says "save" or "save to sushidata"       |
| Pulling prior session memory at the start of a new conversation                             | `sushi-restore` skill — user says "restore" or "pull my memory"    |
| Seeing a breakdown of what Sushidata retrieved vs what Claude built                         | `sushi-savings` skill — user says "savings" or "session report"    |
| Getting an overview of available commands and use case guides                               | `sushi-help` skill — user says "help" or "what can you do"         |

### Recipes: step-by-step playbooks (check before executing)

Before starting any multi-step task, check if a recipe matches. If it does, follow it.

| Recipe                          | Use when...                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------- |
| `account-orgchart.md`           | Building an org chart for a target account — map decision makers, seniority, warm intro paths      |
| `clay-to-sushidata.md`          | Extracting a Clay table, enriching rows with Sushidata swarms, saving results to the context lake  |
| `build-tam.md`                  | Building a total addressable market list from ICP criteria                                        |
| `document-accuracy-review.md`   | Verifying factual accuracy, link integrity, and citation labels in any research document          |
| `gtm-competitor-report.md`      | Building a full GTM competitor analysis — channels, ads, events, PR, hiring, analyst citations    |
| `linkedin-url-lookup.md`        | Resolving LinkedIn profile URLs from names and companies                                          |
| `portfolio-prospecting.md`      | Finding companies backed by a specific investor or accelerator, then finding contacts              |
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
