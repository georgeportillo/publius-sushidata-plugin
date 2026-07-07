---
name: niche-signal-discovery
description: 'Discover niche first-party signals that differentiate Closed Won vs Closed Lost accounts for ICP analysis. Use when the user provides won/lost customer domain lists and wants differential signals (website content, job listings, tech stack, maturity markers) to build account scoring models and prospecting criteria. Triggers: ICP analysis, niche signals, won vs lost analysis, differential signals, signal discovery, ICP signal report, account scoring signals, lead scoring, first-party signals, buyer signals.'
---

# Niche Signal Discovery

Discover differential signals between Closed Won and Closed Lost accounts by extracting multi-page website content and job listings, then computing Laplace-smoothed lift scores to identify what distinguishes buyers from non-buyers.

## Prerequisites

- **Sushidata swarm** — Company research via `/swarm/deploy/` (BASE URL in sushi-research SKILL.md)
- **WebSearch + WebFetch** — Website page discovery and content extraction. Free.
- **Apify MCP tools** — Use exposed Sushidata Apify tools only; do not use dynamic actor discovery.
- **FullEnrich MCP tools** — Email discovery and contact enrichment: `fullenrich_search_people`, `fullenrich_start_enrichment`.
- **Python 3** stdlib only — no pip dependencies for any shipped script.
- **Estimated cost** — ~$0.50–1.00/company (WebFetch free + Apify jobs ~$0.25–0.50 + FullEnrich credits vary by plan). Step 7 contact discovery is additional. **Always get user approval before paid steps.**

## Input requirements

- Won and lost customer domain lists (≥20 won + ≥10 lost for statistical significance)
- **Lookalikes can supplement Won** if Closed Won < 15. Add a Dataset Caveat to the report.
- **Target company context** from Step 0 — what they sell, who they sell to, key personas.

## Pipeline

```
0.    Discover target company (what they sell, who they sell to) via Sushidata swarm
0.5.  Discover ecosystem (competitors, tech stack, buyer personas) via Sushidata swarm
1.    Prepare input CSV (deduplicate within won/lost groups)
1.0.5 Build "do not re-contact" index from user's existing list (scripts/dedupe_utils.py)
1.5.  Generate vertical-specific configs (keywords, tools, job roles)
2.    Multi-page website + job extraction (WebFetch + Apify)
3.    Quality gate — verify file completeness + coverage (>80%)
3.5.  Review configs against enriched data
4.    Differential analysis (scripts/analyze_signals.py)
5.    Generate report — every top signal must include cited evidence
6.    Signal interpretation review
7.    Top 10 net-new prospects [REQUIRED] + contacts/emails [optional, costs credits]
```

**Step 7 is required.** A signal report without 10 actionable companies forces the reader to do their own prospecting pass — exactly the expensive thing they wanted to skip. Contacts/emails are optional only because they cost extra; always offer them.

## Signal reliability hierarchy

Highest → lowest confidence:

1. **Job listings** — active budget + acknowledged pain. Highest-intent.
2. **Analyst validation** (Gartner/Forrester) — typically 4-7x lift, rare in lost.
3. **Compliance infrastructure** (SOC2/GDPR/ISO) — procurement maturity.
4. **Buyer pain language** on careers/blog — operational awareness.
5. **Tech stack tools** (niche SaaS) — infrastructure readiness.
6. **Website product/marketing content** — variable; can be buyer OR competitor.

**When website signals fail:** For B2B back-office tools (AR, billing, compliance), buyers don't publish their pain on marketing pages. Prioritize jobs + tech stack + firmographics for these verticals.

## What NOT to use for scoring

CRM fields populated by AE activity — catalyst note count, OCR-derived counts, MEDDPICC picklists, any "did the AE do X on this opp" field — correlate with win-rate as **engagement artifacts, not causal signals**. They get filled in _after_ the AE decides an opp is worth working. **Never use them as scoring inputs.**

Rule of thumb: every scoring input must be observable BEFORE the AE touches the account. Read `references/scoring-pitfalls.md` for the full list.

## Step 0: Target company discovery

**Do this FIRST.** The entire pipeline adapts based on this discovery; skipping it produces generic/irrelevant signals.

Deploy a Sushidata research swarm:

```json
POST /swarm/deploy/
{
  "query": "Research {{company-domain}}. Summarize: (1) what the company sells, (2) who they sell to and what buyer personas, (3) what makes them different from competitors, (4) example customers. Be specific — avoid generic descriptions.",
  "swarmSize": 3
}
```

Document: (1) product category, (2) target buyer persona, (3) key differentiation, (4) example customers.

## Step 0.5: Ecosystem discovery

Three Sushidata swarms (swarmSize 3 each) or WebSearch queries:

- **Competitors**: `"{{product category}} software alternatives competitors site:g2.com OR site:capterra.com"` → 3-5 names
- **Tech stack**: `"{{buyer persona}} software stack tools"` → 10-15 tools by category
- **Job roles**: `"{{buyer persona}} job titles seniority VP director manager"` → 10-15 title variations

These feed Step 1.5 config generation.

## Step 1: Prepare input CSV

```csv
domain,status
customer1.com,won
non-customer1.com,lost
```

**Deduplicate within the input.** If a domain appears in BOTH won and lost, remove ALL rows for cross-group domains before enrichment:

```python
from collections import Counter
counts = Counter(r['domain'] for r in rows)
duplicate_domains = {d for d, c in counts.items() if c > 1}
# Drop every row in duplicate_domains, not just one copy.
```

## Step 1.0.5: Build "do not re-contact" index

Before any prospects ship in Step 7, dedupe candidates against whatever "already known" list the user provides. **Always ask explicitly**; if the user has no list, note it as a caveat in the final report.

**Order: apex domain first, fuzzy company name as fallback.** Use the shipped helper:

```bash
python3 scripts/dedupe_utils.py --selftest   # one-time sanity check
python3 scripts/dedupe_utils.py \
    --existing customers.csv --candidates prospects_raw.csv \
    --out-actionable prospects_actionable.csv --out-matched already_known.csv
```

Don't silently drop matches — **categorize** them: Net-new / Account-only / Re-engage / Active-open / Current-customer.

**Read `references/dedupe.md`** for the failure modes (raw-string match missing `amsynergy.nikon.com → nikon.com` cost 24 of 50 prospects in one run).

## Step 1.5: Generate vertical-specific configs

Create three JSON files in `output/{{company}}/`:

```
{{company}}-keywords.json    # product category, pain language, competitor names, maturity terms
{{company}}-tools.json       # niche SaaS tools by category
{{company}}-job-roles.json   # buyer persona job titles
```

**Read `references/keyword-catalog.md`** for the JSON schema, generation patterns, and multi-vertical examples.

## Step 2: Website + Job extraction

**Never scrape just the homepage.** Discover multiple relevant pages first, then extract content.

**Step 2a — Discover pages (WebSearch):**

```
WebSearch: site:{{domain}} product OR features OR integrations OR customers OR security OR pricing OR careers OR about
```

Collect the top 5-8 URLs. Adapt by vertical: add `compliance OR audit` for back-office, `documentation OR api` for developer tools.

**Step 2b — Scrape pages (WebFetch):**

For each discovered URL, call WebFetch to extract the page content. If WebFetch returns a shell with no real content (JavaScript-rendered page), use Browser Rendering. If a dedicated web crawler actor is required, follow the missing-actor feedback workflow in `skills/sushi-research/provider-playbooks/apify.md`.

Aggregate all page text into a per-domain JSON record stored in the `website` CSV column:

```json
{
  "data": {
    "results": [
      {"url": "https://example.com/pricing", "title": "Pricing", "text": "<page text>"},
      {"url": "https://example.com/security", "title": "Security", "text": "<page text>"}
    ]
  }
}
```

**Step 2c — Job listings:**

Use WebSearch, WebFetch, Browser Rendering, and focused Sushidata swarms to collect job listings from company career pages and public job boards. Sushidata does not currently expose a general job-listings Apify actor. If a dedicated jobs actor is required, follow the missing-actor feedback workflow in `skills/sushi-research/provider-playbooks/apify.md`.

Store results in the `jobs` CSV column as:
```json
{"result": {"listings": [{"title": "...", "description": "...", "url": "..."}]}}
```

**Total estimated cost: ~$0.50–1.00/company.** Get user approval. Example: "60 companies × ~$0.75 = ~$45."

## Step 3: Quality gate

Verify CSV completeness before running the analysis script:

```bash
INPUT_ROWS=$(wc -l < output/{{company}}-icp-input.csv)
OUTPUT_ROWS=$(wc -l < output/{{company}}-enriched.csv)
echo "Input: $INPUT_ROWS, Output: $OUTPUT_ROWS"  # should match
```

Then spot-check: won rows have job data, website coverage >80%, avg content depth 6-8 pages / 12-20K chars per company.

**Read `references/quality-gate.md`** for the full verification script and the "auto-extracted domain validation" check that has caught up to **53% false-positive rates** in CRM-exported customer lists.

## Step 3.5: Review configs against enriched data

Open the enriched CSV and inspect a sample of 5-10 rows. Check that website columns contain multi-page content and job listings are present for ≥60% of won accounts.

**Red flags:**
- Keyword in <10% of enriched companies → too niche, broaden
- Keyword in >90% → too generic, refine
- Product-category keywords appear frequently in Won → those companies are competitors not buyers
- Job roles missing from actual listings → wrong buyer persona

Fix and regenerate configs if needed.

## Step 4: Differential analysis

The `analyze_signals.py` script expects a CSV with `website` and `jobs` columns (JSON-formatted as in Step 2). Use `--website-col` and `--jobs-col` to pass column indices explicitly:

```bash
python3 scripts/analyze_signals.py \
  --input output/{{company}}-enriched.csv \
  --keywords output/{{company}}-keywords.json \
  --tools output/{{company}}-tools.json \
  --job-roles output/{{company}}-job-roles.json \
  --website-col <N> \
  --jobs-col <M> \
  --output output/{{company}}-analysis.json
```

The script computes substring-match presence, Laplace-smoothed lift, source breakdown (website/jobs/both), tech-stack mentions, job-role prevalence, anti-fit signals, and **per-keyword evidence quotes** (±40 chars with URLs).

## Step 5: Report generation

**Read `references/report-template.md`** for the full report structure. Critical rules:

- Raw counts always (`15% (6)`, not just `15%`); sample sizes in headers (`Won (n=37)`)
- Bold only signals with lift > 2x AND count ≥ 3 companies
- Flag n=1 signals — statistically meaningless
- **Source evidence is mandatory for every top signal** — 3-5 cited quotes per signal with source type, company, page/job title, ±40-char quote, and live URL
- Annotate each evidence quote with ✅ (clear buyer signal) or ⚠️ (vendor-adjacent)

## Step 6: Signal interpretation

**Read `references/signal-interpretation.md`** before writing interpretation columns.

## Step 7: Top 10 net-new prospects (required)

**10 companies are required for every run; contacts + emails are optional.** Always offer contact discovery; only run it if the user approves the spend.

**Companies only (no extra cost):**

Deploy a Sushidata research swarm to find ICP-matching prospects:

```json
POST /swarm/deploy/
{
  "query": "Find 15 companies that match this buyer profile: {{top 3 signals from analysis}}. Target vertical: {{vertical}}. Headcount: {{range}}. Exclude: {{input list domains}}. For each company: name, domain, which signals they exhibit, estimated headcount, why they're a fit.",
  "swarmSize": 8
}
```

Then dedupe against the user's existing list and deliver the top 10:
```bash
python3 scripts/dedupe_utils.py \
    --existing customers.csv --candidates prospects_raw.csv \
    --out-actionable prospects_actionable.csv --out-matched already_known.csv
```

**Companies + contacts + emails (FullEnrich credits required):**

Ask for approval, then for each of the top 10 companies:

1. **FullEnrich people search** — `fullenrich_search_people current_company_domains=[{"value":"{{domain}}"}]` → returns contacts with titles
2. **Filter by persona** — Keep only titles matching buyer persona job titles from Step 0.5
3. **FullEnrich email finder** — `fullenrich_start_enrichment` with first name + last name + domain (or linkedin_url), then poll `fullenrich_get_enrichment`
4. **FullEnrich quality check** — review confidence scores from `fullenrich_get_enrichment` — treat low-confidence or blank results as non-send → mark "(email not found)"
5. **LinkedIn supplement** — if FullEnrich returns <2 contacts at a company, use WebSearch, Browser Rendering, and focused Sushidata swarms for role discovery. If bulk LinkedIn employee scraping is required, follow the missing-actor feedback workflow in `skills/sushi-research/provider-playbooks/apify.md`.

**Read `references/step-7-prospects.md`** for the required output fields, prospect-card skeleton, and the "10 is a ceiling, not a floor" guidance.

## Common pitfalls (top 6 — full list in references/pitfalls.md)

1. **Skipping target discovery (Step 0)** → generic/irrelevant configs.
2. **Homepage-only scraping** → misses pricing, integrations, security, careers.
3. **Generic tech stack** ("AWS", "GitHub", "Slack" appear on most B2B sites) → search for niche SaaS specific to the buyer persona.
4. **Trusting n=1 signals** → require 3+ companies for Tier 1 scoring.
5. **Raw-string dedupe missing parent domains** — `amsynergy.nikon.com ≠ nikon.com`. Always use `extract_apex()` from `scripts/dedupe_utils.py`.
6. **Trusting confirmation-biased CRM fields** — they're downstream of AE engagement, not causal. Read `references/scoring-pitfalls.md`.

## References

- **`references/keyword-catalog.md`** — JSON schema + multi-vertical examples for Step 1.5
- **`references/dedupe.md`** — Step 1.0.5 dedupe failure modes, categorization rules, library usage
- **`references/quality-gate.md`** — Step 3 verification scripts, auto-extracted-domain validation
- **`references/report-template.md`** — Step 5 full report structure, signal-strength scale, all quality rules
- **`references/signal-interpretation.md`** — Step 6 buyer-vs-seller-vs-competitor rules
- **`references/step-7-prospects.md`** — Step 7 prospect-card skeleton, apex validation
- **`references/scoring-pitfalls.md`** — Confirmation-biased CRM fields to exclude from scoring
- **`references/pitfalls.md`** — Full 18-item pitfalls list
- **`references/proven-signals.md`** — Typical lift ranges + 0-100 scoring model guidance
- **`scripts/analyze_signals.py`** — Step 4 differential analysis. Use `--website-col`/`--jobs-col` for column positions.
- **`scripts/dedupe_utils.py`** — Step 1.0.5 deduplication. `extract_apex()`, `norm_name()`, `match_against_existing()`. Stdlib only. `--selftest` for one-time verification.

---

## Save to Context Lake

Save signal analysis results at two points:

**After Step 5 (report complete):**
```json
POST /context/
{
  "serverId": "26",
  "content": "ICP signal analysis complete — {{target company}}. Dataset: {{won_count}} won + {{lost_count}} lost. Top 3 positive signals: {{signal 1 (lift)}} / {{signal 2 (lift)}} / {{signal 3 (lift)}}. Top anti-fit signals: {{list}}. Scoring model: 60+ = Tier 1, 35-59 = Tier 2, <35 = nurture. Full analysis JSON: {{output path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

**After Step 7 (prospects delivered):**
```json
POST /context/
{
  "serverId": "26",
  "content": "Top 10 prospects delivered for {{target company}} ICP. Companies: {{list of domains}}. Contacts found: {{count}}. Dedupe status: {{count net-new vs already-known}}. Signals used for scoring: {{top 3 signals from report}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
