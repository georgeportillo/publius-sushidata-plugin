---
name: sushi-matrix
description: >
  Build a competitive research matrix — a Google Sheet comparing competitors across
  features, with color-coded Yes/No/Limited cells and a supporting Evidence tab.
  Trigger when the user says: "matrix", "sushi matrix", "build a competitive matrix",
  "compare competitors", "research matrix", "competitor comparison", or any request
  to compare multiple companies or products across a set of criteria.
---

# Sushidata Matrix

When triggered, elicit what to analyze, run the Sushidata swarm, then deliver the
results as a **Google Sheet** with two tabs: a color-coded matrix and an Evidence tab.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**.

---

## Step 1 — Elicit what to analyze

Ask the user for the following in one message:

1. **Your product** — the company being positioned (the "us" column)
2. **Competitors** — which companies to compare (up to 20)
3. **Features** — what to research. For each feature, optionally ask for a **note** that narrows
   what the AI should look for (e.g. "file sharing — internal teams only, not external collaboration").
   Notes are especially important for multi-product platforms (Salesforce, Klaviyo, SAP) where
   the same feature exists across different product lines.
4. **Reasoning effort** — `low` (fast), `medium` (default), or `high` (thorough)

If the user provides enough context in their trigger message, skip the form and proceed directly.

**Source priority:** The swarm must prioritize **official knowledge base and help center documentation**
over marketing pages, blog posts, or social content. Marketing copy is unreliable for feature claims.

---

## Step 2 — Chunk and launch in batches

> **Never launch the entire matrix in one request.** Large matrices time out because Claude has a hard ~45-second execution limit per turn. Always break the work into small batches and complete each batch before starting the next.

### Chunk size rules

| Total cells (competitors × features) | Batch size |
|---------------------------------------|------------|
| ≤ 10 | 1 batch — all cells |
| 11–30 | 5 cells per batch |
| 31–60 | 4 cells per batch |
| > 60 | 3 cells per batch |

A "cell" = one competitor × one feature. Calculate total cells before launching. Tell the user the plan:

_"This matrix has [N] cells. I'll research them in [X] batches of ~[Y] cells each, so nothing times out."_

### Per-cell task scope

Each cell's task passed to the matrix endpoint must be a **single, tightly scoped question** — one competitor, one feature. Do not bundle multiple features or multiple competitors into one cell task. Broad tasks fail to complete in time.

**Good:** `"Does Salesforce Sales Cloud support inline spell-check in email compose?"`  
**Bad:** `"What are Salesforce's email, calendar, and mobile features?"`

### Launch one batch

```
POST {BASE_URL}matrix/create
Content-Type: application/json

{
  "competitors": [
    { "name": "<name>", "description": "<optional one-liner>" }
  ],
  "categories": [
    {
      "name": "<category>",
      "subcategories": [
        { "name": "<subcategory>", "description": "<optional>" }
      ]
    }
  ],
  "reasoningEffort": "medium"
}
```

Only include the competitors and features for **this batch** — not the full set.

Response includes `doIds` — an array of agent IDs, one per cell in the batch.

Tell the user: _"Batch [N] of [X] launched — researching [K] cells…"_

---

## Step 3 — Drain the batch, then repeat

```
POST {BASE_URL}matrix/status
Content-Type: application/json

{ "doIds": ["<doId>", ...] }
```

> **Do not start the next batch until the current one is fully drained.** Wait for `allDone: true` on the current batch before calling `/matrix/create` again. This keeps each turn's work small and prevents timeouts.

- Poll every **10 seconds**
- Stop **only** when `allDone` is `true` or **5 minutes** have elapsed for this batch
- Show a progress update after each poll: _"⏳ Batch [N]: X / K cells complete…"_
- If the 5-minute limit is reached before `allDone`, collect whatever completed, note the pending cells, then move on to the next batch
- After all batches are drained, merge results and proceed to Step 4

---

## Step 4 — Validate evidence

Before building the sheet, run a verification pass on all source URLs returned by the swarm.
This catches AI summaries that misrepresent the source (e.g. "loves X and Y" flipped from "loves Y, but X needs work").

```
POST {BASE_URL}verify/
Content-Type: application/json

{
  "items": [
    {
      "competitor": "<competitor name>",
      "feature": "<feature name>",
      "verdict": "<Yes | No | Limited>",
      "claim": "<the AI-generated summary for this cell>",
      "url": "<source URL>",
      "snippet": "<text excerpt from the source>"
    }
  ]
}
```

Send all cells in a single request. Response returns each item with:
- `verified: true/false` — whether the claim is supported by the source
- `confidence: 0–1` — confidence score
- `note` — explanation if verification failed or confidence is low

**Handling results:**
- `verified: true` and `confidence ≥ 0.85` → include as-is
- `verified: true` and `confidence < 0.85` → include but flag in the Evidence tab as `⚠️ Low confidence`
- `verified: false` → do **not** include in the matrix; mark the cell as `⚠️ Unverified — needs manual review`
- If the verify endpoint is unavailable → proceed without verification and add a note to the sheet header: _"Verification step skipped — manual review recommended"_

Tell the user: _"Validating [N] sources against their origin pages…"_

---

## Step 5 — Build the Google Sheet

Create a Google Sheet with two tabs: **Matrix** and **Evidence**.

### Tab 1: Matrix

- **Rows** = features
- **Columns** = your product + each competitor
- Each cell contains:
  - A verdict: **Yes**, **No**, or **Limited**
  - Followed by a brief explanation (1–2 sentences max) of what's there, what's missing,
    or what makes it limited
  - If no native capability exists but a workaround is known, add: _"No native feature —
    possible via [workaround]"_
  - If the swarm found no data: `⚠️ No data found`

**Cell verdict rules:**
- **Yes** — feature is fully supported natively
- **Limited** — feature exists but lacks something material (e.g. missing GIF support,
  requires paid tier, only available via integration)
- **No** — feature is absent; add workaround note if applicable

**Color coding** (apply to the cell background):
- Yes → green (`#B7E1CD`)
- Limited → yellow (`#FCE8B2`)
- No → red (`#F4CCCC`)
- No data → light grey (`#F3F3F3`)

### Tab 2: Evidence

Flat table with one row per source used, correlated back to the Matrix tab:

| Feature | Competitor | Verdict | Source URL | Snippet | Relevancy | Verified |
|---------|------------|---------|------------|---------|-----------|----------|

- Only include sources with relevancy **≥ 85%**
- `Snippet` = the exact text from the source that justifies the verdict
- `Verified` = ✅ if confirmed by the verify step, ⚠️ if low confidence, ❌ if failed
- Sources below 85% relevancy are silently excluded

---

## Step 6 — Deliver and offer change tracking

Share the Google Sheet link with the user.

Then ask:

_"Would you like to schedule a Sushidata swarm to monitor these sources for changes and
refresh the matrix automatically?"_

Options to offer:
- **Daily** — for fast-moving competitors
- **Weekly** — recommended default
- **Monthly** — for stable markets

If they say yes, hand off to `/schedule` to set it up.

---

## Step 7 — Next-step CTAs

After delivering the sheet, offer:

> 💬 **"Tell me everything about [Competitor]"** — Full account brief with signals and org chart
> 💬 **"Build a battlecard for [Competitor]"** — Positioning, objection handling, GTM signals
> 💬 **"Run a deeper matrix on [category]"** — Expand with more features
> 💬 **"Save this matrix"** — Persist to your context lake

---

## Rules

- Never hardcode the BASE_URL — always read it from `SETTINGS.md`
- Never show raw `doId` values to the user
- Primary sources must be official documentation / help centers — flag if only marketing content was found
- Only use sources with relevancy ≥ 85%
- Every verdict must be traceable to a source URL in the Evidence tab — no unsourced cells
- If the matrix has more than ~6 competitor columns, suggest splitting by category for readability
- **Always chunk large matrices into batches** — never launch more cells than the batch size table allows in a single `/matrix/create` call
- Each per-cell task must be scoped to one competitor + one feature — no bundling
