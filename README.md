# Daily Email + Slack Open-Items Workflow

A daily automation that scans **Gmail, Slack, and meeting notes (Fireflies)** and
keeps a single Notion tracker of everything Shaffy still needs to respond to or
act on — pipeline emails, Slack threads to weigh in on, and next-steps committed
to in calls.

## Where things live

- **Tracker (output):** Notion → **📥 Open Items — Daily Tracker**
  - https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144
- **Workflow definition (what runs each morning):** [`daily-open-items.md`](./daily-open-items.md)

## How it runs

The workflow is designed to run **once every morning around 07:00
(Europe/Amsterdam)**. Each run:

1. Reads the current tracker (everything not `Done`).
2. Pulls recent Gmail threads, Slack mentions/DMs, and Fireflies action items.
3. Adds new open items, updates statuses, and marks handled items `Done` —
   deduping on a stable `Ref` key so nothing is duplicated.
4. Posts a short summary.

### Scheduling it

Recommended: a **Claude Code scheduled session/trigger** (Claude Code on the web)
set to a daily 07:00 cron, pointed at this repo, with the instruction:

> Run the workflow in `daily-open-items.md`.

This survives across days and doesn't depend on any single chat session staying
open. (An in-session cron can also be used for testing, but it expires and only
fires while that session is alive.)

## Criteria (summary)

**Include:** pipeline/client emails awaiting a reply or follow-up; Slack
@mentions/DMs with an open question for Shaffy; meeting action items assigned to
Shaffy.
**Exclude:** newsletters, promotions, automated/no-reply mail, calendar
accept/decline notices, workflow error alerts, and anything already handled.

Full logic, schema, and dedup rules are in [`daily-open-items.md`](./daily-open-items.md).

## Security

Credentials never belong in this repo. `accounts.json`, `.env`, and key files are
gitignored. The workflow only uses connected MCP connectors (Gmail, Slack,
Notion, Fireflies) — it does not read or store API keys.
