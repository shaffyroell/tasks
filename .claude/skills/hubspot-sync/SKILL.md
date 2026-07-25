---
name: hubspot-sync
description: W1 — sync every open HubSpot deal from the full conversation (Granola, Slack, Lemlist, email) into HubSpot as dated notes. Advances stage only from "In conversation (Lemlist)" to "Asked for information"/"Demo scheduled" on a genuinely interested reply — never creates deals, never touches any later stage (Contracting onward is manual-only). HubSpot is the single source of truth. Use to keep the pipeline current each morning.
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
the note and let Shaffy move it himself. Never create a deal (W0/
`new-deal-discovery.md` is disabled — list genuine new opportunities in the digest
for manual creation instead). Use only the native Gmail MCP. Before flagging any
deal stale or recommending a downgrade, follow the §3a/§7 verification rule in
`hubspot-sync.md`: do an undated `from:`/`to:` search on that contact and confirm
the actual last message's date — never characterize a thread as quiet without
checking it. Config: `hubspot.json`. Finish with the deal-sync digest.
