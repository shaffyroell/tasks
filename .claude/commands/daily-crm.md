---
description: Daily SwimScore CEO sweep — Lemlist + email + Slack + calls → update HubSpot deals, flag stale deals with a drafted follow-up, and update the SwimScore Notion board
---
Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Work the steps in order; open with a per-source preflight line
(HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion).

1. **Lemlist — new & updated conversations.** Pull the full team inbox
   (`get_inbox_conversations listId=teamConversations`, paginated; read threads via
   `get_inbox_conversation`, note `aiLeadInterest`). New genuine B2B interest with no
   deal → create it (W0 `new-deal-discovery.md`: deal named after the clinic + linked
   contact + company + domain). Existing deal with new content → update it (step 5).
   Negative replies are not deals.

2. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` is the source
   of truth after Lemlist). For every serious conversation (calls held, proposals,
   pricing, scheduling, commitments), find its deal and capture what moved.

2b. **Shopify — check for B2B inbound** from the website (both surfaces): the
   `"New customer message"` contact-form emails to `info@myswimscore.com` in Gmail,
   AND recent Shopify customers/orders (`get-shop-info`, `list-customers`,
   `list-orders`) for clinic/wholesale/partner signal. Keep only B2B intent → match
   to a deal or hand to W0. Ignore individual B2C patient orders.

3. **Slack — sweep the deal channels** in `hubspot.json.slackChannels`: outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + any other account channel. Read
   threads. Pull anything that changes a deal or is an internal action item.

4. **Calls — read all recent Granola notes** for decisions, pain points, next steps.

5. **Update HubSpot per client — only on a development.** Check the deal's existing
   notes first (no duplicates); if something moved, add/refresh a `[hubspot-ingest]`
   note (Background → timestamped Timeline → Latest → Next step) and advance the stage
   on clear evidence. Honor autoApply; never auto-close Won/Lost (list for approval);
   never silently demote an active deal.

6. **Stale check → flag + drafted follow-up** (per `hubspot.json.staleFollowUp`). For
   each open deal, compute last contact date across email/Lemlist/Slack/calls. If
   >14d (high priority >21d) and a nudge is warranted (alive, not awaiting a booked
   call): add a To-Do to the SwimScore Notion board under **New sales**
   (`Follow up with <clinic> — quiet <N>d`), Owner Shaffy, Link the deal, dedup on
   `Ref hubspot:stale:<dealId>`, and **draft a short, specific follow-up message from
   the deal's HubSpot context** into the to-do (only if a real message fits).

7. **Update the SwimScore Notion board from all channels** (W2 `daily-open-items.md`):
   reconcile internal to-dos across the six pillars (Onboarding, New sales, Research,
   Clinic & patient portal, Support, Finance) from business-strategy, product/portal,
   legal, finance, and support signal — check what each item is for, mark Done /
   advance, dedup on `Ref`, add new ones in house style per `STYLE.md`.

End with a digest: deals updated (notes + stage moves, any Closed awaiting approval),
new deals created, **stale deals flagged (with/without drafted message)**, and Notion
to-dos added/advanced/closed per pillar. Call out the top risks and decisions for the CEO.
