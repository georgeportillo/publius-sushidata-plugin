# LimaData Playbook

Use this playbook for LinkedIn-native enrichment and prospecting via LimaData. LimaData excels at company and person enrichment, work email + phone finding, prospecting with LinkedIn URL search, and — uniquely — scraping LinkedIn post activity (posts, comments, reactions).

Tools are called **directly** (no swarm needed for individual lookups). For prospecting over large lists, distribute across parallel swarm workers.

---

## Available Tools

| Tool name                              | What it does                                                                                         |
|----------------------------------------|------------------------------------------------------------------------------------------------------|
| `limadata_enrich_person`               | Enrich a person from email, LinkedIn URL, name + company                                             |
| `limadata_enrich_company`              | Enrich a company from domain or LinkedIn URL                                                         |
| `limadata_find_work_email`             | Find work email from a LinkedIn URL                                                                  |
| `limadata_find_phone`                  | Find phone number from LinkedIn URL, or name + company                                               |
| `limadata_find_profiles`               | Find LinkedIn profile URLs from full name + company domain                                           |
| `limadata_prospect_people_search_url`  | Return people from a LinkedIn / Sales Navigator search URL                                           |
| `limadata_prospect_companies_filter`   | Filter and return matching company profiles from LinkedIn                                            |
| `limadata_prospect_people_filter`      | Filter and return matching person profiles from LinkedIn                                             |
| `limadata_search_posts`                | Search LinkedIn posts by keyword or topic                                                            |
| `limadata_post_comments`               | Get comments on a specific LinkedIn post                                                             |
| `limadata_post_reactions`              | Get reactions on a specific LinkedIn post                                                            |

---

## Core Workflows

### Person enrichment

```json
{
  "email": "jane@example.com",
  "linkedin_url": "https://www.linkedin.com/in/janedoe",
  "include_work_email": true,
  "include_phone": true
}
```

**Note:** `include_phone: true` costs 10 additional credits per call. Only enable when phone is specifically needed.

Minimal input (name + company as fallback):
```json
{
  "name": "Jane Doe",
  "company_name": "Stripe"
}
```

### Company enrichment

```json
{ "domain": "stripe.com" }
```
or
```json
{ "linkedin_url": "https://www.linkedin.com/company/stripe" }
```

Returns: company name, description, industry, headcount, HQ location, LinkedIn data.

### Find work email

```json
{
  "linkedin_url": "https://www.linkedin.com/in/janedoe"
}
```

### Find LinkedIn profiles from name + domain

```json
{
  "full_name": "Jane Doe",
  "company_domain": "stripe.com"
}
```

Returns LinkedIn profile URLs for further enrichment.

### Prospecting

Return people directly from an existing LinkedIn / Sales Navigator search URL (paginate with `page`):
```json
{
  "search_url": "https://www.linkedin.com/sales/search/people?query=...",
  "page": 1
}
```

For filter-based prospecting, use `limadata_prospect_people_filter` or `limadata_prospect_companies_filter`.

---

## LinkedIn Post Intelligence (Unique Capability)

These tools are **not available in any other enrichment provider** in this stack. Use them for ICP signal research, competitor tracking, or finding active voices in a niche.

### Search posts by keyword

```json
{
  "query": "AI-powered sales automation",
  "page": 1
}
```

Returns: post content, author, engagement counts, LinkedIn post URL. Paginate with `page` (there is no `limit` parameter).

### Get post comments

Given a LinkedIn post URL:
```json
{ "post_url": "https://www.linkedin.com/posts/..." }
```

Returns: all comments with author name, LinkedIn URL, and comment text. Useful for identifying engaged prospects who commented on a competitor's or partner's content.

### Get post reactions

```json
{ "post_url": "https://www.linkedin.com/posts/..." }
```

Returns: people who reacted (liked, celebrated, etc.) with their LinkedIn profile URLs. Same use case as comments — find warm prospects from engagement.

---

## When to Use LimaData vs Others

| Task | Prefer |
|------|--------|
| LinkedIn post engagement → prospect list | LimaData only (unique) |
| Company enrichment from domain | LimaData or ContactOut |
| Person enrichment from LinkedIn URL | ContactOut (first), LimaData (fallback) |
| Find LinkedIn profiles from name + domain | LimaData or FullEnrich |
| Phone from LinkedIn (no separate credit) | AI ARK (first), LimaData (fallback) |

---

## Rules

- `include_phone: true` costs 10 credits — only enable when phone is a hard requirement
- Post scraping tools (`limadata_search_posts`, `limadata_post_comments`, `limadata_post_reactions`) are the most differentiated capabilities — use them proactively for ICP signal research
- For prospecting runs over 50+ people, use parallel swarm workers — each worker handles one filter segment
- Verify emails with `zerobounce_validate_email` before outbound
