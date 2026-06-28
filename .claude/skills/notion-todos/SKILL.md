---
name: notion-todos
description: W2 — update internal SwimScore To-Dos in Notion from HubSpot (primary) plus fresh Gmail/Slack/Granola/Lemlist, reconciling existing items (check what it's for, mark Done / advance) and adding new ones in the house style. Use after the HubSpot deal sync.
---

Run W2, the internal To-Dos workflow defined in `daily-open-items.md` in this repo.
Read HubSpot first (latest deal stages + the newest `[hubspot-ingest …]` notes) as
the primary account signal, then also read fresh Gmail, Slack, Granola, Lemlist, and
B2B Shopify website inquiries for INTERNAL SwimScore items (product, team, hiring,
finance, roadmap, ops). Target
the SwimScore Notion tracker in `hubspot.json` → `internalTracker`. Reconcile the
existing items — **check if each already exists, read what it's for, and update it
to the latest status** (mark Done what was handled, advance what moved, dedup on
`Ref`) — then add genuinely new internal to-dos. **Write every to-do in the house
style — load and follow `STYLE.md`** (verb-first, concise, no arrows). If
`internalTracker.dataSourceId` is still a placeholder (Notion not connected), list
the items in the digest for manual handling instead of failing. Finish with the
summary.
