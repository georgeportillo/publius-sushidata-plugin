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

| Command | What it does |
|---|---|
| **restore** | Pull everything saved from your prior sessions back into memory |
| **save** | Save this session's outputs to your context lake — save all or choose what |
| **savings** | See a breakdown of what Sushidata retrieved vs what Claude built this session |
| **help** | Show this screen |

---

#### Research & Intelligence

| Say something like... | What happens |
|---|---|
| "Research [company]" | Deploys a swarm to build a full intel profile |
| "Who are [company]'s competitors" | Identifies and profiles key competitors |
| "Build a GTM competitor report on [company]" | Full channel-by-channel GTM analysis |
| "Build a battlecard for [company]" | Positioning and objection-handling card |
| "Find contacts at [company]" | Discovers key contacts with LinkedIn profiles |
| "Build an org chart for [company]" | Maps the org structure and buying committee |

---

#### Prospecting & Outreach

| Say something like... | What happens |
|---|---|
| "Find me leads for [ICP description]" | Builds a targeted prospect list |
| "Write outreach for [person/company]" | Drafts personalized LinkedIn or email copy |
| "Add these leads to HeyReach" | Pushes contacts into a HeyReach campaign |
| "Enrich these contacts in HubSpot" | Updates HubSpot records with researched data |
| "Find emails for [list]" | Runs Hunter to find verified email addresses |

---

#### ICP & Signal Discovery

| Say something like... | What happens |
|---|---|
| "Analyze my won vs lost accounts" | Discovers niche signals that predict closed-won |
| "What signals should I look for" | Builds a scoring model from your ICP data |
| "Score these accounts" | Classifies accounts by ICP fit |

---

#### Community & Market Intelligence

| Say something like... | What happens |
|---|---|
| "What are people saying about [topic]" | Scrapes Discord, Reddit, forums for signal |
| "Build a TAM report for [segment]" | Sizes the addressable market with sources |
| "Review this document for accuracy" | Verifies claims, links, and citations |

---

#### Scheduled & Automated

| Say something like... | What happens |
|---|---|
| "Run this every Monday morning" | Sets up a recurring scheduled task |
| "Alert me when [condition]" | Creates a trigger-based scheduled task |

---

## Step 2 — Offer use case guides

After the command card, add:

---

Want to go deeper? I can walk you through how to use Sushidata for a specific
use case:

- **GTM Sales** — prospecting, outreach, account research, pipeline enrichment
- **Marketing** — competitor intelligence, community signals, TAM analysis,
  campaign research
- **Competitor Analysis** — GTM reports, battlecards, positioning, tracking

Just say which one and I'll show you the full workflow.

---

## Step 3 — If the user asks for a use case guide

### GTM Sales

Walk through this workflow:

1. **Start with your ICP** — "Describe who you're selling to and I'll find
   matching accounts." Use the niche-signal-discovery skill if they have
   won/lost data.
2. **Build a prospect list** — "Find leads for [ICP]." Deploys swarm,
   returns companies and contacts with LinkedIn profiles.
3. **Research priority accounts** — "Research [company]." Full intel profile
   saved to context lake.
4. **Write outreach** — "Write a LinkedIn message for [person] at [company]."
   Personalized to their role and signals.
5. **Push to HeyReach** — "Add these leads to my HeyReach campaign." Routes
   through the HeyReach playbook.
6. **Save your work** — "Save" at the end of any session to persist everything
   for next time.

---

### Marketing

Walk through this workflow:

1. **Map the competitive landscape** — "Who are [company]'s competitors."
   Profiles key players across SEO, Paid, Social, Events, PR.
2. **Build a GTM competitor report** — "Build a GTM competitor report on
   [company]." Full channel-by-channel analysis with sources.
3. **Listen to the community** — "What are people saying about [topic/pain point]."
   Surfaces signal from Discord, Reddit, forums.
4. **Size the market** — "Build a TAM report for [segment]." Structured market
   sizing with analyst citations.
5. **Track over time** — Schedule a weekly competitor check: "Run a competitor
   update on [company] every Monday."

---

### Competitor Analysis

Walk through this workflow:

1. **Start with a battlecard** — "Build a battlecard for [competitor]."
   Positioning, strengths, weaknesses, objection handling.
2. **Go deeper with a GTM report** — "Build a GTM competitor report on
   [competitor]." Maps every channel they use and how they use it.
3. **Verify what you find** — "Review this document for accuracy." Checks
   all claims, links, and citations before you publish.
4. **Save intel to the context lake** — "Save" after each session. Future
   sessions pick up where this one left off via "restore."
5. **Schedule ongoing tracking** — "Check [competitor]'s hiring page every
   week." Signals intent before press releases do.

---

## Rules

- Always show the full command card first. Never ask what the user wants
  before showing it.
- Only go into use case detail if the user asks. The offer at the end of
  Step 2 is enough.
- Keep the command card intact — do not summarize or shorten it.
- If the user names a use case, go straight to that section. Do not show
  all three unless asked.
