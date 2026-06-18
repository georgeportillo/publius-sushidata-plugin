# ContactOut Playbook

Use this playbook for contact and company research using ContactOut. ContactOut has one of the highest LinkedIn-to-email match rates available and is the preferred first stop for full-profile enrichment from LinkedIn URLs.

Tools are called **directly** (no swarm needed for individual lookups). For lists of 5+ contacts, distribute across parallel swarm workers.

---

## Available Tools

| Tool name                       | What it does                                                                                    | Cost |
|---------------------------------|-------------------------------------------------------------------------------------------------|------|
| `contactout_linkedin_profile`   | Full profile + email + phone from a LinkedIn URL                                                | Email credit + phone credit if found |
| `contactout_email_enrich`       | Emails + profile data from a known email address                                                | 1 email credit + 1 phone credit if found |
| `contactout_people_enrich`      | Profile enrichment from name + company/domain or LinkedIn URL or email or phone                | 1 search credit |
| `contactout_people_search`      | Search contacts by name, title, company, location                                               | 1 search credit per result |
| `contactout_contact_info`       | Email + phone from a LinkedIn URL (single contact)                                             | Email/phone credits |
| `contactout_contact_info_bulk`  | Bulk email + phone lookup for multiple LinkedIn URLs                                           | Credits per contact |
| `contactout_domain_enrich`      | Enrich a company domain → industry, size, social profiles                                      | 1 credit |
| `contactout_company_search`     | Find companies by name, domain, or filters                                                     | 1 credit per result |
| `contactout_people_count`       | Count search results without consuming full search credits                                     | Free |
| `contactout_decision_makers`    | Get ranked decision makers at a company by domain                                              | Credits per result |
| `contactout_email_to_linkedin`  | Reverse: email → LinkedIn URL                                                                   | 1 credit |
| `contactout_contact_checker`    | Check whether a LinkedIn contact is reachable/connected                                        | Free |
| `contactout_email_verifier`     | Email deliverability check                                                                     | 1 credit |
| `contactout_usage_stats`        | Check credit usage and remaining balance                                                       | Free |

---

## Core Workflows

### Profile + email from LinkedIn URL

This is the most common use case. Use `contactout_linkedin_profile`:

```json
{
  "profile": "https://www.linkedin.com/in/janedoe",
  "profile_only": false
}
```

Set `profile_only: true` to get just the profile data (uses 1 search credit instead of email/phone credits).

Returns: `email`, `phone`, `name`, `title`, `company`, `location`, `seniority`, `department`, `linkedin_url`.

### Find decision makers at a company

Use `contactout_decision_makers` with the company domain:

```json
{ "domain": "example.com" }
```

Returns pre-ranked contacts by seniority. Supplement with `lusha_prospect_contacts` and `aiark_people_search` for fuller coverage.

### People search

Use `contactout_people_search` with filters. `job_title`, `company`, and `location` are **arrays** of strings; only `name` is a bare string:

```json
{
  "name": "Jane",
  "job_title": ["VP Sales"],
  "company": ["Acme"],
  "location": ["San Francisco"]
}
```

Check result count first with `contactout_people_count` before consuming credits on a full search.

### Company enrichment

Use `contactout_domain_enrich` to get company metadata from a domain:

```json
{ "domain": "stripe.com" }
```

Returns: industry, company size, HQ location, social links.

### Bulk contact info

For multiple LinkedIn URLs at once, use `contactout_contact_info_bulk` instead of looping `contactout_contact_info`. Pass an array of LinkedIn URLs.

---

## Email Verification

Use `contactout_email_verifier` for a quick deliverability check. For full verification (7 statuses including spamtrap and abuse), prefer `zerobounce_validate_email`.

---

## When to Use ContactOut vs Others

| Task | Prefer |
|------|--------|
| Profile + email from LinkedIn URL | ContactOut (highest LI match rate) |
| Bulk 20+ contacts | FullEnrich (cheaper at scale) |
| Mobile phone from LinkedIn URL | AI ARK or Dropleads |
| Decision makers at a company | ContactOut `decision_makers` + Lusha |
| Company enrichment | LimaData (more data) or ContactOut (faster) |

---

## Rules

- Set `profile_only: true` when you only need profile data — saves email/phone credits
- Use `contactout_people_count` before running expensive searches to gauge result volume
- `contactout_contact_info_bulk` is more efficient than looping `contactout_contact_info` for multiple URLs
- Always verify discovered emails with `zerobounce_validate_email` before outbound
