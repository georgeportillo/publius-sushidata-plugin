# Wiza Playbook

Use this playbook for contact enrichment via Wiza. Wiza is strong at finding emails, phones, and full profiles from LinkedIn URLs, and supports a bulk reveal pattern that is well-suited for high-throughput enrichment.

All three find tools (`wiza_find_phone`, `wiza_find_email`, `wiza_find_profile`) are **async** — they return a `reveal_id` that you then poll with `wiza_get_reveal` to retrieve results.

---

## Available Tools

| Tool name          | What it does                                                                          |
|--------------------|--------------------------------------------------------------------------------------|
| `wiza_find_phone`  | Find phone number for a contact (returns reveal_id)                                  |
| `wiza_find_email`  | Find work (and optionally personal) email for a contact (returns reveal_id)           |
| `wiza_find_profile`| Find full contact profile — name, title, company, email, phone (returns reveal_id)   |
| `wiza_get_reveal`  | Poll for reveal result using a reveal_id returned by any of the three find tools     |

---

## Async Pattern (Required)

All three find tools return a `reveal_id` — not the data directly. You must poll `wiza_get_reveal` to get the result.

**Never** call a find tool and return the reveal_id to the user as the final answer. Always poll until done.

### Example — find email from LinkedIn URL

**Step 1 — Start reveal:**
```json
{
  "profile_url": "https://www.linkedin.com/in/janedoe",
  "accept_work": true,
  "accept_personal": false
}
```

Returns: `{ "reveal_id": "abc123" }`

**Step 2 — Poll for result:**
```json
{ "id": "abc123" }
```

Poll every 5–10 seconds until status is `completed`. Returns the email, phone, or profile data.

---

## Tool Inputs

All three find tools accept the same input fields:

| Field         | Required | Notes |
|---------------|----------|-------|
| `profile_url` | Recommended | LinkedIn profile URL — best results |
| `full_name`   | Optional | Use with `domain` or `company` |
| `domain`      | Optional | Company domain, use with `full_name` |
| `company`     | Optional | Company name, use with `full_name` |
| `email`       | Optional | Only if other inputs unavailable |

**For `wiza_find_email` specifically:**
- `accept_work` — include work emails (default: true)
- `accept_personal` — include personal emails (default: false)

---

## Bulk Enrichment Pattern

For lists of 5+ contacts, use parallel reveals:

1. Start all find calls simultaneously (one per contact) — collect all `reveal_id`s
2. Poll all `reveal_id`s in parallel until each returns completed
3. Aggregate results and save to context lake

This is faster than sequential reveals. Use swarm workers — one per batch of 5–10 contacts.

---

## When to Use Wiza vs Others

| Task | Prefer |
|------|--------|
| Email from LinkedIn URL | ContactOut (first), Wiza (fallback) |
| Phone from LinkedIn URL | AI ARK (first), Dropleads (second), Wiza (third) |
| Full profile from LinkedIn URL | ContactOut or Wiza |
| Bulk reveal of 10+ contacts | Wiza or FullEnrich |

---

## Rules

- Always poll `wiza_get_reveal` after every find call — never leave a reveal_id uncollected
- For bulk runs, start all reveals in parallel before polling any of them (saves wall time)
- Verify returned emails with `zerobounce_validate_email` before outbound
- `profile_url` is strongly recommended for best results — name + domain is a fallback
