---
name: new-deals
description: W0 — discover new opportunities in inbound mail and Lemlist replies that are not yet in HubSpot, and create the deal + link company + link contact + add a thread-summary note. Use to capture fresh pipeline before the deal-sync run.
---

Run W0, the new-deal discovery defined in `new-deal-discovery.md` in this repo.
Preflight (HubSpot + Gmail + Lemlist); pull yesterday's inbound (Gmail `in:inbox`,
native MCP only, default 2-day lookback) and fresh Lemlist replies
(`get_inbox_conversations`, positive `aiLeadInterest`); keep only genuine new
external opportunities that are NOT already in HubSpot (dedup hard against existing
deals/contacts/companies; exclude SaaS/automated, internal, vendors-pitching-us,
recruiting applicants, personal). For each kept opportunity: find-or-create the
company (by domain) and contact (by email) and associate both, create the deal in
`newDealDiscovery.defaultPipeline` at the stage implied by intent, owned by
`defaultOwnerId`, then add a `[new-deal …]` note summarizing the thread. List
low-confidence intros for review instead of creating. Finish with the discovery
digest. Config: `hubspot.json` (`newDealDiscovery`).
