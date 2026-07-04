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
- **Keep the stage honest** — advance/hold per the evidence; never silently
  demote an active client. Detail in `attio-ingest.md`.

## 3. Reconcile action items in Notion (per client)
Run W2 (`daily-open-items.md`): read Attio (the notes/stages you just wrote) plus
the same fresh sources, then for each client's Notion board **see if the action
items already exist** — mark Done what was handled, advance what moved, dedup on
`Ref` — and add genuinely new to-dos, **each written in the house style per
`STYLE.md`** (verb-first, concise, client-safe, no arrows).

## 4. Report
Post the recap as a Slack message to the **`#shaffy-recap`** channel (ID
`C0BEYR36UQ3` in the `techtower-ai` workspace), using this exact structure:

```
*Daily Key-Account Sweep — <date>*

*Key action items:*
• Account: text text
• Account: text text

*Overall goals per account we're working towards:*
• Account: text text
• Account: text text

*Possible blockers to solve:*
• Account: text text
• Account: text text

*Next 2-3 goals to push proactively (post-current-items):*
• Account: goal; goal; goal — the roadmap-level pushes Shaffy should be
  driving next per account, beyond today's open items (upsell/renewal cues,
  scope expansion, next milestone). This is what makes the sweep proactive
  instead of reactive — always fill this in, even on a quiet week.
• Account: goal; goal; goal

*Suggestions to Joep, Roman, Swayam:*
• To Joep: text
• To Roman: text
• To Swayam: text (if nothing surfaced for someone that day, say so explicitly —
  never invent a suggestion just to fill the line)

Notion boards:
• Internal/pipeline: <internalBoard.todoDbUrl from clients.json>
• <Client>: <link to that client's dashboardPageId from clients.json>
  (one line per client entry in clients.json — always list every one, even if
  quiet that day, so the message doubles as a navigation index)
```

**Note on the Slack identity:** the connected Slack integration authenticates as
its own workspace member (`Shaffy`, `U07CJK9H78A`, shaffy.roell@gmail.com) — a
**different** account from Shaffy's actual daily-use account (`Shaffy Roell`,
`U0A2CAKTJA2`, shaffy@techtower.ai). Posting to `#shaffy-recap` (a channel the
connector has joined) sidesteps this; do not send a "DM to self," as that lands
in the connector's own inbox, which Shaffy doesn't check.

Think critically per account the whole way through: progressing, stalling, or at
risk? a commitment slipping? an upsell/renewal cue? That judgement is the job.
Config = committed `attio.json` + `clients.json`. Open with a per-source preflight
line; stop and report if a critical source is down.
