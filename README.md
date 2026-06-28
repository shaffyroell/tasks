# Client Comms Automation — Attio + Notion

Three daily workflows run as a pipeline so the CRM is the source of truth and the
to-do list stays current:

- **W0 — Discover → Attio** ([`new-deal-discovery.md`](./new-deal-discovery.md)):
  scans yesterday's inbound mail for **genuine new opportunities not yet in
  Attio** and **creates the deal + links company + links person + adds a note**.
  Runs first (~06:50).
- **W1 — Ingest → Attio** ([`attio-ingest.md`](./attio-ingest.md)): reads all raw
  comms (**Gmail, Slack, Granola + Fireflies, calendar**) and writes them into
  **Attio** — a dated note on every active deal/client + a deal **stage** move
  when the evidence warrants (existing deals, incl. those W0 just created).
  Runs ~07:00. Attio holds everything.
- **W2 — Act → Notion** ([`daily-open-items.md`](./daily-open-items.md)): derives
  the **per-client To-Dos in Notion**. Hybrid input — reads **Attio (primary)**
  plus fresh Gmail/Slack/Granola/Fireflies, and **reconciles** existing to-dos
  (mark done / advanced) before adding new ones. Runs ~07:15.

```
Gmail · Slack · Granola · Fireflies · Calendar
        │
        ▼   W0  new-deal-discovery.md   (create new deals + link + note)
        ▼   W1  attio-ingest.md         (notes + stage moves on existing deals)
     ┌───────────┐
     │   Attio   │  ← source of truth: deals, stages, raw comms notes
     └───────────┘
        │
        ▼   W2  daily-open-items.md     (+ fresh sources, reconcile)
   Per-client To-Dos in Notion
```

One routine runs all three in order: the `/daily-crm` slash command.

## Where things live

- **W0 — New-deal discovery (Attio):** [`new-deal-discovery.md`](./new-deal-discovery.md)
- **W1 — Ingest to Attio (source of truth):** [`attio-ingest.md`](./attio-ingest.md)
- **W2 — Per-client To-Dos (Notion):** [`daily-open-items.md`](./daily-open-items.md)
- **Tracker (W2 output):** Notion → **📥 Open Items — Daily Tracker**
  - https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144
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

## Attio CRM write-back

For pipeline items, the sweep keeps Attio (TechTower) current: on the matched
person/company/deal it **adds a note** and **ensures a follow-up task**, recording
what it did in the tracker's `Attio` column. It's strictly additive — stage
changes need approval, and nothing is ever created-new or deleted. Add an API
token to `attio.json` (gitignored) to enable it — see
[`ATTIO_SETUP.md`](./ATTIO_SETUP.md).

## How it runs

The workflow is designed to run **once every morning around 07:00
(Europe/Amsterdam)**. Each run:

0. **Preflight** — checks every dependency (Notion, each email account, each
   Slack workspace, Fireflies, Attio) and reports readiness. Missing/expired
   credentials are flagged loudly in the summary; that source is skipped for the
   run rather than failing the whole sweep.
1. Reads the current tracker (everything not `Done`).
2. Pulls recent Gmail threads, Slack mentions/DMs, and Fireflies action items.
3. Adds new open items, updates statuses, and marks handled items `Done` —
   deduping on a stable `Ref` key so nothing is duplicated.
4. Posts a short summary.

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

Credentials never belong in this repo. `accounts.json`, `.env`, and key files are
gitignored. The workflow only uses connected MCP connectors (Gmail, Slack,
Notion, Fireflies) — it does not read or store API keys.
