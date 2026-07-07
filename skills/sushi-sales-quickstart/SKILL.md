---
name: sushi-sales-quickstart
description: 'Run a live GTM sales demo using Sushidata. Use when the user wants to see Sushidata in action, get their first leads, run a quick demo, find contacts for an ICP, build a prospect list, research a target account, or get started with outbound. Triggers: "show me what Sushidata can do", "find me some leads", "get started", "quick demo", "build a prospect list", "research this company for me", "who should I be selling to". Default to Recipe 1 if no context is given.'
---

# Sushidata Sales Quickstart

A live demo of Sushidata's GTM capabilities — runs in under 5 minutes and delivers real results the user can act on immediately. Every finding is saved to the Sushidata context lake so nothing is lost between sessions.

**Always pick the most relevant recipe based on what the user said.** Default to Recipe 1 if no context is given.

---

## Before starting any recipe

If the user hasn't told you what they sell and who they sell to, ask:

> 🎯 **Quick question before we start:** What does your company sell, and who's your ideal buyer?
> *(e.g. "We sell contract management software to VP Legal at Series B+ companies")*
>
> I'll tailor the results to your ICP — or:
>
> 💬 **"Just show me"** — Run a generic demo with a default ICP right now

If they don't answer or say "just show me," proceed with Recipe 1 using the default ICP.

---

## Execution pattern (all recipes)

1. **Tell the user what you're about to do** — explain the goal, data sources, and what they'll get at the end.
2. **Save the request to the context lake** — always save the user's ask before starting.
3. **For each step**: narrate what's happening, run it, show partial results as they come in.
4. **Save all results to the context lake** — every company found, every contact discovered, every email verified.
5. **Show the final table** — clean, scannable, with next-step options offered.

---

## Recipe 1 — Find your first 10 qualified leads (default)

**Goal:** Discover 10 companies matching the user's ICP, find the right buyer at each, verify their email, and offer HeyReach activation.

**Data sources:** Sushidata research swarm → FullEnrich people search → FullEnrich email enrichment

**Steps:**

### Step 1 — Save the request

```json
POST /context/
{
  "serverId": "26",
  "content": "Sales quickstart — Recipe 1. User ICP: {{what the user told you, or 'not specified'}}. Goal: find 10 qualified leads with verified emails.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 2 — Discover 10 ICP-matching companies

Deploy a Sushidata research swarm:

```json
POST /swarm/deploy/
{
  "query": "Find 12 companies that match this buyer profile: {{ICP description — product category, buyer persona, company size, vertical, geography}}. For each company: company name, domain, headcount estimate, why they fit the ICP, 1-2 specific signals (hiring, tech stack, pain language, compliance). Do NOT include companies in the Fortune 500 or any company already mentioned.",
  "swarmSize": 6
}
```

Poll `/swarm/status/` every 30 seconds. Show progress: "✅ 3 / 6 workers done..."

Once complete, synthesize results directly from the `output` fields of completed workers collected during polling — parse out the list of companies from those outputs. Then run `/verify/` on any source URLs before presenting.

Show the user a quick preview: "Found 12 candidates — filtering to the 10 strongest fits..."

### Step 3 — Find buyer contacts via FullEnrich

For each of the 10 companies, search for contacts via FullEnrich:

```
fullenrich_search_people current_company_domains=[{"value":"{{domain}}"}]
```

Filter results by the buyer persona title from the ICP. If FullEnrich returns 0 contacts for a company, note it and move on — do not retry with alternative providers. Over-provision at Step 2 means you have spares.

### Step 4 — Verify emails

For each contact with a found email:

```
# Spot-check FullEnrich confidence scores before outbound activation
fullenrich_get_enrichment enrichment_id={{enrichment_id}}
```

Mark emails with non-send status (`invalid`, `accept_all`, `webmail`, `disposable`) as "(unverified — use with caution)" in the output. Do not drop them — let the user decide.

### Step 5 — Save all results to the context lake

```json
POST /context/
{
  "serverId": "26",
  "content": "Sales quickstart results — Recipe 1 complete. Companies found: {{list of domains}}. Contacts discovered: {{count}}. Verified emails: {{count}}. ICP used: {{ICP description}}. Full CSV available at: {{output path if saved}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 6 — Show results + offer next steps

Display a clean table:

| Company | Domain | Contact | Title | Email | Status |
|---------|--------|---------|-------|-------|--------|
| ... | ... | ... | ... | ... | ✅ Verified |

Then offer:

**What would you like to do next?**

> 💬 **"Add these contacts to HeyReach"** — Launch a LinkedIn outbound campaign right now
> 💬 **"Write personalized emails for these contacts"** — Custom first line per contact based on live research
> 💬 **"Tell me everything about [Company Name]"** — Full account brief: signals, org chart, outreach angle
> 💬 **"Save to HubSpot"** — Create contacts and a deal pipeline in your CRM
> 💬 **"Save these results"** — Persist to your context lake for future sessions

---

## Recipe 2 — Deep-dive a target account

**Goal:** Build a complete account brief for one company — org chart, key personas, pain signals, personalized outreach angle — ready to hand to an AE in under 3 minutes.

**Data sources:** Sushidata research swarm → FullEnrich people search → Apify LinkedIn scraper

**When to use:** User names a specific company ("tell me everything about Acme Corp"), preparing for a discovery call, or following up on Recipe 1 results.

**Steps:**

### Step 1 — Save the request

```json
POST /context/
{
  "serverId": "26",
  "content": "Sales quickstart — Recipe 2. Target account: {{company name / domain}}. Goal: full account brief with org chart, pain signals, and outreach angle.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 2 — Research the company

Deploy a Sushidata swarm:

```json
POST /swarm/deploy/
{
  "query": "Research {{company name or domain}} in depth. Cover: (1) what they sell and to whom, (2) company size and funding stage, (3) tech stack and integrations, (4) recent news, hiring signals, or product changes in the last 90 days, (5) pain points they're likely experiencing based on their job postings and website, (6) who the likely economic buyer and champion would be for {{seller's product category}}, (7) any compliance or regulatory signals. Be specific — include quotes from job postings or website content where relevant.",
  "swarmSize": 6
}
```

Poll and show progress. Present the synthesized research after `/verify/`.

### Step 3 — Find the org chart via FullEnrich

```
fullenrich_search_people current_company_domains=[{"value":"{{domain}}"}]
```

Group contacts by seniority (C-suite, VP, Director, Manager). Flag the likely economic buyer and champion based on the research from Step 2.

### Step 4 — LinkedIn supplement (if FullEnrich returns <5 contacts)

Sushidata does not currently expose a LinkedIn employee-list Apify actor. Use WebSearch, Browser Rendering, and focused Sushidata swarms to identify likely personas and validate names. If bulk LinkedIn employee scraping is required, follow the missing-actor feedback workflow in `sushi-research/provider-playbooks/apify.md`.

### Step 5 — Save to context lake

```json
POST /context/
{
  "serverId": "26",
  "content": "Account brief complete — {{company name}}. Key findings: {{2-3 sentence summary of top signals and buyer profile}}. Contacts found: {{list of names, titles, emails}}. Research saved for future retrieval.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 6 — Present the account brief

Structure the output as:

**🏢 {{Company Name}} — Account Brief**

- **What they do:** ...
- **Size / Stage:** ...
- **Top buying signal:** ...
- **Key pain (for your pitch):** ...
- **Who to target:** Name, Title, Email
- **Personalized first line:** ...
- **Talk track angle:** ...

**What would you like to do next?**

> 💬 **"Write a full outreach sequence for [Name]"** — Personalized email + LinkedIn cadence based on the research
> 💬 **"Add [Name] to HeyReach"** — Launch a LinkedIn outbound sequence now
> 💬 **"Save to HubSpot"** — Create a contact and deal in your CRM
> 💬 **"Tell me everything about [another company]"** — Run this recipe on a different target account
> 💬 **"Save these results"** — Persist the account brief to your context lake

---

## Recipe 3 — Validate ICP against competitor's customers

**Goal:** Find known customers of a named competitor and check whether they match your ICP — surfaces displacement opportunities and migration angles.

**When to use:** User names a competitor ("who uses Competitor X?"), wants to validate ICP assumptions, or is building displacement messaging.

**Steps:**

### Step 1 — Save the request

```json
POST /context/
{
  "serverId": "26",
  "content": "Sales quickstart — Recipe 3. Competitor: {{competitor name}}. Goal: find their known customers, check ICP fit, surface displacement opportunities.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 2 — Find known customers of the competitor

Deploy a Sushidata swarm:

```json
POST /swarm/deploy/
{
  "query": "Find 15 companies that are known customers or users of {{competitor name}}. Sources to check: G2 reviews, Capterra, case studies on their website, customer logos, LinkedIn posts, job postings mentioning the tool, press releases. For each company: name, domain, evidence of competitor usage (quote + source URL), estimated headcount, vertical.",
  "swarmSize": 8
}
```

Run `/verify/` on all evidence URLs. Drop any company where the evidence link doesn't confirm competitor usage.

### Step 3 — Score ICP fit for each confirmed company

For each verified company, ask: does this company match the user's ICP profile (size, vertical, pain, tech maturity)? Score as Tier 1 / Tier 2 / Tier 3 based on ICP criteria.

### Step 4 — Find displacement contacts via FullEnrich

For Tier 1 companies only, run `fullenrich_search_people` to find the buyer persona contacts. These are your highest-priority outreach targets — they have confirmed budget (they're paying a competitor), confirmed pain (they're in the category), and you have a migration angle.

### Step 5 — Save everything to context lake

```json
POST /context/
{
  "serverId": "26",
  "content": "Recipe 3 complete — Competitor displacement for {{competitor}}. Companies found using competitor: {{count}}. Tier 1 ICP matches: {{list of domains}}. Contacts found: {{count}}. Evidence sources verified: {{count}}. Displacement angle: {{1-sentence summary}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

### Step 6 — Present results + displacement messaging

Show a table of Tier 1 companies with competitor evidence and contacts. Then offer:

> **Displacement angle:** "You're paying for {{competitor}} — here's why teams like yours switch to {{your product}}: {{top migration reason from research}}."

**What would you like to do next?**

> 💬 **"Write a displacement email sequence"** — Draft a tailored sequence for switching away from [competitor]
> 💬 **"Add Tier 1 to HeyReach"** — Launch outbound for your highest-priority displacement targets
> 💬 **"Tell me everything about [Company Name]"** — Build a full account brief on any match
> 💬 **"Save these results"** — Persist the displacement list to your context lake

---

## Fallback (if a swarm returns no usable results)

Tell the user what happened, then try a direct `POST /query/` against the context lake first — results from previous sessions may already answer the question. If the lake also returns nothing useful, re-deploy the swarm with `swarmSize` doubled and a more specific query.

If two attempts fail, tell the user and suggest they provide more ICP context or try `/sushi-research` for a fully custom workflow.

---

## Context lake rule

Every recipe saves twice:
1. **Before starting** — the request, ICP, and goal (so future sessions know what was asked)
2. **After completing** — the full results summary (so future sessions can retrieve the findings without re-running)

Never skip the saves. The context lake is what makes Sushidata useful across sessions — without saves, every session starts from zero.

---

## Sushidata commands

Remind the user of these commands they can use anytime:

> 🍣 **Sushidata commands — use anytime:**
>
> 💬 **"save"** — save this session’s outputs to your context lake
> 💬 **"restore"** — pull prior session memory back at the start of any new session
> 💬 **"savings"** — see what Sushidata retrieved vs what Claude built this session
> 💬 **"help"** — see everything Sushidata can do
