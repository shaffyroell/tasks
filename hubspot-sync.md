# Daily HubSpot Deal Sync — keep every deal current from the full conversation

**Run every morning (~07:00 Europe/Amsterdam).** HubSpot is the **single source of
truth**. Act like the person who owns this pipeline: read every channel, note what
moved on each deal, and flag deals that have gone quiet so they get a follow-up.
Config: `hubspot.json`.

> **2026-07-25 — scope narrowed at Shaffy's request.** This workflow no longer
> creates HubSpot deals (W0/`new-deal-discovery.md` is disabled via
> `newDealDiscovery.enabled:false`) and no longer advances stages freely. The
> **only** automated stage change left is `hubspot.json → ingest.stageAdvanceRule`:
> moving a deal from **In conversation (Lemlist)** to **Asked for information** or
> **Demo scheduled** when a Lemlist reply shows real interest. Every other stage
> (Contracting, Portal onboarding, First order placed, Actively ordering, No
> orders (L3M), Closed Lost) is manual-only — log a note describing what happened
> and let Shaffy move the card himself.
>
> **2026-07-25 (2) — a separate automation now creates deals for us.** Shaffy's
> team has wired up their own flow that auto-creates a HubSpot deal for every
> Lemlist reply. This workflow still never creates deals itself — but per step
> 6.6, it now checks every deal it touches in the intake stage(s) below for a
> linked **Company**, and creates+links one if the external flow didn't (that's
> the one narrow exception to "never create records": company-only, only to
> backfill that specific gap).
>
> **2026-08-15 — new intake stage: "Reply (to-be-enriched)".** The external
> automation now lands every fresh reply at **Reply (to-be-enriched)**
> (`4154315512`) instead of straight into **In conversation (Lemlist)**. New
> step 1b below enriches each one — context note, `b2b_type` category,
> `orders_pm` volume (default `1-5` if unknown) — then classifies it into In
> conversation / Asked for information / Demo scheduled using the same
> three-way call as the stage-advance rule in §6.3. The Company-link check in
> §6.6 now applies to both this stage and In conversation (Lemlist). `hs_next_step`
> is also now a short dated log, not free prose — see §6.4.
>
> **Pipeline:** ~~W0 (`new-deal-discovery.md`, create new deals)~~ *(disabled —
> deal creation for Lemlist replies now happens outside this workflow)* →
> **W1 (this, notes + the narrow Lemlist stage move + org-link backfill)** → W2
> (`daily-open-items.md`, Notion to-dos). Run with `/daily-crm`.

---

## 0. Preflight
One-line readiness check: **HubSpot** (write — `get_user_details`), **Gmail**
(native), **Slack**, **Granola**, **Lemlist** (`get_campaigns`), **Shopify**
(`get-shop-info`). Mark each ✅/⚠️. Never move a stage on a partial view if a key
source is down — note the gap.

## 1. Lemlist — new conversations to add, existing to update
- Pull the **full team inbox**: `get_inbox_conversations(listId="teamConversations")`,
  paginated (multiple sender personas). For each conversation in the window or with
  `isYourTurn:true`, read the thread via `get_inbox_conversation(contactId)` (carries
  `aiLeadInterest` positive/neutral/negative).
- **Match** the lead email / company domain to a HubSpot deal.
  - **Existing deal** (the normal case now — a separate automation auto-creates a
    deal for every Lemlist reply, see the banner above) → if there's new content,
    update it (steps 4–5); if the deal is at **In conversation (Lemlist)**, also
    check it has a linked Company (step 6.6) and apply the stage-advance rule
    (step 6.3).
  - **No deal found** (the external flow hasn't caught up yet, or this came
    through email/Shopify instead of Lemlist) + genuine B2B interest → **still do
    not create a deal.** List it in the digest under "New opportunities (not
    created — create manually)" with contact, company, and why it looks genuine.
    Negative replies ("no thanks", "unsubscribe", wrong-email) don't even need
    listing.

## 1b. Enrich "Reply (to-be-enriched)" deals
Per `hubspot.json → ingest.enrichment`. Lemlist's auto-create automation now lands
every fresh reply at this stage instead of directly at "In conversation
(Lemlist)". Pull every deal at `dealstage = 4154315512` and for each:

0. **Screen for a decline first** (`ingest.enrichment.declineHandling`, added
   2026-08-15, this stage only — the one narrow exception to §8's "never touch a
   close state"). Read the reply for unambiguous not-interested language:
   "don't contact me," "not relevant," "unsubscribe," "no thank you," "not
   looking for additional services," etc.
   - **Clearly declined** → write a `[new-deal]` note explaining why it reads as
     a decline, then move the deal straight to **Closed Lost** (`3744104178`).
     Skip steps 2-6 below — not worth enriching a dead lead.
   - **Genuinely ambiguous** (deflects to a colleague, vague brush-off, unclear
     tone — reads negative-ish but isn't a clean decline) → add a
     `[flag-uncertain]` note saying it's probably not relevant and why, **leave
     the stage exactly as-is**, and skip steps 2-6. Shaffy moves it by hand.
     List these separately in the digest from genuine new-deal creates.
   - **Not a decline** (positive, neutral, or a real question) → continue to
     step 1.
1. **Context note** — read the Lemlist thread (`get_inbox_conversation`), write a
   `[new-deal]` note same as §1 (who/company/what they said, channel, date).
2. **Category** — set `b2b_type` from what the practice actually is: IVF clinic,
   Acupuncture Fertility, TRT and men's health, Egg-freezing, Fertility guidance,
   Urologist, OB/GYN, or Family Doctor. Infer from the reply/company content, not
   the campaign name.
3. **Volume** — set `orders_pm` to the closest bucket (`1-5`, `6-10`, `11-25`,
   `26-50`, `51-100`, `100-200`) if the thread mentions a patient count/volume,
   else default to **`1-5`**. Never leave it blank. Also set `amount` to `2500`
   (`hubspot.json → defaultACV`) if not already set.
4. **Classify the stage** — same three-way call as §6.3's stage-advance rule:
   default → In conversation (Lemlist); asks a question/wants info or pricing →
   Asked for information; agrees to/confirms a call → Demo scheduled. A deal only
   stays at Reply (to-be-enriched) if the thread genuinely can't be read yet.
5. **Company** — verify/backfill per §6.6 (now in scope for this stage too).
6. **Fields** — apply the description/next-step refresh per §6.4.

## 2. Shopify — B2B inbound contact-form messages only (check first)
SwimScore's website (www.myswimscore.com) contact form is a real inbound channel for
**B2B clinic/partner** opportunities — screen it **before** the email threads so a
fresh website lead is in the pipeline before you read the follow-ups.
- **Only read inbound contact-form messages.** These arrive as **"New customer
  message …"** emails to `info@myswimscore.com` — search Gmail for them in the
  window. (If a Shopify-native message surface is available, read messages there;
  otherwise the email is the message.)
- **Do NOT read Shopify orders or the customer list.** Orders/customers are B2C
  patient purchases and are **out of scope** — never pull `list-orders` /
  `list-customers` and never create a deal from a patient order.
Keep **only B2B clinic/partner intent** from the contact-form message (a practice
name, provider, wholesale/partnership ask): match it to an existing HubSpot deal, or
hand a genuinely new one to **W0** (deal named after the clinic + contact + company
+ domain). A purely individual B2C message → skip.
*(Worked example already in HubSpot: Epoch Health came in as a Jun 21 website
contact-form message and is now a Pilot-Discussion deal.)*

## 3. Email — read all of Shaffy's threads for serious conversations
**`shaffy@myswimscore.com` is the source of truth after Lemlist** — he follows up
every lead by email. Read every Gmail thread with a new inbound/outbound message in
the last `ingest.lookbackDays` (widen after a gap). For each, find the deal it
belongs to and capture what actually moved: calls held, proposals sent, pricing,
scheduling, commitments.

**Also read threads where Shaffy is only CC'd by a teammate** — these very often
carry pipeline updates. The team emails leads from several personas
(`info@myswimscore.com`, `elara.k@maleswimscore.com`,
`stewart.hill@checkswimscore.com`, `syb@myswimscore.com`, and other `*swimscore*`
sending domains) and CC Shaffy. Search beyond the direct inbox, e.g.
`cc:shaffy@myswimscore.com newer_than:<lookbackDays>d` and threads from those
teammate senders, and fold any deal-relevant development into the right deal's note
— don't skip a thread just because Shaffy wasn't the direct recipient.

### 3a. Verifying recency — never characterize, always check the actual last message
A date-windowed search (`newer_than:Nd`, `cc:` search, importance flags) can surface
a thread without the reader actually noticing that thread's *most recent* message is
new. **"No activity in the window" is a claim, not an observation — it must be
backed by the literal date of the last message in that thread**, not an impression
from skimming. For every deal touched this run, record the concrete fact:
`last_message_date / sender / direction (inbound|outbound)` — not a qualitative
summary like "quiet" or "no activity." If you can't point to that fact, you haven't
actually checked.

When a specific deal is a candidate for a **stale flag, a stage regression, or a
Closed Lost recommendation** (i.e. any negative/downgrade claim — the costliest kind
of mistake, since it tells Shaffy a live conversation is dead), do a **direct,
undated verification pull** before writing anything: search by the contact's exact
email/domain with no date restriction (`from:<email> OR to:<email>`, no
`newer_than`), open the thread, and read the actual last message — sender and date.
Do not rely on a broader dated sweep's characterization for this specific
determination. Re-verify even if the deal was already flagged stale in a prior run —
don't carry an old flag forward without checking it's still true today.

## 4. Slack — sweep the deal/account channels
Read threads (not just top messages) in the channels in `hubspot.json.slackChannels`:
**outbound / Lemlist-replies**, **pipeline-clients**, **business-strategy**,
**clinic-portal-dev**, **wellness-portal-dev**, **legal**, **daily-status** — plus
any other channel where an account is discussed. Pull anything that changes a deal
(a clinic decision, a call recap, a pricing/legal blocker).

## 5. Calls — Granola
Read every Granola meeting note in the window. Calls carry the real decisions, pain
points, and next steps — attach each to its deal.

## 6. Update HubSpot per client — only when there's a development
For each deal with new substantive activity:
1. **Check the deal's existing notes first** (search `notes` associated to the
   deal). If this conversation/call is already captured, **skip** (no duplicates —
   match on thread/meeting + content, not just date).
2. **If there's a development, add/refresh the note** (`[hubspot-ingest YYYY-MM-DD]`)
   in this structure: **Background → Timeline (timestamped; Gmail = source of truth)
   → Latest status → Next step**. Synthesize across all channels; name the channel
   and date per point.
3. **Stage moves — narrow, one rule only.** Per `hubspot.json → ingest.stageAdvanceRule`:
   - **Only touch a deal's stage if it is currently "In conversation (Lemlist)"**
     (`3744632514`) **and** the development is a Lemlist reply showing real
     interest. Every deal already at Asked for information, Demo scheduled,
     Contracting, Portal onboarding, First order placed, Actively ordering, No
     orders (L3M), or Closed Lost — **leave the stage exactly as-is**, no matter
     what happened (call held, proposal sent, pilot started, order placed). Log it
     all in the note; Shaffy moves the card himself.
   - **Reply asks a question / wants pricing or info, no call agreed yet** → move to
     **Asked for information** (`3744632516`).
   - **Reply agrees to, requests, or confirms a call/demo/onboarding time** → move
     to **Demo scheduled** (`3744632517`).
   - Record `old → new — why` in the note either way (including "left unchanged,
     already past In conversation" when that's the case — makes the digest legible).
   - Honor `ingest.autoApply`. Never set Closed Lost or any close state — that's
     always manual.
4. **Refresh Description + Next step — early-funnel deals only** (per
   `hubspot.json → ingest.fieldRefresh` and `pipelines.clinicPartnerships.earlyFunnelStages`
   = Reply (to-be-enriched), Inbound request, In conversation (Lemlist), Asked
   for information, Demo scheduled). If the deal you just wrote a note on is in
   one of those five stages, also update its native `description` and
   `hs_next_step` fields to match the fresh state. `description` = who the
   contact/company is + where things stand, **max 2 sentences**. `hs_next_step`
   (2026-08-15, per Shaffy) is now a **short dated log, not free prose**: prepend
   a new line `M/D: <what happened>. Next step: <the action>` (newest first, no
   year, no leading zeros — `8/14` not `08/14`), keep at most the 4 most recent
   lines (drop the oldest), and replace that day's own line instead of stacking a
   second one if touched twice in a day. Both fields show directly on the
   HubSpot board cards and get unreadable if long. Skip this entirely for any
   deal at Contracting or later — Shaffy keeps those fields current by hand.
5. **Create a HubSpot Task when something is owed on our side — high bar, deal-related only**
   (per `hubspot.json → tasks`, pipeline-wide — not limited to early-funnel deals).
   This is **not** a catch-all for every loose end — a cluttered task list gets
   ignored, which defeats the point. Only create one when it's **clearly
   deal-related** and one of:
   - a person **explicitly asked a question or made a request and we clearly
     haven't answered it yet** (pricing, scope, scheduling, a document — a real,
     specific ask, not just "they might want to hear back eventually");
   - an email (or other channel) surfaces something **clearly important that
     needs flagging** on a deal — a decision point, a real risk, a hard
     deadline — not routine chatter.

   Do **not** create one for: minor/ambiguous items, general nudges ("might be
   worth checking in"), anything already covered by the stale-follow-up flow
   (§7 handles "gone quiet"), or anything you're not confident actually needs
   action. When in doubt, leave it out — mention it in the digest instead of
   creating a task.

   **Before creating any task, verify it isn't already done.** Re-check the
   deal's existing notes and the latest activity on that specific item (email
   thread, Slack, Lemlist) — if the reply already went out or the thing already
   happened, don't create a task for it.

   For each that clears the bar, create a `tasks` object (`manage_crm_objects`,
   associated to the deal): `hs_task_subject` short and specific ("Reply to
   Aliyah — cryo/STD/drug test scope question", not "Follow up"), `hs_task_body`
   = 1-2 sentences of context ending with a dedup line
   `Ref: hubspot:task:<dealId>:<slug>` on its own line, `hs_task_type: TODO` (or
   `EMAIL`/`CALL` if that's specifically the action), `hs_task_priority: HIGH` if
   overdue >7d or blocking a live deal else `MEDIUM`, `hs_timestamp` = today,
   `hubspot_owner_id: 162479602` (Shaffy). **Dedup before creating**: search open
   tasks (`hs_task_status != COMPLETED`) associated with the deal for an
   existing `Ref:` match — skip if found. **If a prior open task's item is now
   resolved** (a fresh note shows the reply went out / the item got done), set
   that task's `hs_task_status` to `COMPLETED` instead of leaving it stale.
   Never auto-complete a task you can't actually verify was resolved.
6. **Verify the deal has an organization linked — Lemlist-replies buckets only**
   (per `hubspot.json → orgLinking`). Shaffy's team runs a separate automation
   that auto-creates a deal for every Lemlist reply, landing it in **Reply
   (to-be-enriched)** (or, for older deals, directly in **In conversation
   (Lemlist)**) — that flow doesn't reliably attach a Company. So whenever this
   sweep touches a deal currently in either of those two stages (note added,
   enrichment pass, stage-advance check, field refresh), also:
   1. **Check for an associated Company**:
      `search_crm_objects(companies, associatedWith: deals EQUAL [dealId])`.
      If one exists, done — nothing to do.
   2. **If none**, find the deal's associated contact(s)
      (`search_crm_objects(contacts, associatedWith: deals EQUAL [dealId])`) and
      take the email domain. Skip personal domains (`orgLinking.personalDomains`)
      — flag those in the digest instead of guessing a company.
   3. **Search for an existing company by that domain**
      (`search_crm_objects(companies, query: domain)` or a `domain` property
      filter). If found, **associate it** to the deal (`manage_crm_objects`
      updateRequest with an `associations` entry) — reuse, don't duplicate.
   4. **If no company exists for that domain, create one** — `{name, domain}`,
      name derived from the deal name (the part before the em dash, if there is
      one) or the domain itself, then associate it to both the deal and the
      contact.
   - This is the **one narrow exception** to "this workflow never creates
     records" — company-only, and only to backfill a gap the external
     deal-creation flow left behind. Still never create a contact or a deal here.
   - Log every backfill in the digest (deal — company created or matched — domain).

## 7. Stale-deal check → flag + suggested follow-up (per `staleFollowUp`)
For every **open** deal (skip `excludeStages` = Closed Won/Lost and any dead/
cold-rejected lead), compute **last contact date** = the most recent inbound or
outbound across email, Lemlist, Slack, and calls.

**Before flagging any deal stale, do the 3a verification pull for that deal's
contact — no exceptions.** A dated sweep missing one reply is exactly how a live,
active negotiation gets wrongly reported as abandoned. Confirm the literal date of
the last message (undated `from:`/`to:` search on the contact's email) before
writing a stale flag, and re-confirm even for a deal already flagged stale in a
previous run.
- If last contact is **older than `thresholdDays` (14d)** — and a nudge is genuinely
  warranted (not already waiting on a booked future call, deal still alive) — **flag
  it**. Past `escalateDays` (21d), mark it high priority.
- **Hand to W2** a follow-up To-Do for the **SwimScore Notion board** under the
  `notionPillar` ("New sales"): title in house style (e.g. `Follow up with <clinic>
  — quiet <N>d (<contact>)`), `Owner` Shaffy, `Link` to the deal, `Ref`
  `hubspot:stale:<dealId>`.
- If `draftSuggestedMessage:true`, **draft a short, specific follow-up message**
  from the deal's HubSpot context (its notes: who they are, last touch, their pain
  points / open question, the agreed next step) — only if a message is actually
  relevant. Put the draft in the To-Do body so Shaffy can review/send. Don't draft a
  generic nudge where none fits — flag without a message instead.

## 8. Idempotency & safety
Dedup notes at the content level; dedup a call by meeting id, a Lemlist reply by
contact id; dedup stale To-Dos on `Ref`; dedup HubSpot Tasks on the `Ref:` line in
`hs_task_body` (§6.5); dedup companies by domain before ever creating one (§6.6 —
always search first, reuse on match). Only write on genuinely new content. Never
delete; **never create a deal or a contact** (W0 is disabled — see the banner at
the top; the one exception is company-only org-linking backfill, §6.6).

## 9. Output — deal-sync digest
**Preflight** ✅/⚠️ · **Deals reviewed** · **Notes added** (deal — gist — channel) ·
**Description/Next step refreshed** (early-funnel deals only, per §6.4) ·
**Tasks created** (deal — subject — owed by whom) + **Tasks completed** (deal —
subject — resolved by what) ·
**Organizations linked/created** (deal — company matched or created — domain,
per §6.6) ·
**Stage moves** (`deal: In conversation → Asked for information/Demo scheduled —
why` — this is the only kind of stage move that should ever appear here, aside
from §1b's decline handling) ·
**Declined deals moved to Closed Lost** (deal — why, §1b) · **Flagged for manual
review** (deal — why, still at Reply (to-be-enriched), §1b) ·
**Stale deals flagged** (deal — days quiet — follow-up drafted? y/n) · **New
opportunities found but NOT created** (contact/company — why it looks genuine — for
Shaffy to add manually) · **Skipped/degraded**.

## Scheduling
Daily ~07:00 Europe/Amsterdam, **Sonnet**, via `/daily-crm` (W1 → W2; W0 disabled).
Enable via `ingest.enabled`.

**Prompt:**
> Run W1, the daily HubSpot deal sync in `hubspot-sync.md`. Preflight; check Lemlist
> (an existing deal now covers almost every reply — a separate automation
> auto-creates one; genuinely new opportunity with no deal → list for manual
> creation, still don't create it), read all of Shaffy's email threads, sweep the
> Slack deal channels, and read Granola call notes; log each deal's HubSpot note
> where there's a development, applying the one narrow stage-advance rule
> (`ingest.stageAdvanceRule`) and leaving every other deal's stage untouched;
> refresh Description + Next step on early-funnel deals only (max 2 sentences
> each — they show on the board cards); for any deal in "In conversation
> (Lemlist)" you touch, verify it has a linked Company and create+link one if the
> external flow didn't (`orgLinking`) — the one narrow exception to never
> creating records; create a HubSpot Task on any deal where something is clearly
> owed on our side (high bar — unanswered question, clearly important email
> flag), verified against existing notes first and deduped on its `Ref:` line,
> completing any such task a fresh note shows was resolved; then flag every open
> deal whose last contact is >14d with a Notion follow-up To-Do + a suggested
> message drafted from HubSpot context. Finish with the deal-sync digest.
