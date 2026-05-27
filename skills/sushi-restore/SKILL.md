---
name: sushi-restore
description: >
  Restore session context from the Sushidata context lake at the start of a
  new session. Trigger when the user says: "restore", "restore my memory",
  "pull my sushidata memory", "what do you know about me", "catch me up",
  "resume from sushidata", "pull context", "load context", or any request
  to recover prior session intelligence at the start of a conversation.
  Run immediately with no clarifying questions.
---

# Sushidata Session Restore

When triggered, pull prior session intelligence from the Sushidata context
lake and brief the user on what was found. This replaces the need to reload
a prior conversation.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

---

## Step 1 — Get the session ID

Run:
```bash
echo $PWD | grep -oP 'local_[a-f0-9-]+'
```

Use the result as the `threadId` for all queries. This is the stable identifier
for this Cowork session.

---

## Step 2 — Query the context lake

Make the following POST request to retrieve everything saved under this session:

```http
POST {BASE_URL}query/
Content-Type: application/json
```

```json
{
  "query": "session restore — all saved context",
  "tenant": "Sushidata",
  "dataspace": "Sushidata Internal",
  "threadId": "<cowork-session-id>",
  "limit": 50
}
```

If the `threadId` query returns fewer than 5 results, run a second broader
query without the `threadId` filter to pull the most recent context saved
across all sessions:

```http
POST {BASE_URL}query/
Content-Type: application/json
```

```json
{
  "query": "recent session intelligence accounts contacts competitors documents",
  "tenant": "Sushidata",
  "dataspace": "Sushidata Internal",
  "limit": 20
}
```

---

## Step 3 — Organize what came back

Group the results into categories:

- **Accounts & Contacts** — companies or people researched, enriched, or
  prospected
- **Competitor Intelligence** — battlecards, GTM reports, channel analysis,
  positioning
- **Documents Produced** — reports, TAM analyses, org charts, outreach
  sequences
- **ICP & Signals** — scoring models, niche signals, won/lost analysis
- **Campaign & Outreach** — lists built, campaigns launched, messages sent
- **Other** — anything that doesn't fit the above

If a category has no results, omit it from the output entirely.

---

## Step 4 — Output the restore brief

Present the findings as a clean, scannable brief:

---

### 🍣 Sushidata Memory Restore

**Session:** [session ID from Step 1, or "new session" if unavailable]
**Date:** [today's date]
**Items recovered:** [total count from both queries, deduplicated]

---

[For each category that has results:]

#### [Category Name]
- [Concise bullet per item — company name, person, document title, or signal
  type. One line each. Include the date it was saved if available.]

---

#### Ready to continue

> Here's what I found in your context lake. Pick up where you left off or
> tell me what you're working on today.

---

## Step 5 — Stay in context

After delivering the brief, treat all recovered items as active context for
the rest of the session. Do not ask the user to re-explain accounts, contacts,
or work that was already summarized in the restore brief.

---

## Rules

- Run immediately when triggered. Do not ask the user any questions first.
- Always attempt the `threadId`-scoped query first before falling back to the
  broad query.
- If both queries return zero results, tell the user plainly: "Nothing was
  found in your Sushidata context lake yet. As you work this session, saves
  will be written back automatically."
- Never fabricate results. Only surface what the context lake actually returned.
- Keep each bullet in the brief to one line. Do not summarize or editorialize
  individual items.
- Omit empty categories entirely — do not show them with a zero or "none".
