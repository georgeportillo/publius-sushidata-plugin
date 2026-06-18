# Enrichment Waterfall Playbook

Use this playbook when you need to find or enrich email addresses, phone numbers, person profiles, or company data using Sushidata's direct enrichment tools. These tools are called **directly** — no swarm needed for individual lookups — but they should be run **in parallel across multiple swarm workers** to maximize coverage when enriching more than a handful of contacts.

The core principle: **no single provider has complete coverage**. Run multiple providers simultaneously, merge results, and let the waterfall produce the best available data per contact.

---

## Available Enrichment Tools

### Person / Contact Tools

| Tool name | Provider | Best for |
| --- | --- | --- |
| `aiark_people_search` | AI ARK | Search 500M+ person profiles by name, title, seniority, department, company, location |
| `aiark_reverse_people_lookup` | AI ARK | Reverse lookup by email or phone → full person profile |
| `aiark_mobile_phone_finder` | AI ARK | Mobile phone from LinkedIn URL or name+domain |
| `hunter_email_finder` | Hunter | Work email from first name + last name + domain/company |
| `hunter_email_enrichment` | Hunter | Profile enrichment from an email address or LinkedIn handle |
| `hunter_combined_enrichment` | Hunter | Person + company enrichment from an email address |
| `contactout_linkedin_profile` | ContactOut | Full profile + email + phone from a LinkedIn URL (costs email + phone credit) |
| `contactout_email_enrich` | ContactOut | Emails + profile from an email address |
| `contactout_people_enrich` | ContactOut | Profile enrichment from name + company/domain |
| `contactout_people_search` | ContactOut | Search contacts by name, title, company, location |
| `contactout_contact_info` | ContactOut | Get contact info (email, phone) from LinkedIn URL |
| `contactout_decision_makers` | ContactOut | Get decision makers at a company by domain |
| `contactout_email_to_linkedin` | ContactOut | Reverse: email → LinkedIn URL |
| `datagma_find_work_email` | Datagma | Work email from first name + last name + domain/LinkedIn slug |
| `datagma_find_people` | Datagma | People search by name, title, company |
| `datagma_search_phone_numbers` | Datagma | Phone from email or social media URL |
| `datagma_enrich` | Datagma | Person enrichment from email or LinkedIn |
| `datagma_job_change_detection` | Datagma | Detect recent job changes for a contact |
| `dropleads_email_finder` | Dropleads | Work email from first + last name + domain (1 credit valid result, 0 for catch-all) |
| `dropleads_mobile_finder` | Dropleads | Mobile phone from LinkedIn URL (3 credits) |
| `fullenrich_start_enrichment` | FullEnrich | **Bulk** enrichment: work email + personal email + mobile for up to 100 contacts at once |
| `fullenrich_get_enrichment` | FullEnrich | Poll results from a bulk enrichment started with `fullenrich_start_enrichment` |
| `fullenrich_reverse_email` | FullEnrich | Start reverse email enrichment (email → person profile) |
| `fullenrich_get_reverse_email` | FullEnrich | Poll results from a reverse email enrichment |
| `fullenrich_search_people` | FullEnrich | Search people with rich filters |
| `limadata_enrich_person` | LimaData | Full person enrichment from email, LinkedIn URL, or name+company (1 credit + optional work email/phone) |
| `limadata_find_work_email` | LimaData | Work email from name + domain/company |
| `limadata_find_phone` | LimaData | Phone from LinkedIn URL or name + company |
| `limadata_find_profiles` | LimaData | Find LinkedIn profiles from name + company |
| `limadata_prospect_people_search_url` | LimaData | Prospect people using a LinkedIn Sales Navigator URL |
| `limadata_prospect_people_filter` | LimaData | Prospect people with programmatic filters |
| `lusha_search_contacts` | Lusha | Look up contact by LinkedIn URL, email, or name+company — returns preview + available data |
| `lusha_enrich_contacts` | Lusha | Full contact enrichment (emails, phones) from Lusha ID |
| `lusha_search_enrich_contacts` | Lusha | Search + enrich in one call |
| `lusha_prospect_contacts` | Lusha | Prospect-mode contact search with ICP filters |
| `lusha_lookalike_contacts` | Lusha | Find contacts similar to a seed list |
| `lusha_signals_contacts` | Lusha | Contact-level buying signals and intent |
| `pdl_person_enrich` | PDL | Enrich from email, LinkedIn URL, phone, or name+company — 3B+ profiles |
| `pdl_person_search` | PDL | Search people via Elasticsearch or SQL against 3B+ profiles |
| `pdl_person_identify` | PDL | Identify up to 20 probable matches from partial info (confidence scored) |
| `wiza_find_email` | Wiza | Work + personal email from LinkedIn URL or name+domain (async reveal — poll with `wiza_get_reveal`) |
| `wiza_find_phone` | Wiza | Mobile phone from LinkedIn URL or name+domain (async reveal — poll with `wiza_get_reveal`) |
| `wiza_find_profile` | Wiza | Full LinkedIn profile data from LinkedIn URL (async reveal) |
| `wiza_get_reveal` | Wiza | Poll the result of any Wiza reveal |

### Company Tools

| Tool name | Provider | Best for |
| --- | --- | --- |
| `aiark_company_search` | AI ARK | Search 70M+ companies by name, domain, industry, size, revenue, tech stack |
| `contactout_company_search` | ContactOut | Find companies by name, domain, or industry |
| `contactout_domain_enrich` | ContactOut | Enrich a company domain → industry, size, social profiles |
| `fullenrich_search_company` | FullEnrich | Company search with rich filters |
| `limadata_enrich_company` | LimaData | Full company enrichment: funding, traffic, tech stack, socials (1 credit) |
| `lusha_search_companies` | Lusha | Look up company by name or domain |
| `lusha_enrich_companies` | Lusha | Full company enrichment from Lusha ID |
| `lusha_prospect_companies` | Lusha | Prospect-mode company search |
| `lusha_lookalike_companies` | Lusha | Find companies similar to a seed list |
| `lusha_signals_companies` | Lusha | Company-level buying signals and intent |
| `pdl_company_enrich` | PDL | Company enrichment from domain or LinkedIn URL |
| `pdl_company_search` | PDL | SQL/Elasticsearch company search across PDL's database |
| `pdl_ip_enrich` | PDL | Identify company from an IP address (reverse-IP enrichment) |

### Email Verification Tools

| Tool name | Provider | Statuses returned |
| --- | --- | --- |
| `zerobounce_validate_email` | ZeroBounce | `valid`, `invalid`, `catch-all`, `unknown`, `spamtrap`, `abuse`, `do_not_mail` |
| `zerobounce_email_finder` | ZeroBounce | Find + verify a work email from name + domain; returns confidence level |
| `zerobounce_domain_search` | ZeroBounce | All known email formats + contacts for a domain |
| `dropleads_email_verifier` | Dropleads | `valid`, `invalid`, `catch_all`, `unknown` + MX record + provider |
| `contactout_email_verifier` | ContactOut | Email deliverability check |
| `contactout_contact_checker` | ContactOut | Check whether a LinkedIn contact is reachable |

---

## Multi-Agent Enrichment Strategy

For lists of 5+ contacts, **do not enrich serially** — deploy parallel swarm workers, one per provider group, then merge. This maximizes coverage without bottlenecking on any single provider's hit rate.

### Recommended agent decomposition

#### Agent 1 — Profile Discovery
**Goal**: Resolve LinkedIn URLs and basic profiles for all contacts on the list.

Use: `aiark_people_search`, `contactout_people_search`, `limadata_find_profiles`, `pdl_person_identify`

Input: name + company or domain per contact
Output: `{ linkedin_url, full_name, title, company }` per contact

#### Agent 2 — Email Waterfall (Work Email)
**Goal**: Find work emails for all contacts using 4–5 providers in sequence per contact. Stop per contact when one provider returns a verified or high-confidence result.

Sequence: `hunter_email_finder` → `datagma_find_work_email` → `dropleads_email_finder` → `limadata_find_work_email` → `zerobounce_email_finder`

For bulk (20+ contacts): use `fullenrich_start_enrichment` instead — submit all at once, poll with `fullenrich_get_enrichment`.

Input: first_name + last_name + domain per contact (use LinkedIn URL if available to improve rates)
Output: `{ email, provider_source, confidence }` per contact

#### Agent 3 — Personal Email + Mobile Phone
**Goal**: Supplement work email with personal email where needed; find mobile for high-priority contacts.

Email: `wiza_find_email` (set `accept_personal: true`), `contactout_linkedin_profile`
Phone: `aiark_mobile_phone_finder`, `dropleads_mobile_finder`, `wiza_find_phone`, `datagma_search_phone_numbers`, `limadata_find_phone`

Wiza returns reveals asynchronously — start all reveals first, then poll with `wiza_get_reveal`.

#### Agent 4 — Deep Profile Enrichment
**Goal**: Fill in title, seniority, department, employment history, skills, and any missing firmographic data.

Use: `pdl_person_enrich`, `limadata_enrich_person`, `contactout_people_enrich`, `lusha_search_enrich_contacts`

Prefer `pdl_person_enrich` when you have an email or LinkedIn URL — it returns the most complete profile. Use `limadata_enrich_person` as a fallback.

#### Agent 5 — Email Verification
**Goal**: Verify all emails discovered in Agents 2 and 3 before any outbound activation.

Run **all emails through ZeroBounce first** — it returns 7 distinct statuses and is the most thorough. Confirm with `dropleads_email_verifier` for any `catch-all` results where you need a second opinion.

Rules:
- ✅ `valid` → safe to send
- ⚠️ `catch-all` → use with caution; confirm with `dropleads_email_verifier`
- ❓ `unknown` → do not send; try alternative provider or drop
- ❌ `invalid`, `spamtrap`, `abuse`, `do_not_mail` → drop immediately

#### Agent 6 — Company Enrichment
**Goal**: Enrich all companies on the list with firmographics, tech stack, funding, and headcount.

Use: `limadata_enrich_company`, `pdl_company_enrich`, `contactout_domain_enrich`

Run in parallel. Merge, preferring fields with higher confidence. LimaData returns traffic + tech stack. PDL returns the most complete firmographic profile.

---

## Merge and Dedupe Logic

After all agents complete, merge per-contact using this priority order:

| Field | Priority |
| --- | --- |
| `linkedin_url` | ContactOut → AI ARK → LimaData → PDL → Wiza |
| `email` (work) | ZeroBounce-verified first; then highest confidence among providers; drop invalid/spamtrap |
| `email` (personal) | Wiza → ContactOut → FullEnrich |
| `phone` (mobile) | AI ARK → Dropleads → Wiza → LimaData → Datagma |
| `title` | PDL → LimaData → ContactOut → AI ARK |
| `company` firmographics | LimaData → PDL → ContactOut |

Dedupe contacts by: `email` → `linkedin_url` → `(full_name + company_domain)` in that order.

---

## Swarm Request Pattern

When deploying agents for enrichment, structure each swarm worker with a narrow, concrete task:

```json
POST /swarm/deploy/
{
  "query": "Email discovery waterfall for the following contacts: [list of name + domain pairs]. For each contact, try these providers in order and stop when one returns a result: (1) hunter_email_finder with first_name + last_name + domain, (2) datagma_find_work_email with firstName + lastName + company domain, (3) dropleads_email_finder with first_name + last_name + company_domain, (4) limadata_find_work_email with name + company_domain. Return per contact: email found, provider that found it, and confidence or status if available. Leave email blank if all providers return nothing.",
  "swarmSize": 4
}
```

```json
POST /swarm/deploy/
{
  "query": "Mobile phone discovery for the following contacts who have LinkedIn URLs: [list of name + linkedin_url pairs]. Try these providers in parallel per contact: (1) aiark_mobile_phone_finder with linkedin URL, (2) dropleads_mobile_finder with linkedin_url, (3) wiza_find_phone with profile_url — start all Wiza reveals first, then poll each with wiza_get_reveal. Return per contact: phone numbers found, provider source, and LinkedIn URL used.",
  "swarmSize": 3
}
```

```json
POST /swarm/deploy/
{
  "query": "Verify all emails in this list before outbound activation: [list of email addresses]. For each: (1) run zerobounce_validate_email and record the full status and sub-status, (2) for any catch-all results, also run dropleads_email_verifier for a second opinion. Return per email: zerobounce status, dropleads status if run, final send recommendation (✅ safe / ⚠️ caution / ❌ drop), and reason.",
  "swarmSize": 3
}
```

---

## Bulk Enrichment (FullEnrich)

For batches of 20–100 contacts, use FullEnrich instead of per-contact waterfalls:

1. Call `fullenrich_start_enrichment` with up to 100 contacts. Include `linkedin_url` per contact whenever available — it significantly improves hit rates.
2. Specify `enrich_fields`: `["contact.work_emails", "contact.personal_emails", "contact.phones"]`
3. Get back an `enrichment_id`.
4. Poll `fullenrich_get_enrichment` with that ID until the enrichment is complete.
5. Merge FullEnrich results with results from other agents.

FullEnrich costs: work email = 1 credit, personal email = 3 credits, mobile phone = 10 credits per contact found.

---

## Decision Makers at a Company

When the task is "find decision makers at [company domain]", use `contactout_decision_makers` with the domain. This returns pre-ranked contacts by seniority. Supplement with `lusha_prospect_contacts` and `aiark_people_search` filtered by company + department for fuller coverage.

---

## Prospecting with People/Company Search

When you have ICP criteria and need a list of matching contacts or companies:

| Goal | Use |
| --- | --- |
| Search 500M+ people by title, company size, location, department | `aiark_people_search` |
| Search 3B+ people via SQL or Elasticsearch | `pdl_person_search` |
| Prospect contacts with seniority + industry filters | `lusha_prospect_contacts` |
| Find people from LinkedIn Sales Navigator URL | `limadata_prospect_people_search_url` |
| Find companies by size, industry, tech stack, revenue | `aiark_company_search` or `pdl_company_search` |
| Find lookalike contacts to a seed list | `lusha_lookalike_contacts` |
| Find lookalike companies to a seed list | `lusha_lookalike_companies` |

---

## Save to Context Lake

After completing any enrichment run, save a summary:

```json
POST /context/
{
  "serverId": "26",
  "content": "Enrichment waterfall complete. Contacts processed: {{count}}. Emails found: {{count}} ({{verified}} verified, {{catch_all}} catch-all, {{invalid}} dropped). Phones found: {{count}}. Providers used: {{list}}. LinkedIn URLs resolved: {{count}}. Output: {{file path}}.",
  "messageId": "msg-<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
