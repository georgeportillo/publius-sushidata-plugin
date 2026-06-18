# AI ARK Playbook

Use this playbook when you need to search 70M+ company profiles or 500M+ person profiles with rich filter criteria, or need to find mobile phones from LinkedIn URLs.

AI ARK tools are called **directly** (no swarm needed for single lookups). For bulk operations, distribute across parallel swarm workers — one worker per filter segment.

---

## Available Tools

| Tool name                     | What it does                                                                                      |
|-------------------------------|---------------------------------------------------------------------------------------------------|
| `aiark_company_search`        | Search 70M+ companies by name, domain, industry, size, revenue, tech stack, keywords             |
| `aiark_people_search`         | Search 500M+ person profiles by name, title, seniority, department, company, location            |
| `aiark_reverse_people_lookup` | Reverse lookup by email or phone → full person profile                                           |
| `aiark_mobile_phone_finder`   | Find a mobile phone number from LinkedIn URL or name + domain                                    |
| `aiark_get_credits`           | Check remaining AI ARK credit balance                                                            |

---

## `aiark_company_search`

Search across 70M+ enriched company profiles.

**Filters available** (all company filters must be nested under the top-level `account` object):
- `name` — company name
- `domain` — company domain
- `linkedin` — company LinkedIn URL
- `industry` — array of industry strings (e.g. `["fintech", "saas"]`)
- `location.country`, `location.state`, `location.city` — arrays
- `employeeSize.start` / `employeeSize.end` — headcount range
- `annualRevenue.start` / `annualRevenue.end` — revenue range in USD
- `keywords` — keyword filters for description/products
- `technologies` — tech stack filters (e.g. `["React", "Salesforce"]`)
- `companyType` — `PRIVATELY_HELD`, `PUBLIC_COMPANY`, `NONPROFIT`, `EDUCATIONAL`, `GOVERNMENT_AGENCY`, `PARTNERSHIP`, `SELF_EMPLOYED`, `SELF_OWNED`

**Example — SaaS companies in NYC with 50–500 employees:**
```json
{
  "account": {
    "industry": ["saas"],
    "location": { "city": ["New York"] },
    "employeeSize": { "start": 50, "end": 500 }
  }
}
```

> The top-level input only accepts `account` (and `lookalikeDomains`). Filters placed at the top level are silently dropped and the search runs unfiltered.

---

## `aiark_people_search`

Search 500M+ person profiles. Combine contact filters with account (company) filters.

**Contact filters:**
- `name` — full name
- `linkedin` — LinkedIn profile URL
- `location.country/state/city` — arrays
- `seniority` — `Owner`, `Founder`, `C-Suite`, `Partner`, `VP`, `Head`, `Director`, `Manager`, `Senior`, `Entry`, `Intern`, `Training`
- `department` — `C-Suite`, `Engineering`, `Finance`, `HR`, `IT`, `Legal`, `Marketing`, `Operations`, `Product Management`, `Sales`, `Support`
- `currentTitle` — job title string
- `skills` — array (e.g. `["Python", "Machine Learning"]`)
- `email` — filter by known email

**Account filters** (nested `account` field — uses `AiArkAccountFilter`): same as `aiark_company_search` filters above.

**Example — VP Sales at SaaS companies in San Francisco:**
```json
{
  "contact": {
    "seniority": ["VP"],
    "department": ["Sales"]
  },
  "account": {
    "industry": ["saas"],
    "location": { "city": ["San Francisco"] }
  }
}
```

---

## `aiark_reverse_people_lookup`

Look up a full person profile from a known email or phone number. Pass it as a single `search` string:

```json
{ "search": "jane@example.com" }
```
or
```json
{ "search": "+15551234567" }
```

Returns: full name, title, company, LinkedIn URL, and profile data.

---

## `aiark_mobile_phone_finder`

Find a mobile phone number. Best results with a LinkedIn URL; also works with name + domain.

```json
{ "linkedin": "https://www.linkedin.com/in/janedoe" }
```
or
```json
{ "name": "Jane Doe", "domain": "example.com" }
```

---

## When to Use AI ARK vs Other Providers

| Task | Prefer |
|------|--------|
| Broad company search by ICP filters | AI ARK or PDL |
| People search by title + company + location | AI ARK or ContactOut |
| Reverse lookup from email | AI ARK or Datagma |
| Mobile phone from LinkedIn URL | AI ARK (primary), Dropleads (fallback) |

---

## Rules

- AI ARK works best when you provide multiple filter criteria — single-field searches return noisy results
- For mobile phones, always try `aiark_mobile_phone_finder` first before Dropleads — it has broader coverage
- Combine with `zerobounce_validate_email` before sending any discovered emails
