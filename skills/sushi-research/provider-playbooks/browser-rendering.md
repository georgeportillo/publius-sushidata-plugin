# Browser Rendering Playbook

Use this playbook when you need to fetch rendered page content from a URL — especially JavaScript-rendered pages that return empty content when fetched as plain HTML.

These tools run a real browser (Cloudflare Browser Rendering / Puppeteer) against the URL and return the fully-rendered result. Use them after a raw fetch or `web-search` returns insufficient content, or when you need to confirm claims against a live page.

---

## Available Tools

| Tool name            | What it does                                                                      |
|----------------------|-----------------------------------------------------------------------------------|
| `get_url_markdown`   | Fetches a URL and returns the page content as clean Markdown — best for reading  |
| `get_url_screenshot` | Takes a screenshot of a URL — best for visual confirmation or inspecting layout  |

> `get_url_html_content` is disabled — use `get_url_markdown` instead (it's far more efficient for reading).

---

## `get_url_markdown`

Returns the rendered page as Markdown. Use for:
- Reading documentation, help center articles, blog posts, and landing pages
- Confirming that a claim made in research is actually present on the source page
- Extracting text from JavaScript-rendered pages (SPAs, React apps)

```json
{ "url": "https://example.com/features" }
```

Returns: the page's full text content in Markdown format.

---

## `get_url_screenshot`

Takes a screenshot of a rendered page. Use for:
- Visually confirming ad creatives or competitor landing pages
- Inspecting UI layout when text alone is insufficient
- Capturing pages that require visual context

```json
{
  "url": "https://example.com",
  "waitUntil": ["networkidle2"],
  "screenshotOptions": { "fullPage": true }
}
```

**Key options:**
- `screenshotOptions.fullPage` — captures the entire scrollable page (nested under `screenshotOptions`; default: false)
- `waitUntil` — an **array** of `load`, `domcontentloaded`, `networkidle0`, `networkidle2` (default `["domcontentloaded"]`) — use `["networkidle2"]` for SPAs
- `waitForTimeout` — milliseconds to wait before capturing (top-level; max 120,000ms)
- `viewport.width` / `viewport.height` — customize viewport size
- `screenshotOptions.omitBackground` — set true to capture transparent backgrounds (nested under `screenshotOptions`)

---

## When to Use Each

| Situation | Tool |
|-----------|------|
| Verifying that a claim appears on a page | `get_url_markdown` |
| Reading help center / documentation | `get_url_markdown` |
| Extracting text from a JS-rendered SPA | `get_url_markdown` |
| Viewing a competitor's ad creative or landing page visually | `get_url_screenshot` |
| Page fails to return useful content via `web-search` | `get_url_markdown` |

---

## Rules

- Always try `web-search` first for general lookups — browser rendering costs more
- Use `waitUntil: ["networkidle2"]` for React/Vue/Angular apps that load content after the initial HTML
- For source verification in `sushi-verify`, use `get_url_markdown` to confirm claims are present in the page text
- If `get_url_markdown` returns empty or minimal content, try again with `waitUntil: ["networkidle2"]` and/or `waitForTimeout: 3000`
