# Attio pipeline clean-up — migration plan (DRY RUN, nothing written yet)

Goal: collapse the messy 23-stage Deals pipeline into **8 clean stages**, and
re-classify deals using **actual email conversations** (last 3–4 months) as the
source of truth — not company domain (Attio domains are incomplete) and not the
old stage label. The #lemlist-all-replies Slack channel is an *index* of who
replied (mostly cold LinkedIn outreach); the genuinely-progressed ones lead into
email/calendar threads, which is where stage is decided.

> Status: **proposal only.** No deal has been modified. Two things gate the
> apply step — see "Blocker" and "Open decisions" at the bottom.

## Target pipeline (8 stages, in order)

| # | Stage | Meaning |
|---|---|---|
| 1 | Meeting Booked | Call is scheduled |
| 2 | Discovery / Qualified | Call happened, real problem exists |
| 3 | Scope Defined | Project shape is clear |
| 4 | Proposal Sent | Proposal / SOW sent |
| 5 | Verbal Yes / Closing | Active closing conversation |
| 6 | Nurture / Recontact Later | Good fit, but stale or timing not now |
| 7 | Won | Closed-won (incl. active & completed clients) |
| 8 | Lost | Bad fit / no longer worth pursuing |

Note: the target list has no separate "Active client" stage, so current active &
completed clients fold into **Won**. Say if you'd rather split that out.

## Mapping: current 23 stages → 8 (starting point, overridden by email evidence)

| Current stage (count) | Default new stage |
|---|---|
| 4.A Proposed to plan a call (20), 4.B Demo planned (3), 2.B Reschedule call (1) | **Meeting Booked** |
| 1. SQL - reply (24), 0. MQL - Potential target (4) — *only if a real call/discovery happened* | **Discovery / Qualified** |
| 5. Contracting - indicated interest (19) — *when scope is clear* | **Scope Defined** |
| 6.C Proposal (5) | **Proposal Sent** |
| 6. Verbal agreed - project based (1), 5. Contracting (active closing) | **Verbal Yes / Closing** |
| 3. Recontact later (19), On hold - reconnect later (30), stale 4.B Shaffy follow-up (13) / 4.B (Joep follow-up) (41), soft/no-progress 1. SQL replies | **Nurture / Recontact Later** |
| 7.B Active Client (7), 7. Active Project Client, 8. Completed Client (13), 8.a Finished/finalize invoicing (2), Won 🎉 | **Won** |
| Lost - not interested (58), Lost - price (4), Lost, Auto-Reply (3), hard-no replies | **Lost** |

The middle band (SQL reply / follow-ups / contracting — roughly 120 deals) is
**not** mechanical: each is decided by reading its email thread. The Won/Lost ends
and the cold mass can largely be rule-based.

### Cold-mass rule (proposed)
- Explicit decline / unsubscribe / "not interested" → **Lost**.
- Everything else stale with no real progress → **Nurture / Recontact Later**.

## Researched sample (email-verified) — proof of method

| Deal | Email evidence | Current | → Proposed |
|---|---|---|---|
| Crewline.ai | Recurring weekly "Techtower/Crewline" standups; AM briefing doc | 7.B Active Client | **Won** |
| Hassans (hassans.gi) | Live weekly check-ins, campaign launched, ongoing scoping w/ partners | 7.B Active Client | **Won** |
| Gullimex | Holland Capital intro + Teams call "Discuss Rapportage automatisering"; project scoping | 6.C Proposal | **Scope Defined / Proposal Sent** (confirm) |
| TwinShape | Via ShiftInvest (Thijs Gitmans), ongoing MCP/technical work | 6. Verbal agreed | needs 1 more read |

## How the full pass will run (once unblocked)
1. Pull every deal (paginate all ~267).
2. Bucket: clear-Won (active/completed clients), clear-Lost (lost/unsubscribe/
   hard-no), and the active middle.
3. For the active middle + any recent (last 3–4 months) deal, read its Gmail
   thread(s) + calendar invites to set the precise stage.
4. Produce a per-deal table (deal → old → new → 1-line evidence) for review.
5. On approval, write `stage` via the Attio connector (update-record), in batches.

## Blocker
The Attio **connector can update deals but cannot create/rename/archive pipeline
stages.** The 8 stages must exist before any deal can be moved into them. Options:
- (a) You add the 8 stages in Attio settings (Deals → Deal stage → add statuses);
  I then migrate all deals and we archive the old 23. ← cleanest
- (b) Reuse & rename 8 existing stages (you rename in settings; I move deals).
- (c) You provide an Attio API token; I create/archive stages + migrate end-to-end.

## Open decisions
1. Stage setup: (a) / (b) / (c) above.
2. Won granularity: single **Won** (folds active+completed clients), or keep a
   separate "Active client" stage?
3. Apply mode: dry-run table for review first, or apply directly in batches?
4. Cold-mass rule: confirm "explicit no → Lost, else → Nurture".
