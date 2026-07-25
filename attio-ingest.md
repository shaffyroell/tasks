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
>
> **2026-07-25 — Description/Next step + Attio Tasks added.** Every genuinely
> active deal (per `attio.json → fieldRefresh.scopeStages`) now also gets its
> native **Deal description** and **Next step** fields kept current (§4), and
> anything clearly owed on TechTower's side gets a native **Attio Task** (§5) —
> not just a note. This mirrors the equivalent HubSpot/SwimScore workflow, with
> one deliberate difference: the field refresh here covers **every active stage
> through "6. Active client,"** not just early-funnel — Shaffy wants ongoing
> clients to carry current context too, not only fresh prospects. All existing
> active deals were backfilled once on 2026-07-25; from here it's incremental.

**Config:** `attio.json` (`attioIngest`, `fieldRefresh`, `tasks`) + `client.json`.
Lookback = `attioIngest.lookbackDays` (default **3 days**; widen after a gap).

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

## 4. Refresh Description + Next step on every active deal touched
(Per `attio.json → fieldRefresh`.) Whenever you write a note on a deal whose
stage is in `fieldRefresh.scopeStages` (`1.  Reply` through `6. Active client`),
also update its native `deal_description` and `next_step` fields to match the
fresh state — **max 2 sentences each**, since they're meant to be scannable on
the deal card, not a full recap (the note already carries the detail).
- `deal_description` = who the contact/company is + where things stand overall.
- `next_step` = the one concrete thing that happens next and who owns it (us or
  them). If nothing is outstanding, say so plainly (e.g. "No open ask — waiting
  on X").
- A deal moving into `8. Completed Client` / `8.a Finished, need to finalize
  invoicing` gets a one-off refresh reflecting that (e.g. next step = "Send
  final invoice"), but isn't part of the daily loop after that.
- Skip Nurture/Lost/Lead/`0. Potential target`/`AI Suggested` — not active
  pipeline, not in scope.

## 5. Create an Attio Task when something is owed on our side — high bar
(Per `attio.json → tasks`, via `mcp__Attio__create-task` /
`mcp__Attio__list-tasks` / `mcp__Attio__update-task`.) Not a catch-all for every
loose end — only when it's **clearly deal-related** and one of:
- a person **explicitly asked a question or made a request and we clearly
  haven't answered it yet** (pricing, scope, scheduling, a document);
- something **clearly important surfaced** that needs flagging — a decision
  point, a real risk, a hard deadline — not routine chatter.

Do **not** create one for minor/ambiguous items, general "might be worth
checking in" nudges, or anything you're not confident actually needs action.
When in doubt, leave it out — mention it in the digest instead.

**Before creating, verify it isn't already done** — re-check the deal's existing
notes and the latest activity on that specific item. Then **dedup**: call
`mcp__Attio__list-tasks(linked_record_object="deals", linked_record_id, 
is_completed=false)` and check each task's content for an existing
`Ref: attio:task:<dealId>:<slug>` match (per `tasks.dedupRefLine`) — skip if
found. Otherwise `mcp__Attio__create-task` with a short specific subject line +
1-2 sentences of context + the `Ref:` line on its own line,
`assignee_workspace_member_id` = `tasks.defaultAssigneeWorkspaceMemberId`,
`linked_record_object`/`linked_record_id` set to the deal, `deadline_at` = today.
**If a prior open task's item is now resolved** (a fresh note shows the reply
went out / the thing got done), call `mcp__Attio__update-task` with
`is_completed:true` instead of leaving it stale — Attio tasks have no editable
body, so completion is the only signal you can update.

## 6. Keep the stage honest
Move the deal's stage when the evidence warrants (call booked → `2. Planning
call`; live scoping → `4. Scope agreed`; proposal pending → `5. Proposal sent`;
explicit no → `Lost - not interested`; gone quiet > `recencyWeeks` → `Nurture /
recontact`). **Never silently demote an active client** on a quiet day — only on
an explicit churn/pause signal. The one standing exception: a deal sitting in an
active stage whose actual reply content is a plain opt-out/negative (unsubscribe,
"not interested," "remove me") was never a real conversation — reclassify it to
`Lost - not interested` on sight, that's hygiene, not demotion. Record old → new
+ the reason in the note. Honor `attioIngest.autoApply` (false = propose in the
digest instead of applying).

## 7. Idempotency
- Dedup notes at the **content** level (step 3.1), not just the date marker.
- Dedup a call by its meeting id — never re-summarize a call already noted.
- Dedup Attio Tasks on the `Ref:` line in the task content (§5).
- Only write when there is genuinely new, un-noted content.

## 8. Output — ingest digest
**Preflight** ✅/⚠️ · **Accounts touched** · **Notes added** (deal + 1-line gist;
active clients called out) · **Description/Next step refreshed** (deal — 1-line
gist, per §4) · **Tasks created** (deal — subject — owed by whom) + **Tasks
completed** (deal — subject — resolved by what) · **Stage moves** (`deal: old →
new — why`, including hygiene reclassifications to Lost) · **Gaps closed**
(meaningful comms that had no note before) · **New deals handed to W0** ·
**Skipped/degraded**.

## Scheduling
Claude Code web routine, **daily ~07:00 Europe/Amsterdam**, **Sonnet**, after W0.
Enable via `attioIngest.enabled`. Hand-off to W2 turns these notes + stages into
the per-client Notion to-dos.
