# HubSpot pipeline setup — the single source of truth

The pipeline sync keeps **every HubSpot deal current** from the full
conversation. Each morning it reads **Granola, Slack, Lemlist, and email**, writes
a dated note to the right deal, and — as of **2026-07-25, at Shaffy's request** —
moves the deal stage in exactly **one** narrow situation (see "How a stage
moves" below). HubSpot holds everything — it's the canonical record.

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
  `Inbound request → In conversation (Lemlist) → Asked for information → Demo
  scheduled → Contracting → Portal onboarding → First order placed → Actively
  ordering (active L3M) → No orders (L3M)` (+ `Closed Lost`).
- **Secondary:** the default **Sales Pipeline** is also mapped, in case a deal
  lives there.
- **Default owner:** Syb Roell (`163314964`) — new deals are assigned here (moot
  while W0 is disabled, see below).

> Stage **internal ids** (e.g. `3744632516` = *Asked for information*) differ from
> their labels, and **labels get renamed in the HubSpot UI without notice** — this
> file was last verified stale on 2026-07-25 (it still said "Discovery Scheduled"
> for that id). The workflows always read the id from `hubspot.json`, never
> hard-code it, but the *label prose* in these docs can still drift. If a stage
> name here looks off, don't trust it — re-run
> `get_properties(objectType="deals", propertyNames=["dealstage","pipeline"])` to
> get the current ids/labels and update `hubspot.json` + this file to match.

## How a stage moves

**2026-07-25 — narrowed at Shaffy's request.** W1 (`hubspot-sync.md`) no longer
advances a deal freely on "clear evidence" across the whole funnel. The **only**
automated stage change left, per `hubspot.json → ingest.stageAdvanceRule`:

- A deal sitting in **In conversation (Lemlist)** gets a Lemlist reply that shows
  real interest →
  - asks a question / wants pricing, no call agreed yet → **Asked for information**
  - agrees to / requests / confirms a call or demo → **Demo scheduled**
- **Every other deal keeps its stage, always** — Contracting, Portal onboarding,
  First order placed, Actively ordering, No orders (L3M): a call held, a proposal
  sent, a pilot started, an order placed, none of it moves the card. W1 logs it in
  a note; Shaffy moves the card by hand. This applies even to a deal that would
  "obviously" advance on the old rules — the automation just doesn't do it anymore.

Other guardrails in `hubspot.json`:

- `ingest.autoApply` — `true` applies the one allowed move; `false` proposes it in
  the digest instead.
- `ingest.allowStageAdvance` — gates that one move.
- `ingest.allowStageClose` — **off by default**: *Closed Lost* is never set
  automatically; always listed for approval. (There's no distinct "Closed Won"
  stage in this pipeline anymore — a won deal just progresses to Portal
  onboarding → First order placed → Actively ordering.)
- Deals are **never silently demoted** — that's always manual too.

## Lemlist → HubSpot mapping

Lemlist replies are read via `get_inbox_conversations` → `get_inbox_conversation`,
which carries `aiLeadInterest` (**positive** ≥4 / neutral / **negative** ≤1). The
lead's email is matched to the HubSpot deal's contact. A positive reply on a deal
still sitting in *In conversation (Lemlist)* advances it one step (see above); a
negative reply is a lost signal (propose *Closed Lost* for approval, never set it
automatically).

> If Lemlist's **native HubSpot integration** is also enabled in your Lemlist
> account, sequence sends/opens/replies additionally appear on the HubSpot contact
> timeline — belt-and-suspenders. The workflow reads Lemlist directly via MCP
> regardless, so it doesn't depend on that sync being on.

## Scheduling

Schedule **one** daily Claude Code web session with the prompt **`/daily-crm`** —
it runs **W1 → W2** in order (sync every deal from all four sources → derive
Notion to-dos). W0 (new-deal creation) is **disabled** as of 2026-07-25 — see
`new-deal-discovery.md`. Daily, **07:00 Europe/Amsterdam**, on **Sonnet**. See
`SCHEDULING.md`.

## Secrets — usually NONE

The core pipeline reads `hubspot.json` from git and all sources from the connected
account. Add an env secret only if a scheduled (headless) run's preflight shows a
source ⚠️ unavailable — then add just that one (e.g. a HubSpot private-app token as
`HUBSPOT_TOKEN`, Gmail via `EMAIL_ACCOUNTS_JSON`, Slack via
`SLACK_WORKSPACES_JSON`). Real tokens go in env secrets or a gitignored
`hubspot.local.json` — **never** in the committed `hubspot.json`.

## Safety

- Writes are conservative: **add a note always; move the stage only in the one
  narrow Lemlist-reply case above.** Never delete; never auto-close; never create
  a deal.
- New-deal creation (W0) is **disabled** (`newDealDiscovery.enabled: false`). W1
  only updates existing deals — a genuine new opportunity gets named in the digest
  for Shaffy to add by hand, not created automatically.
