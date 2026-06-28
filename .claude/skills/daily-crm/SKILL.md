---
name: daily-crm
description: Daily CRM routine. Runs W0 (discover & create new deals in Attio) → W1 (ingest all client comms to Attio as dated notes + stage moves) → W2 (per-client to-dos in Notion). Use this as the scheduled daily run, or when asked to sync the CRM and to-dos for the day.
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
   (mark Done / advance, dedup on `Ref`), add new ones routed per client, **each
   written in the house style per `STYLE.md`** → summary.

Stop and report if W0 or W1 preflight shows a critical source down. Config:
`attio.json` (`newDealDiscovery` + `attioIngest`) + `clients.json`.
