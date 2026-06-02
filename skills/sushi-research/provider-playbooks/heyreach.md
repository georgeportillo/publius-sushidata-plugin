# HeyReach Playbook

Use HeyReach for outbound activation after qualification and verification are complete.

## Key rules

- **Campaigns must be pre-created in the HeyReach UI.** The public API does not expose campaign creation — do not attempt to create campaigns via API tools. Create campaigns in the HeyReach UI first, then reference the resulting `campaign_id`.
- Always list campaigns first and resolve the exact campaign target before any inserts.
- Batch contact writes in small chunks (≤50 contacts) and validate response shape before scaling.
- Pull campaign stats after insert operations to confirm downstream effects.

## Workflow

### 1. List available campaigns

Call `heyreach_list_campaigns` (no required payload) to retrieve all active campaigns and their IDs.

```json
{}
```

Identify the correct `campaign_id` from the response before proceeding.

### 2. Add contacts to a campaign

Call `heyreach_add_to_campaign` with the resolved campaign ID and contact list:

```json
{
  "campaign_id": "12345",
  "contacts": [
    {
      "linkedin_url": "https://www.linkedin.com/in/example",
      "first_name": "Ada",
      "last_name": "Lovelace",
      "email": "ada@example.com"
    }
  ]
}
```

**Only add contacts that have been verified** — do not pass unverified or catch-all emails.

### 3. Confirm stats

After inserting, call `heyreach_get_campaign_stats` (or equivalent) to confirm that the contact count and campaign state reflect the inserts correctly.

## Common pitfalls

- Attempting to create campaigns via API (not supported — always use the UI).
- Inserting contacts without resolving the campaign ID first.
- Skipping stat verification after inserts — downstream sync issues are silent.
- Passing catch-all or unverified emails; use Hunter to verify first.

---

## Save to Context Lake

Save campaign activation results so future sessions can check campaign status without re-querying HeyReach:

```json
POST /context/
{
  "serverId": "26",
  "content": "HeyReach campaign activation complete. Campaign: {{campaign name, ID}}. Contacts added: {{count}}. LinkedIn account used: {{sender name}}. Contacts accepted: {{count from stats check}}. Any rejected: {{count and reason}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
