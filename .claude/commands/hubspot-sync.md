---
description: W1 — sync every HubSpot deal from the full conversation (Slack, Lemlist, email; Granola removed 2026-08-18) — dated notes, action-owed stage classification (Interested, send Information → Interested, send follow-up → Demo scheduled), Description/Next step + touch-tracking refresh on early-funnel deals, HubSpot Tasks (high bar, deal-related only) for anything clearly owed on our side, and Company backfill for deals Lemlist's automation creates without one. Deal/contact creation is never this workflow's job — Lemlist's own automation creates the deal directly on every reply (W0/new-deals retired 2026-08-17)
---
Run W1, the daily HubSpot deal sync defined in `hubspot-sync.md` in this repo.
Start with the Step 0 preflight across all four sources (HubSpot, Gmail, Slack,
Lemlist) and open with the readiness line. Load **every open deal** from
HubSpot (exclude Closed Lost), and for each deal gather the full recent
conversation across **Slack, Lemlist, email, and B2B Shopify website
inquiries** by matching the deal's contact emails / company domain (B2C patient
orders are out of scope — B2B only). Write one consolidated `[hubspot-ingest]` note to
each deal that had fresh activity **only if it isn't already there** (what
happened per channel, decisions/commitments, risks, next step).

**Stage moves are narrow, and classified by action owed, not by the reply's
content** (`hubspot.json → ingest.stageAdvanceRule`, rewritten 2026-08-17): only
touch a deal currently at **"Interested, send Information"** (`3744632514`) or
**"Interested, send follow-up"** (`3744632516`) — every other deal keeps its
current stage, logged in the note but not moved. The question is **"has SwimScore
actually replied to this lead with info/pricing yet?"** — no → Interested, send
Information (also the default landing stage for a fresh un-replied-to touch); yes,
and no meeting booked → Interested, send follow-up; a specific meeting time is
agreed by both sides → Demo scheduled. No interest → don't move it yourself (only
exception: the Reply-to-be-enriched decline screen, §1b) — flag Closed Lost or
Interested (not now) for Shaffy instead.

**On early-funnel deals only** (Inbound request / Interested, send Information /
Interested, send follow-up / Demo scheduled — `pipelines.clinicPartnerships.earlyFunnelStages`),
also refresh the native `description` and `hs_next_step` fields to match, **max 2
sentences each** (they render on the board cards — Contracting-onward deals are
managed by hand, don't touch those). **In the same edit**, backfill the
`touchTracking` fields (`last_touch_date`, `last_touch_direction`, `last_message`,
`initial_reply_lead`, `initial_reply_ss`, `reply_channel` — `hubspot.json →
touchTracking`) — these are not optional and not a separate pass; a live run on
2026-08-17 skipped them on several newly-created deals by treating them as
optional. Before finishing, spot-check with a `last_touch_date NOT_HAS_PROPERTY`
search across early-funnel deals — it should come back empty.

**Create a HubSpot `Task` linked to the deal only for clearly deal-related items —
high bar**: a question/request we clearly haven't answered yet, or something
clearly important flagged in email — never for minor or ambiguous stuff
(`hubspot.json → tasks`). Verify against the deal's existing notes/activity first
that it isn't already resolved, dedupe on a `Ref:` line in the task body, and
complete any existing task a fresh note shows was resolved.

**For any deal at "Interested, send Information" or "Interested, send follow-up"
you touch**, verify it has a linked Company — Lemlist's automation creates a deal
for every reply and doesn't reliably attach one — and create+link one by domain if
missing (`hubspot.json → orgLinking`). **Search for the existing company/contact
by domain/email first** — the automation frequently creates the contact (and
sometimes the company) ahead of the deal, so a blind create produces duplicates.
This is the one narrow exception to never creating records: company-only.

**Never create a deal or a contact.** Lemlist's automation creates the deal
directly on every reply now — this workflow does not create deals under any
circumstance (W0/`new-deal-discovery.md` is retired as of 2026-08-17). If a
genuinely new B2B opportunity has no matching deal, list it in the digest for
Shaffy to review rather than creating it or assuming "the automation hasn't caught
up yet."

Use only the native Gmail MCP. Config: `hubspot.json`. Finish with the deal-sync
digest.
