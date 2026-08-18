---
name: daily-crm
description: Daily SwimScore sweep. Checks Lemlist for new/updated conversations, plus Front's Email Replies shared inbox as a supplementary reply source (added 2026-08-18, filtered for lemwarmup/auto-reply noise) — deals are created automatically by Lemlist's own automation on every reply, this sweep never creates one), enriches every deal Lemlist's automation dropped at "Reply (to-be-enriched)" (context note, B2B_Type category, Orders_PM volume, then classifies into the right stage — added 2026-08-15), reads ALL of Shaffy's emails, sweeps the Slack deal channels, reads Granola call notes, logs HubSpot deal notes on any development (advancing stage between "Interested, send Information" and "Interested, send follow-up"/"Demo scheduled" based on whether SwimScore has actually sent the lead info yet, not on what their reply said — relabeled 2026-08-17; refreshes Description/Next step — a short dated log — on early-funnel deals; backfills touch-tracking fields in the same edit, never a separate pass; creates a HubSpot Task on the deal only for clearly deal-related items we clearly still owe — high bar, checked against existing notes first; backfills a missing Company on any "Reply (to-be-enriched)" or "Interested, send Information" deal it touches, since Lemlist's automation doesn't reliably attach one), flags deals with no contact in 2-3 weeks (adding a Notion To-Do with a follow-up message drafted from HubSpot context), and updates the SwimScore Notion board across all pillars. Run daily.
---

Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Open with a per-source preflight (HubSpot, Gmail, Slack, Granola,
Lemlist, Front, Shopify, Notion); work the steps in order.

> **2026-08-17 — deal creation retired for good; stages relabeled to action-owed.**
> Supersedes the 2026-08-11 "deal creation re-enabled" note below in full: Lemlist's
> automation now creates the HubSpot deal directly on every reply (not just
> catching up eventually) — **this sweep never creates a deal, under any
> circumstance, from any source** (Lemlist, email, Shopify). `newDealDiscovery` in
> `hubspot.json` is `enabled:false`; step 1 below no longer creates anything, it
> only matches and enriches. If a genuinely new B2B opportunity truly has no
> matching deal, list it in the digest for Shaffy — don't create it, and don't
> assume "the automation hasn't caught up yet."
>
> HubSpot also relabeled the two early-funnel stages (same ids, new meaning):
> **"In conversation (Lemlist)" → "Interested, send Information"** (`3744632514`,
> we haven't sent the lead our info/pricing yet) and **"Asked for information" →
> "Interested, send follow-up"** (`3744632516`, we've sent it and they haven't
> booked a meeting). The classification test everywhere below is now **"has
> SwimScore actually replied to this lead with info/pricing yet?"** — not "did
> their message ask a question." No reply sent → Interested, send Information
> (also the default landing/classification for any fresh un-replied-to touch).
> Replied with info, no meeting booked → Interested, send follow-up. A specific
> meeting time agreed by both sides → Demo scheduled (Shaffy's "meeting" bucket,
> label unchanged). No interest → don't move it yourself outside step 2a's decline
> screen — flag Closed Lost or **Interested (not now)** (`4065138365`, e.g. a
> lead we can't service yet) for Shaffy.
>
> **Touch-tracking fields are not optional and not a separate pass — and apply to
> every touch, not just deal creation.** Set `last_touch_date`/
> `last_touch_direction`/`last_message`/`initial_reply_lead`/`initial_reply_ss`/
> `reply_channel` (`hubspot.json → touchTracking`) in the exact same edit as any
> note/stage/field-refresh, on every deal touched, enriched, OR simply
> re-evaluated — not only newly-created ones. Two related bugs found live on
> 2026-08-17: (1) these fields got skipped entirely on newly-created deals by
> treating them as a follow-up step; (2) `Initial_reply_lead`/`Initial_reply_SS`
> were left blank on pre-existing deals because they'd been filed as
> create-time-only — but Lemlist's automation creates every deal now, so nothing
> else ever sets those two. **Always re-derive all five fields from a fresh
> full-thread read (Lemlist + Gmail, both directions) — never trust a stored
> value just because nothing "new" happened today; the value may have been wrong
> since the deal was created.** See step 7 for the full procedure and the
> mandatory end-of-run spot-checks (three, not one).
>
> ~~**2026-08-11 — deal creation re-enabled:** Shaffy asked for new HubSpot deals
> back (had been off 2026-07-25 – 2026-08-11), with one change from the old
> behavior — a new deal never auto-lands past **In conversation (Lemlist)** or
> **Asked for information** (`hubspot.json → newDealDiscovery.stageAssignRule`),
> never further along the pipeline on autopilot.~~ *(retired 2026-08-17, see above)*

1. **Lemlist — new & updated.** Full team inbox (`teamConversations`, paginated;
   `get_inbox_conversation` for the thread + `aiLeadInterest`). **Every reply now
   already has a deal** — Lemlist's automation creates it directly. Match the lead
   email/domain to its HubSpot deal (search by contact email, and by company
   domain — the automation sometimes creates the contact/company ahead of the
   deal) and update it (step 7). **Never create a deal or a contact.** If genuine
   B2B interest truly has no matching deal anywhere, list it in the digest for
   Shaffy to review manually — do not create it yourself. Negative replies aren't
   worth listing.

1b. **Front — Email Replies inbox, supplementary (added 2026-08-18).** Sweep the
   shared **Email Replies** inbox (`inb_nh7fs`, ticket prefix `SU-####`) via
   `mcp__Front__search_conversations` (`filters.inboxId: inb_nh7fs`,
   `scope: all_inboxes`, `filters.after` bounded to the lookback window) — it
   aggregates replies across all of SwimScore's rotating cold-outreach sending
   mailboxes (myswimscore.com/withswimscore.com/swimscoreview.com/
   viewswimscore.com/malescore.com/goswimscore.com). This runs **in addition to**
   step 1's Lemlist check, not instead of it — the two frequently carry the SAME
   underlying reply (Front is just those sending mailboxes viewed through a
   shared inbox), so check the deal's existing notes/touchTracking before logging
   anything, to avoid a duplicate note for one physical message.

   **Filter noise before treating anything as a real reply**: subjects ending
   `- lemwarmup` are Lemlist's own email-warmup network (fake back-and-forth
   threads that build sender reputation, always auto-resolved) — never a real
   lead; anything matching an auto-reply/OOO pattern (`Automatic reply:`,
   `Thank you for your email`, `We will get back to you shortly`,
   `Thank you for contacted...`) isn't a real reply either.

   For every conversation that survives the filter: read it with
   `read_conversation`, match the contact email/domain to its HubSpot deal (same
   matching rule as step 1), and fold it into the same enrich/note/touch-tracking
   pass as any other reply. **Never create a deal or contact from Front** — list
   an unmatched one in the digest like any other source. Front's own ticket
   status (Open/Waiting/Resolved) and `<name> - In Contact` tags are Front-side
   handling state (who's working the thread inside Front), not HubSpot
   dealstage — informational only, never mapped onto dealstage.

2. **Enrich "Reply (to-be-enriched)" deals** (`hubspot.json → ingest.enrichment`,
   added 2026-08-15): Lemlist's automation lands every fresh reply-triggered deal
   at this stage instead of straight into **Interested, send Information**. For
   each deal sitting there:

   a. **Screen for a decline first** (`ingest.enrichment.declineHandling`,
      added 2026-08-15, this stage only): read the reply for unambiguous
      not-interested language ("don't contact me," "not relevant,"
      "unsubscribe," "no thank you," etc.).
      - **Clearly declined** → write a `[new-deal]` note explaining why, then
        move straight to **Closed Lost** — skip the rest of this step. This is
        the one place in the whole workflow the sweep sets a close state on its
        own, scoped tightly to this pre-human-review intake stage.
      - **Genuinely ambiguous** (reads negative-ish but isn't clean — deflects
        to a colleague, vague brush-off) → add a `[flag-uncertain]` note saying
        it's probably not relevant and why, **leave the stage as-is** for
        Shaffy to move by hand, and skip the rest of this step. List these
        separately in the digest.
      - **Not a decline** (positive, neutral, or a real question) → continue below.

   b. Read the Lemlist thread and write a `[new-deal]` context note (who/company/
      what they said, channel, date); set **B2B_Type** (`b2b_type`) from what the
      clinic/practice actually is — IVF clinic, Acupuncture Fertility, TRT and
      men's health, Egg-freezing, Fertility guidance, Urologist, OB/GYN, Family
      Doctor; set **Orders_PM** (`orders_pm`) to the closest volume bucket if the
      thread mentions one, else default to **`1-5`** — never leave it blank; set
      **`amount` to `2500`** (`hubspot.json → defaultACV`) if not already set;
      then classify into the right stage using the **info-owed rule** (no reply
      with info sent yet → Interested, send Information; SwimScore already
      replied with info/pricing → Interested, send follow-up; a meeting time is
      agreed → Demo scheduled). Verify/backfill the Company here too
      (`orgLinking` covers both this stage and Interested, send Information).
      **Backfill touch-tracking fields in this same pass** (step 7's rules).

3. **Shopify — inbound contact-form messages FIRST** (before the email threads):
   read **only** the website `"New customer message"` contact-form submissions
   (arrive as emails to `info@myswimscore.com`). Keep only B2B clinic/partner
   intent → match to an existing deal, or list in the digest if none exists. **Do
   NOT read Shopify orders or customers** — B2C, out of scope. **Never create a
   deal.**

4. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` = source of
   truth after Lemlist): calls held, proposals, pricing, scheduling, commitments →
   attach to the deal. **Also read threads where Shaffy is only CC'd by a teammate**
   (`cc:shaffy@myswimscore.com` + threads from team senders: info@myswimscore.com,
   elara.k@maleswimscore.com, stewart.hill@checkswimscore.com, syb@myswimscore.com,
   other *swimscore* domains) — these often carry pipeline updates. A dated
   `newer_than:Nd` sweep can miss a thread's actual latest message — never report
   "no activity" from an impression; state the literal last-message date/sender.

   **Also search the Sent folder directly** (`in:sent newer_than:Nd`, not just
   `in:inbox`) — `in:inbox` alone misses any reminder/follow-up Shaffy sends
   personally that hasn't gotten a reply yet, since a Sent-only message with no
   inbound reply never shows up under `in:inbox`. Match each Sent recipient to a
   deal and log it as a real outbound touch (note + `hubspot.json → touchTracking`:
   `last_touch_date`/`last_touch_direction: Outbound`/`last_message`) even with no
   new inbound reply — this is exactly the kind of touch that must prevent a false
   stale flag in step 8.

5. **Slack — sweep the deal channels** in `hubspot.json.slackChannels` (outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + others). Read threads.

6. **Calls — read recent Granola notes** for decisions, pain points, next steps.

7. **Update HubSpot per client — only on a development.** Check existing notes
   first (no duplicates); add/refresh a `[hubspot-ingest]` note (Background →
   timestamped Timeline → Latest → Next step). **Stage moves are narrow, two
   stages only, classified by action owed** (`hubspot.json →
   ingest.stageAdvanceRule`, rewritten 2026-08-17): only touch a deal currently at
   **Interested, send Information** or **Interested, send follow-up**. Ask **has
   SwimScore actually sent info/pricing yet** (check for an outbound reply after
   their inbound message — not what their message said): no → stays/moves to
   Interested, send Information; yes and no meeting booked → Interested, send
   follow-up; a specific meeting time agreed by both sides → Demo scheduled. Every
   deal already at Demo scheduled or later keeps its current stage regardless of
   what happened — note it, don't move it. No interest → flag Closed Lost or
   Interested (not now) for Shaffy rather than setting it yourself (exception:
   step 2a's decline screen). Honor autoApply; never touch a close state yourself
   outside that one exception.

   **On early-funnel deals only** (Reply (to-be-enriched) / Inbound request /
   Interested, send Information / Interested, send follow-up / Demo scheduled),
   also refresh the native `description` and `hs_next_step` fields to match —
   **description stays max 2 sentences; `hs_next_step` is a short dated log**
   (`hubspot.json → ingest.fieldRefresh.nextStepFormat`, added 2026-08-15): prepend
   a new line `M/D: <what happened>. Next step: <the action>` (newest first, no
   year), keep at most the 4 most recent lines, replace same-day's line instead of
   stacking a second one. Both render on the HubSpot board cards. Skip this for
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

   **For any deal at "Reply (to-be-enriched)" or "Interested, send Information"
   you touch, verify it has a linked Company** (`hubspot.json → orgLinking`) —
   Lemlist's automation doesn't reliably attach one. **Search for the existing
   company by domain first** — the automation frequently creates the
   contact/company ahead of the deal, so a blind create produces duplicates
   (confirmed live 2026-08-17: created 5 duplicate companies this way in one
   run). If none exists, match or create a company by the contact's email domain
   and associate it — the one narrow exception to never creating records
   (company-only, never a deal or contact).

   **Touch-tracking fields — check and correct on EVERY deal you touch for ANY
   reason (note, stage check, enrichment, field refresh), IN THE SAME EDIT as
   the note/stage/field-refresh above, not a separate pass, and not limited to
   deals with activity in today's lookback window** (`hubspot.json →
   touchTracking`, added 2026-08-16): keep `last_touch_date`,
   `last_touch_direction` (Inbound/Outbound), `Last_Message` (last literal
   message either side sent, prefixed `Client:`/`SwimScore:`), `Initial_reply_lead`
   (the lead's first-ever reply, **no prefix**), `Initial_reply_SS` (SwimScore's
   reply *to* that first reply, **no prefix** — **leave empty if we genuinely
   haven't replied yet**, this is a deliberate follow-up-owed flag), and
   `time_to_first_reply_hrs` (their first reply → our reply to it, NOT their
   reaction time to our cold email — leave blank while `Initial_reply_SS` is
   empty). `reply_channel` (email/linkedin/call) is the *acquisition* channel,
   set once at deal creation, never overwritten by a later touch on a different
   channel.

   **2026-08-17 gotcha, twice in one run — read before touching any deal.**
   "Update touch tracking" had been misread as "update the LATEST-touch fields
   only." `Initial_reply_lead`/`Initial_reply_SS` were then left untouched on
   every deal not created in the current run, because they'd been filed
   mentally as create-time-only. That was true back when this workflow created
   deals; now Lemlist's automation creates every deal, so **nothing else ever
   sets those two fields** — any deal touched without explicitly checking them
   stays permanently blank. A second bug in the same pass: `last_touch_date`/
   `last_touch_direction`/`last_message` were themselves stale on deals with NO
   new activity today, because whatever set them at deal creation had captured
   only the lead's inbound trigger and missed a same-day SwimScore reply that
   came later — and the sweep left them alone since nothing looked "new."
   **The fix: every single time you touch a deal, for any reason, read the
   FULL thread (Lemlist `get_inbox_conversation` + Gmail, both directions) and
   recompute all five fields from that read — never assume a stored value is
   correct just because today's trigger wasn't about that field.** If
   `Initial_reply_lead`/`Initial_reply_SS` are blank and a real exchange
   exists anywhere in the deal's history, backfill both now, regardless of
   whether that exchange happened today.

   Determine the true last touch by checking **both** Lemlist
   (`get_inbox_conversation`, full thread) and Gmail — search **domain-wide**
   (`from:@theirdomain.com OR to:@theirdomain.com`), not just the one contact's
   address, since other people at the same clinic often correspond too and a
   single-address search misses them (fall back to a single-address search
   only on a personal domain like gmail.com, where domain-wide would pull in
   unrelated people). Not routed through Shaffy's inbox only, since a teammate
   (info@, elara.k@, stewart.hill@, syb@) emailing the lead directly is a real
   SwimScore-side touch whether or not Shaffy is cc'd. Always also check Gmail
   for a Calendly booking/acceptance notification — a booking can be the true
   last touch even with no new Lemlist reply. Lemlist sometimes mislabels
   SwimScore's own reply-in-thread as an inbound "emailsReplied" — read the
   actual sender/content, don't trust the activity type blindly.

   **Format rule, violated on a few pre-existing deals in production:**
   `Initial_reply_lead`/`Initial_reply_SS` must be the literal reply text with
   **no** `Client:`/`SwimScore:` prefix — that prefix belongs exclusively on
   `Last_Message`; the property name already establishes the sender for the
   other two.

   **Not optional — before finishing any run, spot-check all three:**
   1. `last_touch_date NOT_HAS_PROPERTY` across early-funnel deals — should be empty.
   2. `initial_reply_ss NOT_HAS_PROPERTY` across early-funnel deals — should be
      empty except deals with a genuinely real reason (no thread exists yet, or
      SwimScore genuinely hasn't replied — which itself belongs in the digest's
      reply-owed list, not silently skipped).
   3. No `initial_reply_lead`/`initial_reply_ss` value starts with `Client:` or
      `SwimScore:`.

   **CRITICAL — never trust search_threads' inline messages as a complete thread**
   (`hubspot.json → touchTracking.sources._CRITICAL_threadTruncation_warning`):
   confirmed in production that `search_threads` silently truncates long
   back-and-forth threads (returned 4-5 of a real 16-message thread for
   Onto.Health, cutting off weeks of activity including the actual last touch,
   with zero indication of truncation). For any thread that looks like an
   ongoing exchange, call `get_thread` on that threadId directly — if it
   overflows to a saved file (common for long threads), read it with
   `jq '[.messages[] | {date, sender, toRecipients, ccRecipients, subject, snippet}] | sort_by(.date)'`
   to get the true last message. One extra call is cheap; a stale last-touch
   date reported as current is not.

8. **Stale check → flag + drafted follow-up** (`hubspot.json.staleFollowUp`):
   **before flagging (or re-affirming) any deal as stale, verify directly** — an
   undated `from:`/`to:` search on that contact's email, reading the actual last
   message's date. Never carry forward a prior run's stale flag unchecked. For
   each open deal confirmed quiet >14d (high >21d) where a nudge is warranted → add
   a To-Do on the SwimScore Notion board under **New sales** (Owner Shaffy, Link the
   deal, dedup `Ref hubspot:stale:<dealId>`) with a short, specific follow-up
   message **drafted from the deal's HubSpot notes** (only if a real message fits).
   **Before creating a new stale To-Do, check for an existing one with the same
   `Ref` and update it in place rather than adding a duplicate** — a live run on
   2026-08-16/17 found two rows for the same deal with different day-counts.

9. **Update the SwimScore Notion board from all channels** (W2,
   `daily-open-items.md`): reconcile internal to-dos across the seven pillars
   (incl. Marketing) from business-strategy, product/portal, legal, finance,
   support — check what each is for, mark Done / advance, dedup on `Ref`, add new
   in house style (`STYLE.md`). Also check goal coverage against the **Key Goals**
   database (see `SWIMSCORE_NOTION.md`) — a goal with no linked to-do, or only
   execution items and nothing measuring progress, needs a new to-do.

End with a CEO digest: **enriched deals** (deal — B2B_Type — Orders_PM — stage it
landed at), **declined deals moved to Closed Lost** (deal — why) and **flagged for
manual review** (deal — why, still at Reply (to-be-enriched)) from step 2a, deals
updated (notes + the info-owed stage moves only + which early-funnel deals had
Description/Next step refreshed), **new opportunities found but with no matching
deal** (for Shaffy to review — never created by this sweep), **HubSpot Tasks
created/completed** (deal — subject — why), organizations linked/created (deal —
company — domain), stale deals flagged (with/without drafted message), **deals
with `Initial_reply_SS` empty** (a lead replied and nobody from SwimScore has
answered yet — this is a same-day follow-up list, distinct from the 14d+ stale
check), and Notion to-dos added/advanced/closed per pillar. Surface the top risks
+ decisions.

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
