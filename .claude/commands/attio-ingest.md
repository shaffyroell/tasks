---
description: W1 — ingest all client comms into Attio (dated notes, Description/Next step refresh, Company/Person linking, Attio Tasks, live/frozen stage moves)
---
Run W1, the Attio ingest defined in `attio-ingest.md` in this repo. Start with the
Step 0 preflight and open with the readiness line; build the working set —
including `Nurture / recontact` and advanced/frozen-stage deals, not just early
pipeline; gather Gmail, Slack, Granola, Fireflies, and calendar activity from
the last 24–48h. For every deal touched: write a dated comms note if not already
captured; refresh `deal_description` + `next_step` (no stage exclusion — every
touched deal gets this); find-or-create + link its Company/Person if missing
(the one exception to never creating records — deal creation stays W0-only);
create an Attio Task for anything clearly owed on our side (high bar, deduped on
its `Ref:` line). Move stage per the live/frozen split: `1.  Reply`,
`2. Planning call`, `3. Demo planned`, and `Nurture / recontact` move per
evidence (including reactivating Nurture on genuine new activity, and
reclassifying a plain opt-out to `Lost - not interested`); `4. Scope agreed`
onward is frozen — never touched by W1 once a deal is there, Shaffy moves it by
hand. Honor `attioIngest.autoApply`. Then post the ingest digest. Use only the
native Gmail MCP. Config: `attio.json` (`attioIngest`, `stagePolicy`,
`fieldRefresh`, `orgLinking`, `tasks`) + `client.json`.
