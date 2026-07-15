---
name: sushi-signals
description: >
  Scan LinkedIn posts, comments, and other sources for buying signals from ICP accounts and prospects.
  Triggers: "check signals", "run signals", "find signals", "signal scan", "buying signals",
  "linkedin signals", "sushi signals", "who just moved jobs", "who's hiring",
  "find pain signals", "prospect signals", "signal detection", "run my signals",
  or any request to detect job change, pain, or hiring signals from online activity.
  Run as a scheduled event (Monday–Friday, 9:00 AM) to surface fresh pro insights before outreach.
---

# Sushidata Signal Detection — LinkedIn Buying Signals

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

**Schedule:** Monday–Friday, 9:00 AM (customer local time)
**Target output:** Ranked list of prospects mapped to their detected buying signal, ready for outreach

Two phases:

1. **Signal Config** (Phase 1) — one-time setup capturing signal history, source preferences, intelligence goals, and classification model.
2. **Signal Scan** (Phase 2) — runs on every subsequent invocation using saved config to surface fresh signals.

---

## Dependencies & Tools

| Capability | Source |
|---|---|
| Context lake CRUD | `sushi-research` SKILL → `/context/`, `/query/`, `/swarm/deploy/` |
| LinkedIn post search | `apify_linkedin_post_search` (via `provider-playbooks/apify.md`) |
| Lead enrichment | `apify_leads_finder` (via `provider-playbooks/apify.md`) |
| Reddit scraping | `apify_reddit_scraper` or swarm web search (via `provider-playbooks/apify.md`) |
| News & press | Swarm web search via `sushi-research` |
| Session persistence | `sushi-save` skill |

---

## Elicitation Forms — Mandatory for All User Input

**CRITICAL:** All configuration questions MUST be collected via the `askQuestions` elicitation tool — never as plain-text questions in the chat.

### Rules

1. **Every question to the user must go through `askQuestions`** — no exceptions during config.
2. **Use `options` with predefined choices** wherever the answer set is known.
3. **Use `multiSelect: true`** for questions where multiple answers are valid.
4. **Always include a skip/decline option** so users aren't forced to answer.
5. **Use `allowFreeformInput: true`** (default) when users may want custom answers.
6. **Set `allowFreeformInput: false`** only for strict yes/no or fixed-choice questions.
7. **Group related questions in a single `askQuestions` call** — batch up to 5 per form.
8. **Use `message` field for context** — brief explanatory markdown below each header.
9. **Use `recommended: true`** on the most common/expected option.

### Form Patterns by Step

Below are the exact elicitation form structures to use for each config step. Render these via `askQuestions` — do NOT convert them to blockquote text or plain chat messages.

---

## Generative UI — Output Standards

### Phase Detection Banner

On every invocation, show one of these before doing anything else:

```
## 📡 Sushidata Signal Detection

**Status:** 🟢 Config loaded — running signal scan
```

or

```
## 📡 Sushidata Signal Detection

**Status:** ⚙️ No config found — running first-time setup (one time only)
```

### Config Progress Tracker

Display and update after each config step in Phase 1:

```
## ⚙️ Signal Config

| Step | Status | Description |
|:---:|:---:|---|
| 1 | ✅ | Signal History |
| 2 | ⏳ | Signal Sources |
| 3 | ⬜ | Intelligence Goals |
| 4 | ⬜ | Signal Classification |
```

Use ✅ (complete), ⏳ (in progress), ⬜ (pending). Update after each step.

### Signal Report Card

One card per matched prospect in Phase 2:

```
---

### 👤 Jane Smith — VP Revenue · Acme Corp

| Field | Value |
|---|---|
| **Signal Type** | 🔥 Motivated Buyer |
| **Signal** | Pain |
| **Evidence** | "We're drowning in manual reporting — open to suggestions" |
| **Source** | LinkedIn post · July 14, 2026 |
| **LinkedIn** | [Profile URL] |
| **Email** | j.smith@acme.com ✅ |
| **Recommended Action** | Lead with reporting automation angle · reach out within 24 hrs |

---
```

Signal type badges: 🔥 Motivated Buyer · 🟢 Buying Signal · 🟡 Qualification Signal · 🔵 Watch & Wait · ⚪ Cold Lead

---

## Phase Detection

On every invocation:

1. Query the context lake: `POST {BASE_URL}query/` with `{ "query": "signal_config sources classification goals worked not_worked" }`
2. If a saved `signal_config` entry is found → skip to **Phase 2**.
3. If missing → run **Phase 1** starting from Step 1.

**Never ask the user which phase to run.** Detection is automatic.

---

# PHASE 1: SIGNAL CONFIG

> Runs once on first invocation. Saves config to the context lake for all future runs.

---

## Step 1 — Signal History

Collect what has and hasn't worked before touching any external sources.

**Elicitation Form — Signal History:**

```
askQuestions([
  {
    header: "Signals that worked",
    question: "What types of signals have led to real conversations or closed deals for you?",
    message: "Select all that apply. These will be prioritized in every scan.",
    multiSelect: true,
    options: [
      { label: "Job change — prospect moved to a new company or role", recommended: true },
      { label: "Pain post — prospect publicly described a struggle or asked for help" },
      { label: "Hiring signal — company posted a role that implies budget or initiative" },
      { label: "Funding announcement — company raised a new round" },
      { label: "Competitor complaint — prospect vented about a competitor product" },
      { label: "Tool evaluation post — prospect asked for vendor recommendations" },
      { label: "Event / conference mention — prospect attended a relevant event" },
      { label: "None of these — I'll describe my own" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Signals that did NOT work",
    question: "What signals have you chased that turned out to be cold or wasted effort?",
    message: "These will be filtered out or deprioritized in your scan results.",
    multiSelect: true,
    options: [
      { label: "Generic company news (awards, press releases)" },
      { label: "Content engagement (likes, reposts) without direct intent" },
      { label: "Job change into an irrelevant role" },
      { label: "Early-stage funding (pre-seed / seed) — too early to buy" },
      { label: "Hiring for non-buyer roles (engineering, design)" },
      { label: "Thought leadership posts without a pain hook" },
      { label: "None — I haven't found a pattern yet" }
    ],
    allowFreeformInput: true
  }
])
```

---

## Step 2 — Signal Sources

**Elicitation Form — Source Preferences:**

```
askQuestions([
  {
    header: "Where should signals come from?",
    question: "Which sources matter most for finding your buyers?",
    message: "Select all that apply. I'll weight the scan toward the sources your ICP actually uses.",
    multiSelect: true,
    options: [
      { label: "LinkedIn posts — public posts from prospects and companies", recommended: true },
      { label: "LinkedIn comments — comments left on industry posts" },
      { label: "LinkedIn job postings — new roles that signal budget or initiative" },
      { label: "Reddit — subreddit posts and comments in your niche" },
      { label: "News & press releases — funding, acquisitions, product launches" },
      { label: "Twitter / X — public posts and threads" },
      { label: "G2 / review sites — reviews mentioning pain or switching intent" },
      { label: "Hacker News — Show HN posts, Ask HN threads" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Specific communities",
    question: "Are there specific subreddits, Slack communities, or forums where your buyers hang out?",
    message: "Examples: r/sales, r/startups, r/devops — or a specific Slack/Discord community name.",
    allowFreeformInput: true
  }
])
```

---

## Step 3 — Intelligence Goals

**Elicitation Form — What to Understand:**

```
askQuestions([
  {
    header: "What should I extract from each signal?",
    question: "When I detect a signal, what do you most want to understand about the prospect?",
    message: "These become the intelligence dimensions shown in every signal report card.",
    multiSelect: true,
    options: [
      { label: "Active pain — what problem are they struggling with right now", recommended: true },
      { label: "Broader need — what initiative or goal is driving their search" },
      { label: "Budget signals — evidence they have money to spend" },
      { label: "Timing — how urgent is this (quarter end, new hire, post-funding)" },
      { label: "Competitive context — are they evaluating other vendors" },
      { label: "Stakeholder map — who else is involved in the decision" },
      { label: "Objections — what concerns or blockers are visible in the signal" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Keywords to always flag",
    question: "Are there specific phrases, pain keywords, or competitor names I should always surface?",
    message: "Examples: 'manual reporting', 'outgrowing HubSpot', 'scaling our SDR team', a competitor name.",
    allowFreeformInput: true
  }
])
```

---

## Step 4 — Signal Classification

**Elicitation Form — Classification Model:**

```
askQuestions([
  {
    header: "Signal classification tiers",
    question: "Which output categories do you want signals sorted into?",
    message: "Every detected signal will be labeled with one of these so you know exactly what action to take.",
    multiSelect: true,
    options: [
      { label: "🔥 Motivated Buyer — explicit pain + actively seeking a solution", recommended: true },
      { label: "🟢 Buying Signal — strong intent indicator, close to purchase" },
      { label: "🟡 Qualification Signal — fits ICP, shows relevant growth or change" },
      { label: "🔵 Watch & Wait — interesting but no urgency yet, monitor over time" },
      { label: "⚪ Cold Lead — fits ICP profile but no active signal detected" }
    ],
    allowFreeformInput: false
  },
  {
    header: "Default first outreach move",
    question: "For Motivated Buyers and Buying Signals, what's your preferred first move?",
    message: "This will auto-populate the Recommended Action field on every high-priority signal card.",
    options: [
      { label: "Personalized direct message on LinkedIn", recommended: true },
      { label: "Cold email with pain-specific hook" },
      { label: "Add to HeyReach campaign immediately" },
      { label: "Just surface them — I'll decide per prospect" }
    ],
    allowFreeformInput: true
  }
])
```

---

## Config Complete — Save + Continue

After all four steps, save to the context lake:

```json
POST {BASE_URL}context/
{
  "content": "signal_config: { worked: [...], not_worked: [...], sources: [...], communities: '...', intelligence_goals: [...], keywords: '...', classifications: [...], priority_action: '...' }",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Confirm:

```
✅ Signal config saved. Running your first scan now...
```

Proceed immediately to Phase 2.

---

# PHASE 2: SIGNAL SCAN

> Runs on every invocation once config exists. Uses saved config to target sources, prioritize signals, and apply the classification model.

---

## Step 1 — Load Config + ICP Targets

Query the context lake for both in one call:

```json
{ "query": "signal_config ICP accounts target companies prospect list" }
```

Extract:
- `signal_config` — sources, keywords, classification model, intelligence goals
- `target_accounts` — saved ICP company names, LinkedIn URLs, or prospect names

If `target_accounts` is empty, ask via `askQuestions`:

```
askQuestions([
  {
    header: "ICP Targets",
    question: "Which companies or people should I scan for signals?",
    message: "Paste LinkedIn company names, profile URLs, or people's names — one per line."
  }
])
```

---

## Step 2 — Build Search Queries

Using `signal_config.sources` and `signal_config.keywords`, build queries per active source:

**LinkedIn posts:** One query per signal type that worked:
- Job change: `"[person name OR company] excited to join"` or `"new role"`
- Pain: `"[pain keyword]"` + ICP industry modifier
- Hiring: `"[company name] hiring [buyer persona role]"`
- Tool eval: `"looking for [product category] alternatives"` or `"alternatives to [competitor]"`

**Reddit:** One query per subreddit from `signal_config.communities`:
- Search `r/[subreddit]` for configured pain keywords

**News/press:** Swarm web search for `"[company name] funding OR acquisition OR launch"`

Run up to **3 queries per active source**. Start narrow, widen only if results are thin.

---

## Step 3 — Scrape Active Sources

For each active source in `signal_config.sources`:

### LinkedIn Posts
```json
{
  "searchQueries": ["<query>"],
  "maxPosts": 15,
  "profileScraperMode": "short",
  "maxReactions": 10
}
```

### LinkedIn Job Postings
Use `apify_leads_finder` with role-based filters or `apify_linkedin_company_scraper` for company job pages.

### Reddit / News
Use swarm web search via `sushi-research` targeting configured communities and keywords.

Collect all results into a raw signal pool. Deduplicate by URL.

---

## Step 4 — Classify Each Signal

Apply the saved `signal_config.classifications` model:

1. **Match against worked signals** — flag items matching patterns marked as working.
2. **Filter against not-worked signals** — discard or deprioritize excluded patterns.
3. **Keyword scan** — flag any content containing terms from `signal_config.keywords`.
4. **Assign class:**

| Class | Criteria |
|---|---|
| 🔥 Motivated Buyer | Explicit pain statement + request for solution or vendor |
| 🟢 Buying Signal | Job change into buyer role, tool eval post, competitor complaint |
| 🟡 Qualification Signal | Hiring for relevant role, funding round, strategic initiative post |
| 🔵 Watch & Wait | General pain content but no active search, or early signal with no urgency |
| ⚪ Cold Lead | ICP account match only — no active signal detected |

Extract intelligence dimensions per `signal_config.intelligence_goals` (pain, timing, budget, competitors) and attach as notes to each classified item.

Discard items below 🔵 Watch & Wait unless the user opted into Cold Leads.

---

## Step 5 — Enrich Prospects

For each flagged item, extract the poster's name and company. Fill missing contact data:

```json
{
  "searchQueries": ["<first name> <last name> <company>"],
  "maxResults": 1
}
```

Pull: full name, title, company, LinkedIn URL, email (if available).

---

## Step 6 — Deliver the Report

Show the Phase Detection Banner, then:

**Summary line:**
```
Found X signals across Y prospects — [N] 🔥 Motivated Buyers · [N] 🟢 Buying Signals · [N] 🟡 Qualification · [N] 🔵 Watch & Wait
```

Render one **Signal Report Card** per prospect, sorted: 🔥 → 🟢 → 🟡 → 🔵 → ⚪

Call out the **Top 3** with a suggested first outreach line using `signal_config.priority_action` + the specific evidence from the signal.

Then ask:

```
askQuestions([
  {
    header: "Next step",
    question: "What would you like to do with these signals?",
    multiSelect: true,
    options: [
      { label: "Save signals to Sushidata", recommended: true },
      { label: "Add top prospects to HeyReach campaign" },
      { label: "Draft outreach messages for top 3" },
      { label: "Export to CSV" },
      { label: "Nothing — just reviewing" }
    ],
    allowFreeformInput: false
  }
])
```

---

## Step 7 — Save to Context Lake (if requested)

```json
POST {BASE_URL}context/
{
  "content": "signal_scan: { date: '<today>', signals: [{ prospect, company, class, signal_type, evidence, post_url, intelligence: { pain, timing, budget, competitors }, recommended_action }] }",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Scheduled Run Behavior

When invoked automatically (Monday–Friday 9:00 AM):

1. Skip Phase 1 — config must already exist.
2. Load config and ICP targets from context lake silently.
3. Run Steps 2–5 silently.
4. Deliver the signal report (Step 6 cards + summary).
5. Auto-save scan results (Step 7) without prompting.
6. If no signals found: `No new signals detected today. Will check again next scheduled run.`
