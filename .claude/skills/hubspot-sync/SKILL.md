---
name: hubspot-sync
description: W1 — sync every open HubSpot deal from the full conversation (Granola, Slack, Lemlist, email) into HubSpot as dated notes. Advances stage only from "In conversation (Lemlist)" to "Asked for information"/"Demo scheduled" on a genuinely interested reply — never creates deals, never touches any later stage (Contracting onward is manual-only). Refreshes Description + Next step (max 2 sentences each, for the board cards) on early-funnel deals. Creates a HubSpot Task, linked to the deal, only for clearly deal-related items where we clearly owe a reply or something clearly important needs flagging — high bar, verified against existing notes/activity first so nothing gets double-flagged, never for minor stuff. Backfills a missing Company (by domain, matched or created) on any "In conversation (Lemlist)" deal it touches, since a separate automation now auto-creates those deals without one. HubSpot is the single source of truth. Use to keep the pipeline current each morning.
---

Run W1, the daily HubSpot deal sync defined in `hubspot-sync.md` in this repo.
Start with the Step 0 preflight across all four sources (HubSpot, Gmail, Slack,
Granola, Lemlist) and open with the readiness line. Load **every open deal** from
HubSpot (exclude Closed Lost), and for each deal gather the full recent
conversation across **Granola, Slack, Lemlist, email, and B2B Shopify website
inquiries** by matching the deal's contact emails / company domain (B2C patient
orders are out of scope — B2B only). Write one consolidated `[hubspot-ingest]` note to
each deal that had fresh activity **only if it isn't already there** (what
happened per channel, decisions/commitments, risks, next step). **Stage moves are
narrow, per `hubspot.json → ingest.stageAdvanceRule`**: only move a deal off "In
conversation (Lemlist)" to "Asked for information" (question/pricing ask, no call
yet) or "Demo scheduled" (call/demo agreed) — on a Lemlist reply showing real
interest, and only if the deal is currently in that first stage. Every deal
already at Contracting, Portal onboarding, First order placed, Actively ordering,
No orders (L3M), or Closed Lost keeps its stage untouched — log the development in
the note and let Shaffy move it himself.

**Description/Next step refresh, early-funnel deals only** (per `hubspot.json →
ingest.fieldRefresh` + `pipelines.clinicPartnerships.earlyFunnelStages`): whenever
you write a note on a deal in Inbound request / In conversation / Asked for info /
Demo scheduled, also update its native `description` and `hs_next_step` — max 2
sentences each, since these show directly on the HubSpot board cards. Skip this
for Contracting-onward deals, which Shaffy manages by hand.

**HubSpot Tasks for anything owed on our side — high bar** (per `hubspot.json →
tasks`, pipeline-wide, not limited to early-funnel): only when it's clearly
deal-related and either a question/request that's clearly still unanswered, or
something clearly important surfaced in email that needs flagging. Not a
catch-all — skip anything minor, ambiguous, or already covered by the stale-flow
follow-up (§7); a cluttered task list gets ignored. **Verify against the deal's
existing notes/activity first that it isn't already done.** For each that
clears the bar, create a `tasks` object linked to the deal — specific subject,
1-2 sentence body ending with a `Ref: hubspot:task:<dealId>:<slug>` dedup line,
`hs_task_type` TODO/EMAIL/CALL as fits, priority HIGH if >7d overdue or blocking
else MEDIUM, due today, owner Shaffy (162479602). Dedup on the `Ref:` line before
creating; mark an existing open task COMPLETED if a fresh note shows its item
got resolved.

**Organization backfill — "In conversation (Lemlist)" deals only** (per
`hubspot.json → orgLinking`): a separate automation, outside this workflow, now
auto-creates a HubSpot deal for every Lemlist reply — but doesn't reliably attach
a Company. For any deal in that exact stage you touch, check for an associated
Company; if missing, take the domain from the deal's contact email (skip personal
domains — flag those instead), search for an existing company by that domain and
associate it, or create one (`{name, domain}`) if none exists. This is the one
narrow exception to never creating records — company-only, never a contact or a
deal.

Never create a deal or a contact (W0/`new-deal-discovery.md` is disabled — list
genuine new opportunities in the digest for manual creation instead). Use only
the native Gmail MCP. Before flagging any deal stale or recommending a downgrade,
follow the §3a/§7 verification rule in `hubspot-sync.md`: do an undated
`from:`/`to:` search on that contact and confirm the actual last message's date —
never characterize a thread as quiet without checking it. Config: `hubspot.json`.
Finish with the deal-sync digest.
