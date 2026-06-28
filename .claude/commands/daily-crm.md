---
description: Daily CRM routine — run W1 (ingest → Attio) then W2 (To-Dos → Notion)
---
Run the two daily workflows in order, so W2 reads what W1 just wrote.

1. **W1 first, to completion** — the Attio ingest defined in `attio-ingest.md`:
   Step 0 preflight (open with the readiness line) → build the working set
   (active clients + deals with fresh activity) → gather Gmail, Slack, Granola,
   Fireflies, and calendar activity from the last 24–48h → write a dated comms
   note to each active deal/client that had activity and move its stage when the
   evidence warrants (§3 rules, §4 safety; honor `attioIngest.autoApply`) →
   post the ingest digest. Use only the native Gmail MCP.

2. **Then W2** — the per-client To-Dos defined in `daily-open-items.md`:
   read Attio first (latest stages + the newest `[attio-ingest …]` notes + open
   tasks) as the primary signal, then also read fresh Gmail, Slack, and meeting
   notes (Granola + Fireflies); reconcile existing per-client to-dos (mark Done
   what was handled, advance what moved, dedup on `Ref`); add genuinely new
   to-dos routed to each client's board → post the summary.

Stop and report if W1's preflight shows a critical source down. Config comes from
`attio.json` (`attioIngest` block) + `client.json`.
