# Web Search Playbook

Use this playbook when you need to search the public web for information not available in Sushidata's context lake or enrichment providers. The `web-search` tool is powered by a private SearXNG instance at `search.sushidata.ai` that aggregates results from multiple search engines.

---

## Available Tools

| Tool name     | What it does                                                                                         |
|---------------|------------------------------------------------------------------------------------------------------|
| `web-search`  | Searches the public web via SearXNG. Returns results, answers, infoboxes, and suggestions.          |

---

## `web-search`

**Parameters:**

| Parameter       | Type     | Default | Description                                                                      |
|-----------------|----------|---------|----------------------------------------------------------------------------------|
| `query`         | string   | —       | The search query (required)                                                      |
| `language`      | string   | —       | Locale filter (e.g. `"en-US"`)                                                   |
| `pageno`        | integer  | `1`     | Page number for pagination                                                       |
| `safesearch`    | `0/1/2`  | `0`     | `0` = off, `1` = moderate, `2` = strict                                         |
| `numberOfResults`| integer | `3`     | Number of results to return (max 5)                                              |

**Example — find pricing page of a competitor:**
```json
{
  "query": "Gong.io pricing plans 2025",
  "numberOfResults": 5
}
```

### Result fields

- `results` — list of web pages with `url`, `title`, `content` snippet, `engine`, `score`
- `answers` — direct answers extracted from search (check first if present)
- `infoboxes` — high-confidence structured answers from trusted sources (check this first — if present, it likely contains the answer)
- `corrections` — query spelling corrections
- `suggestions` — related queries

**Always check `infoboxes` first** — if an infobox is in the response, it is from a high-trust source and likely contains the answer directly.

---

## When to Use `web-search` vs Other Tools

| Task | Prefer |
|------|--------|
| General public information lookup | `web-search` |
| Reading page content after finding URL | `get_url_markdown` |
| Competitor feature claims (from their website) | `web-search` → then `get_url_markdown` for the source page |
| Company/person contact enrichment | Enrichment providers (ContactOut, PDL, etc.) |
| LinkedIn post research | LimaData |
| Ad intelligence | Adyntel |
| Buying intent signals | PredictLeads, TheirStack |
| Previously saved research | Context lake (`/query/` or `/context/`) |

---

## Query Construction Best Practices

- Be specific: `"Gong.io 2025 pricing plans annual billing"` vs `"Gong pricing"`
- Use site operators for targeted searches: `"site:help.gong.io call recording"` is valid in the query field
- For recent news, add the year: `"Salesforce Einstein AI features 2025"`
- For comparison research: `"Gong vs Chorus feature comparison"`
- If results are thin, increase `numberOfResults` to 5 and add `pageno: 2` for a second call

---

## Rules

- Always check context lake (`/query/`) before running `web-search` — prior research may already contain the answer
- After finding a URL in search results, use `get_url_markdown` to read the full page content
- Do not use `web-search` for contact enrichment — use the enrichment provider stack
- `web-search` does not consume enrichment credits — use it freely for general research
- Infoboxes are the most reliable output — prioritize them over raw result snippets
