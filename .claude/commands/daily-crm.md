---
description: Daily CEO sweep — read all email/Slack/Granola/Lemlist, classify each item ACCOUNT vs INTERNAL, route accounts to HubSpot deals and internal items to SwimScore Notion
---
Think like the CEO of this business. HubSpot is the single source of truth for the
pipeline; SwimScore Notion holds internal execution. Work these steps in order.

1. **Read everything (last few days):** all Gmail threads (native MCP), all
   relevant Slack channels (client + internal), all Granola call notes, all fresh
   Lemlist replies (`get_inbox_conversations` → `get_inbox_conversation`, note
   `aiLeadInterest`), and **B2B inbound via the Shopify website** (clinic/wholesale
   inquiries — often arrive as email to info@myswimscore.com). B2C patient orders
   are out of scope — **B2B only**.

2. **Classify each meaningful item — ACCOUNT or INTERNAL.** ACCOUNT = an external
   prospect/client/partner (winning, running, or growing a deal). INTERNAL =
   SwimScore's own ops (product, team, hiring, finance, roadmap). If an item is
   both, log the account side to HubSpot and the internal task to Notion.

3. **ACCOUNT → HubSpot.** Resolve the item to a deal (contact email / company
   domain / name). **Deal exists?** Run W1 (`hubspot-sync.md`): check existing
   notes and add a consolidated `[hubspot-ingest]` note only if missing (what
   happened per channel, decisions/commitments, risks, next step); move
   `dealstage` when warranted (honor autoApply + never auto-close Won/Lost; never
   silently demote an active deal). **No deal yet?** Run W0
   (`new-deal-discovery.md`): create the deal + link the company + link the people
   + add a `[new-deal]` note.

4. **INTERNAL → SwimScore Notion.** Run W2 (`daily-open-items.md`) against
   `internalTracker` in `hubspot.json`: check if the item already exists, read
   what it's for, and update it to the latest status (advance / mark Done), or add
   it new in the house style per `STYLE.md`. If `internalTracker.dataSourceId` is
   still a placeholder (Notion not connected), list internal items in the digest
   for manual handling instead of failing.

End with a CEO digest: **Accounts** (deals reviewed, notes added/gaps closed,
stage moves + any awaiting approval, new deals + who was linked) and **Internal**
(Notion items updated/advanced/closed/added, or listed for manual handling). Call
out the top risks and decisions that need the CEO. Config: committed `hubspot.json`
(+ `clients.json` if agency routing is on).
