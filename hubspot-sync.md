# Daily HubSpot Deal Sync — keep every deal current from the full conversation

**Run every morning (~07:00 Europe/Amsterdam).** HubSpot is the **single source of
truth**. Act like the person who owns this pipeline: read every channel, update
each deal that has a development, and flag deals that have gone quiet so they get a
follow-up. Config: `hubspot.json`.

> **Pipeline:** W0 (`new-deal-discovery.md`, create new deals) → **W1 (this,
> update every deal)** → W2 (`daily-open-items.md`, Notion to-dos). Run all three
> with `/daily-crm`.

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
  - **No deal + genuine B2B interest** → hand to **W0** to create (deal named after
    the clinic/company, link contact + company + domain). Negative replies
    ("no thanks", "unsubscribe", wrong-email) are **not** deals.
  - **Existing deal** → if there's new content, update it (steps 4–5).

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
3. **Keep the stage honest** — advance on clear evidence (call booked → Discovery
   Scheduled; call held → Discovery Completed; proposal sent → Proposal Sent; pilot
   terms → Pilot Discussion; onboarding/live → Pilot Active). Record `old → new — why`
   in the note. Honor `ingest.autoApply`; **never** auto-set Closed Won/Lost
   (`allowStageClose:false`) — list those for approval. Never silently demote an
   active deal.

## 7. Stale-deal check → flag + suggested follow-up (per `staleFollowUp`)
For every **open** deal (skip `excludeStages` = Closed Won/Lost and any dead/
cold-rejected lead), compute **last contact date** = the most recent inbound or
outbound across email, Lemlist, Slack, and calls.
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
delete; never create duplicate deals (that's W0).

## 9. Output — deal-sync digest
**Preflight** ✅/⚠️ · **Deals reviewed** · **Notes added** (deal — gist — channel) ·
**Stage moves** (`deal: old → new — why`) + **proposed Closed Won/Lost (approval)** ·
**Stale deals flagged** (deal — days quiet — follow-up drafted? y/n) · **New
opportunities handed to W0** · **Skipped/degraded**.

## Scheduling
Daily ~07:00 Europe/Amsterdam, **Sonnet**, via `/daily-crm` (W0 → W1 → W2). Enable
via `ingest.enabled`.

**Prompt:**
> Run W1, the daily HubSpot deal sync in `hubspot-sync.md`. Preflight; check Lemlist
> (new → W0, existing → update), read all of Shaffy's email threads, sweep the Slack
> deal channels, and read Granola call notes; update each deal's HubSpot note + stage
> only where there's a development; then flag every open deal whose last contact is
> >14d with a Notion follow-up To-Do + a suggested message drafted from HubSpot
> context. Finish with the deal-sync digest.
