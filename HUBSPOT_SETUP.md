# HubSpot pipeline setup — the single source of truth

The pipeline sync keeps **every HubSpot deal current** from the full
conversation. Each morning it reads **Granola, Slack, Lemlist, and email**, writes
a dated note to the right deal, and moves the **deal stage** when the evidence
warrants. HubSpot holds everything — it's the canonical record.

## What's connected

All five run through the **connected account** in this environment (no tokens to
paste for the core pipeline):

| Capability | Provider | Used for |
|---|---|---|
| CRM (source of truth) | **HubSpot** (MCP) | Deals, stages, notes — read + write |
| Email | **Gmail** (native MCP) | Full email conversation per deal |
| Chat | **Slack** (MCP) | Client / Connect + internal account channels |
| Meeting notes | **Granola** (MCP) | Call decisions and next steps |
| Outreach | **Lemlist** (MCP) | Cold-sequence replies + AI interest scoring |

## The pipeline this is wired to

`hubspot.json` is committed (IDs + flags only, **no secrets**) and holds the live
pipeline pulled from this account:

- **Primary pipeline:** **Clinic Partnerships** (`id 2314112732`) — where the
  active deals live. Stages, in order:
  `Target Identified → Outreach Sent → Discovery Scheduled → Discovery Completed
  → Proposal Sent → Pilot Discussion → Pilot Active → Closed Won` (+ `Closed
  Lost`).
- **Secondary:** the default **Sales Pipeline** is also mapped, in case a deal
  lives there.
- **Default owner:** Syb Roell (`163314964`) — new deals are assigned here.

> Stage **internal ids** (e.g. `3744632516` = *Discovery Scheduled*) differ from
> their labels — the workflows always read the id from `hubspot.json`, never
> hard-code it. If you add/rename a stage in HubSpot, update `hubspot.json` to
> match (re-run `get_properties(objectType="deals", ["dealstage","pipeline"])` to
> get the current ids).

## How a stage moves

W1 (`hubspot-sync.md`) advances a deal only on **clear evidence** from the
conversation — e.g. a Granola note proves a discovery call happened
(→ *Discovery Completed*), a proposal email went out (→ *Proposal Sent*), a
positive Lemlist reply plus a booked call (→ *Discovery Scheduled*). Guardrails in
`hubspot.json`:

- `ingest.autoApply` — `true` applies moves; `false` proposes them in the digest.
- `ingest.allowStageAdvance` — gates forward moves.
- `ingest.allowStageClose` — **off by default**: *Closed Won* / *Closed Lost* are
  never set automatically; they're always listed for your approval.
- Active deals are **never silently demoted** on a quiet day — only on an explicit
  pause/churn signal.

## Lemlist → HubSpot mapping

Lemlist replies are read via `get_inbox_conversations` → `get_inbox_conversation`,
which carries `aiLeadInterest` (**positive** ≥4 / neutral / **negative** ≤1). The
lead's email is matched to the HubSpot deal's contact. A positive reply is a
buying signal (advance the stage); a negative reply is a lost signal (propose
*Closed Lost* for approval).

> If Lemlist's **native HubSpot integration** is also enabled in your Lemlist
> account, sequence sends/opens/replies additionally appear on the HubSpot contact
> timeline — belt-and-suspenders. The workflow reads Lemlist directly via MCP
> regardless, so it doesn't depend on that sync being on.

## Scheduling

Schedule **one** daily Claude Code web session with the prompt **`/daily-crm`** —
it runs **W0 → W1 → W2** in order (discover new deals → sync every deal from all
four sources → derive Notion to-dos). Daily, **07:00 Europe/Amsterdam**, on
**Sonnet**. See `SCHEDULING.md`.

## Secrets — usually NONE

The core pipeline reads `hubspot.json` from git and all sources from the connected
account. Add an env secret only if a scheduled (headless) run's preflight shows a
source ⚠️ unavailable — then add just that one (e.g. a HubSpot private-app token as
`HUBSPOT_TOKEN`, Gmail via `EMAIL_ACCOUNTS_JSON`, Slack via
`SLACK_WORKSPACES_JSON`). Real tokens go in env secrets or a gitignored
`hubspot.local.json` — **never** in the committed `hubspot.json`.

## Safety

- Writes are conservative: **add a note + advance a stage on clear evidence**.
  Never delete; never auto-close-won/lost; never create duplicate deals.
- New-deal creation is W0's job alone (`newDealDiscovery.enabled`); W1 only
  updates existing deals.
