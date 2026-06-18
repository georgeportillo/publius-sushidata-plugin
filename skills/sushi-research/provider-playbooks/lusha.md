# Lusha Playbook

Use this playbook for contact and company research using Lusha. Lusha is particularly strong at B2B prospecting, lookalike search (find people similar to a known profile), and intent signals — making it uniquely powerful for targeted GTM campaigns.

Tools are called **directly** (no swarm needed for individual lookups). For prospecting or lookalike runs over large datasets, distribute across parallel swarm workers.

---

## Available Tools

| Tool name                      | What it does                                                                                          |
|--------------------------------|-------------------------------------------------------------------------------------------------------|
| `lusha_search_contacts`        | Look up a specific contact by LinkedIn URL, email, name + company                                    |
| `lusha_search_companies`       | Look up a specific company by domain, name, or Lusha ID                                              |
| `lusha_enrich_contacts`        | Enrich one or more contacts to get full email + phone profiles                                       |
| `lusha_enrich_companies`       | Enrich one or more companies to get full company profiles                                            |
| `lusha_search_enrich_contacts` | Search + enrich contacts in one call (most efficient for single contacts)                           |
| `lusha_search_enrich_companies`| Search + enrich companies in one call                                                               |
| `lusha_prospect_contacts`      | Prospect contacts by ICP filters (title, seniority, dept, industry, location, headcount, etc.)      |
| `lusha_prospect_companies`     | Prospect companies by ICP filters (industry, size, revenue, tech stack, location)                  |
| `lusha_lookalike_contacts`     | Find contacts similar to one or more known Lusha contact IDs                                        |
| `lusha_lookalike_companies`    | Find companies similar to one or more known Lusha company IDs                                       |
| `lusha_signals_contacts`       | Find contacts showing buying intent signals (job changes, hiring, promotions)                       |
| `lusha_signals_companies`      | Find companies showing buying intent signals (headcount growth, new hires, tech changes)            |
| `lusha_account_usage`          | Check Lusha credit usage and remaining balance                                                      |

---

## Core Workflows

### Single contact lookup

For a specific person, use `lusha_search_enrich_contacts` (most efficient):

```json
{
  "linkedinUrl": "https://www.linkedin.com/in/janedoe"
}
```

Or by name + company:
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "companyDomain": "stripe.com"
}
```

Returns: email, phone, full profile.

### Company lookup

```json
{
  "companyDomain": "stripe.com"
}
```

### ICP Prospecting

Use `lusha_prospect_contacts` for building a filtered list. All filter criteria must be nested under the required `filters` object, and keys are **plural** (`titles`, `seniorities`, `departments`, `industries`, `locations`, `companySizes`, `companyDomains`, `technologies`, `signals`):

```json
{
  "filters": {
    "titles": ["VP Sales"],
    "seniorities": ["VP", "Director"],
    "departments": ["Sales"],
    "industries": ["SaaS", "Fintech"],
    "locations": ["United States"],
    "companySizes": [{ "min": 50, "max": 500 }]
  }
}
```

Returns: list of matching contacts with LinkedIn URLs. Follow up with `lusha_enrich_contacts` for email + phone.

### Lookalike Search

Find contacts similar to a known champion. Lookalike search takes an array of **Lusha contact IDs** (`ids`) — not LinkedIn URLs. Run `lusha_search_contacts` first to resolve a profile to its Lusha contact ID:

```json
{ "ids": ["<lushaContactId>"] }
```

This returns people with similar titles, industries, company sizes, and locations. Ideal for cloning a high-value ICP.

For companies (array of **Lusha company IDs**):
```json
{ "ids": ["<lushaCompanyId>"] }
```

Returns companies similar in size, industry, and tech stack to the input company.

### Intent Signals

Find contacts showing behavioral buying signals. Use `signalTypes` (an array) and optionally nest other criteria under `filters`:

```json
{
  "signalTypes": ["promotion", "companyChange"],
  "filters": { "industries": ["SaaS"], "locations": ["United States"] }
}
```

Valid `signalTypes` values are exactly: `promotion`, `companyChange`, `allSignals`.

Use intent signals to prioritize outreach timing — a new VP of Sales is 5x more likely to buy than an established one.

---

## When to Use Lusha vs Others

| Task | Prefer |
|------|--------|
| Lookalike search from a champion profile | Lusha (unique capability) |
| Intent signals (job changes, growth) | Lusha or PredictLeads |
| ICP prospecting with tight filters | Lusha or ContactOut |
| Single contact email + phone | ContactOut (highest LI match), Lusha (fallback) |
| Bulk 20+ contacts | FullEnrich (cheaper) or Lusha |

---

## Rules

- Use `lusha_search_enrich_contacts` for individual contacts — it's one call instead of two
- Lookalike search needs Lusha IDs, not URLs — resolve a champion via `lusha_search_contacts` first to get its `id`, then pass it in `ids`
- Lusha signals are time-sensitive — run them weekly for the best timing window
- Check credits with `lusha_account_usage` before large prospecting runs
- Verify emails with `zerobounce_validate_email` before outbound
