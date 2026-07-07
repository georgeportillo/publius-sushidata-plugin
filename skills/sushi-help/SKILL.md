---
name: sushi-help
description: >
  Show available Sushidata commands and how to use them. Trigger when the
  user says: "help", "sushidata help", "what can you do", "what commands
  are available", "how do I use sushidata", "show me what sushidata can do",
  "what skills do you have", "commands", or any request for an overview of
  capabilities. Run immediately — show the command list first, then offer
  to explore use cases.
---

# Sushidata Help

When triggered, immediately show the full command reference. After delivering
it, offer to go deeper on use cases — do not ask first.

---

## Step 1 — Output the command reference

Show this card exactly:

---

### 🍣 Sushidata — What I Can Do

#### Commands you can use anytime

| Command      | What it does                                                                        |
| ------------ | ----------------------------------------------------------------------------------- |
| **restore**  | Pull everything saved from your prior sessions back into memory                     |
| **save**     | Save this session's outputs to your context lake — save all or choose what          |
| **savings**  | See a breakdown of what Sushidata retrieved vs what Claude built this session       |
| **matrix**   | Launch a competitor/category research matrix and render a spreadsheet with evidence |
| **sequence** | Run a multi-step research pipeline where each step builds on the last               |
| **help**     | Show this screen                                                                    |

---

#### Research & Intelligence

| Say something like...                                      | What happens                                                       |
| ---------------------------------------------------------- | ------------------------------------------------------------------ |
| "Research [company]"                                       | Deploys a swarm to build a full intel profile                      |
| "Who are [company]'s competitors"                          | Identifies and profiles key competitors                            |
| "Build a GTM competitor report on [company]"               | Full channel-by-channel GTM analysis                               |
| "Run a research matrix for [competitors] and [categories]" | Launches a research matrix and returns a spreadsheet with evidence |
| "Build a battlecard for [company]"                         | Positioning and objection-handling card                            |
| "Find contacts at [company]"                               | Discovers key contacts with LinkedIn profiles                      |
| "Build an org chart for [company]"                         | Maps the org structure and buying committee                        |

---

#### Prospecting & Outreach

| Say something like...                 | What happens                                 |
| ------------------------------------- | -------------------------------------------- |
| "Find me leads for [ICP description]" | Builds a targeted prospect list              |
| "Write outreach for [person/company]" | Drafts personalized LinkedIn or email copy   |
| "Add these leads to HeyReach"         | Pushes contacts into a HeyReach campaign     |
| "Enrich these contacts in HubSpot"    | Updates HubSpot records with researched data |
| "Find emails for [list]"              | Runs FullEnrich to find email addresses      |

---

#### ICP & Signal Discovery

| Say something like...             | What happens                                    |
| --------------------------------- | ----------------------------------------------- |
| "Analyze my won vs lost accounts" | Discovers niche signals that predict closed-won |
| "What signals should I look for"  | Builds a scoring model from your ICP data       |
| "Score these accounts"            | Classifies accounts by ICP fit                  |

---

#### Community & Market Intelligence

| Say something like...                  | What happens                               |
| -------------------------------------- | ------------------------------------------ |
| "What are people saying about [topic]" | Scrapes Discord, Reddit, forums for signal |
| "Build a TAM report for [segment]"     | Sizes the addressable market with sources  |
| "Review this document for accuracy"    | Runs `/sushi-verify` — checks claims, links, and citations |

---

#### Scheduled & Automated

| Say something like...           | What happens                           |
| ------------------------------- | -------------------------------------- |
| "Run this every Monday morning" | Sets up a recurring scheduled task     |
| "Alert me when [condition]"     | Creates a trigger-based scheduled task |

---

## Step 2 — Offer starting points as CTAs

After the command card, add this section exactly — use the CTA format so the user can click without typing:

---

**Where do you want to start?**

> 💬 **"Research my competitors"** — Competitive battlecard with real-time Sushidata research
> 💬 **"Find me leads for [describe your ICP]"** — Qualified prospect list with verified emails in under 5 minutes
> 💬 **"Run a matrix on [competitors] across [categories]"** — Side-by-side comparison spreadsheet with evidence per cell
> 💬 **"Tell me everything about [company name]"** — Full account brief: signals, contacts, org chart, outreach angle
> 💬 **"Restore my prior sessions"** — Pull your saved research back into context

---

## Step 3 — If the user asks for a use case guide

### GTM Sales

Walk through this workflow — click any step to start:

> 💬 **"Find me leads for [your ICP description]"** — Build a qualified prospect list with verified emails
> 💬 **"Tell me everything about [company name]"** — Full account brief: signals, org chart, outreach angle
> 💬 **"Write a LinkedIn message for [person] at [company]"** — Personalized outreach drafted from live research
> 💬 **"Add these leads to my HeyReach campaign"** — Push contacts to a LinkedIn sequence
> 💬 **"Save"** — Persist this session to your context lake

---

### Marketing

Walk through this workflow — click any step to start:

> 💬 **"Who are [company]'s competitors"** — Profile key players across SEO, Paid, Social, Events, PR
> 💬 **"Build a GTM competitor report on [company]"** — Full channel-by-channel analysis with sources
> 💬 **"What are people saying about [topic]"** — Surfaces signal from Discord, Reddit, and forums
> 💬 **"Build a TAM report for [segment]"** — Structured market sizing with analyst citations
> 💬 **"Run a competitor update on [company] every Monday"** — Set up ongoing tracking

---

### Competitor Analysis

Walk through this workflow — click any step to start:

> 💬 **"Build a battlecard for [competitor]"** — Positioning, strengths, weaknesses, objection handling
> 💬 **"Build a GTM competitor report on [competitor]"** — Every channel they use and how they use it
> 💬 **"Run a matrix on [competitors] across [categories]"** — Side-by-side spreadsheet with live evidence
> 💬 **"Review this document for accuracy"** — Runs `/sushi-verify` to check all claims, links, and citations before publishing
> 💬 **"Check [competitor]'s hiring page every week"** — Schedule ongoing tracking for early intent signals

---

## Rules

- Always show the full command card first. Never ask what the user wants
  before showing it.
- Only go into use case detail if the user asks. The offer at the end of
  Step 2 is enough.
- Keep the command card intact — do not summarize or shorten it.
- If the user names a use case, go straight to that section. Do not show
  all three unless asked.
