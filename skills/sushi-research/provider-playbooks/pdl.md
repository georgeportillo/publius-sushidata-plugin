# People Data Labs (PDL) Playbook

Use this playbook for high-volume, high-confidence B2B data using People Data Labs. PDL has one of the largest person and company databases available (3B+ person records, 100M+ companies). It is particularly strong for SQL-style search queries with complex filter combinations.

A unique capability: `pdl_ip_enrich` — reverse-IP to company identification, useful for de-anonymizing website traffic.

Tools are called **directly** (no swarm needed for individual lookups). For bulk search queries, distribute across parallel swarm workers.

---

## Available Tools

| Tool name            | What it does                                                                                          |
|----------------------|-------------------------------------------------------------------------------------------------------|
| `pdl_person_enrich`  | Enrich a person profile from email, LinkedIn URL, name + company, or phone                           |
| `pdl_person_search`  | Search 3B+ person profiles with SQL-style filter queries                                           |
| `pdl_person_identify`| Identify a person from a fuzzy combination of signals (name, email, phone, social handles)           |
| `pdl_company_enrich` | Enrich a company from domain, name, LinkedIn URL, or ticker symbol                                   |
| `pdl_company_search` | Search 100M+ companies with SQL-style filter queries                                                 |
| `pdl_ip_enrich`      | Reverse-IP lookup — identify the company associated with an IP address                               |

---

## Core Workflows

### Person enrichment

```json
{
  "email": "jane@example.com"
}
```

Or by LinkedIn URL:
```json
{
  "profile": "https://www.linkedin.com/in/janedoe"
}
```

Returns: full name, current title, company, location, education, skills, social profiles, and work history.

### Person search (SQL-style)

PDL uses an Elasticsearch-style query syntax. This is the most powerful way to build ICP lists:

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "job_title_role": "sales" } },
        { "match": { "job_title_levels": "vp" } },
        { "match": { "industry": "software" } }
      ],
      "filter": [
        { "term": { "location_country": "united states" } }
      ]
    }
  },
  "size": 25,
  "from": 0
}
```

Pagination: increment `from` by `size` to page through results.

### Person identify (fuzzy match)

Use when you have partial information and want to confirm a person's identity:

```json
{
  "name": "Jane Doe",
  "company": "Stripe",
  "location": "San Francisco"
}
```

Returns: best-match profile with a confidence score.

### Company enrichment

```json
{ "website": "stripe.com" }
```

Or by name:
```json
{ "name": "Stripe" }
```

Returns: industry, size, revenue, HQ, social profiles, tech stack, funding data.

### IP Enrichment (reverse-IP to company)

Identify the company associated with an IP address — de-anonymize website visitors:

```json
{ "ip": "123.45.67.89" }
```

Returns: company name, domain, industry, size, and location. Use this for website visitor de-anonymization workflows.

---

## When to Use PDL vs Others

| Task | Prefer |
|------|--------|
| High-volume person search with complex filters | PDL (SQL-style, 3B records) |
| Reverse-IP to company | PDL (unique capability) |
| Person enrichment from email | PDL or ContactOut |
| Company enrichment from domain | PDL or LimaData |
| Fuzzy identity resolution | PDL `person_identify` |
| Mobile phone finding | AI ARK or Dropleads (PDL not specialized here) |

---

## Rules

- PDL's query syntax is Elasticsearch-style — use `bool`, `must`, `filter`, `match`, `term` correctly
- Use `from` + `size` for pagination — PDL does not use continuation tokens
- `pdl_ip_enrich` is the only tool in the stack for reverse-IP company identification — use it for website visitor workflows
- PDL returns social handles and work history that other providers often don't — useful for signal research
- Verify emails with `zerobounce_validate_email` before outbound
