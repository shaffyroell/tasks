---
description: Daily CRM routine — W0 (new deals) → W1 (ingest → Attio) → W2 (To-Dos → Notion)
---
Run the three daily workflows in order, so each reads what the previous wrote.

0. **W0 first** — new-deal discovery in `new-deal-discovery.md`: preflight → pull
   yesterday's inbound (Gmail, native MCP) → keep only genuine NEW opportunities
   not already in Attio (dedup hard; exclude SaaS/internal/vendors/recruiting/
   personal) → for each, create the deal + link company + link person + add a
   `[new-deal …]` thread-summary note (low-confidence intros listed for review,
   not created) → discovery digest.

1. **Then W1** — Attio ingest in `attio-ingest.md`: preflight → working set
   (active clients + deals with fresh activity, including anything W0 just
   created) → gather Gmail/Slack/Granola/Fireflies/calendar → write a dated comms
   note to each active deal/client with activity and move its stage when evidence
   warrants (honor `attioIngest.autoApply`) → ingest digest. Native Gmail only.

2. **Then W2** — per-client To-Dos in `daily-open-items.md`: read Attio first
   (latest stages + the newest `[attio-ingest …]`/`[new-deal …]` notes + open
   tasks), then fresh Gmail/Slack/Granola/Fireflies; reconcile existing to-dos
   (mark Done / advance, dedup on `Ref`), add new ones routed per client → summary.

Stop and report if W0 or W1 preflight shows a critical source down. Config:
`attio.json` (`newDealDiscovery` + `attioIngest`) + `client.json`.
