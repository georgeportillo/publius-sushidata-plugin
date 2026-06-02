# Hunter Playbook

Use this playbook when a Sushidata research or GTM task needs professional email discovery, contact discovery, or send-gate verification through Hunter.io.

Claude should not behave like a raw Hunter MCP operator. Claude should package the contact or verification need into a Sushidata swarm request, using the Hunter-specific constraints below so Sushidata can execute the right work and return usable results.

## How Hunter Works Inside Sushidata

Sushidata's `chainfuse-api` MCP server exposes Hunter through `HunterTool` in `src/mcp/tools/hunter.mts`.

The available Hunter-backed capabilities are:

| Capability | Sushidata tool name | Hunter endpoint | Use for |
| --- | --- | --- | --- |
| Domain contact discovery | `hunter_domain_search` | `domain-search` | Find professional email addresses and domain metadata for a company domain |
| Named-person email lookup | `hunter_email_finder` | `email-finder` | Find a specific person's professional email from first name, last name, and domain or company |
| Email verification | `hunter_email_verify` | `email-verifier` | Verify whether an email is valid, deliverable, and safe for outbound |
| Person enrichment | `hunter_email_enrichment` | `people/find` | Enrich a person by email or LinkedIn handle — returns name, location, employment, social profiles, timezone |
| Company enrichment | `hunter_company_enrichment` | `companies/find` | Enrich a company by domain — returns industry, HQ, tech stack, funding, social profiles, employee count |
| Combined enrichment | `hunter_combined_enrichment` | `combined/find` | Enrich both a person and their company in one call from a single email address |

Important implementation constraints:

- `hunter_domain_search` accepts only `domain`.
- `hunter_email_finder` requires `first_name`, `last_name`, and either `domain` or `company`.
- `hunter_email_verify` requires `email`.
- `hunter_email_enrichment` requires either `email` or `linkedin_handle` (or both).
- `hunter_company_enrichment` requires `domain`.
- `hunter_combined_enrichment` requires `email`.
- Department filters, seniority filters, pagination, and ICP-count endpoints are not exposed.

## Agent Behavior

Claude should infer what the user needs, then ask Sushidata to run the appropriate Hunter-backed workflow through a swarm.

Do not ask the user to choose a Hunter tool. If multiple Hunter-backed steps are useful, include all relevant steps in the Sushidata swarm request and ask Sushidata to merge, dedupe, and verify the results.

Do not improvise unavailable Hunter capabilities. If the user needs a capability Sushidata does not expose, clearly name the missing capability and help the user send feedback to `support@sushidata.com`. If Google/Gmail is connected in the current environment, offer to draft and send the feedback email.

## Swarm Request Pattern

When Hunter is needed, deploy a Sushidata swarm with a concrete task description. Include:

- target companies and domains
- target personas or titles
- whether named contacts are already known
- which Hunter-backed steps to use
- verification requirements
- output schema
- non-send rules

Example:

```json
POST /swarm/deploy/
{
  "query": "For these target accounts, find likely buyer contacts and verify sendable corporate emails using Sushidata's Hunter-backed tools. Accounts: {{company/domain list}}. Target personas: {{roles/titles}}. For each domain, use domain-level discovery first. Select contacts whose titles match the personas. For named contacts missing emails, use named-person email lookup with first name, last name, and domain. Verify every candidate email before marking it sendable. Return company, domain, contact name, title, email, verification status, non-send reason if any, and source notes. Treat invalid, accept_all, webmail, disposable, blocked, smtp_check=false, or non-valid results as non-send.",
  "swarmSize": 3
}
```

For a single known person:

```json
POST /swarm/deploy/
{
  "query": "Find and verify the professional email for {{first name}} {{last name}} at {{company/domain}} using Sushidata's Hunter-backed named-person lookup. Prefer domain over company if both are available. Then verify the email before returning it. Return email, verification status, score if available, accept_all/webmail/disposable/block/smtp_check flags, and whether it is sendable.",
  "swarmSize": 2
}
```

For verification only:

```json
POST /swarm/deploy/
{
  "query": "Verify these candidate emails using Sushidata's Hunter-backed email verification. Emails: {{email list}}. Return each email with status/result, score, accept_all, webmail, disposable, block, smtp_check, and a sendable yes/no decision. Treat anything other than valid as non-send unless explicitly overridden by the user.",
  "swarmSize": 2
}
```

## Core Workflow

### 1. Domain-Level Discovery

Use this when the user has company domains and wants contacts.

Ask Sushidata to use domain-level discovery for each domain:

```json
{
  "domain": "stripe.com"
}
```

Because the Sushidata schema exposes only `domain`, the swarm request should tell Sushidata to filter the returned contacts by persona after retrieval, using returned fields like title, seniority, department, LinkedIn URL, email metadata, and organization metadata.

### 2. Named-Person Lookup

Use this when the user already knows the person and needs a professional email.

Ask Sushidata to use named-person lookup with domain when possible:

```json
{
  "first_name": "Patrick",
  "last_name": "Collison",
  "domain": "stripe.com"
}
```

If the domain is unavailable, company can be used:

```json
{
  "first_name": "Patrick",
  "last_name": "Collison",
  "company": "Stripe"
}
```

Prefer domain over company because domain is the more stable identity key.

### 3. Email Verification

Every email that may be used for outbound must be verified before activation.

Ask Sushidata to verify:

```json
{
  "email": "patrick@stripe.com"
}
```

Treat these outputs as non-send by default:

| Signal | Meaning |
| --- | --- |
| `status` or `result` is not `valid` | Not safe enough for outbound by default |
| `accept_all: true` | Catch-all domain - high bounce or uncertainty risk |
| `webmail: true` | Personal address - usually wrong for B2B outbound |
| `disposable: true` | Throwaway address - do not send |
| `block: true` | Blocked or risky verification state |
| `smtp_check: false` | SMTP validation did not pass |

Only mark addresses sendable when Hunter verifies them as valid unless the user explicitly overrides the send gate.

---

### 4. Person Enrichment

Use this when you have an email or LinkedIn handle and want full profile data for a contact — name, location, employment, social profiles, timezone.

Ask Sushidata to enrich by email:

```json
{
  "email": "matt@hunter.io"
}
```

Or by LinkedIn handle:

```json
{
  "linkedin_handle": "matttharp"
}
```

Returns: `name`, `email`, `location`, `timeZone`, `geo`, `bio`, `employment` (company domain, title, role, seniority), `linkedin`, `twitter`, `github`, `facebook`, `phone`, `avatar`.

Use this to fill in gaps after domain discovery or named lookup — especially to confirm job titles, seniority, and current employer before outbound.

---

### 5. Company Enrichment

Use this when you have a domain and want full firmographic data — industry, HQ, headcount, tech stack, funding, social profiles.

Ask Sushidata to enrich by domain:

```json
{
  "domain": "hunter.io"
}
```

Returns: `name`, `legalName`, `domain`, `description`, `industry`, `foundedYear`, `location`, `geo`, `metrics` (employees, revenue, funding raised, market cap), `tech`, `techCategories`, `linkedin`, `twitter`, `crunchbase`, `fundingRounds`, `tags`, `category` (sector, GICS, SIC, NAICS).

Use this to validate ICP fit before running email discovery or outbound.

---

### 6. Combined Enrichment

Use this when you have a contact's email and want both their personal profile and their company's firmographic data in a single call.

Ask Sushidata to run combined enrichment:

```json
{
  "email": "matt@hunter.io"
}
```

Returns a combined `person` and `company` object. Equivalent to running person enrichment + company enrichment together. Use when you want to confirm both the contact and the account in one step.

## Output Expectations

Ask Sushidata to return contact results in a structured shape:

| Field | Required |
| --- | --- |
| `company` | yes |
| `domain` | yes |
| `contact_name` | when found |
| `title` | when found |
| `email` | when found |
| `verification_status` | for every email |
| `sendable` | yes/no for every email |
| `non_send_reason` | required when `sendable=false` |
| `source_notes` | brief notes on whether the result came from domain search, named lookup, or verification |

For outbound workflows, also ask Sushidata to dedupe contacts by email and by normalized full name + domain.

## Recommended Order

1. Ask Sushidata to run domain-level discovery for known account domains.
2. Ask Sushidata to filter returned contacts by the target persona.
3. Ask Sushidata to use named-person lookup for relevant known contacts that are missing emails.
4. Ask Sushidata to verify every candidate email.
5. Optionally enrich key contacts with person enrichment and key accounts with company enrichment to validate ICP fit and fill profile gaps.
6. Use combined enrichment when you have a contact's email and want both person + company data in one step.
7. Cross-check names, titles, domains, and LinkedIn URLs with Sushidata research, Browser Rendering, Apify, or first-party sources before campaign activation.

## Pitfalls

- Treating this as a direct Hunter tool-use playbook instead of a Sushidata swarm-packaging playbook.
- Asking the user to choose a Hunter tool. Claude should choose the workflow.
- Requesting unsupported Hunter options such as department filters, seniority filters, pagination, or ICP-count endpoints.
- Skipping email verification before outbound.
- Treating Hunter as authoritative for identity. Always confirm that the person, company, and domain match the target account before activation.
- Using `hunter_combined_enrichment` or `hunter_email_enrichment` without a valid email — these require a real email address, not a domain or name.

---

## Save to Context Lake

After Sushidata returns Hunter-backed results, save the discovery and verification summary. Hunter calls may consume credits, and storing results prevents re-running the same lookups in future Sushidata sessions:

```json
POST /context/
{
  "serverId": "26",
  "content": "Hunter-backed Sushidata workflow complete. Task: {{contact discovery / named email lookup / email verification}}. Domain(s): {{list}}. Contacts discovered: {{count}}. Emails found: {{count}}. Verified sendable: {{count}}. Non-send or risky: {{count and reasons}}. Output: {{CSV or JSON path if saved}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
