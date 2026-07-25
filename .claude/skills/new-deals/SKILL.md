---
name: new-deals
description: "DISABLED 2026-07-25 at Shaffy's request — the daily sweep no longer creates HubSpot deals. W0 used to discover new opportunities in inbound mail and Lemlist replies and create the deal; it's now off (hubspot.json → newDealDiscovery.enabled:false). Genuine new opportunities get listed in the W1 digest for manual creation instead. Do not invoke this to create deals."
---

> **⚠️ Disabled.** Do not create a deal if this skill is invoked. Read
> `new-deal-discovery.md`'s ICP criteria (§2) to identify genuine new
> opportunities from the same inbound sources, and list them (contact, company,
> why they look genuine) for Shaffy to add manually instead.

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
