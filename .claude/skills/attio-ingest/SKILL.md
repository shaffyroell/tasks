---
name: attio-ingest
description: W1 — ingest all client communication (Gmail, Slack, Granola, Fireflies, calendar) into Attio as dated notes, keep every active deal's Description/Next step fields current, create Attio Tasks for what's owed on our side, and move deal stages when the evidence warrants. Use to keep the CRM current as the source of truth.
---

Run W1, the Attio ingest defined in `attio-ingest.md` in this repo. Start with the
Step 0 preflight and open with the readiness line; build the working set (active
clients + deals with fresh activity); gather Gmail, Slack, Granola, Fireflies, and
calendar activity from the last 24–48h; write a dated comms note to each active
deal/client that had activity; refresh the deal's native `deal_description` +
`next_step` fields (§4, `fieldRefresh.scopeStages`); create an Attio Task for
anything clearly owed on our side (§5, high bar, deduped on its `Ref:` line,
completed once resolved); move its stage when the evidence warrants, including
reclassifying a plain opt-out sitting in an active stage to `Lost - not
interested` (§6 rules and §7 safety; honor `attioIngest.autoApply`); then post
the ingest digest. Use only the native Gmail MCP. Config: `attio.json`
(`attioIngest`, `fieldRefresh`, `tasks`) + `client.json`.
