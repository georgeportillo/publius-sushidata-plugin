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

## Step 2 — Launch the matrix

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

Response includes `doIds` — an array of agent IDs, one per cell in the matrix.

Tell the user: _"Matrix launched — researching [N] cells across [X] competitors and [Y] features. This may take a few minutes…"_

---

## Step 3 — Poll for results

```
POST {BASE_URL}matrix/status
Content-Type: application/json

{ "doIds": ["<doId>", ...] }
```

> **Be patient — do not impose any self-imposed time limits.** The matrix deployment sends out parallel swarm workers and may take several minutes to fully dispatch and complete. Do **not** stop early, do **not** give up, and do **not** apply any internal timeout of your own. The only hard limit is **5 minutes of total wall time** — keep polling until `allDone` is `true` or that limit is reached. Treat slow or zero progress as completely normal.

- Poll every **10 seconds**
- Stop **only** when `allDone` is `true` or **5 minutes** have elapsed
- Show a progress update after each poll: _"⏳ X / N cells complete…"_
- If the 5-minute limit is reached before `allDone`, proceed with whatever results are available and note how many cells are still pending

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
