---
name: sushi-verify
description: >
  Verify factual claims and source links in any research document or competitor matrix.
  Checks that AI-generated summaries accurately reflect their cited sources, flags broken
  or misrepresented links, corrects mislabeled reports, and produces a usable-yield score.
  Trigger when the user says: "verify", "check this", "fact check", "verify claims",
  "accuracy review", "check sources", "validate this document", or any request to confirm
  that research findings are backed by real, accessible evidence.
---

# Sushidata Verify

Run a structured accuracy review on a research document, competitor matrix cell, or any
set of claims with source URLs. Returns a usable-yield score, a list of flagged issues,
and corrections applied.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**.

---

## Step 1 — Identify what to verify

Ask the user in one message:

1. **What to verify** — paste content, share a file, or point to a Google Doc / Sheet
2. **Focus** — factual claims, link integrity, label accuracy, or all three (default: all)
3. **Domain** — what area (competitor GTM, market sizing, battlecard, matrix cells) so
   queries target the right context

If the user provides enough context in the trigger message, skip the form and proceed.

---

## Step 2 — Query Sushidata context lake first

Before doing any external verification, check whether Sushidata already holds verified
facts about the subject — this avoids redundant swarm work.

```
POST {BASE_URL}query/
Content-Type: application/json

{ "query": "<subject of document> verified facts sources" }
```

Cross-check the document's claims against the response. Flag any discrepancy as a
candidate correction before proceeding.

---

## Step 3 — Run the Sushidata verify endpoint

Send all claims with their source URLs in a single batch:

```
POST {BASE_URL}verify/
Content-Type: application/json

{
  "items": [
    {
      "claim": "<the AI-generated or human-written claim>",
      "url": "<source URL cited for this claim>",
      "snippet": "<exact text from the source, if available>",
      "context": "<feature / section this claim belongs to>"
    }
  ]
}
```

Response returns each item with:
- `verified: true/false` — whether the claim is supported by the source
- `confidence: 0–1` — confidence score
- `note` — explanation if verification failed or confidence is low

**Handling results:**
- `verified: true`, `confidence ≥ 0.85` → confirmed ✅
- `verified: true`, `confidence < 0.85` → low confidence ⚠️ — flag for manual review
- `verified: false` → failed ❌ — do not present as fact; find a replacement or remove

---

## Step 4 — Chase broken and failed links

For every link that returned `verified: false` or could not be fetched:

### 4a. Direct fetch
Attempt to retrieve the page and confirm the claim is present in the content.
If the page is client-rendered and returns empty content, note it as ⚠️ requires browser.

### 4b. Find a replacement
```
WebSearch: "[organization] [claim keywords] [year] site:[authoritative-domain]"
WebSearch: "[claim keywords] site:globenewswire.com OR site:prnewswire.com"
```

**For analyst reports** (Gartner, Forrester, IDC):
- Verify the exact document type — MarketScape ≠ Survey Spotlight ≠ Wave
- If paywalled, link the abstract page and classify as ⚠️ paywalled

**For competitor feature claims:**
- Primary source must be the competitor's own knowledge base or help center docs
- Marketing pages and blog posts are not sufficient — flag if only those are available

**For job postings:**
```
WebSearch: "[company] [job title] site:greenhouse.io OR site:lever.co OR site:linkedin.com/jobs"
```

**For event sponsorships:**
Always verify the exact tier on the event's own sponsor page — the original claim may
have the wrong label (e.g. "Gold" vs. "Lanyard").

---

## Step 5 — Apply corrections

For each confirmed issue:

| Issue type | Correction |
|------------|------------|
| Wrong label (e.g. report type, sponsor tier) | Update text and source link |
| Broken link | Replace with verified URL |
| Claim not supported by cited source | Remove or replace with a supported source |
| Paywalled / auth-gated | Keep claim, add ⚠️ with note on access |
| Unverifiable | Add ⚠️ inline, note the source was context lake or swarm only |
| AI summary flips meaning (e.g. "loves X and Y" from "loves Y, X needs work") | Correct the summary to match the source |

**The last item is the most critical** — always check that positive/negative polarity
in the summary matches the source, not just that the subject is mentioned.

---

## Step 6 — Produce the audit summary

```
## Accuracy Review Summary

**Subject**: [document / feature / section]
**Review date**: [date]
**Usable-yield score**: [N]/100

### Confirmed ✅
- [N] claims fully verified against primary sources

### Low confidence ⚠️
- [list each with reason and confidence score]

### Failed / corrected ❌
- [list each with what was wrong and what was changed]

### Could not verify
- [list with explanation — paywalled, auth-gated, no source found]
```

---

## Step 7 — Save to context lake

```
POST {BASE_URL}context/
Content-Type: application/json

{
  "content": "Accuracy review completed: [subject]. Usable-yield: N/100. Confirmed: N. Low confidence: N. Failed/corrected: N. Key corrections: [2–3 most significant].",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<current UTC ISO 8601 timestamp>",
  "channelId": "claude-session"
}
```

---

## Link classification reference

| Symbol | Meaning |
|--------|---------|
| ✅ | Publicly accessible, claim confirmed in content |
| ⚠️ paywalled | Abstract visible, full content requires payment |
| ⚠️ auth-gated | Requires login (LinkedIn, X, Gartner) |
| ⚠️ low confidence | Verified true but confidence < 0.85 |
| ⚠️ unverifiable | No primary source found — context lake or swarm only |
| ❌ failed | Claim contradicted by or absent from the cited source |

---

## Rules

- Never hardcode the BASE_URL — always read it from `SETTINGS.md`
- Always query the context lake before running external verification
- Primary sources for feature claims must be official docs / help centers — never marketing pages alone
- Polarity errors (AI flipping positive/negative) are treated as factual errors, not language issues
- A usable-yield score below 70 should prompt the user to consider re-running the relevant swarm
