# HubSpot pipeline setup — the single source of truth

The pipeline sync keeps **every HubSpot deal current** from the full
conversation. Each morning it reads **Granola, Slack, Lemlist, and email**, writes
a dated note to the right deal, and — as of **2026-07-25, at Shaffy's request** —
moves the deal stage in exactly **one** narrow situation (see "How a stage
moves" below), refreshes the board-card fields on early-funnel deals (see
"Description + Next step"), creates a HubSpot Task on a deal whenever something
is clearly owed on our side (see "HubSpot Tasks"), and backfills a missing
Company on deals a separate Lemlist automation creates (see "Organization
backfill"). HubSpot holds everything — it's the canonical record.

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

## Description + Next step (board-card fields)

**Added 2026-07-25, per Shaffy.** Whenever W1 writes a `[hubspot-ingest]` note on
a deal that's in one of the four **early-funnel stages**
(`pipelines.clinicPartnerships.earlyFunnelStages` — Inbound request, In
conversation (Lemlist), Asked for information, Demo scheduled), it also refreshes
that deal's native `description` and `hs_next_step` (labeled "Next step" in the
UI) fields. **Max 2 sentences each** — these render directly on the board cards,
so they need to stay short and scannable, not turn into another note. Deals at
Contracting or later are *not* touched this way; Shaffy keeps those two fields
current by hand.

A one-time backfill populated these fields for all 41 open/lost deals on
2026-07-25; going forward it's incremental, refreshed only on deals that get a
fresh note.

## HubSpot Tasks — what SwimScore owes

**Added 2026-07-25, per Shaffy.** When the sweep finds something clearly
outstanding on our side, it creates a native HubSpot `Task` linked to the deal
(`hubspot.json → tasks`) — pipeline-wide, not limited to early-funnel deals —
so it shows up in HubSpot's own task queue instead of only the daily digest.

**High bar, deliberately.** This is not a catch-all for every loose thread — only:
- a person **explicitly asked a question or made a request** that's **clearly
  still unanswered**, or
- something **clearly important** surfaces in email that needs flagging (a
  decision point, a real risk, a hard deadline).

Skip anything minor, ambiguous, routine, or already covered by the stale-deal
follow-up flow (`staleFollowUp` — that's for "gone quiet", not "owed a reply"). A
task list cluttered with tiny items gets ignored, which defeats the point.

**Before creating one, the sweep re-checks the deal's existing notes and latest
activity** for that specific item — if the reply already went out or the thing
already happened, no task gets created. Tasks are deduped on a `Ref:
hubspot:task:<dealId>:<slug>` line in the task body (search open, non-completed
tasks on the deal for a match before creating); an existing open task gets marked
`COMPLETED` if a fresh note shows its item was resolved — never marked complete
without that evidence. Default owner is `162479602` ("Support SwimScore" —
shaffy@myswimscore.com's HubSpot user record), due today, priority HIGH if >7
days overdue or blocking a live deal, else MEDIUM.

## Organization backfill (Lemlist-replies bucket)

**Added 2026-07-25 (2), per Shaffy.** Shaffy's team runs a **separate
automation** (outside this workflow, outside our control) that now auto-creates
a HubSpot deal for every Lemlist reply, landing it straight in **In conversation
(Lemlist)**. This workflow still never creates deals — but that automation
doesn't reliably attach a Company, so whenever W1 touches a deal sitting in that
exact stage (a note, the stage-advance check, the field refresh), it also:

1. Checks whether the deal has an associated Company.
2. If not, takes the domain from the deal's associated contact's email (skipping
   personal domains — gmail.com, yahoo.com, etc. — those get flagged instead of
   guessed).
3. Searches for an existing company by that domain and **associates** it if
   found (reuse, never duplicate).
4. If none exists, **creates** one (`{name, domain}`) and associates it to both
   the deal and the contact.

This is the **one narrow exception** to "this workflow never creates records" —
company-only, and only to backfill a gap the external flow left behind. It never
creates a contact or a deal. Verified 2026-07-25: all 19 deals then sitting in
"In conversation (Lemlist)" already had a company linked (they predated the new
automation) — so this is a forward-looking safety net, not a backlog to clear.

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
  narrow Lemlist-reply case above; refresh Description/Next step only on
  early-funnel deals; create a Task only for a high-bar, clearly-deal-related,
  clearly-still-owed item, verified against existing notes first.** Never delete;
  never auto-close; never create a deal.
- New-deal creation (W0) is **disabled** (`newDealDiscovery.enabled: false`). W1
  only updates existing deals — a genuine new opportunity gets named in the digest
  for Shaffy to add by hand, not created automatically.
