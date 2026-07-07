# Step 7 — Top 10 net-new prospects (required deliverable)

A signal report without a companion prospect list is incomplete. The signals tell you what to look for; Step 7 produces the actual "here are 10 real companies the user should pursue" list. **This is a hard requirement of the pipeline.**

## What's required vs. what's optional

| Output                                                                                     | Status                      | Why                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------ | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **10 net-new companies** with descriptions, signal scores, matched signals, cited evidence | **REQUIRED** — every run    | A signal report without target companies forces the user to do their own prospecting pass, which is the expensive thing they wanted to skip.                                                        |
| **Contacts + corporate emails** at those companies                                         | **OPTIONAL** — credit-aware | Contact discovery uses FullEnrich calls and additional research. Always offer it; only run it if the user approves the spend. Default to companies-only if the user hasn't said. |

When contacts are skipped, the prospect cards still need to ship — they just include "(contacts not enriched — ask to add contacts)" in place of the contact bullets.

## What every prospect card must contain

1. **Company identity** — name, apex domain, 1-sentence description (location, what they do)
2. **Category label** from Step 1.0.5: Net-new / Account-only / Re-engage; excluded items dropped entirely
3. **Signal score** + the 4–6 matched signals from Step 4/5, shown as inline code
4. **3–5 cited evidence quotes** from the company's own website or job listings (same format as Step 5) proving the signals actually hit
5. **(if `--contacts` is on)** 1–3 named contacts with full name (linked to LinkedIn), title, and corporate email validated against the company's apex domain. "(email not found)" annotation when the waterfall returned nothing — be honest about gaps rather than shipping false positives.

## Sushidata prospect workflow

**Companies only (default):**

1. Start from `prospects_actionable.csv` after dedupe.
2. Rank candidates by signal score, evidence quality, and category label.
3. Select the top 10 companies with the clearest matched signals.
4. Produce prospect cards even when contacts are not enriched.

**Companies + contacts + emails (only after user approval):**

1. Use the buyer-persona job titles from this run's Step 0/0.5 ecosystem discovery. Do not reuse another vertical's roles.
2. For each top company, call `fullenrich_search_people` with the company domain.
3. Select likely contacts from returned names, titles, seniority, department, LinkedIn URLs, and email metadata.
4. For any named contact without an email, use `fullenrich_start_enrichment` with first name + last name + domain (or linkedin_url), then poll `fullenrich_get_enrichment`.
5. Spot-check FullEnrich confidence scores before publishing or activating — treat low-confidence results as non-send.
6. For companies where FullEnrich returns no useful contacts, use WebSearch, Browser Rendering, and focused Sushidata swarms to identify likely personas. If a LinkedIn employee-scraping actor is required, follow the missing-actor feedback workflow in `skills/sushi-research/provider-playbooks/apify.md`.

Vertical examples for persona-role targeting:

- creative ops: "Creative Director, Brand Manager, Content Operations Lead, Marketing Operations"
- AR automation: "AR Manager, Accounts Receivable Specialist, Controller, Finance Director"
- sales engagement: "SDR Manager, Sales Operations, VP Sales, Head of Sales Development"
- developer tools: "Staff Engineer, Platform Engineer, DevOps Lead, Engineering Manager"
- metal AM: "Design Engineer, Mechanical Design Engineer, Additive Manufacturing Engineer, DfAM Engineer"

**Domain-match validation is mandatory.** Always check that the returned email's apex domain matches the company's apex domain before publishing. Providers return stale addresses often enough that this is the difference between a usable list and an embarrassing one. On one real run:

- A contact at X-Bow Systems came back with `@orbitalatk.com` (his previous employer, acquired into Northrop Grumman years earlier)
- Another came back with `@governors-america.com` (a completely unrelated company)
- A third came back with a personal `@googlemail.com`

Use `extract_apex()` from `scripts/dedupe_utils.py` on both the email domain and the company domain; if they don't match, mark `email_source=apex_mismatch` and publish "(email not found)" instead. Keep the raw value in `raw_email` for auditing.

## Output card skeleton

Render each prospect as a card (heading + description + signals + evidence + contacts), not a wide table. Cards make it possible to include the required 3–5 evidence quotes per prospect without blowing up layout.

```markdown
### [company name] — score [N] [category badge]

_1-sentence description of the company._

Domain: `apex.com`

Matched signals: `signal1`, `signal2`, `signal3`, `signal4`

**Cited evidence:**

- [📄 website] [page title]: "...exact quote around keyword..."
  https://apex.com/source-page
- [💼 job] [job title]: "...exact quote from listing..."
  https://linkedin.com/jobs/view/...

**Contacts:** (only if --contacts was on)

- **Full Name** — Role · ✉ `email@apex.com`
  https://linkedin.com/in/profile
- **Other Name** — Role · (email not found)
  https://linkedin.com/in/profile
```

## How many is "10"

10 is a ceiling, not a floor. If the actionable pool (post-dedupe, scored) has fewer than 10 companies that pass the minimum score threshold, ship whatever you have and explain the shortfall — don't pad with low-confidence entries. If the pool has more than 10, prefer top-scoring first, then break ties on category preference (net-new > account-only > re-engage) to surface the cleanest outbound targets.
