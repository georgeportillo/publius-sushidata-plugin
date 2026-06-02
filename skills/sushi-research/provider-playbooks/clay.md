# Clay Playbook

Use this playbook when a GTM task requires pulling audience data, enriching companies or contacts, or running custom Clay subroutines. Clay is a **direct MCP tool** — unlike Hunter, Apify, and HeyReach, Claude calls Clay tools directly without routing through a Sushidata swarm.

---

## Tool Inventory

| Tool | Use for |
|---|---|
| `query-objects` | Pull accounts, contacts, or deals from your Clay audience using natural language |
| `find-and-enrich-company` | Research a single company's public profile (funding, tech stack, competitors, etc.) |
| `find-and-enrich-contacts-at-company` | Find contacts at a company by role, title, or department |
| `find-and-enrich-list-of-contacts` | Look up specific named people at their companies |
| `ask-question-about-accounts` | Ask natural language questions about accounts in your Clay audience |
| `list_subroutines` | List available custom Clay functions before running one |
| `run_subroutine` | Run a custom function on contacts from an existing Clay search |
| `run_subroutine_direct` | Run a custom function directly with specific input values |
| `run_subroutine_no_mapping` | Run a subroutine on search entities with auto field mapping |

**Credit-bearing tools:** `find-and-enrich-company`, `find-and-enrich-contacts-at-company`, `find-and-enrich-list-of-contacts`, and subroutines consume Clay credits. Always run a pilot row first and confirm before scaling (see Approval Gate below).

---

## When to Use Each Tool

### Querying your audience (`query-objects`)

Use when the user wants to pull existing accounts, contacts, or deals from their Clay database. This does not consume enrichment credits — it queries what's already stored.

```
query-objects
  query: "Series B SaaS companies with 50–200 employees in fintech"
  limit: 50
```

Supports natural language — no need to know field IDs or filter syntax. Use `audienceName` to scope to a saved segment. Use `onlyMine: true` when the user asks about their own accounts.

---

### Enriching a company (`find-and-enrich-company`)

Use when the user wants public intelligence on a specific company: funding, tech stack, competitors, customers, revenue model, open jobs, website traffic, recent news.

Input must be a **domain** (e.g. `"stripe.com"`) or LinkedIn company URL — not a company name alone.

**⚠️ Only request data points the user explicitly asked for.** Each enrichment costs credits.

Available company data points:
- `Headcount Growth`
- `Recent News`
- `Investors`
- `Company Competitors`
- `Company Customers`
- `Tech Stack`
- `Website Traffic`
- `Open Jobs`
- `Revenue Model`
- `Annual Revenue`
- `Latest Funding`

```
find-and-enrich-company
  companyIdentifier: "stripe.com"
  companyDataPoints: [{ type: "Tech Stack" }, { type: "Latest Funding" }]
```

---

### Finding contacts at a company (`find-and-enrich-contacts-at-company`)

Use when the user wants to find people by role or department at a specific company. Input is a domain or LinkedIn company URL.

Available contact filters: `job_title_keywords`, `job_title_exclude_keywords`, `profile_keywords`, `locations`, `locations_exclude`, `current_role_min_months_since_start_date`, `current_role_max_months_since_start_date`, `certification_keywords`, `languages`, `school_names`.

Available contact data points: `Email`, `Summarize Work History`, `Find Thought Leadership`.

```
find-and-enrich-contacts-at-company
  companyIdentifier: "stripe.com"
  contactFilters:
    job_title_keywords: ["VP Sales", "Head of Sales"]
    locations: ["United States"]
  dataPoints:
    contactDataPoints: [{ type: "Email" }]
```

**Compound titles stay as one string:** `"VP Finance"` → `["VP Finance"]`, not `["VP", "Finance"]`.

---

### Finding specific named contacts (`find-and-enrich-list-of-contacts`)

Use when the user names specific people. Input is an array of `{ contactName, companyIdentifier }` pairs. Do not use this to enrich contacts from an existing search — use `add-contact-data-points` instead.

```
find-and-enrich-list-of-contacts
  contactIdentifiers:
    - contactName: "Jane Smith"
      companyIdentifier: "stripe.com"
    - contactName: "Alex Chen"
      companyIdentifier: "openai.com"
```

---

### Asking questions about accounts (`ask-question-about-accounts`)

Use when the user wants insight on accounts already in their Clay audience — deal status, stakeholder mapping, relationship health. Requires account IDs.

Call `query-objects` first to get `entityId` values, then pass them as `accountIds`.

```
ask-question-about-accounts
  accountIds: [12345, 67890]
  question: "Who are the key decision makers and what's the current deal status?"
```

---

### Running custom subroutines

**Before running any subroutine:** call `list_subroutines` to see what's available and what inputs each requires.

| Scenario | Tool |
|---|---|
| User provides specific values (a LinkedIn URL, a name, an email) | `run_subroutine_direct` |
| Running on contacts from an existing Clay search | `run_subroutine` or `run_subroutine_no_mapping` |

```
list_subroutines  ← always call first

run_subroutine_direct
  subroutine_id: "<id from list_subroutines>"
  inputs:
    linkedin_url: "https://linkedin.com/in/someone"
    full_name: "Jane Smith"
```

---

## Integration with Sushidata

Clay and Sushidata are complementary:

- **Clay** supplies structured audience data — your existing accounts, enriched contacts, CRM relationships.
- **Sushidata swarms** supply deep research — web intelligence, signal discovery, competitor analysis, personalization.
- **The context lake** ties sessions together — save Clay outputs to Sushidata so future sessions don't re-query from scratch.

### Recommended pattern: Clay → Sushidata → Context Lake

1. **Pull from Clay** — use `query-objects` or `find-and-enrich-contacts-at-company` to get your base list of companies or contacts.
2. **Research with Sushidata** — deploy a swarm to enrich each company with signals not available in Clay: recent news, hiring patterns, pain language, competitor activity.
3. **Save to the context lake** — POST the combined output to `/context/` so future sessions can retrieve it instantly without re-running Clay or swarms.

```json
POST /context/
{
  "serverId": "26",
  "content": "Clay pull + Sushidata enrichment complete. Companies: [list]. Contacts: [count]. Key signals: [summary]. Full CSV at: [path].",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Approval Gate (credit-bearing calls)

1. Run a **pilot** on 1–2 rows or a single company first.
2. Show the result with the assumed data points and estimated scope.
3. Wait for explicit approval before running on the full list.

Never add enrichment data points "to be helpful" — only what the user explicitly asked for.

---

## Common Pitfalls

- **Don't pass company names to `find-and-enrich-company`** — input must be a domain (`"stripe.com"`) or LinkedIn company URL. Convert known names; ask if ambiguous.
- **Don't use `find-and-enrich-company` for your own accounts** — use `query-objects` + `ask-question-about-accounts` for accounts already in your Clay audience.
- **Don't add unrequested data points** — each enrichment costs credits. Only request what the user asked for.
- **Don't filter subroutine results in chat** — always re-call the tool with narrower inputs. Filtering in prose is lossy.
- **Don't skip `list_subroutines`** — subroutine input names vary; never guess them.
