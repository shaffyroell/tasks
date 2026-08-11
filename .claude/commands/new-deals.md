---
description: "W0 — discover new opportunities in mail + Lemlist replies and create the deal (re-enabled 2026-08-11). Config: hubspot.json → newDealDiscovery.enabled:true."
---

> **Re-enabled 2026-08-11.** Disabled 2026-07-25 – 2026-08-11. Deals are created
> directly again — no more "list for manual add." The one change from the old
> (pre-2026-07-25) behavior: a new deal only ever lands in **In conversation
> (Lemlist)** or **Asked for information**, per `stageAssignRule` below — never
> further along the pipeline on autopilot.

Run W0, the new-deal discovery defined in `new-deal-discovery.md` in this repo.
Preflight (HubSpot + Gmail + Lemlist); pull yesterday's inbound (Gmail `in:inbox`,
native MCP only, default 2-day lookback) and fresh Lemlist replies
(`get_inbox_conversations`, positive `aiLeadInterest`); keep only genuine new
external opportunities that are NOT already in HubSpot (dedup hard against existing
deals/contacts/companies; exclude SaaS/automated, internal, vendors-pitching-us,
recruiting applicants, personal). For each kept opportunity: find-or-create the
company (by domain) and contact (by email) and associate both, create the deal in
`newDealDiscovery.defaultPipeline` at the stage from
`newDealDiscovery.stageAssignRule` — **Asked for information** if the reply asks a
question/requests info or pricing, else **In conversation (Lemlist)** (the
default; never auto-assign further) — owned by `defaultOwnerId`, then add a
`[new-deal …]` note summarizing the thread. List low-confidence intros for review
instead of creating. Finish with the discovery digest. Config: `hubspot.json`
(`newDealDiscovery`).
