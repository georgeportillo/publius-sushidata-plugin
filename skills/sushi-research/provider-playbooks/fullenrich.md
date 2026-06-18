# FullEnrich Playbook

Use this playbook for bulk email and phone enrichment via FullEnrich. FullEnrich is an aggregator that pulls from 15+ underlying providers and is the most cost-effective option for waterfall enrichment at scale.

The key pattern is **async**: call `fullenrich_start_enrichment` to kick off the job, then poll `fullenrich_get_enrichment` until complete. Never wait synchronously in a single swarm task.

---

## Available Tools

| Tool name                      | What it does                                                                                   |
|--------------------------------|-----------------------------------------------------------------------------------------------|
| `fullenrich_start_enrichment`  | Start a bulk enrichment job (email + phone from LinkedIn URLs)                                 |
| `fullenrich_get_enrichment`    | Poll enrichment job status and retrieve results                                               |
| `fullenrich_reverse_email`     | Reverse email lookup — enrich a person from their email address (async, start step)           |
| `fullenrich_get_reverse_email` | Poll and retrieve reverse email enrichment results                                             |
| `fullenrich_search_people`     | Search people by name, title, company, or LinkedIn filters                                    |
| `fullenrich_search_company`    | Search companies by name, domain, industry, or size                                           |
| `fullenrich_get_credits`       | Check remaining FullEnrich credit balance                                                     |

---

## Core Pattern: Bulk Enrichment (Async)

FullEnrich enrichment is always async. Use this two-step pattern:

### Step 1 — Start enrichment job

```json
{
  "name": "Q3 Target Accounts",
  "data": [
    { "linkedin_url": "https://www.linkedin.com/in/janedoe" },
    { "linkedin_url": "https://www.linkedin.com/in/johnsmith" }
  ]
}
```

The `name` appears in the FullEnrich dashboard. The contacts array field is `data` (max 100 per request). Each item accepts `first_name` + `last_name` + `domain`/`company_name`, OR `linkedin_url`. Returns an `enrichment_id`.

### Step 2 — Poll for results

```json
{ "enrichment_id": "abc123" }
```

Poll every 10 seconds until `status` is `FINISHED` (other statuses: `CREATED`, `IN_PROGRESS`). Returns enriched profiles with email and phone when found.

---

## Reverse Email Lookup

Find a person profile from a known email address. Also async. Requires a `name` label plus a `data` array of `{ email }` objects:

**Start:**
```json
{
  "name": "Reverse email batch",
  "data": [{ "email": "jane@example.com" }]
}
```

**Poll:**
```json
{ "enrichment_id": "xyz789" }
```

---

## People Search

Search by filter criteria synchronously. Every filter is an array of `{ value }` objects (optionally `exact_match`/`exclude`); within a category values are OR, across categories logic is AND:

```json
{
  "current_position_titles": [{ "value": "VP Sales" }],
  "current_company_domains": [{ "value": "stripe.com" }]
}
```

Returns LinkedIn URLs + basic profile data. Feed results into `fullenrich_start_enrichment` for email/phone.

---

## Company Search

Filters are arrays of `{ value }` objects; headcount/year filters are arrays of `{ min, max }` ranges:

```json
{
  "names": [{ "value": "Stripe" }],
  "industries": [{ "value": "Financial Services" }],
  "headcounts": [{ "min": 100, "max": 1000 }]
}
```

---

## When to Use FullEnrich vs Others

| Task | Prefer |
|------|--------|
| Bulk 10+ contacts email + phone | FullEnrich (most cost-efficient at scale) |
| Single contact enrichment | ContactOut or Lusha (faster, synchronous) |
| Reverse email → profile | FullEnrich or Datagma |
| Finding emails from names only | Hunter or Datagma (no LinkedIn URL needed) |

---

## Rules

- Never start a FullEnrich job and immediately return — always poll `fullenrich_get_enrichment` until done
- For bulk runs, kick off multiple jobs simultaneously via parallel swarm workers — each worker handles one batch and polls its own job
- Check credits before large runs with `fullenrich_get_credits`
- Verify returned emails with `zerobounce_validate_email` before outbound (FullEnrich aggregates — quality varies by provider)
- FullEnrich handles its own internal waterfall — no need to manually loop other providers if you use it
