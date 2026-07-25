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
> **2026-07-25 — Description/Next step, Attio Tasks, org-linking, and a
> live/frozen stage split, all added.** Per `attio.json`:
> - **Every deal you write a note on gets `deal_description` + `next_step`
>   refreshed (§4) — no stage exclusion.** Early pipeline, Nurture, or an
>   advanced client, all the same rule.
> - **Every deal you touch gets checked for a linked Company + Person, and
>   either gets created if missing (§5, `orgLinking`)** — the one narrow
>   exception to "W1 never creates records" (deal creation stays W0-only).
> - **Anything clearly owed on TechTower's side gets a native Attio Task (§6)**,
>   regardless of stage.
> - **Stage moves follow `attio.json → stagePolicy`: "live" stages
>   (`1.  Reply`, `2. Planning call`, `3. Demo planned`, `Nurture / recontact`)
>   move per evidence — including reactivating a Nurture deal straight back
>   into active pipeline on genuine new activity. "Frozen" stages (`4. Scope
>   agreed` onward — Scope agreed, Proposal sent, Active client, Completed
>   Client, invoicing) never get their stage touched by W1**; Shaffy manages
>   those by hand (§7).

**Config:** `attio.json` (`attioIngest`, `stagePolicy`, `fieldRefresh`,
`orgLinking`, `tasks`) + `client.json`. Lookback = `attioIngest.lookbackDays`
(default **3 days**; widen after a gap).

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
it to W0 (don't force-fit it onto another deal). **A deal currently in `Nurture /
recontact` or a frozen/advanced stage is still a valid match** — new activity on
a parked or established deal matters just as much as on a fresh prospect; don't
skip clustering just because the stage looks settled.

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

## 4. Refresh Description + Next step on every deal you touch — no stage exclusion
(Per `attio.json → fieldRefresh`.) Whenever you write a note on a deal — **any
deal, any stage**: early pipeline, `Nurture / recontact`, or an advanced/frozen
one like `6. Active client` — also update its native `deal_description` and
`next_step` fields to match the fresh state. Max 2 sentences each, since they're
meant to be scannable on the deal card, not a full recap (the note already
carries the detail).
- `deal_description` = who the contact/company is + where things stand overall.
- `next_step` = the one concrete thing that happens next and who owns it (us or
  them). If nothing is outstanding, say so plainly (e.g. "No open ask — waiting
  on X"). For a deal just parked in Nurture, say what it's waiting on to revive.
- This is independent of the stage-move rules in §7 — refreshing the fields on
  a frozen-stage deal is always fine even though you'd never touch its stage.

## 5. Link the deal to its Company and Person — find-or-create, don't just flag
(Per `attio.json → orgLinking`.) Whenever you touch a deal (note written, field
refresh, stage move), check it has a linked **Company** and a linked **Person**
covering the actual counterpart in the conversation:
1. **Company** — if `associated_company` is empty, search `companies` by the
   contact's email domain. Reuse an exact match; otherwise create
   `{name, domains:[domain]}` and link it. **Skip for a personal-domain email**
   (`orgLinking.personalDomains`) — link the person, leave the company blank,
   flag it in the digest instead of guessing an employer.
2. **Person** — if the actual counterpart isn't in `associated_people`, search
   `people` by email. Reuse an exact match; otherwise create
   `{name:[{first_name,last_name,full_name}], email_addresses:[email]}` and
   link it to the deal.
Always search before creating — never duplicate a company or person that
already exists. This is the one exception to W1 never creating records: it
creates/links **companies and people**, never a **deal** (that's still W0-only,
`attioIngest.createNewDeals` stays `false`).

## 6. Create an Attio Task when something is owed on our side — high bar
(Per `attio.json → tasks`, via `mcp__Attio__create-task` /
`mcp__Attio__list-tasks` / `mcp__Attio__update-task`.) Not a catch-all for every
loose end — only when it's **clearly deal-related** and one of:
- a person **explicitly asked a question or made a request and we clearly
  haven't answered it yet** (pricing, scope, scheduling, a document);
- something **clearly important surfaced** that needs flagging — a decision
  point, a real risk, a hard deadline — not routine chatter.

Applies regardless of stage — a frozen-stage client waiting on us for something
deserves a task just as much as an early prospect. Do **not** create one for
minor/ambiguous items, general "might be worth checking in" nudges, or anything
you're not confident actually needs action. When in doubt, leave it out —
mention it in the digest instead.

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

## 7. Keep the stage honest — live stages move, frozen stages don't
(Per `attio.json → stagePolicy`.) Every stage is either **live** or **frozen**:

- **Live** (`1.  Reply`, `2. Planning call`, `3. Demo planned`,
  `Nurture / recontact`) — W1 moves these per evidence:
  - call booked → `2. Planning call`
  - demo/scoping call booked or held → `3. Demo planned`
  - live scoping/handshake reached → `4. Scope agreed` — **this is the one
    crossing into a frozen stage**; once there, hands off (see below).
  - explicit opt-out/negative reply → `Lost - not interested` (hygiene, not
    demotion — this was never a real conversation).
  - no reply/meeting for more than `recencyWeeks` → `Nurture / recontact`.
  - **A `Nurture / recontact` deal that gets genuinely new activity
    (`stagePolicy.nurtureReactivation`) moves back into whichever live stage
    the fresh evidence fits** — usually `1.  Reply` for a bare reply,
    `2. Planning call` if a call gets booked directly. Say explicitly in the
    note why it's back and what's new — this should never read as a silent
    flip.
  - `Lost - not interested` deals are **not** auto-reactivated — flag a
    positive reversal in the digest for Shaffy to move by hand instead.
- **Frozen** (`4. Scope agreed`, `5. Proposal sent`, `6. Active client`,
  `8. Completed Client`, `8.a Finished, need to finalize invoicing`) — **W1
  never changes `stage` on a deal already here, no matter what happened** (a
  call held, a proposal countered, an order placed). Log it all in the note;
  Shaffy moves the card. **Never silently demote an active client** — that's
  exactly the failure mode this freeze prevents.

Record every move as old → new + the reason in the note. Honor
`attioIngest.autoApply` (false = propose in the digest instead of applying).

## 8. Idempotency
- Dedup notes at the **content** level (step 3.1), not just the date marker.
- Dedup a call by its meeting id — never re-summarize a call already noted.
- Dedup Attio Tasks on the `Ref:` line in the task content (§6).
- Dedup org-linking by searching first (§5) — never create a duplicate
  company/person that already matches by domain/email.
- Only write when there is genuinely new, un-noted content.

## 9. Output — ingest digest
**Preflight** ✅/⚠️ · **Accounts touched** · **Notes added** (deal + 1-line gist;
active clients called out) · **Description/Next step refreshed** (deal — 1-line
gist, per §4) · **Companies/people linked or created** (deal — what, per §5) ·
**Tasks created** (deal — subject — owed by whom) + **Tasks completed** (deal —
subject — resolved by what) · **Stage moves** (`deal: old → new — why`,
including Nurture reactivations and hygiene reclassifications to Lost) ·
**Lost deals flagged for manual reactivation** (deal — why) · **Gaps closed**
(meaningful comms that had no note before) · **New deals handed to W0** ·
**Skipped/degraded**.

## Scheduling
Claude Code web routine, **daily ~07:00 Europe/Amsterdam**, **Sonnet**, after W0.
Enable via `attioIngest.enabled`. Hand-off to W2 turns these notes + stages into
the per-client Notion to-dos.
