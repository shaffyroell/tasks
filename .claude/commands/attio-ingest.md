---
description: W1 — ingest all client comms into Attio (dated notes + stage moves)
---
Run W1, the Attio ingest defined in `attio-ingest.md` in this repo. Start with the
Step 0 preflight and open with the readiness line; build the working set (active
clients + deals with fresh activity); gather Gmail, Slack, Granola, Fireflies, and
calendar activity from the last 24–48h; write a dated comms note to each active
deal/client that had activity and move its stage when the evidence warrants (per
§3 rules and §4 safety; honor `attioIngest.autoApply`); then post the ingest
digest. Use only the native Gmail MCP. Config: `attio.json` (`attioIngest`) +
`client.json`.
