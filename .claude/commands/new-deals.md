---
description: "W0 — RETIRED 2026-08-17. Lemlist's own automation now creates the HubSpot deal directly on every reply, so this workflow no longer runs. Config: hubspot.json → newDealDiscovery.enabled:false."
---

> **RETIRED 2026-08-17 — supersedes the 2026-08-11 re-enable note below.**
> Lemlist's automation now creates the HubSpot deal the moment a lead replies,
> every time — not an occasional gap for this workflow to catch. `/daily-crm` no
> longer calls W0. **Do not run this command.** If invoked anyway: do nothing but
> report that deal creation is retired and point to `daily-crm.md` (step 1) for
> the current behavior (match the reply to its existing deal, never create one).
>
> ~~**Re-enabled 2026-08-11.** Disabled 2026-07-25 – 2026-08-11. Deals are created
> directly again — no more "list for manual add." The one change from the old
> (pre-2026-07-25) behavior: a new deal only ever lands in **In conversation
> (Lemlist)** or **Asked for information**, per `stageAssignRule` below — never
> further along the pipeline on autopilot.~~

See `new-deal-discovery.md` for the full historical record of what this workflow
used to do — kept for reference only, none of it should run. Config:
`hubspot.json` (`newDealDiscovery.enabled:false`).
