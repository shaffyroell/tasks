# Daily Email + Slack Open-Items Workflow

A daily automation that scans **Gmail, Slack, and meeting notes (Fireflies +
Granola)** and keeps two things current: **Attio** as the primary CRM log (each
pipeline contact exists, is linked to the right deal, and has a note + follow-up
task) and a **Notion tracker** as the human-facing view of everything Shaffy still
needs to respond to or act on — pipeline emails, Slack threads to weigh in on, and
next-steps committed to in calls.

## Where things live

- **Tracker (output):** Notion → **📥 Open Items — Daily Tracker**
  - https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144
- **Workflow definition (what runs each morning):** [`daily-open-items.md`](./daily-open-items.md)
- **Adding more Slack workspaces:** [`SLACK_SETUP.md`](./SLACK_SETUP.md)
- **Adding more email accounts:** [`EMAIL_SETUP.md`](./EMAIL_SETUP.md)
- **Attio CRM write-back:** [`ATTIO_SETUP.md`](./ATTIO_SETUP.md)
- **Supported tools & connections:** [`CONNECTIONS.md`](./CONNECTIONS.md)
- **Onboard a new client (plug-and-play):** [`ONBOARDING.md`](./ONBOARDING.md)
- **Scheduling:** [`SCHEDULING.md`](./SCHEDULING.md)
- **Roadmap & related workflows:** [`ROADMAP.md`](./ROADMAP.md)

## Slack coverage

TechTower Slack works out of the box via the connected Slack connector. To sweep
**additional** workspaces in the same run, add one `xoxp-` user token per
workspace to `slack-workspaces.json` — see [`SLACK_SETUP.md`](./SLACK_SETUP.md).

**Long-term**, prefer **Slack Connect** — bring client conversations into shared
channels inside your own Slack so the sweep needs only one connection you own,
with no per-client tokens. Per-workspace tokens are the fallback for clients whose
internal workspace you're embedded in. Full rationale in
[`CONNECTIONS.md`](./CONNECTIONS.md).

## Email coverage

This instance is **TechTower-only**: email is scoped to `stack.email.domains`
(techtower.ai, techtowerops.com), so even though the connected inbox also receives
SwimScore mail, that's ignored here — **SwimScore runs as its own duplicated
workflow in a separate Claude account**. To add a genuinely separate mailbox you
*send* from within the same business, connect it as its own account in
`email-accounts.json` (reply-detection needs its Sent mail) — see
[`EMAIL_SETUP.md`](./EMAIL_SETUP.md).

## Attio CRM log (primary)

For pipeline items, the sweep keeps Attio (TechTower) current as the **system of
record**: it **ensures the person exists** (creating them if missing) and **links
them to the right deal** — and when it spots a **new pipeline conversation in
email** with no deal yet, it **opens one deal for that company** (one per company,
deduped by domain) at the **right stage**, inferred from the email plus any
calendar invite you sent and follow-ups. It then **adds a note** and **ensures a
follow-up task**, recording what it did in the tracker's `Attio` column.
Guardrails stay conservative: never a second deal per company, never advance an
existing deal's stage without approval, never delete; weak/ambiguous signals are
flagged `(review)` rather than guessed. Add an API token to `attio.json`
(gitignored, or the `ATTIO_JSON` secret for scheduled runs) to enable it — see
[`ATTIO_SETUP.md`](./ATTIO_SETUP.md).

## How it runs

The workflow is designed to run **once every morning around 07:00
(Europe/Amsterdam)**. Each run:

0. **Preflight** — checks every dependency (Notion, each email account, each
   Slack workspace, Fireflies, Granola, Attio) and reports readiness.
   Missing/expired credentials are flagged loudly in the summary; that source is
   skipped for the run rather than failing the whole sweep.
1. Reads the current tracker (everything not `Done`).
2. Pulls recent Gmail threads, Slack mentions/DMs, and meeting action items from
   Fireflies **and** Granola.
3. Adds new open items, updates statuses, and marks handled items `Done` —
   deduping on a stable `Ref` key so nothing is duplicated.
4. Syncs pipeline into Attio (ensure person → find/create the company's one deal
   at the right stage → link people → note + task).
5. Posts a short summary.

### Scheduling it

Run it as a **Claude Code web scheduled session** — daily 07:00 Europe/Amsterdam,
pointed at this repo, prompt *"Run the workflow in `daily-open-items.md`."* Full
click-through + the env secrets to set are in [`SCHEDULING.md`](./SCHEDULING.md).

This survives across days and doesn't depend on any chat session staying open.
(An in-session cron only fires while that session is alive — it won't persist, so
it's for testing only.)

## Criteria (summary)

**Include:** pipeline/client emails awaiting a reply or follow-up; Slack
@mentions/DMs with an open question for Shaffy; meeting action items assigned to
Shaffy.
**Exclude:** newsletters, promotions, automated/no-reply mail, calendar
accept/decline notices, workflow error alerts, and anything already handled.

Full logic, schema, and dedup rules are in [`daily-open-items.md`](./daily-open-items.md).

## Security

Credentials never belong in this repo. `accounts.json`, `.env`, `attio.json`, and
key files are gitignored. The workflow runs on connected MCP connectors (Gmail,
Slack, Notion, Fireflies, Granola); the only API token it reads is the Attio key
(from `attio.json` locally, or the `ATTIO_JSON` secret for scheduled runs), used
solely to write the CRM log.
