# ZeroBounce Playbook

Use this playbook for email validation and domain-level email intelligence using ZeroBounce. ZeroBounce is the final verification step before any outbound send — always run discovered emails through ZeroBounce to avoid bounces, protect sender reputation, and filter spam traps.

Tools are called **directly** (no swarm needed for single validations). For bulk validation runs, use parallel swarm workers.

---

## Available Tools

| Tool name                   | What it does                                                                                    |
|-----------------------------|------------------------------------------------------------------------------------------------|
| `zerobounce_validate_email` | Validate email deliverability — 7 status categories including spamtrap and abuse detection     |
| `zerobounce_email_finder`   | Find a person's email address from name + domain or company name                              |
| `zerobounce_domain_search`  | Find all email patterns and known addresses associated with a domain or company                |
| `zerobounce_get_credits`    | Check remaining ZeroBounce credit balance                                                     |
| `zerobounce_api_usage`      | View API usage history for a date range                                                       |

---

## `zerobounce_validate_email`

The primary use case — validate email deliverability before sending.

```json
{
  "email": "jane.doe@example.com",
  "ip_address": "123.45.67.89",
  "timeout": 30
}
```

**Parameters:**
- `email` — required
- `ip_address` — optional; adds geo-location data to the result
- `timeout` — 3–60 seconds (default varies); if exceeded, returns `unknown`/`greylisted`
- `activity_data` — optional; appends activity data to results

**7 status values and what to do:**

| Status | Meaning | Action |
|--------|---------|--------|
| `valid` | Deliverable | ✅ Send |
| `invalid` | Non-existent address | ❌ Remove from list |
| `catch-all` | Domain accepts everything — deliverability unknown | ⚠️ Send with caution or skip |
| `spamtrap` | Known spam trap | 🚫 Never send |
| `abuse` | Known complaint address | 🚫 Never send |
| `do_not_mail` | Role address (info@, admin@) or opted out | ❌ Skip |
| `unknown` | Server didn't respond (timeout) | ⚠️ Retry or treat as risky |

**Decision rule:** Only send to `valid`. Treat `catch-all` as low-confidence and skip unless the lead is high priority.

---

## `zerobounce_email_finder`

Find a person's email from their name + company. ZeroBounce's finder returns validated results only.

```json
{
  "domain": "example.com",
  "first_name": "Jane",
  "last_name": "Doe"
}
```

Or by company name (if domain is unknown):
```json
{
  "company_name": "Acme Corporation",
  "first_name": "Jane",
  "last_name": "Doe"
}
```

**Middle name** is optional and can improve accuracy for common first+last combinations.

---

## `zerobounce_domain_search`

Discover all known email addresses and email format patterns for a domain.

```json
{ "domain": "stripe.com" }
```

Or by company name:
```json
{ "company_name": "Stripe" }
```

Returns: email format patterns (e.g. `{first}.{last}@company.com`), known valid addresses. Use this to infer email patterns for a company before running finder tools.

---

## `zerobounce_api_usage`

Review credit consumption history.

```json
{
  "start_date": "2025-01-01",
  "end_date": "2025-01-31"
}
```

---

## When to Validate

**Always validate:**
- Any email returned by enrichment providers before outbound
- Emails from `zerobounce_email_finder` or `zerobounce_domain_search` (validate even ZeroBounce-found emails)
- Emails older than 90 days (people change jobs — emails go stale)

**Skip validation only if:**
- The email was just returned by `zerobounce_validate_email` with status `valid` (no need to re-validate immediately)

---

## Rules

- **Never send to `spamtrap` or `abuse` status** — these will permanently damage sender reputation
- Treat `catch-all` as unverifiable — only send if the lead is extremely high-value and risk is acceptable
- For large lists (50+ emails), run validation in parallel via swarm workers — one worker per batch of 10
- Check credit balance with `zerobounce_get_credits` before bulk validation runs
- ZeroBounce validation is the final step in every enrichment waterfall, not optional
