---
description: Daily SwimScore CEO sweep, CONTEXT-ONLY (rewritten 2026-08-18) — Lemlist (first replies, email + LinkedIn) + Front's Email Replies inbox (supplementary, email-only follow-ups) + email + Slack + calls → match every reply to its deal, or CREATE one if a genuinely new B2B reply has no match anywhere (lands at "Reply (to-be-enriched)", no owner, and stays there — this workflow never moves any deal's stage, ever); analyzes all emails from the last 2 days and adds one HubSpot activity note per individual email observed (that email's own text, not the chain) + a same-pass touch-tracking update; refreshes Description/Next step + HubSpot Tasks for anything clearly owed + Company backfill on Reply-to-be-enriched deals; flags stale deals with a drafted follow-up; and updates the SwimScore Notion board
---
Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Work the steps in order; open with a per-source preflight line
(HubSpot, Gmail, Slack, Granola, Lemlist, Front, Shopify, Notion).

> **2026-08-18 (2) — NON-NEGOTIABLE four-point check, every single run, no
> exceptions, no scope-cutting for time.** Found live: a run skipped the actual
> deal-matching/creation check, eyeballed 3 example deals, wrongly generalized
> "looks handled" to the whole batch, and reported no new deals needed —
> **22 genuine B2B replies from one day had a contact record but zero linked
> deal.** Re-verified three independent ways (by contact association, by
> company association, by scanning every deal created in the window) before
> the gap was believed and fixed. That verification depth — not a spot-check
> of a few examples — is the bar every run must clear:
> 1. **Are there new replies in Lemlist (email or LinkedIn)?** Pull the full
>    team inbox for the lookback window, both channels, not email-only.
> 2. **Does the deal exist in HubSpot? Search deeply.** For every contact from
>    step 1 (and 3/4 below): search by the contact record's association to a
>    deal AND by the company record's association to a deal — a deal can be
>    linked to a colleague/company contact instead of the one who replied, or
>    to the company with no contact link at all. If genuinely no deal exists
>    anywhere (**the common case — check don't assume**), create one
>    (`newDealDiscovery`): Reply-to-be-enriched, no owner, relevant context
>    (notes, touch-tracking, company link). **`hubspot_owner_id` gets
>    auto-populated by HubSpot on create even when omitted from the create
>    call** — always re-fetch every newly created deal's owner field and clear
>    it in a follow-up update; never assume omitting the property on create was
>    enough.
> 3. **Are there new replies from today in Front or Shaffy's Gmail (sent +
>    inbox)?** For each: check whether it's already logged as a deal activity
>    note (dedup per `perMessageNoteRule`); if not, add it to the relevant deal
>    (note + `last_touch_date`/`last_touch_direction`/`last_message` etc.).
> 4. **Are there new replies in Shopify, and has each been followed up with in
>    Front?**
>
> These four are the floor for every run, not a nice-to-have subset — if time
> pressure means the full per-email note/field-refresh pass (steps 2/7 below)
> can't be completed exhaustively for every deal, say so explicitly in the
> digest and name what's outstanding; never silently narrow scope on the
> deal-existence check itself.
>
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
> action-owed.**~~ ~~**2026-07-25 — scope narrowed at Shaffy's request:** this
> sweep no longer creates HubSpot deals, and no longer advances stages
> freely.~~ ~~**2026-07-25 (2):** a separate automation now auto-creates a
> HubSpot deal for every Lemlist reply.~~ ~~**2026-08-11 — deal creation
> re-enabled:** a new deal never auto-lands past In conversation (Lemlist) or
> Asked for information.~~ *(all superseded 2026-08-18, see above — deal
> creation is back on with a different landing rule, and stage movement of any
> kind is now fully retired, not just narrowed)*

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
   (`shaffy@myswimscore.com` is the source of truth after Lemlist). Every
   individual email in that window gets its own `[hubspot-ingest]` note (that
   email's own text, not the chain) plus a touch-tracking update — calls held,
   proposals, pricing, scheduling, commitments all attach to the deal this way.
   **Also read threads where Shaffy is only CC'd by a teammate**
   (`cc:shaffy@myswimscore.com` + threads from team senders: info@myswimscore.com,
   elara.k@maleswimscore.com, stewart.hill@checkswimscore.com, syb@myswimscore.com,
   other *swimscore* domains) — these often carry pipeline updates. A dated
   `newer_than:2d` sweep can miss a thread's actual latest message — never
   report "no activity" from an impression; state the literal last-message
   date/sender for every deal touched.

   **Also search the Sent folder directly** (`in:sent newer_than:2d`, not just
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
   Interested, send Information / Interested, send follow-up / Demo scheduled —
   `pipelines.clinicPartnerships.earlyFunnelStages`), also refresh the native
   `description` and `hs_next_step` fields to match — **max 2 sentences each**,
   since they render on the HubSpot board cards. Skip this for Contracting-onward
   deals; Shaffy manages those fields by hand. This is context, not a stage
   move — do it regardless of what stage the deal is sitting at.

   **Enforce no-owner on every Reply (to-be-enriched) deal you touch**
   (`hubspot.json → ownerPolicy`): check `hubspot_owner_id`; if set, clear it
   in the same edit as whatever else you're updating on that deal. Never set an
   owner on a deal you create — omit `hubspot_owner_id` entirely.

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
   (`hubspot.json → touchTracking`): `last_touch_date`, `last_touch_direction`,
   `last_message` (prefixed `Client:`/`SwimScore:`), `initial_reply_lead`,
   `initial_reply_ss` (**no prefix on these two** — leave `initial_reply_ss`
   empty only if we genuinely haven't replied yet, a deliberate follow-up-owed
   flag), `time_to_first_reply_hrs`, `reply_channel`.

   **Every email in the 2-day window updates these fields — not just the
   "most important" one.** If a deal had 3 emails inside the window, all 3 get
   their own note (step above) and the touch-tracking fields end up reflecting
   whichever of those 3 (or an earlier Lemlist/Front message) is truly
   chronologically last.

   **2026-08-17 gotcha, twice in one run.** "Update touch tracking" had been
   misread as "update the latest-touch fields only" — `initial_reply_lead`/
   `initial_reply_ss` were left blank on every deal not created in the current
   run because they'd been filed as create-time-only. **Fix: every time you
   touch a deal, for any reason, read the FULL thread (Lemlist + Gmail + Front,
   all directions) and recompute all five fields from that read — never assume
   a stored value is correct just because today's trigger wasn't about that
   field.**

   Determine the true last touch by checking Lemlist, Front (per step 1b's
   rules), and Gmail domain-wide, and always check for a Calendly
   booking/acceptance too.

   **Not optional — before finishing any run, spot-check all four:**
   `last_touch_date NOT_HAS_PROPERTY` and `initial_reply_ss NOT_HAS_PROPERTY`
   across early-funnel deals should both come back empty (except deals with a
   genuinely real reason — no thread exists, or a reply is genuinely still
   owed, which belongs in the digest); no `initial_reply_lead`/
   `initial_reply_ss` value should start with `Client:` or `SwimScore:`; and
   `hubspot_owner_id HAS_PROPERTY` across Reply-to-be-enriched deals should be
   empty — anything found gets cleared, not left for next run.

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

End with a digest: **deals created** (deal — company — source, all landed at
Reply-to-be-enriched with no owner), **emails logged** (deal — count of notes
added this run), **declines flagged with Closed Lost suggested** (deal — why —
still at Reply-to-be-enriched, awaiting Shaffy's move) and **flagged for
manual review** (deal — why) from step 2a, **deals where context was added but
the conversation has clearly moved past Reply-to-be-enriched** (deal — what
happened — e.g. "call confirmed for 8/27," "info sent, awaiting response" —
call these out clearly so Shaffy knows what's ready to move by hand),
**HubSpot Tasks created/completed** (deal — subject — why), organizations
linked/created (deal — company — domain), owners cleared (deal), **stale deals
flagged (with/without drafted message)**, and Notion to-dos added/advanced/closed
per pillar. Call out the top risks and decisions for the CEO — especially any
deal that's clearly ready for Shaffy to move by hand.

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
