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
> **Pipeline:** ~~W0 (`new-deal-discovery.md`, create new deals)~~ *(disabled)* →
> **W1 (this, notes + the narrow Lemlist stage move)** → W2
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
  - **No deal + genuine B2B interest** → **do not create a deal.** List it in the
    digest under "New opportunities (not created — create manually)" with contact,
    company, and why it looks genuine, so Shaffy can add it himself. Negative
    replies ("no thanks", "unsubscribe", wrong-email) don't even need listing.
  - **Existing deal** → if there's new content, update it (steps 4–5), and if the
    deal is currently at **In conversation (Lemlist)** apply the stage-advance rule
    below (step 6.3).

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
contact id; dedup stale To-Dos on `Ref`. Only write on genuinely new content. Never
delete; **never create a deal** (W0 is disabled — see the banner at the top).

## 9. Output — deal-sync digest
**Preflight** ✅/⚠️ · **Deals reviewed** · **Notes added** (deal — gist — channel) ·
**Stage moves** (`deal: In conversation → Asked for information/Demo scheduled —
why` — this is the only kind of stage move that should ever appear here) ·
**Stale deals flagged** (deal — days quiet — follow-up drafted? y/n) · **New
opportunities found but NOT created** (contact/company — why it looks genuine — for
Shaffy to add manually) · **Skipped/degraded**.

## Scheduling
Daily ~07:00 Europe/Amsterdam, **Sonnet**, via `/daily-crm` (W1 → W2; W0 disabled).
Enable via `ingest.enabled`.

**Prompt:**
> Run W1, the daily HubSpot deal sync in `hubspot-sync.md`. Preflight; check Lemlist
> (new opportunity with no deal → list for manual creation, don't create it;
> existing deal → update), read all of Shaffy's email threads, sweep the Slack
> deal channels, and read Granola call notes; log each deal's HubSpot note where
> there's a development, applying the one narrow stage-advance rule
> (`ingest.stageAdvanceRule`) and leaving every other deal's stage untouched; then
> flag every open deal whose last contact is >14d with a Notion follow-up To-Do + a
> suggested message drafted from HubSpot context. Finish with the deal-sync digest.
