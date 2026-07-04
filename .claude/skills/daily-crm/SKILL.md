---
name: daily-crm
description: Daily SwimScore sweep. Checks Lemlist for new/updated conversations, reads ALL of Shaffy's emails, sweeps the Slack deal channels, reads Granola call notes, updates HubSpot deal notes + stages on any development, flags deals with no contact in 2-3 weeks (adding a Notion To-Do with a follow-up message drafted from HubSpot context), and updates the SwimScore Notion board across all pillars. Run daily.
---

Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Open with a per-source preflight (HubSpot, Gmail, Slack, Granola,
Lemlist, Shopify, Notion); work the steps in order.

1. **Lemlist — new & updated.** Full team inbox (`teamConversations`, paginated;
   `get_inbox_conversation` for the thread + `aiLeadInterest`). New genuine B2B
   interest with no deal → create it via W0 (`new-deal-discovery.md`): deal named
   after the clinic + linked contact + company + domain. Existing deal with new
   content → update (step 5). Negative replies are not deals.

2. **Shopify — inbound contact-form messages FIRST** (before the email threads):
   read **only** the website `"New customer message"` contact-form submissions
   (arrive as emails to `info@myswimscore.com`). Keep only B2B clinic/partner intent
   → match to a deal or hand to W0. **Do NOT read Shopify orders or customers** —
   B2C, out of scope.

3. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` = source of
   truth after Lemlist): calls held, proposals, pricing, scheduling, commitments →
   attach to the deal. **Also read threads where Shaffy is only CC'd by a teammate**
   (`cc:shaffy@myswimscore.com` + threads from team senders: info@myswimscore.com,
   elara.k@maleswimscore.com, stewart.hill@checkswimscore.com, syb@myswimscore.com,
   other *swimscore* domains) — these often carry pipeline updates. A dated
   `newer_than:Nd` sweep can miss a thread's actual latest message — never report
   "no activity" from an impression; state the literal last-message date/sender.

4. **Slack — sweep the deal channels** in `hubspot.json.slackChannels` (outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + others). Read threads.

5. **Calls — read recent Granola notes** for decisions, pain points, next steps.

6. **Update HubSpot per client — only on a development.** Check existing notes
   first (no duplicates); add/refresh a `[hubspot-ingest]` note (Background →
   timestamped Timeline → Latest → Next step) and advance the stage on clear
   evidence. Honor autoApply; never auto-close Won/Lost; never silently demote.

7. **Stale check → flag + drafted follow-up** (`hubspot.json.staleFollowUp`):
   **before flagging (or re-affirming) any deal as stale, verify directly** — an
   undated `from:`/`to:` search on that contact's email, reading the actual last
   message's date. Never carry forward a prior run's stale flag unchecked. For
   each open deal confirmed quiet >14d (high >21d) where a nudge is warranted → add
   a To-Do on the SwimScore Notion board under **New sales** (Owner Shaffy, Link the
   deal, dedup `Ref hubspot:stale:<dealId>`) with a short, specific follow-up
   message **drafted from the deal's HubSpot notes** (only if a real message fits).

8. **Update the SwimScore Notion board from all channels** (W2,
   `daily-open-items.md`): reconcile internal to-dos across the six pillars from
   business-strategy, product/portal, legal, finance, support — check what each is
   for, mark Done / advance, dedup on `Ref`, add new in house style (`STYLE.md`).

End with a CEO digest: deals updated (notes + stage moves, Closed awaiting
approval), new deals, stale deals flagged (with/without drafted message), and
Notion to-dos added/advanced/closed per pillar. Surface the top risks + decisions.

9. **Post the recap to `#daily-recap`** — *Short-term goals* by category +
   *To dos (specific)*, plus the Notion link. Filter to-dos by business judgment,
   not by mechanically dumping every open/High-priority Notion item — only what
   moves the needle this week (pipeline decisions, live customer-facing issues,
   genuinely blocked items). Batch the rest of the stale deals instead of listing
   each one, and **suppress routine dev-team-execution items** (most of Clinic &
   patient portal / Support) — Dmytro/Harsh close those out themselves in their own
   channels; only surface one if it's blocked, needs Shaffy's decision, or is a
   live patient-facing issue right now.
