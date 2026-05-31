---
name: sushi-matrix
description: >
  Build a competitive research matrix — a spreadsheet comparing competitors across
  categories, with real evidence per cell. Trigger when the user says: "matrix",
  "sushi matrix", "build a competitive matrix", "compare competitors", "research matrix",
  "competitor comparison", or any request to compare multiple companies or products
  across a set of criteria.
---

# Sushidata Matrix

When triggered, ask the user what they want to analyze, launch the matrix, and render the
results as a formatted spreadsheet with evidence clearly shown in every cell.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**.

---

## Step 1 — Elicit what to analyze

Ask the user for the following. Keep it brief — one message, all fields:

1. **Competitors** — which companies or products to compare (up to 20)
2. **Categories** — what dimensions to research (e.g. Pricing, Integrations, Support)
   - For each category, optionally ask for subcategories (specific questions within it)
3. **Reasoning effort** — `low` (fast), `medium` (default), or `high` (thorough)

If the user provides enough context in their trigger message (e.g. "compare Salesforce, HubSpot, and Pipedrive on pricing and integrations"), skip the form and proceed directly.

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

Tell the user: _"Matrix launched — researching [N] cells across [X] competitors and [Y] categories. This may take up to 60 seconds…"_

---

## Step 3 — Poll for results

```
POST {BASE_URL}matrix/status
Content-Type: application/json

{ "doIds": ["<doId>", ...] }
```

- Poll every **3 seconds**
- Stop when `allDone` is `true` or **60 seconds** have elapsed
- Show a progress update after each poll: _"⏳ X / N cells complete…"_
- If the 60-second limit is reached before `allDone`, proceed with whatever results are available and note how many cells are still pending

---

## Step 4 — Render the spreadsheet

Build a Markdown table where:

- **Rows** = competitors
- **Columns** = subcategories (or categories if no subcategories were given)
- Each cell shows `result` from that agent's output, followed by evidence links on a new line

**Column layout:**

| Competitor | [Subcategory 1]        | [Subcategory 2]        | …   |
| ---------- | ---------------------- | ---------------------- | --- |
| Company A  | Finding. [source](url) | Finding. [source](url) | …   |

**Evidence rules:**

- Include every `url` from `stepsAndSources` as a linked `[linkText](url)` inline below the result
- Only include URLs from entries where `url` is non-null
- If a cell has no evidence, write `—`
- If a cell's status is `errored` or `unknown`, write `⚠️ No data`

**After the table**, add a brief summary paragraph highlighting the most significant differences across competitors. Then offer next-step CTAs:

---

**What would you like to do next?**

> 💬 **"Tell me everything about [Competitor Name]"** — Full account brief with signals, contacts, and org chart
> 💬 **"Find contacts at [Competitor Name]"** — Discover and verify buyer emails
> 💬 **"Build a battlecard for [Competitor Name]"** — Positioning, objection handling, and GTM signals
> 💬 **"Run a deeper matrix with [additional categories]"** — Expand the comparison with more dimensions
> 💬 **"Save this matrix"** — Persist the results to your context lake

---

## Rules

- Never hardcode the BASE_URL — always read it from `SETTINGS.md`
- Never show raw `doId` values to the user
- If the matrix has more than ~6 columns, break it into multiple tables by category for readability
- Always show evidence — a matrix with no sources is not useful
