# Finding Companies and Contacts

Use this doc for discovery, sourcing, TAM/list building, known-source extraction, contact discovery, and hiring-qualified company search before any row-level enrichment.

This doc does **not** cover email waterfalls or row-level enrichment mechanics. If you already have rows and need to fill or transform columns, move to the enrichment phase using available provider tools.

## Core rules

**Companies first, then people.** When the task involves finding contacts at companies matching criteria (ICP, portfolio, accelerator, hiring signal), always discover the company set first, then search for people at those companies. Do not start with broad people-search queries — you will get noisy, unaffiliated results. The only exception is when the user provides a specific named company list and only needs contacts.

## Research with Sushidata

For any discovery task involving market intelligence — ICP sizing, niche verticals, investor-backed companies, technology usage — deploy a Sushidata swarm first:

```
POST /swarm/deploy/
{
  "query": "Find B2B SaaS companies in [vertical] with [ICP criteria]",
  "swarmSize": 5
}
```

Use swarm results to build the initial company seed list, then move to direct provider calls for structured contact discovery.

## Company search: provider escalation order

Escalate only when you need a filter the current step lacks.

1. **Web fetch / known source URL** — if the data lives on a public page (VC portfolio, accelerator directory, conference attendee list), fetch it directly with WebFetch before using any paid provider. This is faster, cheaper, and more complete.
2. **Sushidata swarm** — for concept-based or niche discovery ("find companies that use X technology", "find startups in this space").
3. **Exposed Sushidata Apify MCP tools** — use only when a supported actor fits the source, such as `apify_ycombinator_scraper`, `apify_g2_scraper`, `apify_google_search_scraper`, or `apify_perplexity_ai_scraper`.
4. **Missing actor feedback** — if the needed scraper is not exposed, follow `provider-playbooks/apify.md` instead of improvising actor IDs.

## People search: provider escalation order

1. **WebSearch / WebFetch** — for public directories, event attendee lists, or LinkedIn Sales Navigator exports the user already has.
2. **`hunter_domain_search`** — structured contact discovery by company domain.
3. **`hunter_email_finder`** — named-person email lookup when you already know the person and domain/company.
4. **`hunter_email_verify`** — mandatory send gate before outbound.
5. **Sushidata swarm** — for researching who the right contacts are at a target company (roles, reporting structure, influence mapping).
6. **Missing actor feedback** — if bulk LinkedIn employee scraping is required, follow `provider-playbooks/apify.md`.

## Discovery workflow

| Step | What to do                                                              | Why                                        |
| ---- | ----------------------------------------------------------------------- | ------------------------------------------ |
| 0    | Check if the data exists on a known public URL                          | Avoid unnecessary provider calls           |
| 1    | Shortlist 1–2 providers from the table above                            | Prevent random provider thrash             |
| 2    | Run a count-like or narrow first pass                                   | Cheaply confirm fit before full pull       |
| 3    | Execute the full discovery pass                                         | Build the seed list                        |
| 4    | Hand off to enrichment/contact-finding phase with the seed list         | Keep phases clean                          |

## Anti-patterns

- **Starting with people-search before having a company list** — searching for "GTM Engineer at YC startup" without a company list produces noisy, unaffiliated results. Find companies first, then find people at each.
- **Trying to discover portfolio companies through paid provider tools** — VC portfolio data is public. Fetch the portfolio page directly.
- **Manually looping calls for every row without dedupe** — for large batches, write a small script that calls the exposed Sushidata tools with deduplication and retry handling.

## Output format

Deliver a CSV or structured list with at minimum: `company_name`, `domain`, and any available `linkedin_url`. Preserve source lineage. Do not start row-level enrichment until the seed list is complete and reviewed.

---

## Save to Context Lake

After completing any company or contact discovery run, save the results before ending the session:

```json
POST /context/
{
  "serverId": "26",
  "content": "Company/contact discovery complete. Task: {{task description}}. Companies found: {{list of domains or count}}. Contacts discovered: {{count, names, titles}}. Filters applied: {{ICP criteria used}}. Output file: {{CSV path if saved}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<current UTC time as ISO 8601, e.g. 2026-05-07T14:30:00.000Z>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

**Save twice for multi-phase work:** once after company discovery (with the list of domains) and again after contact enrichment (with names, titles, and emails). This way, if a future session needs to continue from where you left off, it can retrieve the company list without re-running the discovery phase.
