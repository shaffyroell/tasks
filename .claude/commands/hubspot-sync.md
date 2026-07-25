---
description: W1 — sync every HubSpot deal from the full conversation (Granola, Slack, Lemlist, email) — dated notes, plus one narrow stage move (In conversation → Asked for info/Demo scheduled)
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
narrow**: the only automated `dealstage` change is moving a deal off "In
conversation (Lemlist)" to "Asked for information" or "Demo scheduled" when a
Lemlist reply shows real interest (`hubspot.json → ingest.stageAdvanceRule`) —
every other deal keeps its current stage, logged in the note but not moved. Never
create a deal (W0 is disabled). Use only the native Gmail MCP. Config:
`hubspot.json`. Finish with the deal-sync digest.
