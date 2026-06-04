---
name: sushi-sequence
description: >
  Run a multi-step agent sequence where each step's output feeds the next.
  Trigger when the user says: "sequence", "run a sequence", "multi-step research",
  "step by step analysis", "run steps", "build a sequence", "sequence this",
  "staged research", "pipeline research", or any task that requires ordered,
  dependent research phases (step 2 builds on step 1's output). Also use when
  a research task is too broad for a single swarm and benefits from decomposition
  into sequential phases. Do NOT use for one-shot research — use sushi-research instead.
---

# Sushidata Sequence

A **sequence** is a multi-step agent pipeline where steps run serially and each step automatically inherits the summarized output from all prior steps. Use it when a research task requires ordered phases — e.g., first profile a market, then find companies in it, then find contacts at those companies.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

**Required header on all requests**: `Content-Type: application/json`

---

## Choose Your Mode

| Mode | When to use |
|------|-------------|
| **AI-planned** — `POST {BASE_URL}sequence/` | You describe the goal; an orchestrator LLM designs the steps |
| **Manual steps** — `POST {BASE_URL}sequence/steps/` | You define each step's prompts and output format explicitly |

Both modes return a `sequenceId`. Poll `POST {BASE_URL}sequence/status/` with that ID until complete.

---

## Mode 1 — AI-Planned Sequence

The endpoint takes your query, an orchestrator designs the step workflow, then launches execution.

```json
POST {BASE_URL}sequence/
{
  "query": "Build a full competitive intelligence package on Gong.io: market position, GTM motion, and key contacts at their top 5 competitors",
  "swarmSize": 5,
  "contextAnswers": []
}
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `query` | ✅ | Natural language research goal — be as specific as possible |
| `swarmSize` | No | Workers per step (2–20, default 5) |
| `contextAnswers` | No | Array of pre-answered context questions (see below) |

**`contextAnswers` shape** (optional, to guide the orchestrator):
```json
[
  {
    "questionId": "q1",
    "questionTitle": "What industry is the target in?",
    "answer": "B2B SaaS — sales intelligence",
    "skipped": false
  }
]
```

**Response:**
```json
{
  "sequenceId": "019...",
  "steps": [
    { "name": "Market Landscape", "outputSchemaPreset": "markdown", "swarmSize": 5 },
    { "name": "Competitor Profiles", "outputSchemaPreset": "autoTable", "swarmSize": 5 },
    { "name": "Key Contacts", "outputSchemaPreset": "findPeople", "swarmSize": 5 }
  ]
}
```

Show the user the planned steps before polling starts.

---

## Mode 2 — Manual Steps

You define each step explicitly. Use when you know exactly what output format and focus each step should have.

```json
POST {BASE_URL}sequence/steps/
{
  "name": "Gong Competitive Intelligence",
  "swarmSize": 5,
  "steps": [
    {
      "name": "Market Overview",
      "systemPrompt": "You are a B2B market analyst. Produce a structured overview of the sales intelligence software market.",
      "userPrompt": "Analyze the sales intelligence / revenue intelligence market: key players, estimated market size, growth drivers, and buyer segments.",
      "outputSchemaPreset": "markdown",
      "swarmSize": 5,
      "reasoningEffort": "medium"
    },
    {
      "name": "Competitor Battle Cards",
      "systemPrompt": "You are a competitive analyst. Use the market context from the previous step to build head-to-head battle cards against Gong.io.",
      "userPrompt": "Build battle cards for Gong.io vs Chorus, Clari, and Salesloft. Cover positioning, pricing, strengths, weaknesses, and objection handling.",
      "outputSchemaPreset": "battleCard",
      "swarmSize": 5
    }
  ]
}
```

**Top-level fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `steps` | ✅ | Array of step objects (1–10 steps) |
| `name` | No | Sequence display name |
| `swarmSize` | No | Default workers per step (2–20, default 5) — overridden per-step |

**Per-step fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `systemPrompt` | ✅ | The agent's role and context for this step |
| `userPrompt` | ✅ | The specific task for this step |
| `outputSchemaPreset` | ✅ | Output format (see presets below) |
| `name` | No | Display label for the step |
| `swarmSize` | No | Override workers for this step (2–20) |
| `reasoningEffort` | No | `low`, `medium` (default), or `high` |
| `maxSteps` | No | Max agent steps per worker (default 40) |
| `maxOutputTokens` | No | Max tokens per worker (default 30,000) |

**Response:**
```json
{
  "sequenceId": "019...",
  "steps": [ ... ]
}
```

---

## Output Schema Presets

| Preset | Best for |
|--------|----------|
| `markdown` | Default — prose reports, summaries, analysis |
| `autoTable` | Structured comparative data (feature matrices, pricing tables) |
| `battleCard` | Head-to-head competitive battle cards |
| `ciClaimsMap` | FUD / claims analysis with severity ratings |
| `findPeople` | Listing people with names, titles, companies, LinkedIn profiles |
| `dynamicChart` | Data visualization (Apache ECharts) — runs single agent, no swarm |
| `socialCards` | Social media carousel content |

> When in doubt, use `markdown`. Use `autoTable` only when output is genuinely structured tabular data.
> `dynamicChart` steps automatically run without a swarm regardless of `swarmSize`.

---

## Step 3 — Polling

After deploying, poll every 10–15 seconds until `status === "complete"` or `status === "error"`.

```json
POST {BASE_URL}sequence/status/
{ "sequenceId": "<id from deploy response>" }
```

**Response shape:**
```json
{
  "status": "running",
  "sequenceId": "019...",
  "currentStepIndex": 1,
  "totalSteps": 3,
  "stepStates": [
    { "status": "complete" },
    { "status": "running" },
    { "status": "pending" }
  ],
  "currentStepProgress": {
    "total": 5,
    "completed": 3,
    "hasErrors": false
  },
  "message": "Workers running..."
}
```

**Step statuses (in order):** `pending` → `orchestrating` → `running` → `summarizing` → `complete` / `error`

**Sequence statuses:** `running` | `complete` | `error`

**When complete**, the response includes `publicUrl`:
```json
{
  "status": "complete",
  "sequenceId": "019...",
  "publicUrl": "/public/019..."
}
```

**Polling rules:**
- Poll every 10–15 seconds — not faster
- Show progress to the user: _"Step 2 of 3 running — 3 of 5 workers done..."_
- Hard wall time limit: **10 minutes per step** (the server will self-timeout stale steps)
- If the entire sequence takes more than 30 minutes, surface whatever is available and note which steps are incomplete
- On `status: "error"`, surface the `error` message to the user and stop polling

---

## Full Workflow

```
User requests a sequence
        │
        ▼
Choose mode:
  AI-planned → POST {BASE_URL}sequence/
  Manual     → POST {BASE_URL}sequence/steps/
        │
        ▼
Receive sequenceId + planned steps
Show user the step plan
        │
        ▼
Poll POST {BASE_URL}sequence/status/ every 10–15s
Show step-by-step progress
        │
        ▼
status === "complete"
        │
        ▼
Surface publicUrl to user
Save outputs to context lake → POST {BASE_URL}context/
```

---

## When to Use Sequences vs Single Swarms

| Use sequences when... | Use swarm instead when... |
|-----------------------|--------------------------|
| Step 2 needs step 1's output as input | All research can happen in parallel |
| Research has natural phases (profile → find → enrich) | Single-focus research question |
| Output format changes per phase (table, then prose, then people list) | One type of output is sufficient |
| Task has 2–4 distinct, ordered objectives | A single comprehensive swarm can cover it |

---

## Example Sequences

### Competitive Intelligence Pipeline
1. `markdown` — Market landscape and key player summary
2. `autoTable` — Feature-by-feature comparison matrix
3. `battleCard` — Battle cards for top 3 competitors

### ICP Research → Contacts Pipeline
1. `markdown` — Profile the ICP segment and identify 5–10 target companies
2. `findPeople` — Find VP Sales / CRO contacts at each company

### Market Signal Pipeline
1. `markdown` — Research recent funding + hiring signals in the target space
2. `ciClaimsMap` — Map competitor claims and positioning weaknesses
3. `markdown` — Synthesize into a prioritized outreach strategy

---

## Rules

- Always show the step plan to the user before polling begins
- Mention that step N will automatically receive step N-1's summarized output as context — Claude doesn't need to pass it manually
- The server enforces a 10-minute per-step stale timeout; you do not need to implement your own
- After a sequence completes, save the outputs to the context lake with `POST {BASE_URL}context/`
- Always provide the `publicUrl` to the user when the sequence completes — they can share or view the full output there
