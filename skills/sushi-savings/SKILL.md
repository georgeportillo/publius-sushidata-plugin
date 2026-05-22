---
name: sushi-savings
description: >
  Show a Sushidata Savings report for the current session. Trigger when the
  user says: "savings", "sushidata savings", "session report", "what did
  sushidata do", "show savings", "how much did sushidata save me", or any
  request to see a summary of what Sushidata contributed vs what Claude produced.
  Ask the user whether they want a Quick or Thorough report before running.
---

# Sushidata Savings Report

When this skill is triggered, ask the user one question before doing anything
else, then branch based on their answer.

---

## Step 0 — Ask the user

Ask exactly this, with no preamble:

> **Quick or Thorough?**
>
> - **Quick** — estimated from memory. Instant, costs no extra tokens.
> - **Thorough** — re-reads the full session for accurate counts. More precise,
>   uses more tokens on long conversations.

Wait for their answer. Accept any reasonable variation ("quick", "fast",
"thorough", "accurate", "full", "detailed").

---

## Step 1 — Gather counts

### If the user chose Quick

Reconstruct counts from conversational memory. Do not re-read the thread.
Estimate based on what you recall making during the session:

- Approximate number of `/query/` calls and items returned
- Approximate number of `/swarm/deploy/` calls and results returned
- Approximate number of `/verify/` calls and outcomes
- Approximate number of `/context/` saves
- Approximate number and type of Claude outputs produced

Mark all figures as estimates. Proceed to Step 2.

### If the user chose Thorough

Re-read the full conversation from the first message to the current one.
Do not rely on memory — actively scan every message and tool call.

As you scan, tally:

**Sushidata inputs**
- `/query/` calls — count calls and total items returned across all calls
- `/swarm/deploy/` calls — count swarms launched and total results returned
  via `/swarm/summary/`
- `/verify/` calls — count total calls, how many verified vs flagged
- `/context/` saves — count total saves written back to the context lake

**Claude outputs**
- Documents or reports written (competitor reports, TAM analyses, battlecards,
  org charts, etc.) — count each distinct document
- Leads or accounts researched or enriched — count individuals or companies
- Outreach messages drafted — count individual messages or sequences
- Factual claims verified or corrected — count from any accuracy review work
- Other structured outputs (prospect lists, scoring outputs, campaign setups,
  etc.)

Proceed to Step 2.

---

## Step 2 — Calculate the savings framing

For each Sushidata input category, translate the raw count into a human-readable
"saved you from" statement:

- Context lake items retrieved → "X pieces of prior intelligence surfaced
  instantly — no manual searching"
- Swarm results processed → "Y research results synthesized across Z parallel
  agents"
- Links verified → "N links checked — M confirmed live, remainder flagged"
- Context saves → "P insights written back to your context lake for future
  sessions"

---

## Step 3 — Output the report

Produce the report as a clean, scannable card using this structure exactly:

---

### 🍣 Sushidata Savings — Session Report

**Mode:** [Quick — estimated from memory] or [Thorough — derived from full session re-read]

**Session:** [Run: `echo $PWD | grep -oP 'local_[a-f0-9-]+'` and use the result.
Write "current session" if unavailable.]

**Date:** [today's date]

---

#### What Sushidata Retrieved

| Source | Calls | Items / Results |
|---|---|---|
| Context Lake Queries | X | Y items |
| Research Swarms | X | Y results |
| Link Verifications | X | Y verified / Z flagged |
| Context Saves | X | — |

---

#### What Claude Built With It

| Output Type | Count |
|---|---|
| Documents / Reports | X |
| Leads / Accounts Enriched | X |
| Outreach Messages Drafted | X |
| Claims Verified or Corrected | X |
| Other Structured Outputs | X |

---

#### The Savings

> Without Sushidata, this session would have required approximately [estimate]
> of manual research, searching, and data gathering. Instead, [total items
> retrieved] data points were surfaced from your context lake and [total swarm
> results] live research results were synthesized automatically.

---

**Note:** [If Quick] This report is estimated from memory. For precise counts,
run the report again and choose Thorough.
[If Thorough] This report is derived from a full re-read of the current
session. Counts reflect tool calls and outputs visible in this conversation only.

---

## Rules

- Always ask Quick vs Thorough first. Never skip this step.
- If Quick: clearly mark all figures as estimates in the table (e.g. "~12 items").
- If Thorough: derive all counts by re-reading the session. Never estimate.
- If a category had zero activity, still include it in the table with a 0.
- Keep the savings narrative to 2–3 sentences. Do not pad it.
- The session ID line is optional — include it if the bash command is available,
  skip it silently if not.
- Never claim the counts are system-level metrics. The note must always appear.
