# Changelog

All notable changes to the sushidata-gtm plugin are documented here.

---

## [0.4.8] — 2026-06-18

### Added

**New Apify capability: Google Maps Email Extractor (`apify_google_maps_contact_details`)**

Adds `lukaskrivka/google-maps-with-contact-details` as a new Apify-backed tool. Scrapes Google Maps places and extracts contact details from their websites: email addresses, phone numbers, and social media links. Supports keyword + location search, direct Maps URLs, star-rating and website filters, and an optional per-place employee leads enrichment add-on.

Updated files:
- `sushi-research/provider-playbooks/apify.md` — added capability table row, Capability Selection Guide row, and full Google Maps Contact Details workflow section with three example patterns (keyword search, direct URL, leads enrichment)

---

## [0.4.7] — 2026-06-09

### Changed

**Removed `/swarm/summary/` endpoint — synthesize directly from `/swarm/status/` outputs**

`/swarm/summary/` has been removed from the workflow. It was slow and added unnecessary latency. Claude now synthesizes results directly from the `output` fields collected on completed workers during the `/swarm/status/` polling loop — no extra API call required.

Updated files:
- `sushi-research/SKILL.md` — replaced section 5 (swarm/summary) with a direct synthesis step; updated polling rules and decision flow diagram
- `sushi-research-quickstart/SKILL.md` — updated post-poll step to synthesize from worker outputs
- `sushi-sales-quickstart/SKILL.md` — updated ICP prospecting recipe to synthesize from worker outputs
- `sushi-sales-onboarding/SKILL.md` — updated dependencies table and polling step
- `sushi-research/recipes/gtm-competitor-report.md` — removed swarm/summary call, replaced with synthesis step

---

## [0.2.1] — 2026-05-20

### Added

**New skills**

- `sushi-sales-quickstart` — Three hardcoded live-demo recipes: ICP prospecting (swarm → Hunter email waterfall → HeyReach offer), single-account deep dive (swarm → org chart → account brief), and competitor displacement (swarm → tier scoring → contacts for Tier 1). Uses Sushidata API endpoints throughout. Every recipe saves its request and results to the context lake.

- `sushi-research-quickstart` — Competitive intelligence entry point for new users. Given a company name, deploys a Sushidata research swarm to identify and profile at least three competitors, then delivers a structured battlecard. Designed to demonstrate Sushidata value in under five minutes.

- `niche-signal-discovery` — Sushidata-native niche signal discovery skill. Discovers differential signals (website content, job listings, tech stack, maturity markers) between Closed Won and Closed Lost accounts using Laplace-smoothed lift scores. Uses Sushidata swarm, WebSearch/WebFetch, and Apify MCP tools. Includes two pure-Python stdlib scripts (`analyze_signals.py`, `dedupe_utils.py`) and nine reference documents.

**New recipes (inside `sushi-research`)**

- `gtm-competitor-report.md` — End-to-end workflow for building a GTM competitor analysis document: context lake query, swarm research, LinkedIn Ads JSON parsing, channel-by-channel document writing, source verification loop (event sponsorship tiers, analyst citation types, job posting URLs, LinkedIn/X post URLs), structural review checklist, and context lake save. Derived directly from the ZeroFox GTM analysis session.

- `document-accuracy-review.md` — Verification workflow wrapping the `document-accuracy-review` skill with Sushidata context lake queries, WebFetch link verification, client-rendered page escalation (Claude in Chrome), link classification system (verified / paywalled / auth-gated / unverifiable / broken), factual claim verification by domain (analyst reports, personnel, events, metrics), and per-document-type error patterns.

- `scheduled-tasks.md` — Playbook for setting up recurring and one-time automated GTM tasks via Cowork's scheduler (cron and fire-at patterns).

**New job doc**

- `jobs/writing-outreach.md` — Sushidata outreach-writing workflow. Uses a Python loop with the Anthropic SDK (`claude-haiku-4-5-20251001`) for per-row copy generation. Covers one-shot emails, four-step sequences, ICP tier classification, multi-criteria lead scoring, and HeyReach activation. Includes context lake saves for research batch and scored results.

### Changed

**`sushi-research` SKILL.md**
- BASE URL updated to `https://dashboard.sushidata.ai/public/019dff6e-988f-71e2-8aa0-1be949e8421b/`
- Tenant updated to `Sushidata`, Dataspace set to `Sushidata Internal`
- Frontmatter description expanded to cover: community signals, campaign performance, competitor battlecards, GTM competitor reports, outreach copy, and document accuracy review
- Recipes routing table updated with two new entries: `gtm-competitor-report.md` and `document-accuracy-review.md`

**Context lake audit — all actionable docs**
- Added `POST /context/` save blocks to every recipe, playbook, quickstart, and job doc
- Each save is tailored to that workflow's output (domain lists, contact counts, campaign IDs, actor names, etc.)
- Pattern: save the user's request before starting, save results with evidence links after completing

**`niche-signal-discovery` reference docs**
- `keyword-catalog.md`: prompt examples use `Sushidata swarm:` equivalents
- `quality-gate.md`: enrichment timing guidance uses "WebFetch and Apify actor runs complete synchronously..."
- `pitfalls.md`: buffer flush note updated for Sushidata tooling

**`plugin.json`**
- Added `homepage`, `repository`, and `license` fields (required for validation)
- Version bumped to `0.2.1`

### Fixed

- Plugin ZIP structure: previously packed with a `sushidata-gtm/` wrapper directory, causing validation failure. Rebuilt so `.claude-plugin/plugin.json` and `skills/` are at the root of the ZIP.

---

## [0.1.0] — 2026-05-19

### Added

Initial plugin build for Sushidata GTM workflows.

**`sushi-research` skill** — Core research and GTM execution skill. Covers the full Sushidata API workflow: `/context/` (save), `/query/` (fast lookup), `/swarm/deploy/` + `/swarm/status/` + `/swarm/summary/` (parallel research), `/verify/` (link validation). Includes decision flow, context saving rules, and transparency guidelines.

**Provider playbooks**
- `hunter.md` — domain search → email finder → email verify chain
- `heyreach.md` — LinkedIn campaign insertion (≤50 contacts per call, list-first pattern)
- `hubspot.md` — contact/company/note upsert patterns, deal creation
- `apify.md` — curated Sushidata MCP actor tools exposed by `ApifyTools`

**Recipes**
- `account-orgchart.md` — Map decision makers and warm intro paths for a target account
- `build-tam.md` — Build a total addressable market list from ICP criteria
- `linkedin-url-lookup.md` — Resolve LinkedIn profile URLs from names and companies
- `portfolio-prospecting.md` — Find companies backed by a specific investor, then find contacts
- `small-business-prospecting.md` — Find local small businesses using Maps-style search

**Validation scripts**
- `validate-emails.py` — Flag rows where enriched email domain doesn't match company domain
- `validate-linkedin-names.py` — Validate scraped LinkedIn profile names against source names (handles accents, hyphenated names, 50+ nickname pairs, initials); includes eval mode against 52 fixture test cases

**`finding-companies-and-contacts.md`** — Phase doc for the companies-first discovery order, over-provisioning rule, and approval gate for paid actions.
