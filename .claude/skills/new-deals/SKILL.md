---
name: new-deals
description: "W0 — discover new opportunities in inbound mail and Lemlist replies and create the deal + link company + link person + add a thread-summary note (re-enabled 2026-08-11). New deals land only in In conversation (Lemlist) or Asked for information — never further along the pipeline on autopilot. Use to capture fresh pipeline before the ingest run."
---

> **Re-enabled 2026-08-11.** Disabled 2026-07-25 – 2026-08-11 (Shaffy: no
> auto-created deals). Deals are created directly again — no more "list for
> manual add." The one change from the old (pre-2026-07-25) behavior: a new deal
> only ever lands in **In conversation (Lemlist)** or **Asked for information**,
> per `hubspot.json → newDealDiscovery.stageAssignRule` — never further along the
> pipeline on autopilot; Shaffy moves a deal past that himself.

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
default) — owned by `defaultOwnerId`, then add a `[new-deal …]` note summarizing
the thread. List low-confidence intros for review instead of creating. Finish
with the discovery digest. Config: `hubspot.json` (`newDealDiscovery`).
