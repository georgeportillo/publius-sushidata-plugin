---
name: sushi-signals
description: >
  Scan LinkedIn posts and comments for buying signals from ICP accounts and prospects.
  Triggers: "check signals", "run signals", "find signals", "signal scan", "buying signals",
  "linkedin signals", "sushi signals", "who just moved jobs", "who's hiring",
  "find pain signals", "prospect signals", "signal detection", "run my signals",
  or any request to detect job change, pain, or hiring signals from LinkedIn activity.
  Run as a scheduled event (Monday–Friday, 9:00 AM) to surface fresh pro insights before outreach.
---

# Sushidata Signal Detection — LinkedIn Buying Signals

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

**Schedule:** Monday–Friday, 9:00 AM (customer local time)
**Target output:** Ranked list of prospects mapped to their detected buying signal, ready for outreach

---

## Signal Types

| Signal | What to Look For | Why It Matters |
|---|---|---|
| **Job Change** | Post announcing new role, new company, "excited to join", "starting at", "my next chapter" | New exec = new budget, new priorities, open to new vendors |
| **Pain Post** | Post describing a challenge, frustration, asking for advice, "struggling with", "looking for a better way" | Active pain = high intent to buy |
| **Hiring Signal** | Job posting for a role that implies the pain the product solves (e.g., hiring a Head of RevOps) | Budget allocated, initiative confirmed |

---

## Dependencies & Tools

| Capability | Source |
|---|---|
| Context lake CRUD | `sushi-research` SKILL → `/context/`, `/query/`, `/swarm/deploy/` |
| LinkedIn post search | `apify_linkedin_post_search` (via `provider-playbooks/apify.md`) |
| Lead enrichment | `apify_leads_finder` (via `provider-playbooks/apify.md`) |
| Session persistence | `sushi-save` skill |

---

## Step 0 — Load ICP Targets

Query the Sushidata context lake for saved ICP accounts and prospect lists:

```json
{
  "query": "ICP accounts target companies prospect list signal tracking",
  "top_k": 20
}
```

If no ICP is saved yet, ask the user via `askQuestions`:

```
header: ICP Targets
question: Which companies or people should I scan for signals?
message: Paste LinkedIn company names, profile URLs, or people's names — one per line. I'll search their recent posts and comments for buying signals.
```

Load the results as `target_accounts`.

---

## Step 1 — Build Search Queries

For each target account or persona archetype in `target_accounts`, construct LinkedIn post search queries:

- **Job Change**: `"[person name] new role"` or `"[company name] just joined"`
- **Pain**: `"[ICP industry] [pain keyword]"` — use pain keywords from the saved ICP profile (e.g., `"revenue operations struggling"`, `"sales velocity bottleneck"`)
- **Hiring**: `"[company name] hiring [target role]"` or `"[target role] [ICP industry] job"`

Run up to **3 queries per signal type** to avoid over-calling. Start broad, then refine.

---

## Step 2 — Scrape LinkedIn Posts

For each query, call `apify_linkedin_post_search`:

```json
{
  "searchQueries": ["<query from Step 1>"],
  "maxPosts": 15,
  "profileScraperMode": "short",
  "maxReactions": 10
}
```

Collect all returned posts into a raw signal pool. Deduplicate by post URL.

---

## Step 3 — Classify Each Post

For every post in the signal pool, apply this classification logic:

### Job Change Detection
Flag if ANY of the following appear in the post text or author headline:
- "excited to join", "starting at", "new role", "new chapter", "just joined"
- "my next adventure", "I'm thrilled to announce"
- Author's current title doesn't match their previous title (if profileScraperMode returns both)

### Pain Signal Detection
Flag if the post text contains:
- Question-form posts asking for vendor/tool recommendations
- "struggling with", "looking for a better way", "anyone else dealing with", "pain point"
- Posts in the comment thread where the OP is asking for solutions
- Explicit mentions of the pain categories defined in the ICP profile from the context lake

### Hiring Signal Detection
Flag if:
- Post is a job posting from an ICP company
- Role title matches a buyer persona or signals budget/initiative
  (e.g., "VP of Sales", "Head of RevOps", "Director of Demand Gen", "Growth Lead")
- Hiring volume is unusual (multiple open roles at once)

Discard posts that don't match any signal type.

---

## Step 4 — Enrich Prospects

For each flagged post, extract the poster's name and company. If email or LinkedIn URL is missing, enrich using `apify_leads_finder`:

```json
{
  "searchQueries": ["<first name> <last name> <company>"],
  "maxResults": 1
}
```

Pull: full name, title, company, LinkedIn URL, email (if available).

---

## Step 5 — Map Prospects to Signals

Build the signal map table. For each matched prospect, output one row:

| Prospect | Title | Company | Signal Type | Evidence | LinkedIn Post | Recommended Action |
|---|---|---|---|---|---|---|
| Jane Smith | VP Revenue | Acme Corp | Pain | "We're drowning in manual reporting" | [link] | Lead with reporting automation angle |
| Tom Lee | — | BetaCo | Hiring | Posted Head of RevOps role | [link] | Reach out before new hire starts |
| Sara Cruz | New: CRO at Delta | Delta Inc | Job Change | "Thrilled to join Delta as CRO" | [link] | Intro email within 48 hrs |

Sort by signal priority: **Job Change → Pain → Hiring**.

---

## Step 6 — Deliver the Report

Present the signal map table to the user with:

1. **Summary line**: `Found X signals across Y prospects — [N] job changes, [N] pain posts, [N] hiring signals.`
2. The ranked signal map table (Step 5).
3. **Top 3 priority prospects** called out with a brief recommended first line for outreach.

Then ask:

```
header: Next Step
question: What would you like to do with these signals?
options:
  - label: Save signals to Sushidata
  - label: Add top prospects to HeyReach campaign
  - label: Draft outreach messages for top 3
  - label: Export to CSV
  - label: Nothing — just reviewing
```

---

## Step 7 — Save to Context Lake (if requested)

Save the signal map to Sushidata under a structured context entry:

```json
{
  "type": "signal_scan",
  "date": "<today's date>",
  "signals": [
    {
      "prospect": "<name>",
      "company": "<company>",
      "signal_type": "job_change | pain | hiring",
      "evidence": "<quote or description>",
      "post_url": "<url>",
      "recommended_action": "<text>"
    }
  ]
}
```

Use `sushi-save` skill to write the entry.

---

## Scheduled Run Behavior

When invoked automatically (e.g., Monday–Friday 9:00 AM):

1. Skip Step 0 — use saved ICP targets from context lake without asking.
2. Run Steps 1–5 silently.
3. Surface the signal report directly (Step 6 table + summary).
4. Auto-save the scan results to the context lake (Step 7) without prompting.
5. If no signals found: output `No new signals detected today. Will check again next scheduled run.`
