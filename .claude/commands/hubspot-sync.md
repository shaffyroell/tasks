---
description: W1 — sync every HubSpot deal from the full conversation (Granola, Slack, Lemlist, email) — dated notes + stage moves
---
Run W1, the daily HubSpot deal sync defined in `hubspot-sync.md` in this repo.
Start with the Step 0 preflight across all four sources (HubSpot, Gmail, Slack,
Granola, Lemlist) and open with the readiness line. Load **every open deal** from
HubSpot (exclude Closed Won/Lost), and for each deal gather the full recent
conversation across **Granola, Slack, Lemlist, email, and B2B Shopify website
inquiries** by matching the deal's contact emails / company domain (B2C patient
orders are out of scope — B2B only). Write one consolidated `[hubspot-ingest]` note to
each deal that had fresh activity **only if it isn't already there** (what
happened per channel, decisions/commitments, risks, next step), and move its
`dealstage` when the evidence warrants — honoring `ingest.autoApply`,
`allowStageAdvance`, and `allowStageClose` (never auto-close Won/Lost). Use only
the native Gmail MCP. Config: `hubspot.json`. Finish with the deal-sync digest.
