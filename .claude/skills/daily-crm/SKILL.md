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
- **Refresh Description + Next step** on every deal in `attio.json →
  fieldRefresh.scopeStages` you touch (native `deal_description`/`next_step`
  fields, ≤2 sentences each) — keeps the deal card itself current, not just the
  note history.
- **Create an Attio Task** when something is clearly owed on our side (high bar
  — an unanswered question, a clearly important flag), deduped on its `Ref:`
  line and completed once a fresh note shows it's resolved. Detail in
  `attio-ingest.md` §5.
- **Keep the stage honest** — advance/hold per the evidence; never silently
  demote an active client. The one exception: a deal in an active stage whose
  reply is a plain opt-out (unsubscribe, "not interested") gets reclassified to
  `Lost - not interested` on sight — that's hygiene, not demotion. Detail in
  `attio-ingest.md`.

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
