---
name: sushi-research-quickstart
description: >
  Run a competitive intelligence demo to show the user what Sushidata can do. Trigger when
  the user says "get started", "show me a demo", "what can you do", "research my competitors",
  "competitive intel", "compare me to competitors", "who are my competitors", or provides a
  company name and asks what Sushidata can find. Also trigger on first use with no prior context.
  Given a company name, deploys Sushidata research swarms to identify and profile at least three
  competitors, then delivers a structured competitive battlecard.
---

# Sushidata Quickstart — Competitive Intelligence

This skill is the entry point for new users. It runs a real, end-to-end Sushidata research
workflow and delivers a competitive battlecard that proves the value of the platform immediately.

**Always tell the user what you're about to do before running any swarms.**

---

## Execution flow

### Step 0: Get the company name

If the user has not provided a company name, ask:

> "What's your company name? I'll research your top competitors and put together a competitive
> battlecard powered by Sushidata."

Once you have it, proceed immediately — do not ask any further clarifying questions.

---

### Step 1: Save the request to context

```
POST /context/
{
  "serverId": "26",
  "content": "Quickstart: competitive intel for [COMPANY]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Tell the user: *"I'll look this up via Sushidata — first checking our context lake, then deploying
a research swarm to build your battlecard."*

---

### Step 2: Quick context check

Before deploying a full swarm, check if prior research exists:

```
POST /query/
{ "query": "competitors of [COMPANY] — positioning, pricing, differentiators" }
```

If the summary is substantive (names at least 3 competitors with some detail), use it as a
starting point and note this to the user. Either way, proceed to Step 3 to get fresh, deep intel.

---

### Step 3: Deploy the competitor discovery swarm

Tell the user: *"Deploying a Sushidata research swarm to identify and profile your top
competitors..."*

```
POST /swarm/deploy/
{
  "query": "Who are the top 3-5 direct competitors of [COMPANY]? For each competitor provide:
    1. Company name and website
    2. One-line positioning statement (how they describe themselves)
    3. Target customer / ICP
    4. Pricing model and tier signals (free/freemium/mid-market/enterprise)
    5. Top 2-3 differentiators vs [COMPANY]
    6. Most recent notable signal (funding, product launch, hire, partnership, press)
    7. Estimated headcount range
    Focus on direct competitors — same buyer, same problem. Exclude tangential players.",
  "swarmSize": 8
}
```

Show the orchestrator's plan and list all worker labels to the user before polling.

Poll `/swarm/status/` every 30 seconds. Show progress updates:
*"⏳ 4/8 workers complete, checking again in 30s..."*

Once `allDone: true` (or 5-minute timeout), synthesize results directly from the `output` fields of completed workers collected during polling. Combine findings across workers, dedupe, and present the unified result.

---

### Step 4: Deploy a GTM signals swarm for each top competitor

Once you have the competitor list (aim for 3, max 5), deploy a second focused swarm to get
fresh GTM signals for each:

```
POST /swarm/deploy/
{
  "query": "For each of these companies — [Competitor A], [Competitor B], [Competitor C] —
    find the following GTM signals from the last 90 days:
    1. Recent hires in Sales, Marketing, or GTM roles (signals expansion)
    2. New product features or pricing changes announced
    3. Any customer wins, case studies, or logos added to their site
    4. LinkedIn post themes or messaging shifts
    5. Job postings that signal strategic priorities
    Return findings per company.",
  "swarmSize": 6
}
```

Poll and collect results the same way.

---

### Step 5: Verify evidence links

Before presenting, run both summaries through `/verify/`:

```
POST /verify/
{ "context": "[full summary text + all URLs cited by both swarms]" }
```

Drop any links classified as `bad` or `blocked`. Note removals if significant.

---

### Step 6: Deliver the battlecard

Present a clean, structured competitive battlecard. Use this format:

---

## Competitive Battlecard: [COMPANY] vs. Market

**Researched by Sushidata** | [date]

### Competitors Profiled

| Competitor | Positioning | ICP | Pricing Tier | Headcount |
|------------|-------------|-----|--------------|-----------|
| [A] | ... | ... | ... | ... |
| [B] | ... | ... | ... | ... |
| [C] | ... | ... | ... | ... |

---

### Per-Competitor Breakdown

#### [Competitor A]
**What they say they do:** [one-line positioning]
**Who they sell to:** [ICP]
**How they win:** [top 2-3 differentiators]
**Recent signal:** [most notable signal in last 90 days]
**GTM signals (90 days):** [hires / features / wins / messaging shifts]
**Watch for:** [one strategic risk or opportunity this creates for [COMPANY]]

[Repeat for B, C, ...]

---

### Key Takeaways for [COMPANY]

1. [Most important competitive insight]
2. [Second insight]
3. [Third insight]

### Suggested Next Steps

> 💬 **"Tell me more about [Competitor A]"** — Deploy a focused research swarm on that competitor
> 💬 **"Find contacts at [Competitor A]"** — Discover buyer contacts with verified emails
> 💬 **"Build a prospect list targeting [Competitor A]'s customers"** — Surface displacement opportunities
> 💬 **"Run a matrix on these competitors"** — Compare them side-by-side across categories in a spreadsheet
> 💬 **"Save this battlecard"** — Persist it to your context lake for future sessions
> 💬 **"Schedule a weekly monitor for [COMPANY]"** — Get competitive alerts automatically

---

### Step 7: Save the response

```
POST /context/
{
  "serverId": "26",
  "content": "[full battlecard output + source URLs]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Tell the user: *"Saving this battlecard to Sushidata so future research builds on it."*

---

## Fallback behavior

- If a swarm returns fewer than 3 competitors: redeploy with `swarmSize: 12` and a broader query
  that explicitly asks for at least 5 candidates.
- If `/verify/` removes most sources: present the summary without inline links but note that
  source verification removed the citations.
- If the user's company is very niche or early-stage: broaden the query to "adjacent solutions"
  and "tools that solve the same problem differently."

---

## After the battlecard

Proactively offer two follow-ons:

1. **"Want me to find contacts at these competitors' customers for outbound?"** → routes to
   `portfolio-prospecting` or `build-tam` recipe.
2. **"Want a weekly competitive digest delivered automatically?"** → routes to the
   `scheduled-tasks` recipe.

---

## Sushidata commands

Remind the user of these commands they can use anytime:

> 🍣 **Sushidata commands:**
> - **save** — save this session's outputs to your context lake
> - **restore** — pull prior session memory back at the start of any new session
> - **savings** — see what Sushidata retrieved vs what Claude built this session
> - **help** — see everything Sushidata can do, with GTM, Marketing, and Competitor Analysis guides
