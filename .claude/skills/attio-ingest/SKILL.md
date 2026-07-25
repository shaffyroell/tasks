---
name: attio-ingest
description: W1 — ingest all client communication (Gmail, Slack, Granola, Fireflies, calendar) into Attio as dated notes, keep every touched deal's Description/Next step current and its Company/Person linked (find-or-create), create Attio Tasks for what's owed on our side, and move deal stages per a live/frozen split (early pipeline + Nurture move freely, Scope agreed onward is manual-only). Use to keep the CRM current as the source of truth.
---

Run W1, the Attio ingest defined in `attio-ingest.md` in this repo. Start with the
Step 0 preflight and open with the readiness line; build the working set —
**including deals in `Nurture / recontact` and advanced/frozen stages, not just
early pipeline** — and gather Gmail, Slack, Granola, Fireflies, and calendar
activity from the last 24–48h. For every deal with new activity: write a dated
comms note if it isn't already captured; refresh its native `deal_description` +
`next_step` fields (§4, no stage exclusion — applies to every deal touched);
find-or-create and link its Company + Person if missing (§5, `orgLinking` — the
one exception to never creating records, deal creation stays W0-only); create an
Attio Task for anything clearly owed on our side (§6, high bar, deduped on its
`Ref:` line, completed once resolved, applies regardless of stage). Move stage
per `attio.json → stagePolicy` (§7): **live** stages (`1.  Reply`,
`2. Planning call`, `3. Demo planned`, `Nurture / recontact`) move per evidence
— including reactivating a Nurture deal on genuine new activity, and
reclassifying a plain opt-out to `Lost - not interested` as hygiene — but once a
deal reaches `4. Scope agreed` or later (**frozen**: Scope agreed, Proposal
sent, Active client, Completed Client, invoicing), its stage is never touched
again by W1, Shaffy moves it by hand. Honor `attioIngest.autoApply`. Then post
the ingest digest. Use only the native Gmail MCP. Config: `attio.json`
(`attioIngest`, `stagePolicy`, `fieldRefresh`, `orgLinking`, `tasks`) +
`client.json`.
