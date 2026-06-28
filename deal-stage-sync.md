# Daily Deal-Stage + Notes Sync (Attio = source of truth)

**Run this every morning (~7:15 Europe/Amsterdam), just after the open-items
sweep.** It reads the last day of communication across **Gmail, Slack, meeting
notes (Granola + Fireflies), and calendar**, then for each affected deal it
**moves the stage when the evidence warrants** and **appends a dated note**
summarizing what was discussed — so Attio is the single source of truth for all
client communication, including active clients (Hassans, Crewline, Norvell
Jefferson, Wietec, …).

> **How this differs from `daily-open-items.md`.** That sweep is *additive only*
> (note + follow-up task; stage changes need approval) and its output is the
> Notion tracker. **This** workflow is the opt-in **stage automation**: its whole
> job is to keep Attio deal **stages** and **notes** current. It is gated by its
> own config flag (`dealStageSync.enabled`) and writes only to Attio.

**Config:** same files as the rest of the repo — `attio.json` (API token +
mapping) and `client.json` (owner, stack, lookbacks). Add the `dealStageSync`
block shown in `attio.example.json`. Values below are the **TechTower "client
zero"** instance as a worked example.

---

## 0. Preflight

Check each dependency and open the run summary with a readiness line. A
missing/expired source is **skipped for the run** (flag it loudly) rather than
failing the whole sync. Note which signals are degraded — a stage move should
never be made on a partial view if a key source is down (see §4 safety).

- **Attio** (`stack.crm`, required) — token valid, `deals` object reachable.
- **Gmail** (`stack.email`) — connected inbox, scoped to `stack.email.domains`.
- **Slack** (`stack.chat`) — connected workspace (+ any `slack-workspaces.json`).
- **Meetings** (`stack.meetings`) — **Granola and/or Fireflies** connected.
- **Calendar** — optional; used to detect booked/held calls.

## 1. Build the working set (which deals to look at)

Don't re-evaluate all ~290 deals daily. Each run looks at:

1. **All active clients & live deals** — every deal whose stage is `6. Active
   client`, `7. Active Project Client`, `8. Completed Client`, `8.a Finished…`,
   or any active pipeline stage (`1.–5.`, `2. Planning call`, `3. Demo planned`,
   `4. Scope agreed`, `5. Proposal sent`). These always get a freshness check and
   a comms note when there's new activity.
2. **Any deal with fresh activity** in the lookback window (default **2 days**;
   first run after a gap: widen to `lookbackDays`). Match activity → deal via the
   sender/participant email → `people.email_addresses` → linked deal, or the
   company `domains` → `companies` → deal, or a name/alias hit.
3. **Genuine new inbound** not yet a deal → hand to the **new-deal path** (§6c).

> Relies on people↔deal and company↔deal links being populated. The pipeline
> cleanup (deal names normalized, domains backfilled, people/companies linked,
> values + `type_of_client_8` set) is a prerequisite and is now done — keep it
> healthy; unlinked records degrade matching.

## 2. Gather communication per deal (last 24–48h)

For each deal in the working set, pull what's new since the last run:

- **Gmail** — latest thread(s) with the deal's contact(s); read the most recent
  messages, who replied last, and the date. (`mcp__Gmail__search_threads` +
  `get_thread`. Native Gmail only.)
- **Slack** — messages in shared/Connect channels or DMs with the contact, and
  internal mentions of the account. (`Slack` connector; extra workspaces via
  `slack-workspaces.json`.)
- **Meeting notes** — **Granola** (`mcp__Granola__*`: recent meetings, transcript/
  summary) and **Fireflies** (`mcp__Fireflies__*`: recent transcripts, summaries,
  action items). Match a meeting to the deal by attendee email / company.
- **Calendar** — upcoming or just-held calls with the contact (booked = a real
  scheduling signal).

## 3. Decide the stage (email/meetings are the leading indicator)

Use the **most recent real interaction** to set the stage; fall back to the
existing CRM note only when no live signal exists. Today's date drives recency
(`recencyWeeks`, default 6). Use **only** these existing Attio status titles:

| Signal in the last interaction | New stage |
|---|---|
| Explicit decline / "not interested" / unsubscribe / "not a fit" | `Lost - not interested` |
| A call/meeting scheduled or actively being arranged | `2. Planning call` (or `3. Demo planned` for a product demo) |
| Live, concrete scoping/requirements (≤ `recencyWeeks`) | `4. Scope agreed` |
| Formal proposal/pricing sent, awaiting decision | `5. Proposal sent` |
| Genuinely positive reply, no call yet, recent | `1.  Positive reply` *(two spaces after "1.")* |
| Verbal yes / actively closing | keep advancing toward `6. Active client` |
| Kickoff done / ongoing delivery | `6. Active client` → `8. Completed Client` as work wraps |
| Interest but gone quiet / "reconnect later" / no reply > `recencyWeeks` | `Nurture / recontact` |
| Unclear and not demonstrably active | leave unchanged (don't churn) |

**Forward-bias for active clients:** never silently demote an active/completed
client to Nurture from a quiet day — only move a client backward on an explicit
signal (e.g. churn/pause). Quiet ≠ lost for someone already engaged.

## 4. Safety rules (before any write)

- **Evidence required.** Only move a stage when a concrete, dated interaction
  supports it. No interaction → no move (still add a comms note if there *was*
  activity).
- **One-step, explainable moves.** Prefer single-stage transitions; every move is
  justified in its note (old → new + the evidence + date).
- **Degraded run = conservative.** If a key source failed preflight, do **not**
  move stages that depend on it; note-only for the affected deals.
- **Never auto-close-won/lost a client** without an explicit human-readable
  signal in the thread.
- **Reversible + logged.** The note records the prior stage, so any move can be
  undone. The daily digest lists every move.
- Respect `dealStageSync.autoApply`: `true` = apply moves directly; `false` =
  write the note + propose the move in the digest for approval (notes still post).

## 5. Idempotency / dedup

- Stamp every note this workflow writes with a marker:
  `[stage-sync YYYY-MM-DD]`. Before writing, check the deal's notes — if a
  stage-sync note for **today** already exists, update/skip rather than duplicate.
- Dedup a meeting summary by the meeting id; don't re-summarize a call already
  noted on a prior run.
- Only write a note when there is genuinely new content since the last marker.

## 6. Write to Attio

**(a) Stage move** — `update-record(object="deals", record_id, {stage: "<title>"})`
when §3+§4 warrant it.

**(b) Comms note (always, when there's new activity)** —
`create-note(parent_object="deals", parent_record_id, title, content)` with:
> `[stage-sync YYYY-MM-DD] <Source(s)>`
> • **What happened:** 2–4 lines — call notes (from Granola/Fireflies), email
>   summary, Slack highlights.
> • **Decisions / asks:** what was agreed or requested.
> • **Stage:** `<old>` → `<new>` (reason) — or "unchanged".
> • **Next step / owner + date.**
This runs for **active clients and live deals every day they have activity**, so
the deal record is a complete running log of the relationship.

**(c) New inbound not yet in CRM** — if `dealStageSync.createNewDeals` is true,
create the deal (name, stage from intent, owner, comment), then find-or-create
the **person** (by email) and **company** (by domain) and link both, and add the
first comms note. Otherwise, list it in the digest for review. (This is the same
create+link path used in the pipeline backfill.)

## 7. Output — daily digest

Post a short summary (Slack channel `dealStageSync.digestChannel`, else the
open-items tracker, else the run reply):

- **Preflight:** ✅/⚠️ per source.
- **Moves:** `Deal: old → new — why` (one line each).
- **Notes added:** count, with the active-client log entries called out.
- **New inbound:** created or proposed.
- **Skipped/degraded:** anything a down source blocked.

---

## Scheduling

Run as a **Claude Code web scheduled session** on this repo (see `SCHEDULING.md`),
**daily ~07:15 Europe/Amsterdam** (stagger ~15 min after the open-items run so
they don't overlap), on **Sonnet** (retrieval + structured writes; no Opus needed
— Granola/Fireflies summaries and scoped lookbacks, not full transcripts).

**Prompt:**
> Run the deal-stage + notes sync defined in `deal-stage-sync.md` in this repo.
> Start with the Step 0 preflight and open with the readiness line, build the
> working set, gather Gmail/Slack/Granola/Fireflies/calendar activity, then apply
> stage moves + comms notes to Attio per the rules, and finish with the digest.

**Verify after the first run:** the digest opens with a Preflight line, each
source shows ✅, stage moves look right on a couple of deals, and active clients
(Hassans, Crewline, Norvell Jefferson, Wietec) each got a dated note for any day
they had activity. If a connector shows ⚠️ in the headless run, give it a
token-based credential (Gmail via `EMAIL_ACCOUNTS_JSON`, Slack via
`SLACK_WORKSPACES_JSON`, Attio via `ATTIO_JSON`; Granola/Fireflies via their own
tokens) — the preflight tells you exactly which.

## One-off vs daily

This daily job is **incremental** — it reacts to *new* activity. The big one-time
reclassification of the whole back-catalog (the 146→Nurture / Lost / live-stage
pass) is **not** repeated daily. If you want a periodic deep re-sweep of stale
deals, schedule a **weekly** variant with a wider lookback instead.
