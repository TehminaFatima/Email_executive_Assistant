# AI Email Executive Assistant (n8n Workflow)

An n8n automation that watches a Gmail inbox, triages incoming email with GPT-4.1, and routes each message to the right follow-up action — drafting a reply, creating a calendar event, logging a task, or syncing the contact to a CRM — then posts a daily digest to Slack.

## What it does

1. **Watches Gmail** for new unread inbox mail (polls every minute).
2. **Preprocesses** each email — extracts sender, subject, clean body text (signature stripped), and attachment metadata.
3. **Deduplicates** against a Google Sheet log so the same message is never processed twice.
4. **Classifies** the email with GPT-4.1 into:
   - Priority (`High` / `Medium` / `Low`)
   - Category (`Client` / `Internal` / `Vendor` / `Personal` / `Newsletter` / `Spam`)
   - Sentiment
   - Whether it needs a reply, a calendar event, or a task
   - A one-line summary
5. **Routes** the classified email down up to four parallel branches:
   - **Reply drafting** — GPT-4.1 writes a reply, saved as a Gmail draft (never auto-sent)
   - **Calendar extraction** — pulls meeting details (title, time, attendees, location) and creates a Google Calendar event
   - **Task logging** — appends an action item row to the tracking sheet
   - **CRM sync** — upserts the sender as a contact in HubSpot (for `Client` category emails)
6. **Logs** every processed email (metadata + actions taken) to a Google Sheet.
7. **Sends a Slack digest** summarizing everything processed in the run.

## Architecture

```
Gmail Trigger → Preprocess → Dedup check (Sheets) → If new →
  Classify (GPT-4.1) → Switch (multi-route) →
    ├─ Draft Reply (GPT-4.1) → Gmail Draft
    ├─ Extract Event (GPT-4.1) → Google Calendar
    ├─ Log Task → Google Sheets
    └─ Sync Contact → HubSpot
  → Merge → Write Log (Sheets) → Build Digest → Slack
```

## Requirements

- [n8n](https://n8n.io) (self-hosted or cloud)
- An OpenAI API key with access to `gpt-4.1`
- Google account with:
  - Gmail API access (OAuth2)
  - Google Sheets API access (OAuth2)
  - Google Calendar API access (OAuth2)
- A Slack workspace + bot token (OAuth2) with permission to post to a channel
- A HubSpot account (OAuth2) — optional, only needed for the CRM sync branch

## Setup

1. Import `workflow_sanitized.json` into n8n (**Workflows → Import from File**).
2. Re-connect each credential placeholder to your own accounts:
   - Gmail (trigger + draft creation)
   - Google Sheets (dedup check, task logging, run log)
   - OpenAI (classification, reply drafting, event extraction)
   - Google Calendar (event creation)
   - HubSpot (contact sync)
   - Slack (digest notification)
3. Create a Google Sheet with the following columns and point the Sheets nodes at it:
   `timestamp, message_id, thread_id, from_email, subject, priority, category, actions_taken, reply_status, calendar_event_id, task_id, crm_synced, error_flag, error_detail`
4. Update the Slack node to point at your own channel.
5. Activate the workflow.

## Notes

- **Drafts, not sends** — the workflow creates Gmail drafts for replies rather than sending automatically, so a human always reviews before anything goes out.
- **Error tolerance** — most nodes have `continueOnFail` enabled so a single failed branch (e.g. a malformed AI response) doesn't stop the whole run; failures are flagged in the log sheet instead.
- **Idempotency** — the Sheet-based dedup check prevents reprocessing the same `message_id` on retriggers or polling overlaps.
- This repo's workflow JSON has had all credential IDs, spreadsheet/document IDs, webhook IDs, Slack channel ID, and the n8n instance ID redacted. You'll need to supply your own when importing.

## License

MIT (or update to your preferred license).
