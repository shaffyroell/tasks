---
name: daily-crm
description: Daily SwimScore sweep. Checks Lemlist for new/updated conversations, creates a HubSpot deal for genuine new opportunities (landing at In conversation (Lemlist) or Asked for information, never further on autopilot — re-enabled 2026-08-11), enriches every deal Lemlist's automation dropped at "Reply (to-be-enriched)" (context note, B2B_Type category, Orders_PM volume, then classifies into the right stage — added 2026-08-15), reads ALL of Shaffy's emails, sweeps the Slack deal channels, reads Granola call notes, logs HubSpot deal notes on any development (advancing stage only from "In conversation (Lemlist)" to "Asked for information"/"Demo scheduled" on a genuinely interested reply; refreshes Description/Next step — now a short dated log — on early-funnel deals; creates a HubSpot Task on the deal only for clearly deal-related items we clearly still owe — high bar, checked against existing notes first; backfills a missing Company on any "Reply (to-be-enriched)" or "In conversation (Lemlist)" deal it touches, since a separate automation now auto-creates those deals without one), flags deals with no contact in 2-3 weeks (adding a Notion To-Do with a follow-up message drafted from HubSpot context), and updates the SwimScore Notion board across all pillars. Run daily.
---

Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Open with a per-source preflight (HubSpot, Gmail, Slack, Granola,
Lemlist, Shopify, Notion); work the steps in order.

> **2026-08-11 — deal creation re-enabled:** Shaffy asked for new HubSpot deals
> back (had been off 2026-07-25 – 2026-08-11), with one change from the old
> behavior — a new deal never auto-lands past **In conversation (Lemlist)** or
> **Asked for information** (`hubspot.json → newDealDiscovery.stageAssignRule`),
> never further along the pipeline on autopilot. Stage moves on *existing* deals
> are still limited to the one rule in step 7 (plus the enrichment classification
> in step 2, for deals still sitting at "Reply (to-be-enriched)").

1. **Lemlist — new & updated.** Full team inbox (`teamConversations`, paginated;
   `get_inbox_conversation` for the thread + `aiLeadInterest`). New genuine B2B
   interest with no deal → **create the deal** (W0, `new-deal-discovery.md`):
   find-or-create the company + contact, land it at **Asked for information** if
   the reply asks a question/requests info or pricing, else **In conversation
   (Lemlist)** (the default), set **`amount` to `2500`** (`hubspot.json →
   defaultACV`) so it counts toward the weighted-pipeline total, and add a
   `[new-deal]` note summarizing the thread.
   Thin/low-confidence intros still just get listed in the digest for review.
   Existing deal with new content → update (step 6). Negative replies are not deals.

2. **Enrich "Reply (to-be-enriched)" deals** (`hubspot.json → ingest.enrichment`,
   added 2026-08-15): Lemlist's auto-create automation now lands every fresh
   reply-triggered deal at this stage instead of straight into **In conversation
   (Lemlist)**. For each deal sitting there:

   a. **Screen for a decline first** (`ingest.enrichment.declineHandling`,
      added 2026-08-15, this stage only): read the reply for unambiguous
      not-interested language ("don't contact me," "not relevant,"
      "unsubscribe," "no thank you," etc.).
      - **Clearly declined** → write the `[new-deal]` note explaining why, then
        move straight to **Closed Lost** — skip the rest of this step. This is
        the one place in the whole workflow the sweep sets a close state on its
        own, scoped tightly to this pre-human-review intake stage.
      - **Genuinely ambiguous** (reads negative-ish but isn't clean — deflects
        to a colleague, vague brush-off) → add a `[flag-uncertain]` note saying
        it's probably not relevant and why, **leave the stage as-is** for
        Shaffy to move by hand, and skip the rest of this step. List these
        separately in the digest.
      - **Not a decline** (positive, neutral, or a real question) → continue below.

   b. Read the Lemlist thread and write a `[new-deal]` context note (same as
      step 1); set **B2B_Type** (`b2b_type`) from what the clinic/practice
      actually is — IVF clinic, Acupuncture Fertility, TRT and men's health,
      Egg-freezing, Fertility guidance, Urologist, OB/GYN, Family Doctor; set
      **Orders_PM** (`orders_pm`) to the closest volume bucket if the thread
      mentions one, else default to **`1-5`** — never leave it blank; set
      **`amount` to `2500`** (`hubspot.json → defaultACV`) if not already set;
      then classify into the right stage with the same three-way call as step
      7's stage-advance rule (default → In conversation (Lemlist); asks a
      question/wants info or pricing → Asked for information; agrees to/confirms
      a call → Demo scheduled). Verify/backfill the Company here too
      (`orgLinking` now covers both this stage and In conversation (Lemlist)).

3. **Shopify — inbound contact-form messages FIRST** (before the email threads):
   read **only** the website `"New customer message"` contact-form submissions
   (arrive as emails to `info@myswimscore.com`). Keep only B2B clinic/partner intent
   → match to a deal or hand to W0. **Do NOT read Shopify orders or customers** —
   B2C, out of scope.

4. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` = source of
   truth after Lemlist): calls held, proposals, pricing, scheduling, commitments →
   attach to the deal. **Also read threads where Shaffy is only CC'd by a teammate**
   (`cc:shaffy@myswimscore.com` + threads from team senders: info@myswimscore.com,
   elara.k@maleswimscore.com, stewart.hill@checkswimscore.com, syb@myswimscore.com,
   other *swimscore* domains) — these often carry pipeline updates. A dated
   `newer_than:Nd` sweep can miss a thread's actual latest message — never report
   "no activity" from an impression; state the literal last-message date/sender.

5. **Slack — sweep the deal channels** in `hubspot.json.slackChannels` (outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + others). Read threads.

6. **Calls — read recent Granola notes** for decisions, pain points, next steps.

7. **Update HubSpot per client — only on a development.** Check existing notes
   first (no duplicates); add/refresh a `[hubspot-ingest]` note (Background →
   timestamped Timeline → Latest → Next step). **Stage moves are narrow** — only
   ever move a deal off **In conversation (Lemlist)** to **Asked for information**
   or **Demo scheduled**, and only on a Lemlist reply showing real interest
   (`hubspot.json → ingest.stageAdvanceRule`). Every deal already past that first
   stage keeps its current stage regardless of what happened — note it, don't move
   it. Honor autoApply; never touch a close state.

   **On early-funnel deals only** (Reply (to-be-enriched) / Inbound request / In
   conversation / Asked for info / Demo scheduled), also refresh the native
   `description` and `hs_next_step` fields to match — **description stays max 2
   sentences; `hs_next_step` is a short dated log** (`hubspot.json →
   ingest.fieldRefresh.nextStepFormat`, added 2026-08-15): prepend a new line
   `M/D: <what happened>. Next step: <the action>` (newest first, no year), keep
   at most the 4 most recent lines, replace same-day's line instead of stacking a
   second one. Both render on the HubSpot board cards. Skip this for
   Contracting-onward deals.

   **Create a HubSpot Task linked to the deal only for clearly deal-related items
   where we clearly owe a reply, or something clearly important surfaced in
   email — high bar** (`hubspot.json → tasks`). Not a catch-all — skip minor or
   ambiguous items and anything the stale-follow-up flow (step 8) already covers;
   a cluttered task list gets ignored. **Verify against the deal's existing
   notes/activity first that it isn't already done.** **Before creating, always
   search the deal's existing open tasks for a `Ref:` match and skip if found —
   never create a duplicate task for the same item.** Complete an existing task
   if a fresh note shows its item got resolved.

   **For any deal at "Reply (to-be-enriched)" or "In conversation (Lemlist)" you
   touch, verify it has a linked Company** (`hubspot.json → orgLinking`) — a
   separate automation now auto-creates these deals and doesn't reliably attach
   one. If missing, match or create a company by the contact's email domain and
   associate it — the one narrow exception to never creating records
   (company-only, never a deal or contact).

   **Touch-tracking fields — update on every deal with a new development**
   (`hubspot.json → touchTracking`, added 2026-08-16): keep `last_touch_date`,
   `last_touch_direction` (Inbound/Outbound), `Last_Message` (last literal
   message either side sent, prefixed `Client:`/`SwimScore:`), `Initial_reply_lead`
   (the lead's first-ever reply, set once), `Initial_reply_SS` (SwimScore's reply
   *to* that first reply — **leave empty if we haven't replied yet**, this is a
   deliberate follow-up-owed flag), `time_to_first_reply_hrs` (our response
   latency: their first reply → our reply to it, NOT their reaction time to our
   cold email — leave blank while `Initial_reply_SS` is empty), and
   `Lemlist_campaign_reply` current. `reply_channel` (email/linkedin/call) is the
   *acquisition* channel and is set once at deal creation, never overwritten by a
   later touch on a different channel. Determine the true last touch by checking
   **both** Lemlist (`get_inbox_conversation`, full thread) and Gmail — search
   **domain-wide** (`from:@theirdomain.com OR to:@theirdomain.com`), not just the
   one contact's address, since other people at the same clinic often correspond
   too and a single-address search misses them (fall back to a single-address
   search only on a personal domain like gmail.com, where domain-wide would pull
   in unrelated people). Not routed through Shaffy's inbox only, since a teammate
   (info@, elara.k@, stewart.hill@, syb@) emailing the lead directly is a real
   SwimScore-side touch whether or not Shaffy is cc'd. Always also check Gmail for
   a Calendly booking/acceptance notification — a booking can be the true last
   touch even with no new Lemlist reply. Lemlist sometimes mislabels SwimScore's
   own reply-in-thread as an inbound "emailsReplied" — read the actual
   sender/content, don't trust the activity type blindly.

8. **Stale check → flag + drafted follow-up** (`hubspot.json.staleFollowUp`):
   **before flagging (or re-affirming) any deal as stale, verify directly** — an
   undated `from:`/`to:` search on that contact's email, reading the actual last
   message's date. Never carry forward a prior run's stale flag unchecked. For
   each open deal confirmed quiet >14d (high >21d) where a nudge is warranted → add
   a To-Do on the SwimScore Notion board under **New sales** (Owner Shaffy, Link the
   deal, dedup `Ref hubspot:stale:<dealId>`) with a short, specific follow-up
   message **drafted from the deal's HubSpot notes** (only if a real message fits).

9. **Update the SwimScore Notion board from all channels** (W2,
   `daily-open-items.md`): reconcile internal to-dos across the seven pillars
   (incl. Marketing) from business-strategy, product/portal, legal, finance,
   support — check what each is for, mark Done / advance, dedup on `Ref`, add new
   in house style (`STYLE.md`). Also check goal coverage against the **Key Goals**
   database (see `SWIMSCORE_NOTION.md`) — a goal with no linked to-do, or only
   execution items and nothing measuring progress, needs a new to-do.

End with a CEO digest: **new deals created** (deal — contact — company — stage —
why), **enriched deals** (deal — B2B_Type — Orders_PM — stage it landed at),
**declined deals moved to Closed Lost** (deal — why) and **flagged for manual
review** (deal — why, still at Reply (to-be-enriched)) from step 2a, deals
updated (notes + the narrow stage moves only + which early-funnel deals had
Description/Next step refreshed), low-confidence opportunities found but NOT
created (for manual review), HubSpot Tasks created/completed (deal — subject —
why), organizations linked/created (deal — company — domain), stale deals flagged
(with/without drafted message), **deals with `Initial_reply_SS` empty** (a lead
replied and nobody from SwimScore has answered yet — this is a same-day
follow-up list, distinct from the 14d+ stale check), and Notion to-dos
added/advanced/closed per pillar. Surface the top risks + decisions.

10. **Post the recap to `#daily-recap`**, in order:
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
