# Datagma Playbook

Use this playbook for multi-signal enrichment using Datagma. Datagma is strong at work email finding from name + company, general person/company enrichment, phone number search (reverse and forward), job change detection, and Twitter/X profile lookups — capabilities not available in most other providers.

Tools are called **directly** (no swarm needed for single lookups).

---

## Available Tools

| Tool name                        | What it does                                                                                          |
|----------------------------------|-------------------------------------------------------------------------------------------------------|
| `datagma_find_work_email`        | Find work email from first name + last name + company domain/name                                    |
| `datagma_enrich`                 | Enrich a person or company from email, LinkedIn URL, company name, website, or SIREN number          |
| `datagma_find_people`            | Find people at a company by name                                                                     |
| `datagma_job_change_detection`   | Detect job changes for a person by LinkedIn URL or profile data                                      |
| `datagma_search_phone_numbers`   | Forward phone lookup — find phone number(s) for a person                                             |
| `datagma_search_by_phone`        | Reverse phone lookup — identify a person from a phone number                                         |
| `datagma_get_twitter_by_username`| Look up Twitter/X profile by username (handle)                                                       |
| `datagma_get_twitter_by_email`   | Look up Twitter/X profile by email address                                                           |
| `datagma_get_credit`             | Check remaining Datagma credit balance                                                               |

---

## Core Workflows

### Find work email (name + company)

```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "company": "stripe.com"
}
```

Or use `fullName` instead of `firstName`/`lastName`. The `company` field accepts domain or company name. Use `linkedInSlug` for higher accuracy when you have a LinkedIn company URL.

### Person/company enrichment

`datagma_enrich` accepts flexible inputs — use whichever you have:

```json
{ "data": "jane@example.com" }
```
or
```json
{ "data": "https://www.linkedin.com/in/janedoe" }
```
or
```json
{ "data": "stripe.com" }
```

Use `countryCode` to improve accuracy and avoid disambiguating namesakes. Use `companyKeyword` when multiple companies share a similar name.

### Job change detection

```json
{ "data": "https://www.linkedin.com/in/janedoe" }
```

Returns current and previous positions, detects recent role changes. Useful for timing outreach around career transitions.

### Phone lookup

**Forward** (name → phone):
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "company": "example.com"
}
```

**Reverse** (phone → person):
```json
{ "phone": "+15551234567" }
```

Reverse phone via `datagma_search_by_phone` is unique to Datagma — not available in most other enrichment providers.

### Twitter/X lookups

By handle:
```json
{ "username": "elonmusk" }
```

By email (great for ICP research when you know email but not social handle):
```json
{ "email": "jane@example.com" }
```

---

## When to Use Datagma vs Others

| Task | Prefer |
|------|--------|
| Work email from name + company | Datagma, FullEnrich, Hunter (all strong) |
| Reverse phone → person | Datagma (unique capability) |
| Twitter/X profile from email | Datagma (unique capability) |
| Job change detection | Datagma, LimaData |
| Person enrichment from LinkedIn URL | ContactOut (first), Datagma (fallback) |

---

## Rules

- For work email finding: prefer Datagma when you have a clear name + company domain combo without a LinkedIn URL
- `datagma_search_by_phone` is rare — use it when you have an inbound phone number and need to identify the person
- Check credits with `datagma_get_credit` before large enrichment runs
- Verify emails returned by `datagma_find_work_email` with `zerobounce_validate_email`
