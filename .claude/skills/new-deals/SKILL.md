---
name: new-deals
description: W0 — discover new opportunities in inbound mail that are not yet in Attio, and create the deal + link company + link person + add a thread-summary note. Use to capture fresh pipeline before the ingest run.
---

Run W0, the new-deal discovery defined in `new-deal-discovery.md` in this repo.
Preflight (Attio + Gmail); pull yesterday's inbound (Gmail `in:inbox`, native MCP
only, default 2-day lookback); keep only genuine new external opportunities in
TechTower's services that are NOT already in Attio (dedup hard against existing
deals/people/companies; exclude SaaS/automated, internal, vendors-pitching-us,
recruiting applicants, personal/SwimScore). For each kept opportunity: create the
deal (stage from intent, owner from `newDealDiscovery.ownerId`), find-or-create
the company (by domain) and person (by email) and link both, then add a
`[new-deal …]` note summarizing the thread. List low-confidence intros for review
instead of creating. Finish with the discovery digest. Config: `attio.json`
(`newDealDiscovery`) + `client.json`.
