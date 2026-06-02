# Intent Signals Playbook

Use this playbook when a Sushidata task involves identifying buying signals, hiring signals, technology intelligence, or news-driven trigger events at target companies. The two providers covered here — **PredictLeads** and **TheirStack** — work best together and should be deployed in parallel for maximum signal coverage.

**When to use this playbook:**
- You need to find companies showing active buying intent or growth signals
- You are researching what technology stack a company or prospect uses
- You need to surface news events at companies (funding, hiring, expansion, partnerships, leadership changes)
- You are building a signal-driven outreach trigger (e.g. "contact them the week after a funding round or major hire")
- You want to find companies currently hiring for roles that indicate budget or tooling change

---

## Available Signal Tools

### PredictLeads Tools

| Tool name | What it returns |
| --- | --- |
| `predictleads_company` | Company profile: name, description, location, parent/subsidiary, ticker |
| `predictleads_job_openings` | Current job postings — hiring signal by role, seniority, category, date range |
| `predictleads_technology_detections` | ~54,000 technologies detected via scripts, DNS, cookies, job descriptions — first/last seen |
| `predictleads_news_events` | Structured news events: funding, expansion, partnerships, acquisitions, hires, launches, layoffs — with confidence score + source |
| `predictleads_financing_events` | Financing-specific events with amount, round, date |
| `predictleads_discover_companies` | Find companies that match specific signal criteria (filters by event type, technology, industry) |
| `predictleads_discover_news_events` | Search for news events across all companies |
| `predictleads_connections` | Company connections and subsidiary network |
| `predictleads_companies_using_technology` | Find all companies in PredictLeads that use a specific technology |
| `predictleads_discover_job_openings` | Search job openings across all companies by category, location, title patterns |
| `predictleads_job_openings_deleted` | Jobs that were just closed (can signal role was filled or budget frozen) |
| `predictleads_extended_technology_detection` | Technology adoption history — when a company first/last used a tech |
| `predictleads_website_evolution` | How a company's website has changed over time |
| `predictleads_products` | Products and product launches linked to a company |
| `predictleads_api_subscription` | Check PredictLeads API subscription status |

**Job categories for `predictleads_job_openings`:**
`administration`, `consulting`, `data_analysis`, `design`, `directors`, `education`, `engineering`, `finance`, `healthcare_services`, `human_resources`, `information_technology`, `internship`, `legal`, `management`, `marketing`, `military_and_protective_services`, `operations`, `purchasing`, `product_management`, `quality_assurance`, `research`, `sales`, `software_development`, `support`

**News event categories for `predictleads_news_events`:**
`receives_financing`, `hires`, `launches`, `expands_offices_in`, `increases_headcount_by`, `acquires`, `partners_with`, `wins_contract`, `invests_into`, `promotes`, `opens_new_location`, `identified_as_competitor_of`, `integrates_with`, `spins_off_company`, `has_revenue`, `has_valuation`, `goes_public`, `files_suit_against`, `declares_bankruptcy`, `decreases_headcount_by`, `closes_offices_in`, `retires_from`

---

### TheirStack Tools

| Tool name | What it returns |
| --- | --- |
| `theirstack_job_search` | Search jobs across thousands of sites with filters: title, company, tech, location, seniority, salary, remote, employment type. Costs 1 credit per job. |
| `theirstack_company_search` | Find companies by tech stack, hiring signals, firmographics. Rich filters: tech slug, description keyword, funding stage, employee count, revenue, YC. Costs 3 credits per company. |
| `theirstack_technographics` | Full technology stack for a specific company by domain |
| `theirstack_buying_intents` | Buying intent signals inferred from job postings and tech detection |
| `theirstack_credit_balance` | Check remaining TheirStack credits |
| `theirstack_credits_consumption` | Check credit usage |

---

## Multi-Agent Signal Strategy

When running a signal intelligence campaign against a company or company list, split work across parallel agents:

### Agent 1 — Company Signal Snapshot (PredictLeads)
**Goal**: For each target company domain, pull all major signal categories.

```
For each company domain:
1. predictleads_news_events — filter by: receives_financing, hires, increases_headcount_by, launches, expands_offices_in, partners_with, wins_contract (last 90 days)
2. predictleads_job_openings — filter by sales, marketing, management, information_technology categories (recent only)
3. predictleads_financing_events — last 12 months
```

Output per company: signal score (High / Medium / Low), top triggers, date of most recent signal.

### Agent 2 — Technology Stack (TheirStack + PredictLeads)
**Goal**: Identify what tools each company uses — informs whether they are a buyer or already have a competing solution.

```
For each domain:
1. theirstack_technographics — full current stack
2. predictleads_technology_detections — with last_seen_at_from = 90 days ago (what they actively use)
3. predictleads_extended_technology_detection — adoption + churn history for specific technologies
```

Output per company: current tech stack, recently added tools (flag as expansion signal), recently removed tools (flag as churn/displacement opportunity).

### Agent 3 — Hiring Signal Prospecting (TheirStack)
**Goal**: Find net-new companies to prospect based on what they are hiring for.

```
theirstack_job_search with:
- job_title_or: [roles that signal budget ownership or tool evaluation]
- job_technology_slug_or: [competitor or adjacent tech slugs]
- posted_at_max_age_days: 30
- company_country_code_or: ["US"] (or target countries)
```

Hiring for "Head of Revenue Operations", "Marketing Operations Manager", "CRM Administrator", "VP Sales", "Chief Revenue Officer" → strong outreach trigger.

### Agent 4 — Discover Companies by Technology (PredictLeads)
**Goal**: Surface a list of companies using a specific technology — for competitive displacement or integration-adjacent prospecting.

```
predictleads_companies_using_technology with the technology name
predictleads_discover_companies with relevant event category filters
theirstack_company_search with company_technology_slug_or = [target tech slugs]
```

Cross-reference all three lists. Companies appearing in all three are highest-confidence prospects.

### Agent 5 — Buying Intent (TheirStack)
**Goal**: Pull active buying intent signals for companies already in your pipeline or on a watch list.

```
theirstack_buying_intents for each target domain
```

Use as a daily/weekly refresh on watched companies to surface the right moment to reach out.

---

## Signal Scoring Framework

After agents complete, score each company:

| Signal | Weight |
| --- | --- |
| Financing round (receives_financing) in last 30 days | +5 |
| Hiring: sales / marketing / revenue roles | +3 |
| Hiring: tech roles matching your ICP's stack | +2 |
| Office expansion or headcount increase | +2 |
| Partnership or integration announcement | +2 |
| Product launch | +2 |
| Competitor technology detected (displacement opportunity) | +3 |
| Technology recently removed (churn signal) | +4 |
| Active buying intent from TheirStack | +5 |

Total score → **Tier A** (12+): immediate outreach. **Tier B** (6–11): monitor weekly. **Tier C** (<6): cold for now.

---

## Swarm Request Patterns

**For a company signal sweep (single domain):**

```json
POST /swarm/deploy/
{
  "query": "Run a full buying-signal sweep for [company domain]. Use: (1) predictleads_news_events for categories receives_financing, hires, increases_headcount_by, launches, expands_offices_in, partners_with — last 90 days. (2) predictleads_job_openings for sales, marketing, management, information_technology, software_development categories. (3) theirstack_technographics for their full tech stack. (4) theirstack_buying_intents. Return: a timeline of recent signals, their tech stack highlights, and a recommended outreach hook based on the strongest trigger found.",
  "swarmSize": 4
}
```

**For technology-based company discovery:**

```json
POST /swarm/deploy/
{
  "query": "Find companies using [technology name] that are also actively hiring in sales or marketing roles. Step 1: use predictleads_companies_using_technology for [technology name]. Step 2: use theirstack_company_search with company_technology_slug_or = ['[tech slug]'] and posted_at_max_age_days = 30. Step 3: for the top 20 matching companies, run predictleads_news_events to find any receiving financing or increasing headcount in last 60 days. Return: a ranked list of companies by signal strength — company name, domain, tech detected, hiring signal, and any news event.",
  "swarmSize": 5
}
```

**For a batch of accounts — weekly signal refresh:**

```json
POST /swarm/deploy/
{
  "query": "Run a weekly signal refresh on this account list: [domains]. For each domain: (1) predictleads_news_events last 7 days — categories: receives_financing, hires, increases_headcount_by, launches, partners_with, wins_contract. (2) theirstack_buying_intents. Return only companies with at least one signal this week. Output: company | signal type | signal detail | date | recommended action.",
  "swarmSize": 5
}
```

---

## PredictLeads vs. TheirStack — When to Use Which

| Task | Best provider |
| --- | --- |
| Technology stack detection (~54k technologies) | PredictLeads |
| Technology stack cross-referenced with hiring | TheirStack |
| Find companies using a specific technology | Both — run in parallel |
| News events (funding, launches, leadership changes) | PredictLeads |
| Buying intent scoring | TheirStack |
| Job signal prospecting (job title patterns) | TheirStack (richer job filters) |
| Job signal for a single company | PredictLeads (per-company API) |
| Financing events history | PredictLeads |
| Discover new companies to prospect by signal | Both — PredictLeads `discover_companies` + TheirStack `company_search` |

---

## Save to Context Lake

After any signal sweep, save findings:

```json
POST /context/
{
  "serverId": "26",
  "content": "Signal sweep complete for {{company count}} companies. Tier A (immediate outreach): {{list}}. Tier B (monitor): {{list}}. Key triggers: {{top signals found}}. Tech stack highlights: {{key technologies}}. Sources: PredictLeads news events, PredictLeads job openings, TheirStack technographics, TheirStack buying intents.",
  "messageId": "msg-<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
