---
description: Daily SwimScore CEO sweep — Lemlist + email + Slack + calls → match every reply to the deal Lemlist's own automation already created (this sweep never creates one), log HubSpot deal notes (+ info-owed stage move between "Interested, send Information"/"Interested, send follow-up"/"Demo scheduled" + Description/Next step + touch-tracking refresh on early-funnel deals + HubSpot Tasks for anything clearly owed on our side + Company backfill on Lemlist-created deals), flag stale deals with a drafted follow-up, and update the SwimScore Notion board
---
Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Work the steps in order; open with a per-source preflight line
(HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion).

> **2026-08-17 — deal creation retired for good; stages relabeled to action-owed.**
> Supersedes the 2026-08-11 note below in full: Lemlist's automation now creates
> the HubSpot deal directly on every reply (not just catching up eventually) —
> **this sweep never creates a deal, under any circumstance, from any source**
> (Lemlist, email, Shopify). `newDealDiscovery` in `hubspot.json` is
> `enabled:false`; step 1 below no longer creates anything, it only matches and
> enriches. If a genuinely new B2B opportunity truly has no matching deal, list it
> in the digest for Shaffy — don't create it, and don't assume "the automation
> hasn't caught up yet."
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
> ~~**2026-07-25 — scope narrowed at Shaffy's request:** this sweep no longer
> creates HubSpot deals, and no longer advances stages freely.~~ ~~**2026-07-25
> (2):** a separate automation now auto-creates a HubSpot deal for every Lemlist
> reply.~~ ~~**2026-08-11 — deal creation re-enabled:** Shaffy asked for new
> HubSpot deals back, with one change from the old behavior — a new deal never
> auto-lands past In conversation (Lemlist) or Asked for information, never
> further along the pipeline on autopilot.~~ *(all superseded 2026-08-17, see
> above — deal creation is retired again, permanently, and the stage names/logic
> changed)*

1. **Lemlist — new & updated.** Full team inbox (`teamConversations`, paginated;
   `get_inbox_conversation` for the thread + `aiLeadInterest`). **Every reply now
   already has a deal** — Lemlist's automation creates it directly. Match the lead
   email/domain to its HubSpot deal (search by contact email, and by company
   domain — the automation sometimes creates the contact/company ahead of the
   deal) and update it (step 7). **Never create a deal or a contact.** If genuine
   B2B interest truly has no matching deal anywhere, list it in the digest for
   Shaffy to review manually — do not create it yourself. Negative replies aren't
   worth listing.

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

4. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` is the source
   of truth after Lemlist). For every serious conversation (calls held, proposals,
   pricing, scheduling, commitments), find its deal and capture what moved. **Also
   read threads where Shaffy is only CC'd by a teammate** (`cc:shaffy@myswimscore.com`
   + threads from team senders: info@myswimscore.com, elara.k@maleswimscore.com,
   stewart.hill@checkswimscore.com, syb@myswimscore.com, other *swimscore* domains) —
   these often carry pipeline updates. A dated `newer_than:Nd` sweep can miss a
   thread's actual latest message — never report "no activity" from an impression;
   state the literal last-message date/sender for every deal touched.

   **Also search the Sent folder directly** (`in:sent newer_than:Nd`, not just
   `in:inbox`) — inbox-only misses any reminder/follow-up Shaffy sends personally
   that hasn't gotten a reply yet, since a Sent-only message never shows up under
   `in:inbox`. Match each Sent recipient to a deal and log it as a real outbound
   touch (note + touchTracking fields) even with no new inbound reply — this must
   prevent a false stale flag in step 8.

5. **Slack — sweep the deal channels** in `hubspot.json.slackChannels`: outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + any other account channel. Read
   threads. Pull anything that changes a deal or is an internal action item.

6. **Calls — read all recent Granola notes** for decisions, pain points, next steps.

7. **Update HubSpot per client — only on a development.** Check the deal's existing
   notes first (no duplicates); if something moved, add/refresh a `[hubspot-ingest]`
   note (Background → timestamped Timeline → Latest → Next step). **Stage moves are
   narrow, two stages only, classified by action owed** (`hubspot.json →
   ingest.stageAdvanceRule`, rewritten 2026-08-17): only touch a deal currently at
   **Interested, send Information** or **Interested, send follow-up**. Ask **has
   SwimScore actually sent info/pricing yet** (check for an outbound reply after
   their inbound message — not what their message said): no → stays/moves to
   Interested, send Information; yes and no meeting booked → Interested, send
   follow-up; a specific meeting time agreed by both sides → Demo scheduled. Every
   deal already at Demo scheduled or later keeps its current stage regardless of
   what happened — just log the note. No interest → flag Closed Lost or Interested
   (not now) for Shaffy rather than setting it yourself (exception: step 2a's
   decline screen). Honor autoApply; never touch a close state yourself outside
   that one exception.

   **On early-funnel deals only** (Reply (to-be-enriched) / Inbound request /
   Interested, send Information / Interested, send follow-up / Demo scheduled —
   `pipelines.clinicPartnerships.earlyFunnelStages`), also refresh the native
   `description` and `hs_next_step` fields to match — **max 2 sentences each**,
   since they render on the HubSpot board cards. Skip this for Contracting-onward
   deals; Shaffy manages those fields by hand.

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
   touchTracking`): `last_touch_date`, `last_touch_direction`, `last_message`
   (prefixed `Client:`/`SwimScore:`), `initial_reply_lead`, `initial_reply_ss`
   (**no prefix on these two** — leave `initial_reply_ss` empty only if we
   genuinely haven't replied yet, a deliberate follow-up-owed flag),
   `time_to_first_reply_hrs`, `reply_channel`.

   **2026-08-17 gotcha, twice in one run.** "Update touch tracking" had been
   misread as "update the latest-touch fields only" — `initial_reply_lead`/
   `initial_reply_ss` were left blank on every deal not created in the current
   run because they'd been filed as create-time-only. Lemlist's automation
   creates every deal now, so nothing else ever sets those two — any deal
   touched without explicitly checking them stays permanently blank. Separately,
   `last_touch_date`/`last_touch_direction`/`last_message` were themselves stale
   on deals with no new activity today, because whatever set them at creation
   captured only the lead's inbound trigger and missed a same-day SwimScore
   reply that came later. **Fix: every time you touch a deal, for any reason,
   read the FULL thread (Lemlist + Gmail, both directions) and recompute all
   five fields from that read — never assume a stored value is correct just
   because today's trigger wasn't about that field.**

   Determine the true last touch by checking **both** Lemlist and Gmail
   domain-wide, and always check for a Calendly booking/acceptance too.

   **Not optional — before finishing any run, spot-check all three:**
   `last_touch_date NOT_HAS_PROPERTY` and `initial_reply_ss NOT_HAS_PROPERTY`
   across early-funnel deals should both come back empty (except deals with a
   genuinely real reason — no thread exists, or a reply is genuinely still
   owed, which belongs in the digest); and no `initial_reply_lead`/
   `initial_reply_ss` value should start with `Client:` or `SwimScore:`.

   **Create a HubSpot Task linked to the deal only for clearly deal-related items
   where we clearly owe a reply, or something clearly important surfaced in
   email — high bar** (`hubspot.json → tasks`, pipeline-wide). Not a catch-all:
   skip minor or ambiguous items, and anything the stale-follow-up flow (step 8)
   already covers — a cluttered task list gets ignored. **Verify against the
   deal's existing notes/activity first that it isn't already done.** Dedupe on a
   `Ref:` line in the task body; complete an existing task if a fresh note shows
   its item got resolved.

8. **Stale check → flag + drafted follow-up** (per `hubspot.json.staleFollowUp`).
   **Before flagging or re-affirming any deal as stale, verify directly**: an undated
   `from:`/`to:` search on that contact's email, reading the actual last message's
   date — never carry forward a prior run's stale flag unchecked. For each open deal
   confirmed quiet, compute last contact date across email/Lemlist/Slack/calls. If
   >14d (high priority >21d) and a nudge is warranted (alive, not awaiting a booked
   call): add a To-Do to the SwimScore Notion board under **New sales**
   (`Follow up with <clinic> — quiet <N>d`), Owner Shaffy, Link the deal, dedup on
   `Ref hubspot:stale:<dealId>`, and **draft a short, specific follow-up message from
   the deal's HubSpot context** into the to-do (only if a real message fits).
   **Check for an existing to-do with the same `Ref` first and update it in
   place** — a live run on 2026-08-16/17 found two rows for the same deal with
   different day-counts.

9. **Update the SwimScore Notion board from all channels** (W2 `daily-open-items.md`):
   reconcile internal to-dos across the seven pillars (Onboarding, New sales,
   Marketing, Research, Clinic & patient portal, Support, Finance) from
   business-strategy, product/portal, legal, finance, and support signal — check
   what each item is for, mark Done / advance, dedup on `Ref`, add new ones in
   house style per `STYLE.md`. Also check goal coverage against the **Key Goals**
   database (linked from `SWIMSCORE_NOTION.md`) — a goal with no linked to-do, or
   only execution items and nothing that measures progress, needs a new to-do.

End with a digest: **enriched deals** (deal — B2B_Type — Orders_PM — stage it
landed at), deals updated with notes (+ the info-owed stage moves only + which
early-funnel deals had their Description/Next step refreshed), **new
opportunities found but with no matching deal** (for Shaffy to review — never
created by this sweep), **HubSpot Tasks created/completed** (deal — subject —
why), **stale deals flagged (with/without drafted message)**, and
Notion to-dos added/advanced/closed per pillar. Call out the top risks and
decisions for the CEO.

10. **Post the recap to `#daily-recap` in Slack**, in this order:

   **a. "Updates from yesterday"** — one line per source, factual, scoped to the
   last calendar day (not a re-hash of the multi-day CRM sweep):
   - **Orders** — count from `hubspot.json.slackChannels.newOrders`
     (`#0-new-order-received`, automated Shopify feed). Just the count + pointer to
     the channel; never pull Shopify orders/customers into HubSpot (B2C stays out
     of scope for deals).
   - **Support questions** — count from `hubspot.json.slackChannels.hubspotInboxLive`
     (`#hubspot-inbox-live-responses`). Count + pointer; flag only if something
     looks urgent (refund demand, angry customer, security issue).
   - **Lemlist-replies** — count from the `#lemlist-replies`/outbound channel for
     the day, and call out how many are genuinely worth a follow-up (positive
     `aiLeadInterest`) vs declines/automated.
   - **Pipeline** — the single most consequential deal development from
     yesterday, if one exists, framed as what Shaffy needs to do about it (e.g. a
     reply that raises a pricing/decision point) — verify it directly (§3a in
     `hubspot-sync.md`), don't guess.
   - **Finance** — anything time-sensitive that landed (e.g. a compliance/KYC
     request) — read the actual email/thread before summarizing; don't paraphrase
     from memory or assume specifics (who/what) without checking.
   Skip a bullet entirely if there's nothing real to report — don't pad it.

   **b. "Short-term goals"** (by category: Pipeline, Marketing, Product/Onboarding,
   Support, ...) **and "To dos (specific)"**, plus links to the Notion To-Dos board
   and Key Goals database. **Filter the to-dos by business judgment, not by
   mechanically listing every open/High-priority Notion item** — the full board is
   one click away, so this list should only carry what actually moves the needle
   this week: revenue/pipeline decisions, live customer-facing issues, and anything
   genuinely blocked or needing Shaffy's input. Two defaults that follow from this:
   - **Don't list every stale deal individually** — name the highest-value one(s),
     batch the rest as a cleanup pass.
   - **Suppress routine dev-team-execution items** (most of Clinic & patient
     portal / Support) — Dmytro and Harsh close these out themselves in their own
     channels without needing Shaffy's attention. Only surface a dev item if it's
     (a) blocked or stalled past a normal turnaround, (b) needs a decision only
     Shaffy can make, or (c) is a live patient/customer-facing issue right now
     (e.g. a data bug affecting current users) — not just "high priority" in
     Notion. Everything else stays tracked in Notion, just not in the Slack recap.
