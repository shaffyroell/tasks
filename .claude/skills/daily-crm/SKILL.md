---
name: daily-crm
description: Daily key-account sweep. Reads ALL recent email, Slack, and call notes (Granola/Fireflies), links each to the right Attio deal (adds a note only if it isn't already there), keeps stages honest, then reconciles per-client action items in Notion. Use as the scheduled daily run.
---

Act like the person who **manages these key accounts**. Don't transcribe — read
everything, understand what's happening with each account, and make Attio + Notion
reflect it. Work the four steps below, in order.

## 1. Read everything from the last few days
- **All emails** — every Gmail thread with a new inbound/outbound message
  (native Gmail MCP only). Read the substance, commitments, and asks.
- **All Slack** — sweep every client / Slack-Connect channel **and** the internal
  channels where account work is discussed (e.g. `#<client>-internal`), last few
  days; read threads, not just top messages.
- **All call notes** — every Granola note and Fireflies transcript/summary in the
  window; these hold the real decisions, read them in full.

## 2. Link each conversation to the right Attio deal — note it if missing
For every meaningful communication, find its account's **deal** (participant
email → person → deal; company domain → company → deal; or name/alias). Then:
- **Check the deal's existing notes first.** If this conversation/call is already
  captured, **skip it** (no duplicates — match on the thread/meeting + content,
  not just the date).
- **If it's missing, add a note** (`create-note` on the deal) summarizing: what
  happened, decisions & commitments (what we owe / they owe, by when), risks/
  blockers, and the next step. If a meaningful comm has no note, that's a gap —
  close it.
- Run W0 (`new-deal-discovery.md`) first for genuinely NEW opportunities with no
  deal yet (create deal + link company + person + note); don't force-fit new
  inbound onto an existing deal.
- **Refresh Description + Next step** on **every deal you touch, any stage** —
  early pipeline, Nurture, or an advanced client (native `deal_description`/
  `next_step` fields, ≤2 sentences each) — keeps the deal card itself current,
  not just the note history. No stage exclusion. Detail in `attio-ingest.md` §4.
- **Link the deal's Company + Person, find-or-create if missing** — the one
  exception to never creating records in Attio (deal creation stays W0-only).
  Detail in `attio-ingest.md` §5.
- **Create an Attio Task** when something is clearly owed on our side (high bar
  — an unanswered question, a clearly important flag), deduped on its `Ref:`
  line and completed once a fresh note shows it's resolved. Applies regardless
  of stage. Detail in `attio-ingest.md` §6.
- **Keep the stage honest per the live/frozen split** (`attio.json →
  stagePolicy`): `1.  Reply`, `2. Planning call`, `3. Demo planned`, and
  `Nurture / recontact` move per the evidence — including moving a Nurture
  deal back into active pipeline on genuine new activity, and reclassifying a
  plain opt-out to `Lost - not interested` as hygiene. **`4. Scope agreed`
  onward is frozen — never move that stage automatically**, Shaffy handles
  those by hand; never silently demote an active client. Detail in
  `attio-ingest.md` §7.

## 3. Reconcile action items in Notion (per client)
Run W2 (`daily-open-items.md`): read Attio (the notes/stages you just wrote) plus
the same fresh sources, then for each client's Notion board **see if the action
items already exist** — mark Done what was handled, advance what moved, dedup on
`Ref` — and add genuinely new to-dos, **each written in the house style per
`STYLE.md`** (verb-first, concise, client-safe, no arrows).

## 4. Report
End with a digest: accounts touched, notes added (with gaps closed), stage moves,
new deals created, and the Notion to-dos added/updated/closed per client.

Think critically per account the whole way through: progressing, stalling, or at
risk? a commitment slipping? an upsell/renewal cue? That judgement is the job.
Config = committed `attio.json` + `clients.json`. Open with a per-source preflight
line; stop and report if a critical source is down.
