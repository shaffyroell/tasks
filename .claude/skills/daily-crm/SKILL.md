---
name: daily-crm
description: Daily SwimScore sweep, CONTEXT-ONLY (rewritten 2026-08-18). Checks Lemlist (first replies, email + LinkedIn) and Front's Email Replies inbox (supplementary, email-only follow-ups) for new/updated conversations; creates a HubSpot deal when a genuinely new B2B reply has no match anywhere — every deal this workflow creates, and every deal Lemlist's automation creates, lands at and STAYS at "Reply (to-be-enriched)" with NO owner, since this workflow never moves a deal's stage under any circumstance (a clean decline gets suggested as Closed Lost in the note/digest, never set). Reads all new emails from the last 2 days across Gmail/Lemlist/Front and adds one HubSpot activity note per individual email observed (that email's own text only, not the quoted chain) plus a same-pass touch-tracking field update; sweeps the Slack deal channels; reads Granola call notes; refreshes Description/Next step on early-funnel deals; creates a HubSpot Task only for clearly deal-related items still owed — high bar, checked against existing notes first; backfills a missing Company on any Reply-to-be-enriched deal it touches; flags deals with no contact in 2-3 weeks (Notion To-Do with a drafted follow-up); updates the SwimScore Notion board. Run daily.
---

Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Open with a per-source preflight (HubSpot, Gmail, Slack, Granola,
Lemlist, Front, Shopify, Notion); work the steps in order.

> **2026-08-18 — this workflow is CONTEXT-ONLY: it never moves a deal's stage,
> for any reason, ever.** Supersedes the 2026-08-17 "deal creation retired" note
> below in full, and reverses it: this workflow now **creates** a HubSpot deal
> whenever a genuinely new B2B reply (Lemlist, Front, Gmail, or Shopify) has no
> matching deal anywhere — because Lemlist's own automation only covers
> Lemlist-tracked replies, and a genuine B2B reply can now also arrive as a
> Front-only email follow-up or a Shopify contact-form message that Lemlist
> never sees. **Every deal this workflow creates, and every deal Lemlist's own
> automation creates, lands at and STAYS at "Reply (to-be-enriched)"**
> (`4154315512`) — this workflow adds context (notes, `b2b_type`, `orders_pm`,
> Company links, touch-tracking, description/next-step) but never advances,
> never closes, never sets any stage on any deal, at any point in this
> workflow, for any reason. **Every one of these deals must carry NO owner**
> (`hubspot_owner_id` blank, `hubspot.json → ownerPolicy`) — check and clear it
> on every touch; Shaffy assigns ownership by hand once he's reviewed a deal and
> is ready to move it. The **one** narrow allowance for a clean, unambiguous
> decline: **suggest** Closed Lost in the note and the digest — never set it.
> `newDealDiscovery` in `hubspot.json` is `enabled:true` again, landing every
> new deal at `4154315512`, never further along the pipeline, ever.
>
> **Analyze all emails from the last 2 days, every run** (`hubspot.json →
> ingest.lookbackDays`, narrowed from 3 to 2) — across Gmail, Lemlist, and
> Front. **Every individual email you see must correspond to (1) its own
> HubSpot activity note, containing ONLY that email's own text — the new
> content that person or SwimScore actually typed, never the quoted reply
> chain beneath it — and (2) an updated touch-tracking field set, in the same
> pass** (`hubspot.json → ingest.perMessageNoteRule` /
> `touchTracking._2026-08-18_everyEmail_note`). This replaces the old "one
> consolidated note per deal per run" model: if 3 emails came in on one deal
> inside the window, that's 3 notes, each carrying just its own message text,
> dedup'd by date+direction+first-line so a re-run doesn't double-log the same
> message.
>
> **Touch-tracking fields are not optional and not a separate pass — and apply
> to every touch, not just deal creation.** Set `last_touch_date`/
> `last_touch_direction`/`last_message`/`initial_reply_lead`/`initial_reply_ss`/
> `reply_channel` (`hubspot.json → touchTracking`) in the exact same edit as any
> note/field-refresh, on every deal touched, enriched, OR simply re-evaluated —
> not only newly-created ones. **Always re-derive all five fields from a fresh
> full-thread read (Lemlist + Gmail + Front, all directions) — never trust a
> stored value just because nothing "new" happened today; the value may have
> been wrong since the deal was created.** See step 7 for the full procedure
> and the mandatory end-of-run spot-checks.
>
> ~~**2026-08-17 — deal creation retired for good; stages relabeled to
> action-owed.**~~ ~~**2026-08-11 — deal creation re-enabled** (had been off
> 2026-07-25 – 2026-08-11), a new deal never auto-lands past In conversation
> (Lemlist) or Asked for information.~~ *(both superseded 2026-08-18, see
> above — deal creation is back on with a different landing rule, and stage
> movement of any kind is now fully retired, not just narrowed)*

1. **Lemlist — new & updated. Check this FIRST, for first replies.** Full team
   inbox (`teamConversations`, paginated; `get_inbox_conversation` for the thread
   + `aiLeadInterest`). **Lemlist is the primary source for a lead's first reply
   on either channel a campaign runs on — email AND LinkedIn** — check both, not
   email only. Match the lead email/domain to its HubSpot deal (search by
   contact email, and by company domain — the automation sometimes creates the
   contact/company ahead of the deal) and add context (step 7). **If genuine
   B2B interest truly has no matching deal anywhere, CREATE one** (`hubspot.json
   → newDealDiscovery`) — landing at **Reply (to-be-enriched)**, **no owner**,
   never any further stage. Negative replies aren't worth creating a deal for;
   note the reason if a deal already exists, otherwise skip.

1b. **Front — Email Replies inbox, supplementary, email-only (added 2026-08-18).**
   Checked AFTER step 1, not instead of it. **Front is scoped to email only** (it
   has no LinkedIn visibility) and its real value is catching **email follow-ups
   that may never surface in Lemlist** — a lead replying again later in the same
   email thread, or on a side thread, after their tracked Lemlist reply, isn't
   always still visible in Lemlist's inbox view. Sweep the shared **Email
   Replies** inbox (`inb_nh7fs`, ticket prefix `SU-####`) via
   `mcp__Front__search_conversations` (`filters.inboxId: inb_nh7fs`,
   `scope: all_inboxes`, `filters.after` bounded to the 2-day lookback window) —
   it aggregates replies across all of SwimScore's rotating cold-outreach
   sending mailboxes (myswimscore.com/withswimscore.com/swimscoreview.com/
   viewswimscore.com/malescore.com/goswimscore.com). Front and Lemlist
   frequently carry the SAME underlying first reply (Front is just those sending
   mailboxes viewed through a shared inbox), so check the deal's existing
   notes/touchTracking before logging anything, to avoid a duplicate note for one
   physical message — but don't assume overlap; a later email-only follow-up may
   be genuinely new information step 1 never saw.

   **Filter noise before treating anything as a real reply**: subjects ending
   `- lemwarmup` are Lemlist's own email-warmup network (fake back-and-forth
   threads that build sender reputation, always auto-resolved) — never a real
   lead; anything matching an auto-reply/OOO pattern (`Automatic reply:`,
   `Thank you for your email`, `We will get back to you shortly`,
   `Thank you for contacted...`) isn't a real reply either.

   For every conversation that survives the filter: read it with
   `read_conversation`, match the contact email/domain to its HubSpot deal (same
   matching rule as step 1), and fold it into the same enrich/note/touch-tracking
   pass as any other reply. **If a genuine B2B reply has no matching deal
   anywhere, CREATE one** (same rule as step 1 — Reply-to-be-enriched, no owner,
   never further). Front's own ticket status (Open/Waiting/Resolved) and
   `<name> - In Contact` tags are Front-side handling state (who's working the
   thread inside Front), not HubSpot dealstage — informational only, never
   mapped onto dealstage.

   **Mandatory verification rules, found live 2026-08-18 (`hubspot.json →
   ingest.sources.front._2026-08-18_gotcha_note`) — a manual spot-check of 30
   deals HubSpot showed as `last_touch_direction: Outbound` (waiting on the lead)
   found 3 that were flat wrong: one had DECLINED 2 days earlier, one had
   CONFIRMED A CALL DATE, one had said yes and was waiting on a booking link we
   owed them.** None were caught because the sweep trusted HubSpot's stored
   field instead of reading Front directly. Going forward, every single run:
   1. **Never use HubSpot's stored `last_touch_direction`/`last_touch_date` to
      decide whether a deal needs checking** — that's exactly the field being
      verified; using it to skip the check is circular. Query Front for every
      deal with a conversation touched in the lookback window regardless of what
      HubSpot currently shows.
   2. **Front's own `kind`/`origin.kind` fields lie about direction** — they
      routinely mislabel SwimScore's own outbound reply-in-thread as
      inbound-from-customer (mirrors the Lemlist `emailsReplied` mislabeling
      warning in step 7, but for Front — confirmed on nearly every SwimScore
      auto-reply in this campaign). Never trust `kind` for direction; read the
      message content and signature to determine who actually sent it.
   3. **A contact can have multiple Front conversations** — a domain/name search
      can return 2-3 threads (older campaign touches) or an outright false-match
      on a different contact. The conversation with the newest `updatedAt` is
      **not** reliably the one with the newest real message (a stale thread got
      re-touched by system processing and sorted above a real reply in one live
      case). Read every conversation the search returns for that contact,
      confirm by content it's actually them, and take the true
      chronologically-latest message across all of them.
   4. **Whatever you find — a decline, a confirmed call, a booking owed —
      NEVER move dealstage.** Note it (that email's own text, per the digest
      below); for a clean decline, suggest Closed Lost in the note and the
      digest; that's the ceiling of what this workflow does to dealstage.

2. **Enrich "Reply (to-be-enriched)" deals** (`hubspot.json → ingest.enrichment`):
   every deal this workflow creates, and every deal Lemlist's automation
   creates, lands here — and **stays here**. For each deal sitting there:

   a. **Screen for a decline first** (`ingest.enrichment.declineHandling`): read
      the reply for unambiguous not-interested language ("don't contact me,"
      "not relevant," "unsubscribe," "no thank you," etc.).
      - **Clearly declined** → write a `[hubspot-ingest]` note (that email's own
        text) explaining why it reads as a decline, and **suggest Closed Lost**
        in the note and the digest — **never set the stage**. Skip the rest of
        this step (b2b_type/orders_pm/company-backfill not worth it for a dead
        lead); the owner-check below still applies.
      - **Genuinely ambiguous** (reads negative-ish but isn't clean — deflects
        to a colleague, vague brush-off) → add a `[flag-uncertain]` note saying
        it's probably not relevant and why, and skip the rest of this step.
        List these separately in the digest.
      - **Not a decline** (positive, neutral, or a real question) → continue below.

   b. Write a `[hubspot-ingest]` note per email observed (perMessageNoteRule —
      that email's own text, who/company/what they said, channel, date); set
      **B2B_Type** (`b2b_type`) from what the clinic/practice actually is — IVF
      clinic, Acupuncture Fertility, TRT and men's health, Egg-freezing,
      Fertility guidance, Urologist, OB/GYN, Family Doctor; set **Orders_PM**
      (`orders_pm`) to the closest volume bucket if the thread mentions one,
      else default to **`1-5`** — never leave it blank; set **`amount` to
      `2500`** (`hubspot.json → defaultACV`) if not already set. Verify/backfill
      the Company (`orgLinking`). **Backfill touch-tracking fields in this same
      pass** (step 7's rules). **Check and clear `hubspot_owner_id` if set**
      (`ownerPolicy`) — this deal must have no owner. **Do NOT classify or move
      the stage** — no matter how far the conversation has actually progressed
      (info sent, a call proposed, even a call confirmed), the deal stays at
      Reply (to-be-enriched); that classification is Shaffy's call once he's
      reviewed the context this step adds.

3. **Shopify — inbound contact-form messages FIRST** (before the email threads):
   read **only** the website `"New customer message"` contact-form submissions
   (arrive as emails to `info@myswimscore.com`). Keep only B2B clinic/partner
   intent → match to an existing deal, or **create one** (Reply-to-be-enriched,
   no owner) if none exists. **Do NOT read Shopify orders or customers** — B2C,
   out of scope.

4. **Email — read ALL of Shaffy's threads from the last 2 days**
   (`shaffy@myswimscore.com` = source of truth after Lemlist): every individual
   email in that window gets its own `[hubspot-ingest]` note (that email's own
   text, not the chain) plus a touch-tracking update — calls held, proposals,
   pricing, scheduling, commitments all attach to the deal this way. **Also
   read threads where Shaffy is only CC'd by a teammate**
   (`cc:shaffy@myswimscore.com` + threads from team senders: info@myswimscore.com,
   elara.k@maleswimscore.com, stewart.hill@checkswimscore.com, syb@myswimscore.com,
   other *swimscore* domains) — these often carry pipeline updates. A dated
   `newer_than:2d` sweep can miss a thread's actual latest message — never
   report "no activity" from an impression; state the literal last-message
   date/sender for every deal touched.

   **Also search the Sent folder directly** (`in:sent newer_than:2d`, not just
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

7. **Update HubSpot per client — add context, never move a stage.** Check
   existing notes first (dedup per message, per `perMessageNoteRule` — same
   date+direction+first-line means skip); write one `[hubspot-ingest]` note
   PER EMAIL observed (that email's own text only, not the quoted chain) for
   any deal with activity in the 2-day lookback window. **This workflow never
   sets a deal's stage, for any reason, at any point** — not on info being
   sent, not on a meeting being proposed or confirmed, not on a clean decline.
   `hubspot.json → ingest.allowStageAdvance` is `false`; `stageAdvanceRule` is
   retired (kept only as historical reference for what the classification used
   to look like). The one narrow allowance: a clean, unambiguous decline gets
   **suggested** as Closed Lost in the note and digest — never set. A soft "not
   now" (e.g. service unavailable in their state) → suggest **Interested (not
   now)** (`4065138365`) in the note, same rule, never set it yourself.

   **On early-funnel deals only** (Reply (to-be-enriched) / Inbound request /
   Interested, send Information / Interested, send follow-up / Demo scheduled),
   also refresh the native `description` and `hs_next_step` fields to match —
   **description stays max 2 sentences; `hs_next_step` is a short dated log**
   (`hubspot.json → ingest.fieldRefresh.nextStepFormat`): prepend a new line
   `M/D: <what happened>. Next step: <the action>` (newest first, no year),
   keep at most the 4 most recent lines, replace same-day's line instead of
   stacking a second one. Both render on the HubSpot board cards. Skip this for
   Contracting-onward deals. This is context, not a stage move — do it
   regardless of what stage the deal is sitting at.

   **Enforce no-owner on every Reply (to-be-enriched) deal you touch**
   (`hubspot.json → ownerPolicy`): check `hubspot_owner_id`; if set, clear it
   in the same edit as whatever else you're updating on that deal. Never set an
   owner on a deal you create — omit `hubspot_owner_id` entirely.

   **Create a HubSpot Task linked to the deal only for clearly deal-related items
   where we clearly owe a reply, or something clearly important surfaced in
   email — high bar** (`hubspot.json → tasks`). Not a catch-all — skip minor or
   ambiguous items and anything the stale-follow-up flow (step 8) already covers;
   a cluttered task list gets ignored. **Verify against the deal's existing
   notes/activity first that it isn't already done.** **Before creating, always
   search the deal's existing open tasks for a `Ref:` match and skip if found —
   never create a duplicate task for the same item.** Complete an existing task
   if a fresh note shows its item got resolved.

   **For any deal at "Reply (to-be-enriched)" you touch, verify it has a linked
   Company** (`hubspot.json → orgLinking`) — neither Lemlist's automation nor
   this workflow's own deal creation reliably attaches one. **Search for the
   existing company by domain first** — deal-creation flows frequently create
   the contact/company ahead of the deal, so a blind create produces duplicates
   (confirmed live 2026-08-17: created 5 duplicate companies this way in one
   run). If none exists, match or create a company by the contact's email domain
   and associate it — the one narrow exception to never creating records
   (company-only, never a deal or contact).

   **Touch-tracking fields — check and correct on EVERY deal you touch for ANY
   reason (note, enrichment, field refresh), IN THE SAME EDIT as the
   note/field-refresh above, not a separate pass, scoped to the 2-day lookback
   window but never skipping a deal just because it "looked" unchanged**
   (`hubspot.json → touchTracking`): keep `last_touch_date`,
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

   **Every email in the 2-day window updates these fields — not just the
   "most important" one.** If a deal had 3 emails inside the window, all 3 get
   their own note (step above) and the touch-tracking fields end up reflecting
   whichever of those 3 (or an earlier Lemlist/Front message) is truly
   chronologically last.

   **2026-08-17 gotcha, twice in one run — read before touching any deal.**
   "Update touch tracking" had been misread as "update the LATEST-touch fields
   only." `Initial_reply_lead`/`Initial_reply_SS` were then left untouched on
   every deal not created in the current run, because they'd been filed
   mentally as create-time-only. **The fix: every single time you touch a
   deal, for any reason, read the FULL thread (Lemlist `get_inbox_conversation`
   + Gmail + Front, all directions) and recompute all five fields from that
   read — never assume a stored value is correct just because today's trigger
   wasn't about that field.** If `Initial_reply_lead`/`Initial_reply_SS` are
   blank and a real exchange exists anywhere in the deal's history, backfill
   both now, regardless of whether that exchange happened today.

   Determine the true last touch by checking Lemlist (`get_inbox_conversation`,
   full thread), Front (`read_conversation`, per step 1b's rules — kind field
   lies, check every matching conversation), and Gmail — search **domain-wide**
   (`from:@theirdomain.com OR to:@theirdomain.com`), not just the one contact's
   address, since other people at the same clinic often correspond too (fall
   back to a single-address search only on a personal domain like gmail.com).
   Not routed through Shaffy's inbox only, since a teammate (info@, elara.k@,
   stewart.hill@, syb@) emailing the lead directly is a real SwimScore-side
   touch whether or not Shaffy is cc'd. Always also check Gmail for a Calendly
   booking/acceptance notification — a booking can be the true last touch even
   with no new Lemlist/Front reply. Lemlist sometimes mislabels SwimScore's own
   reply-in-thread as an inbound "emailsReplied" — read the actual
   sender/content, don't trust the activity type blindly.

   **Format rule, violated on a few pre-existing deals in production:**
   `Initial_reply_lead`/`Initial_reply_SS` must be the literal reply text with
   **no** `Client:`/`SwimScore:` prefix — that prefix belongs exclusively on
   `Last_Message`; the property name already establishes the sender for the
   other two.

   **Not optional — before finishing any run, spot-check all four:**
   1. `last_touch_date NOT_HAS_PROPERTY` across early-funnel deals — should be empty.
   2. `initial_reply_ss NOT_HAS_PROPERTY` across early-funnel deals — should be
      empty except deals with a genuinely real reason (no thread exists yet, or
      SwimScore genuinely hasn't replied — which itself belongs in the digest's
      reply-owed list, not silently skipped).
   3. No `initial_reply_lead`/`initial_reply_ss` value starts with `Client:` or
      `SwimScore:`.
   4. `hubspot_owner_id HAS_PROPERTY` across Reply-to-be-enriched deals — should
      be empty; anything found gets cleared, not left for next run.

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

End with a CEO digest: **deals created** (deal — company — source, all landed
at Reply-to-be-enriched with no owner), **emails logged** (deal — count of
notes added this run), **declines flagged with Closed Lost suggested** (deal —
why — still at Reply-to-be-enriched, awaiting Shaffy's move) and **flagged for
manual review** (deal — why) from step 2a, **deals where context was added but
the conversation has clearly moved past Reply-to-be-enriched** (deal — what
happened — e.g. "call confirmed for 8/27," "info sent, awaiting response" —
call these out clearly so Shaffy knows what's ready to move by hand),
**HubSpot Tasks created/completed** (deal — subject — why), organizations
linked/created (deal — company — domain), owners cleared (deal), stale deals
flagged (with/without drafted message), **deals with `Initial_reply_SS` empty**
(a lead replied and nobody from SwimScore has answered yet — this is a
same-day follow-up list, distinct from the 14d+ stale check), and Notion
to-dos added/advanced/closed per pillar. Surface the top risks + decisions —
especially any deal that's clearly ready for Shaffy to move by hand.

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
