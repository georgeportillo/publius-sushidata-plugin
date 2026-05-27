# Google Ads Transparency Playbook

Use this playbook when a Sushidata research or GTM task needs competitor paid ad intelligence from Google's Ads Transparency Center — creative formats, longevity, recency, and messaging patterns.

Claude should not behave like a raw Ads Transparency operator. Claude should package the research need into a Sushidata swarm request, using the constraints below so Sushidata can execute the right work and return usable results.

---

## How Ads Transparency Works Inside Sushidata

Sushidata's `chainfuse-api` MCP server exposes Google Ads Transparency through `AdsTransparencyTool`.

| Capability | Sushidata tool name | Use for |
|---|---|---|
| Google ad creative search | `ads_transparency` | Find ad creatives associated with a domain from Google's Ads Transparency Center |

**Input:** `domain` — a valid domain string (e.g. `"salesforce.com"`, `"hubspot.com"`). No protocol, no path, no `www.` prefix.

**Output fields per creative:**
- `position` — ranking position in results
- `id` — Google creative ID
- `target_domain` — the domain the ad targets
- `advertiser.name` / `advertiser.id` — the registered advertiser
- `first_shown_datetime` / `last_shown_datetime` — ISO timestamps of ad activity window
- `total_days_shown` — total number of days the creative has run
- `format` — ad format code: `"1"` = Image, `"2"` = Video, `"3"` = HTML/Responsive
- `img_src` — image URL if available
- `details_link` — preview URL if available

**Important constraints:**
- One call per domain — do not batch multiple domains in a single tool call
- Returns up to 100 creatives per call (first page)
- Google Search and Display Network only — LinkedIn Ads, Meta Ads, and DSP placements are not included
- `img_src` and `details_link` may be null for some creatives — this is normal
- `search_information.total_results` reflects Google's total index count, not just the returned page

---

## Agent Behavior

Claude should infer what the user needs, then ask Sushidata to run `ads_transparency` through a swarm.

Do not ask the user to choose a tool or format. If multiple competitor domains are needed, include all domains in the swarm request and ask Sushidata to analyze and compare results across them.

Do not improvise unavailable capabilities. `ads_transparency` covers Google only. If the user needs LinkedIn Ads data, use `apify_linkedin_ad_library_scraper` (see `apify.md`). If the capability is missing entirely, name it and help the user send feedback to `support@sushidata.com`.

---

## Swarm Request Pattern

When Ads Transparency data is needed, deploy a Sushidata swarm with a concrete task description. Include:

- target domains to search
- what intelligence to extract (creative count, longevity, format mix, messaging themes)
- how to classify paid maturity
- output schema

**Example — single competitor:**

```json
POST /swarm/deploy/
{
  "query": "Use the ads_transparency tool to research [competitor_domain]'s Google ad activity. For each creative returned: note the format (1=image, 2=video, 3=HTML), first_shown and last_shown dates, total_days_shown, and img_src or details_link if available. Then produce: (1) total creatives on record, (2) count of active creatives with last_shown within the last 30 days, (3) format mix breakdown, (4) list of evergreen creatives with total_days_shown > 90 and their messaging if img_src is available, (5) paid maturity classification — Active & Scaled (50+ creatives, recent, mixed formats) / Active & Early (<20, recent, mostly image) / Paused (last_shown >90 days ago) / Minimal (<5 results). Save findings to context lake.",
  "swarmSize": 3
}
```

**Example — multi-competitor comparison:**

```json
POST /swarm/deploy/
{
  "query": "Use ads_transparency to research paid ad activity for these competitors: [domain_a], [domain_b], [domain_c]. For each: total creatives, active count (last 30 days), format mix, longest-running creative, and paid maturity classification. Then compare: which competitor has the most sustained paid investment? Which has the freshest creative? Which appears to have paused? Return a structured comparison table plus a summary of messaging themes where img_src data is available.",
  "swarmSize": 5
}
```

---

## How to read the output

### Volume signal
`search_information.total_results` indicates sustained paid investment. 100+ creatives = active, scaled program. <10 = early stage, paused, or primarily other channels.

### Longevity signal
`total_days_shown > 90` = evergreen creative. Companies don't run underperforming ads for 90+ days. These creatives contain their most validated messaging — note format, CTA language, and landing page offer type.

### Recency signal
`last_shown_datetime` within 30 days = currently active. More than 90 days ago = likely paused or shifted budget.

### Format mix
Count by format code. Mixed `"1"` + `"2"` signals mature paid program. Only `"1"` = performance-only or early stage. `"3"` = Display Network / responsive investment.

### Paid maturity classification

| Classification | Signal |
|---|---|
| Active & Scaled | 50+ total, recent last_shown, mixed formats |
| Active & Early | <20 total, recent last_shown, mostly image |
| Paused or winding down | last_shown >90 days ago |
| Minimal paid presence | <5 total results |

---

## GTM competitor report integration

When building a GTM competitor report (see `recipes/gtm-competitor-report.md`), the Paid Advertising section should be populated from this playbook. Use the swarm output to populate:

- Total creatives on record and active count
- Format mix
- Evergreen creative descriptions and messaging themes
- Paid maturity classification
- Notable shifts in format or recency vs. prior research

If `total_results` is 0, note it explicitly: *"No Google ad creatives on record — company may rely on organic, LinkedIn Ads (see apify.md), or other paid channels not visible in Google Ads Transparency."*

---

## Common pitfalls

- **Don't include protocol or path in domain** — `"hubspot.com"` not `"https://www.hubspot.com/products"`
- **Don't treat 0 results as no paid activity** — the company may be active on LinkedIn Ads, Meta, or programmatic channels
- **Don't skip the maturity classification** — raw creative counts without context aren't useful in a report
- **Don't claim an ad is "active" without checking last_shown_datetime** — total_results can be high even for a company that paused paid 6 months ago
- **Don't paginate unless asked** — for most GTM research, the first 100 creatives are sufficient
