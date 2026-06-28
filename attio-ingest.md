# W1 — Daily Attio Ingest (read everything, think like an account manager)

**Run this every morning (~7:00 Europe/Amsterdam), after W0.** Your job is to act
like **the person who manages these key accounts**: read *all* of the last few
days' communication, understand what's actually happening with each account, and
make sure **Attio reflects it** — a note on the right deal for every meaningful
conversation, and the stage kept honest.

> **This is comms-FIRST, not deal-first.** Start from the communications (every
> email, Slack message, and call), then attach each to the right deal — **don't**
> start from a list of deals and hope activity matches. If you read a meaningful
> client email/Slack/call and its deal has no note for it, that is a **miss** —
> always close that gap.
>
> **Pipeline:** W0 (`new-deal-discovery.md`) → **W1 (this)** → W2
> (`daily-open-items.md`). W0 creates new deals; W1 logs all comms + stages on
> existing deals; W2 turns it into per-client Notion to-dos.

**Config:** `attio.json` (`attioIngest`) + `client.json`. Lookback =
`attioIngest.lookbackDays` (default **3 days**; widen after a gap).

---

## 0. Preflight
Check each source and open with a readiness line: **Attio** (write), **Gmail**
(native), **Slack**, **Granola + Fireflies**, **calendar**. Skip a down source
loudly; never move a stage on a partial view if a key source is down.

## 1. READ EVERYTHING in the window (don't pre-filter to a working set)
- **Email** — every Gmail thread with a new inbound or outbound message in the
  last `lookbackDays`. Read the substance, who said what, dates, and any
  commitments/asks. (Native Gmail MCP only.)
- **Slack** — sweep **all** relevant channels from the last few days: each client
  / Slack-Connect channel **and** the internal channels where account work is
  discussed (e.g. `#<client>-internal`). Read threads, not just top messages.
- **Calls** — **every** Granola meeting note and **every** Fireflies transcript/
  summary in the window. These carry the real decisions; read them in full.
- **Calendar** — calls held or booked (a scheduling signal).

## 2. Cluster each communication to its account/deal
Group everything by company / person, then resolve each cluster to its Attio
**deal**:
- participant email → `people.email_addresses` → linked deal, or
- company domain → `companies.domains` → deal, or
- a name / alias match.
Use judgement: an intro thread, a call, and a Slack update about the same client
are **one account's** activity. If a real client/opportunity has **no deal**, hand
it to W0 (don't force-fit it onto another deal).

## 3. For each account with new substantive activity — note it **if not already there**
1. **Check first.** Read the deal's recent notes in Attio (search/list its notes).
   Decide whether *this* conversation/call is **already captured** — match by the
   meeting or thread and its content, not merely by date. **If it's already noted,
   skip** (no duplicate).
2. **If it's missing, add the note.** `create-note(parent_object="deals",
   parent_record_id, title, content)`, stamped `[attio-ingest YYYY-MM-DD]`:
   - **What happened** — synthesize the email/Slack/call: the gist, not a dump.
   - **Decisions & commitments** — what was agreed, what *we* owe them, what
     *they* owe us, by when.
   - **Risks / blockers** — anything that could stall or lose the account.
   - **Next step** — concrete action + owner + date.
   Write it the way an account manager briefs themselves before a call.
3. **Think critically, per account:** is this client progressing, stalling, or at
   risk? Is a commitment slipping? Is there an upsell or a renewal cue? Capture it
   — that judgement is the point, not transcription.

## 4. Keep the stage honest
Move the deal's stage when the evidence warrants (call booked → `2. Planning
call`; live scoping → `4. Scope agreed`; proposal pending → `5. Proposal sent`;
explicit no → `Lost - not interested`; gone quiet > `recencyWeeks` → `Nurture /
recontact`). **Never silently demote an active client** on a quiet day — only on
an explicit churn/pause signal. Record old → new + the reason in the note. Honor
`attioIngest.autoApply` (false = propose in the digest instead of applying).

## 5. Idempotency
- Dedup notes at the **content** level (step 3.1), not just the date marker.
- Dedup a call by its meeting id — never re-summarize a call already noted.
- Only write when there is genuinely new, un-noted content.

## 6. Output — ingest digest
**Preflight** ✅/⚠️ · **Accounts touched** · **Notes added** (deal + 1-line gist;
active clients called out) · **Stage moves** (`deal: old → new — why`) · **Gaps
closed** (meaningful comms that had no note before) · **New deals handed to W0** ·
**Skipped/degraded**.

## Scheduling
Claude Code web routine, **daily ~07:00 Europe/Amsterdam**, **Sonnet**, after W0.
Enable via `attioIngest.enabled`. Hand-off to W2 turns these notes + stages into
the per-client Notion to-dos.
