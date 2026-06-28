# W1 — Daily Attio Ingest (all raw comms → Attio, the source of truth)

**Run this every morning (~7:00 Europe/Amsterdam), first in the pipeline.** It
reads the last day of communication across **Gmail, Slack, meeting notes (Granola
+ Fireflies), and calendar** and writes it **all into Attio**: a dated note on
every active deal/client that had activity, plus a **stage move** when the
evidence warrants. Attio becomes the complete raw record of every client
interaction — the source of truth everything else reads from.

> **Two-workflow architecture.**
> - **W1 — this file (`attio-ingest.md`) → Attio.** Ingest *all* raw comms as
>   notes and keep deal **stages** current. Runs first.
> - **W2 — `daily-open-items.md` → Notion.** Derives **per-client To-Dos**,
>   reading Attio (primary) plus fresh sources, and reconciles existing to-dos
>   (mark done / advanced). Runs ~15 min after W1.
>
> W1 owns *everything that goes into Attio*; W2 owns *what Shaffy needs to do*.
> Keeping these separate means one ingestion point (no double-writing the CRM)
> and one to-do brain.

**Config:** `attio.json` (token + the `attioIngest` block — see
`attio.example.json`) and `client.json` (owner, stack, lookbacks). Values below
are the **TechTower "client zero"** instance as a worked example.

---

## 0. Preflight

Check each dependency and open the run summary with a readiness line. A
missing/expired source is **skipped for the run** (flag it loudly) rather than
failing the whole ingest. A stage move is never made on a partial view if a key
source is down (see §4).

- **Attio** (required) — token valid, `deals` object reachable.
- **Gmail** (`stack.email`) — connected inbox, scoped to `stack.email.domains`.
- **Slack** (`stack.chat`) — connected workspace (+ any `slack-workspaces.json`).
- **Meetings** — **Granola and/or Fireflies** (both, if both are connected).
- **Calendar** — optional; detects booked/held calls.

## 1. Build the working set (which deals to touch)

Don't re-scan all ~290 deals daily. Each run looks at:

1. **All active clients & live deals** — stage in `6. Active client`,
   `7. Active Project Client`, `8. Completed Client`, `8.a Finished…`, or any
   active pipeline stage (`1.–5.`). These get a freshness check and a comms note
   on every day they have activity (Hassans, Crewline, Norvell Jefferson, Wietec…).
2. **Any deal with fresh activity** in the lookback (default **2 days**; widen to
   `lookbackDays` after a gap). Match activity → deal via sender/participant email
   → `people.email_addresses` → linked deal, or company `domains` → `companies` →
   deal, or a name/alias hit.
3. **Genuine new inbound** not yet a deal → new-deal path (§6c).

> Relies on people↔deal / company↔deal links staying populated (done in the
> one-off cleanup). Unlinked records degrade matching — keep links healthy.

## 2. Gather communication per deal (last 24–48h)

For each deal in the working set, pull what's new since the last run:

- **Gmail** — latest thread(s) with the contact(s): most recent messages, who
  replied last, the date. (`mcp__Gmail__search_threads` + `get_thread`, native.)
- **Slack** — shared/Connect channels + DMs with the contact, and internal
  mentions of the account.
- **Meeting notes** — **Granola** (`mcp__Granola__*`) and **Fireflies**
  (`mcp__Fireflies__*`): recent meeting summaries / transcripts / action items,
  matched to the deal by attendee email / company.
- **Calendar** — upcoming or just-held calls (booked = a real scheduling signal).

## 3. Decide the stage (email/meetings lead)

Use the **most recent real interaction**; fall back to the existing CRM note only
when no live signal exists. Recency = `recencyWeeks` (default 6). Use **only**
these existing Attio status titles:

| Signal in the last interaction | New stage |
|---|---|
| Explicit decline / not interested / unsubscribe / not a fit | `Lost - not interested` |
| Call/meeting scheduled or being arranged | `2. Planning call` (or `3. Demo planned` for a product demo) |
| Live, concrete scoping/requirements (≤ `recencyWeeks`) | `4. Scope agreed` |
| Formal proposal/pricing sent, awaiting decision | `5. Proposal sent` |
| Genuinely positive reply, no call yet, recent | `1.  Positive reply` *(two spaces after "1.")* |
| Verbal yes / actively closing | advance toward `6. Active client` |
| Kickoff done / ongoing delivery | `6. Active client` → `8. Completed Client` as work wraps |
| Interest but quiet / "reconnect later" / no reply > `recencyWeeks` | `Nurture / recontact` |
| Unclear and not demonstrably active | leave unchanged |

**Forward-bias for active clients:** never silently demote an active/completed
client to Nurture on a quiet day — move a client backward only on an explicit
signal (churn/pause). Quiet ≠ lost for someone already engaged.

## 4. Safety (before any write)

- **Evidence required** for a stage move (a concrete, dated interaction). No
  interaction → no move (still note any activity).
- **One-step, explainable moves**, justified in the note (old → new + evidence).
- **Degraded run = conservative:** a source down at preflight → note-only for
  deals that depend on it; no stage move.
- **Never auto-close-won/lost a client** without an explicit human-readable signal.
- **Reversible + logged:** the note records the prior stage; the digest lists
  every move.
- Respect `attioIngest.autoApply`: `true` = apply moves; `false` = note + propose
  the move in the digest for approval (notes still post either way).

## 5. Idempotency / dedup

- Stamp every note with `[attio-ingest YYYY-MM-DD]`. If today's marker already
  exists on the deal, update/skip rather than duplicate.
- Dedup meeting summaries by meeting id (don't re-summarize a noted call).
- Write a note only when there is genuinely new content since the last marker.

## 6. Write to Attio

**(a) Stage move** — `update-record(object="deals", record_id, {stage:"<title>"})`
when §3 + §4 warrant it.

**(b) Comms note — the core of "all raw info to Attio"** —
`create-note(parent_object="deals", parent_record_id, title, content)`, run for
**every active client / live deal that had activity**, even with no stage change:
> `[attio-ingest YYYY-MM-DD] <Source(s)>`
> • **What happened:** 2–4 lines — call notes (Granola/Fireflies), email summary,
>   Slack highlights.
> • **Decisions / asks:** what was agreed or requested.
> • **Stage:** `<old>` → `<new>` (reason) — or "unchanged".
> • **Next step / owner + date.**
This makes each deal a complete running log — the raw layer W2 reads from.

**(c) New inbound not yet in CRM** — if `attioIngest.createNewDeals` is true,
create the deal (name, stage from intent, owner, comment), find-or-create the
**person** (by email) and **company** (by domain), link both, add the first note.
Else list it in the digest for review.

## 7. Output — ingest digest

Post a short summary (`attioIngest.digestChannel`, else the run reply):
**Preflight** ✅/⚠️ per source · **Moves** (`Deal: old → new — why`) · **Notes
added** (count, active-client log entries called out) · **New inbound** (created/
proposed) · **Skipped/degraded**.

## Scheduling

Claude Code web scheduled session on this repo (`SCHEDULING.md`), **daily ~07:00
Europe/Amsterdam**, **Sonnet**. Enable via `attioIngest.enabled` in `attio.json`.

**Prompt:**
> Run W1, the Attio ingest defined in `attio-ingest.md` in this repo. Start with
> the Step 0 preflight and open with the readiness line, build the working set,
> gather Gmail/Slack/Granola/Fireflies/calendar activity, write stage moves +
> comms notes to Attio per the rules, and finish with the digest.

## Daily vs. periodic

This is **incremental** — it reacts to *new* activity. The one-off back-catalog
reclassification (146→Nurture / Lost / live-stage) is not repeated daily. For a
periodic deep re-sweep of stale deals, schedule a **weekly** wider-lookback run.
