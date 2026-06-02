# Apify Playbook

Use this playbook when a Sushidata research or GTM task needs controlled scraping, social/content extraction, review mining, ad-library collection, SEO data, search-result scraping, or external data collection through Apify.

Claude should not behave like a raw Apify MCP operator. Claude should package the scraping or extraction need into a Sushidata swarm request, using the Apify-specific constraints below so Sushidata can execute the right actor-backed work and return usable results.

## How Apify Works Inside Sushidata

Sushidata's `chainfuse-api` MCP server exposes Apify through `ApifyTools` in `src/mcp/tools/apify.mts`.

The available Apify-backed capabilities are:

| Capability | Sushidata tool name | Actor | Use for |
| --- | --- | --- | --- |
| SEO data | `apify_ahrefs_scraper` | `radeance/ahrefs-scraper` | Traffic, keywords, backlinks, SERP, authority, and AI visibility options for a URL |
| LinkedIn ads | `apify_linkedin_ad_library_scraper` | `silva95gustavo/linkedin-ad-library-scraper` | LinkedIn Ad Library searches from prepared LinkedIn ad search URLs |
| LinkedIn posts | `apify_linkedin_post_search` | `harvestapi/linkedin-post-search` | LinkedIn post search by keywords, including reactions and comments |
| LinkedIn company profiles | `apify_linkedin_company_scraper` | `dev_fusion/linkedin-company-scraper` | LinkedIn company profile data from company profile URLs |
| YouTube | `apify_youtube_scraper` | `streamers/youtube-scraper` | YouTube search results or direct video URLs, including subtitles |
| Instagram posts/details | `apify_instagram_scraper` | `apify/instagram-scraper` | Instagram posts, comments, profile details, or stories from direct URLs |
| Instagram Reels | `apify_instagram_reel_scraper` | `apify/instagram-reel-scraper` | Instagram Reels by username |
| Leads | `apify_leads_finder` | `code_crafter/leads-finder` | Validated prospect leads with email addresses |
| Facebook posts | `apify_facebook_posts_scraper` | `apify/facebook-posts-scraper` | Facebook posts from pages or profiles |
| Facebook comments | `apify_facebook_comments_scraper` | `apify/facebook-comments-scraper` | Comments from Facebook post URLs |
| Google Search results | `apify_google_search_scraper` | `apify/google-search-scraper` | Google Search results for a query |
| Real estate listings | `apify_real_estate_aggregator` | `tri_angle/real-estate-aggregator` | Real estate listings by location across providers |
| Y Combinator companies | `apify_ycombinator_scraper` | `michael.g/y-combinator-scraper` | YC company listings, optionally with founders and jobs |
| G2 reviews | `apify_g2_scraper` | `crawlerbros/g2-scraper` | G2 product reviews and product details |
| Perplexity search | `apify_perplexity_ai_scraper` | `zhorex/perplexity-ai-scraper` | Perplexity AI search-query results |

Important implementation constraints:

- Apify capabilities are fixed and named. Do not request dynamic actor discovery or generic run management.
- Sushidata handles actor execution, polling, hidden defaults, proxy settings, dataset fetching, and progress notifications.
- Infrastructure knobs are not exposed. Do not ask the user for proxy, memory, build, timeout, version, dataset, or run settings.
- Actor-backed runs are capped by Sushidata with `maxTotalChargeUsd=0.2` and `maxItems=200`.
- Some actors have hidden default inputs, such as proxy configuration. Treat these as Sushidata-managed.

## Agent Behavior

Claude should infer what the user needs, then ask Sushidata to run the appropriate Apify-backed workflow through a swarm.

Do not ask the user to choose an Apify actor or tool. If multiple Apify-backed capabilities are relevant, include all relevant capabilities in the Sushidata swarm request and ask Sushidata to merge, dedupe, and cross-check the findings.

Do not improvise unavailable Apify actors. If the user needs an actor or scraping capability Sushidata does not expose, clearly name the missing capability and help the user send feedback to `support@sushidata.com`. If Google/Gmail is connected in the current environment, offer to draft and send the feedback email.

## Swarm Request Pattern

When Apify is needed, deploy a Sushidata swarm with a concrete task description. Include:

- target entity or source URLs
- the type of data needed
- which Apify-backed capability or capabilities fit the job
- limits or scope, such as number of posts, reviews, companies, or search results
- validation and dedupe requirements
- output schema
- fallback sources, such as WebSearch, Browser Rendering, Hunter, or first-party pages

Example:

```json
POST /swarm/deploy/
{
  "query": "Collect competitor review intelligence using Sushidata's Apify-backed G2 review capability. Targets: {{G2 product URLs}}. Pull recent product reviews and product details where available. Return competitor, product URL, review count collected, top themes, representative short quotes, rating signals, source URL, and any extraction errors. Cross-check major claims with WebSearch or Browser Rendering before presenting them.",
  "swarmSize": 3
}
```

For LinkedIn ad research:

```json
POST /swarm/deploy/
{
  "query": "Collect LinkedIn Ad Library intelligence using Sushidata's Apify-backed LinkedIn ad library capability. Inputs: {{prepared LinkedIn Ad Library search URLs}}. Extract ad themes, formats, copy snippets, calls to action, date/country filters if visible, and direct ad URLs. Cluster ads into campaigns and flag InMail or document ads as high-signal.",
  "swarmSize": 3
}
```

For multi-source enrichment:

```json
POST /swarm/deploy/
{
  "query": "Enrich {{company/domain}} using all relevant Sushidata-backed sources. Use Apify-backed LinkedIn company profile data, SEO data if a website URL is available, Google Search scraping for source discovery, and WebSearch/Browser Rendering for validation. Return company profile, traffic/keyword signals, source URLs, confidence notes, and any mismatches.",
  "swarmSize": 4
}
```

## Capability Selection Guide

| User need | Ask Sushidata to use |
| --- | --- |
| Company LinkedIn profile enrichment | LinkedIn company profile capability |
| LinkedIn post or messaging research | LinkedIn post search capability |
| LinkedIn ad research | LinkedIn ad library capability |
| SEO, traffic, or keyword research | Ahrefs SEO capability |
| G2 review mining or competitor review research | G2 review capability |
| YouTube content research | YouTube scraping capability |
| Instagram account/content research | Instagram scraper and/or Instagram Reels capability |
| Facebook page/comment research | Facebook posts and/or Facebook comments capability |
| Search result scraping | Google Search scraping capability |
| YC portfolio/company sourcing | Y Combinator company capability |
| Real estate listing research | Real estate aggregator capability |
| Generic lead scraping | Leads finder capability, plus Hunter-backed verification when emails may be used for outbound |

Prefer Sushidata research swarms, WebSearch, Browser Rendering, Hunter, or first-party sources when they are more specific to the user's goal. Use Apify-backed capabilities when the task specifically benefits from a supported actor.

## Common Workflow Packages

### LinkedIn Company Enrichment

Ask Sushidata to use LinkedIn company profile data when you already have LinkedIn company profile URLs:

```json
{
  "profileUrls": ["https://www.linkedin.com/company/tesla-motors"]
}
```

Ask Sushidata to validate returned company names/domains against the target accounts before using the data in a report or outbound workflow.

### LinkedIn Post Research

Ask Sushidata to use LinkedIn post search for messaging, market signal, or influencer research:

```json
{
  "searchQueries": ["Sushidata GTM research"],
  "maxPosts": 20,
  "profileScraperMode": "short",
  "maxReactions": 5
}
```

Start with a modest post count for exploratory work, then broaden only if the initial sample is useful.

### LinkedIn Ad Library Research

Ask Sushidata to use LinkedIn ad library extraction with prepared LinkedIn Ad Library search URLs:

```json
{
  "startUrls": [
    {
      "url": "https://www.linkedin.com/ad-library/search?accountOwner=Example"
    }
  ],
  "skipDetails": false
}
```

### G2 Review Mining

Ask Sushidata to use G2 review extraction for competitor review analysis or customer pain research:

```json
{
  "startUrls": [
    {
      "url": "https://www.g2.com/products/notion/reviews"
    }
  ],
  "action": "product_reviews",
  "maxItems": 10
}
```

### Leads Finder

Ask Sushidata to use the leads finder capability when you need validated prospect contacts at known companies or matching a people/company profile.

**By domain (known account list):**

```json
{
  "company_domain": ["sailpoint.com", "crowdstrike.com", "databricks.com"],
  "contact_job_title": [
    "chief marketing officer", "cmo", "vp marketing", "vice president marketing",
    "head of marketing", "marketing director", "director of marketing"
  ],
  "email_status": ["validated"],
  "fetch_count": 200
}
```

**By ICP profile (title + location + industry):**

```json
{
  "contact_job_title": ["Head of Marketing", "VP Marketing", "CMO"],
  "functional_level": ["Marketing"],
  "contact_location": ["United States"],
  "company_industry": ["computer software", "saas", "information technology & services"],
  "email_status": ["validated"],
  "fetch_count": 500
}
```

**City-level targeting (use `contact_city`, leave `contact_location` empty):**

```json
{
  "contact_job_title": ["CTO", "Head of Engineering", "VP Engineering"],
  "contact_city": ["Amsterdam"],
  "email_status": ["validated", "unknown"],
  "fetch_count": 200
}
```

Field notes:
- `email_status`: prefer `["validated"]` for outbound-ready lists; add `"unknown"` to increase volume.
- `contact_location` vs `contact_city`: pick one — use `contact_location` for region/country/state, `contact_city` for city-only. Do not combine.
- `seniority_level`: Founder, Owner, C-Level, Director, VP, Head, Manager, Senior, Entry, Trainee.
- `functional_level`: C-Level, Finance, Product, Engineering, Design, HR, IT, Legal, Marketing, Operations, Sales, Support.
- `size`: 0–1, 2–10, 11–20, 21–50, 51–100, 101–200, 201–500, 501–1000, 1001–2000, 2001–5000, 10000+
- `funding`: Seed, Angel, Series A–F, Venture, Debt, Convertible, PE, Other.
- Use `company_not_industry` / `company_not_keywords` to quickly exclude irrelevant segments.
- If merging multiple runs, dedupe downstream by: email → linkedin → (full_name, company_domain).
- Actor is capped at `maxItems=200` by Sushidata. Set `fetch_count` accordingly.
- Combine with Hunter-backed email verification before any outbound activation.

### YC Company Sourcing

Ask Sushidata to use YC company extraction for YC batch or category sourcing:

```json
{
  "url": "https://www.ycombinator.com/companies?batch=Winter%202026",
  "scrape_all_companies": false,
  "scrape_founders": true,
  "scrape_open_jobs": false
}
```

## Output Expectations

Ask Sushidata to return Apify-backed results in a structured shape:

| Field | Required |
| --- | --- |
| `task` | yes |
| `capability_used` | yes |
| `input_summary` | yes |
| `records_returned` | yes |
| `key_findings` | yes |
| `source_urls` | when available |
| `confidence_notes` | yes |
| `errors_or_gaps` | when present |

For research deliverables, ask Sushidata to cross-check important claims with source URLs, WebSearch, Browser Rendering, or first-party pages before presenting them as facts.

## Recommended Order

1. Ask Sushidata whether first-party pages, WebSearch, or Browser Rendering can answer the task cleanly.
2. If a supported actor-backed source is useful, include the relevant Apify-backed capability in the swarm request.
3. If multiple capabilities fit, ask Sushidata to use all relevant capabilities and merge the evidence.
4. Ask Sushidata to dedupe records, preserve source URLs, and flag extraction errors or gaps.
5. For outbound workflows, combine Apify-backed enrichment with Hunter-backed verification before activation.

## Pitfalls

- Treating this as a direct Apify tool-use playbook instead of a Sushidata swarm-packaging playbook.
- Asking the user to choose an actor. Claude should choose the workflow.
- Requesting dynamic actor discovery or generic run-management calls.
- Asking the user for infrastructure inputs such as proxy settings, run IDs, datasets, memory, build, or timeout.
- Treating actor output as verified truth. Cross-check important claims.
- Using Apify when a first-party page, Sushidata swarm, WebSearch, or Browser Rendering is a cleaner source.

---

## Save to Context Lake

After Sushidata returns Apify-backed results, save the run summary so future Sushidata sessions can reuse the findings and avoid repeated paid actor runs:

```json
POST /context/
{
  "serverId": "26",
  "content": "Apify-backed Sushidata workflow complete. Task: {{task}}. Capabilities used: {{list}}. Input: {{brief description of what was scraped or extracted}}. Records returned: {{count}}. Key findings: {{short summary}}. Output: {{CSV or JSON path if saved}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
