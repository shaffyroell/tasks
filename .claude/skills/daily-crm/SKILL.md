---
name: daily-crm
description: Daily SwimScore sweep. Checks Lemlist for new/updated conversations, reads ALL of Shaffy's emails, sweeps the Slack deal channels, reads Granola call notes, logs HubSpot deal notes on any development (advancing stage only from "In conversation (Lemlist)" to "Asked for information"/"Demo scheduled" on a genuinely interested reply — never creates deals, never touches later stages; refreshes Description/Next step on early-funnel deals; creates a HubSpot Task on the deal only for clearly deal-related items we clearly still owe — high bar, checked against existing notes first; backfills a missing Company on any "In conversation (Lemlist)" deal it touches, since a separate automation now auto-creates those deals without one), flags deals with no contact in 2-3 weeks (adding a Notion To-Do with a follow-up message drafted from HubSpot context), and updates the SwimScore Notion board across all pillars. Run daily.
---

Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Open with a per-source preflight (HubSpot, Gmail, Slack, Granola,
Lemlist, Shopify, Notion); work the steps in order.

> **2026-07-25 — scope narrowed at Shaffy's request:** no new HubSpot deals get
> created, and stage moves are limited to one rule — see step 6.

1. **Lemlist — new & updated.** Full team inbox (`teamConversations`, paginated;
   `get_inbox_conversation` for the thread + `aiLeadInterest`). New genuine B2B
   interest with no deal → **don't create it** — list it in the digest for Shaffy
   to add manually (contact + company + why it looks genuine). Existing deal with new
   content → update (step 6). Negative replies are not deals.

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
   timestamped Timeline → Latest → Next step). **Stage moves are narrow** — only
   ever move a deal off **In conversation (Lemlist)** to **Asked for information**
   or **Demo scheduled**, and only on a Lemlist reply showing real interest
   (`hubspot.json → ingest.stageAdvanceRule`). Every deal already past that first
   stage keeps its current stage regardless of what happened — note it, don't move
   it. Honor autoApply; never touch a close state.

   **On early-funnel deals only** (Inbound request / In conversation / Asked for
   info / Demo scheduled), also refresh the native `description` and
   `hs_next_step` fields to match — **max 2 sentences each**, since they render on
   the HubSpot board cards. Skip this for Contracting-onward deals.

   **Create a HubSpot Task linked to the deal only for clearly deal-related items
   where we clearly owe a reply, or something clearly important surfaced in
   email — high bar** (`hubspot.json → tasks`). Not a catch-all — skip minor or
   ambiguous items and anything the stale-follow-up flow (step 7) already covers;
   a cluttered task list gets ignored. **Verify against the deal's existing
   notes/activity first that it isn't already done.** **Before creating, always
   search the deal's existing open tasks for a `Ref:` match and skip if found —
   never create a duplicate task for the same item.** Complete an existing task
   if a fresh note shows its item got resolved.

   **For any deal at "In conversation (Lemlist)" you touch, verify it has a
   linked Company** (`hubspot.json → orgLinking`) — a separate automation now
   auto-creates these deals and doesn't reliably attach one. If missing, match or
   create a company by the contact's email domain and associate it — the one
   narrow exception to never creating records (company-only, never a deal or
   contact).

7. **Stale check → flag + drafted follow-up** (`hubspot.json.staleFollowUp`):
   **before flagging (or re-affirming) any deal as stale, verify directly** — an
   undated `from:`/`to:` search on that contact's email, reading the actual last
   message's date. Never carry forward a prior run's stale flag unchecked. For
   each open deal confirmed quiet >14d (high >21d) where a nudge is warranted → add
   a To-Do on the SwimScore Notion board under **New sales** (Owner Shaffy, Link the
   deal, dedup `Ref hubspot:stale:<dealId>`) with a short, specific follow-up
   message **drafted from the deal's HubSpot notes** (only if a real message fits).

8. **Update the SwimScore Notion board from all channels** (W2,
   `daily-open-items.md`): reconcile internal to-dos across the seven pillars
   (incl. Marketing) from business-strategy, product/portal, legal, finance,
   support — check what each is for, mark Done / advance, dedup on `Ref`, add new
   in house style (`STYLE.md`). Also check goal coverage against the **Key Goals**
   database (see `SWIMSCORE_NOTION.md`) — a goal with no linked to-do, or only
   execution items and nothing measuring progress, needs a new to-do.

End with a CEO digest: deals updated (notes + the narrow stage moves only + which
early-funnel deals had Description/Next step refreshed), new opportunities found
but NOT created (for manual add), HubSpot Tasks created/completed (deal —
subject — why), organizations linked/created (deal — company — domain), stale
deals flagged (with/without drafted message), and Notion to-dos
added/advanced/closed per pillar. Surface the top risks + decisions.

9. **Post the recap to `#daily-recap`**, in order:
   a. **"Updates from yesterday"** — one factual line per source, scoped to the
   last calendar day: **Orders** (count from `#0-new-order-received`, per
   `hubspot.json.slackChannels.newOrders`); **Support questions** (count from
   `#hubspot-inbox-live-responses`, `hubspotInboxLive`); **Lemlist-replies**
   (count + how many are worth following up); **Pipeline** (the single most
   consequential deal development, verified directly — don't guess); **Finance**
   (anything time-sensitive, read the actual thread before summarizing). Skip a
   bullet if there's nothing real — don't pad it.
   b. **"Short-term goals"** by category + **"To dos (specific)"**, plus the
   Notion To-Dos and Key Goals links. Filter to-dos by business judgment, not by
   mechanically dumping every open/High-priority Notion item — only what moves the
   needle this week (pipeline decisions, live customer-facing issues, genuinely
   blocked items). Batch the rest of the stale deals instead of listing each one,
   and **suppress routine dev-team-execution items** (most of Clinic & patient
   portal / Support) — Dmytro/Harsh close those out themselves in their own
   channels; only surface one if it's blocked, needs Shaffy's decision, or is a
   live patient-facing issue right now.
