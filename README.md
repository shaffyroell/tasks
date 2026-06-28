# Client Pipeline Automation — HubSpot (single source of truth)

A daily **CEO sweep** reads every conversation, decides what matters, and keeps the
systems honest: the **pipeline lives in HubSpot**, internal execution lives in
**SwimScore Notion**. It runs as a pipeline of workflows:

- **CEO sweep — classify & route** (`/daily-crm`): reads **all** email, Slack,
  Granola, and Lemlist; classifies each meaningful item as **ACCOUNT** or
  **INTERNAL**, then routes it (W0/W1 for accounts → HubSpot; W2 for internal →
  Notion). This is the one routine to schedule.
- **W0 — Discover → HubSpot** ([`new-deal-discovery.md`](./new-deal-discovery.md)):
  scans yesterday's inbound mail + fresh Lemlist replies for **genuine new
  opportunities not yet in HubSpot** and **creates the deal + links company +
  links contact + adds a note**. Runs first (~06:50).
- **W1 — Sync every deal → HubSpot** ([`hubspot-sync.md`](./hubspot-sync.md)):
  walks **every open deal** and reads the **full recent conversation across
  Granola, Slack, Lemlist, and email**, then writes a dated note on the deal and
  **moves the stage** when the evidence warrants. Runs ~07:00. HubSpot holds
  everything.
- **W2 — Internal → Notion** ([`daily-open-items.md`](./daily-open-items.md)):
  for INTERNAL SwimScore items, reconciles the **SwimScore Notion** tracker —
  checks what each item is for and updates it to the latest status before adding
  new ones. Runs ~07:15.

```
Gmail · Slack · Granola · Lemlist
        │
        ▼   classify each item:  ACCOUNT ───────┐         INTERNAL ──┐
        ▼   W0 new-deal-discovery.md            │                    │
        ▼   W1 hubspot-sync.md                  ▼                    ▼
                                         ┌───────────┐        ┌──────────────┐
                                         │  HubSpot  │        │  SwimScore   │
                                         │  (deals,  │        │   Notion     │
                                         │  stages,  │        │ (internal    │
                                         │  notes)   │        │  to-dos) W2  │
                                         └───────────┘        └──────────────┘
                                       single source of truth
```

One routine runs the whole thing in order: the **`/daily-crm`** slash command.

## Where things live

- **CEO orchestration:** `.claude/commands/daily-crm.md` (`/daily-crm`)
- **W0 — New-deal discovery (HubSpot):** [`new-deal-discovery.md`](./new-deal-discovery.md)
- **W1 — HubSpot deal sync (source of truth):** [`hubspot-sync.md`](./hubspot-sync.md)
- **W2 — Internal SwimScore To-Dos (Notion):** [`daily-open-items.md`](./daily-open-items.md)
- **Pipeline config (committed, IDs + flags):** [`hubspot.json`](./hubspot.json)
- **HubSpot setup & pipeline map:** [`HUBSPOT_SETUP.md`](./HUBSPOT_SETUP.md)
- **House writing style (to-dos):** [`STYLE.md`](./STYLE.md)
- **Adding more Slack workspaces:** [`SLACK_SETUP.md`](./SLACK_SETUP.md)
- **Adding more email accounts:** [`EMAIL_SETUP.md`](./EMAIL_SETUP.md)
- **Supported tools & connections:** [`CONNECTIONS.md`](./CONNECTIONS.md)
- **Scheduling:** [`SCHEDULING.md`](./SCHEDULING.md)

## The sources

All read through the connected account in this environment:

- **Email (Gmail, native MCP)** — the full email conversation per deal.
- **Slack** — client / Slack-Connect channels + the internal channels where the
  business is run.
- **Granola** — call notes; the real decisions.
- **Lemlist** — cold-sequence replies via `get_inbox_conversations` →
  `get_inbox_conversation`, with AI interest scoring (`aiLeadInterest`
  positive/neutral/negative) used as a buying/lost signal.
- **Shopify (B2B inbound only)** — inbound inquiries via the www.myswimscore.com
  website (clinic/wholesale interest) are often **B2B clinic leads** and are
  treated as deal signals. Individual **B2C patient orders are out of scope** —
  this pipeline is B2B-only.

## HubSpot as the single source of truth

For every open deal, W1 **adds a consolidated note** (what happened per channel,
decisions/commitments, risks, next step) and **moves the deal stage** on clear
evidence. It's conservative: stage advances are gated by config, **Closed
Won/Lost are never set automatically** (listed for approval), active deals are
never silently demoted, and nothing is ever deleted. The live pipeline (Clinic
Partnerships) and all stage ids are mapped in [`hubspot.json`](./hubspot.json) —
see [`HUBSPOT_SETUP.md`](./HUBSPOT_SETUP.md).

## Internal items → SwimScore Notion

Items that are internal to SwimScore (product, team, hiring, finance, roadmap, ops)
are routed to the **SwimScore Notion** tracker (`hubspot.json` →
`internalTracker`). W2 checks whether each item already exists, reads what it's
for, and updates it to the latest status. *(Notion isn't connected via MCP yet —
set `internalTracker.dataSourceId` to the SwimScore Notion data-source id to
enable; until then, internal items are listed in the digest for manual handling.)*

## How it runs

Run as a **Claude Code web scheduled session** — daily 07:00 Europe/Amsterdam,
pointed at this repo, prompt **`/daily-crm`**, on **Sonnet**. Each run opens with a
**Preflight** line (HubSpot, Gmail, Slack, Granola, Lemlist, Notion) and skips any
down source loudly rather than failing the whole sweep. Full click-through in
[`SCHEDULING.md`](./SCHEDULING.md).

## Security

Credentials never belong in this repo. `hubspot.json` holds only IDs + flags (no
tokens) and is committed; real tokens go in env secrets or a gitignored
`hubspot.local.json`. `accounts.json`, `.env`, and key files are gitignored. The
workflow uses connected MCP connectors (HubSpot, Gmail, Slack, Granola, Lemlist)
and does not read or store API keys.
