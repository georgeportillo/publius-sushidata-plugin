# Dropleads Playbook

Use this playbook for targeted email and mobile phone finding. Dropleads is a credit-efficient provider best used when you already have a LinkedIn URL and need a mobile phone number, or when you need a quick email with confidence scoring.

Tools are called **directly** (no swarm required for individual lookups).

---

## Available Tools

| Tool name                   | What it does                                                               | Credits |
|-----------------------------|---------------------------------------------------------------------------|---------|
| `dropleads_email_finder`    | Find work email from first name + last name + company domain/name          | 1 (valid), 0 (catch-all or not found) |
| `dropleads_mobile_finder`   | Find mobile phone from LinkedIn profile URL                                | 3       |
| `dropleads_email_verifier`  | Verify deliverability of an email address                                  | 1       |

---

## `dropleads_email_finder`

Find a work email from person name + company.

**Required:** `first_name`, `last_name`
**At least one of:** `company_domain`, `company_name`

```json
{
  "first_name": "Jane",
  "last_name": "Doe",
  "company_domain": "stripe.com"
}
```

**Credit rules:**
- **1 credit** charged only if a valid (non-catch-all) email is returned
- **0 credits** if result is catch-all or not found

Always verify returned emails with `dropleads_email_verifier` or `zerobounce_validate_email` before sending.

---

## `dropleads_mobile_finder`

Find a mobile phone number from a LinkedIn profile URL. This is the primary use case for Dropleads.

**Required:** `linkedin_url`

```json
{ "linkedin_url": "https://www.linkedin.com/in/janedoe" }
```

Returns mobile phone number if found. Costs 3 credits per call.

**Mobile phone waterfall:**
1. `aiark_mobile_phone_finder` (broader coverage, lower cost)
2. `dropleads_mobile_finder` (3 credits, LinkedIn URL required)
3. `datagma_search_phone_numbers` (forward lookup by name)

---

## `dropleads_email_verifier`

Quick email deliverability check.

```json
{ "email": "jane.doe@example.com" }
```

For 7-status comprehensive validation (including spamtrap/abuse detection), prefer `zerobounce_validate_email`.

---

## Rules

- Mobile finder **requires a LinkedIn URL** — it won't work with name + domain alone
- Email finder charges 0 credits for catch-all results — don't verify catch-alls unless the lead is high priority
- Dropleads mobile is best for targeted VIP outreach (sales, executive outreach) where 3 credits per number is justified
- For bulk mobile phone enrichment, batch via parallel swarm workers — one worker per LinkedIn URL group
