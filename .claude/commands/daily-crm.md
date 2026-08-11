---
description: Daily SwimScore CEO sweep — Lemlist + email + Slack + calls → create deals for genuine new opportunities (landing at In conversation or Asked for information), log HubSpot deal notes (+ narrow Lemlist stage move + Description/Next step refresh on early-funnel deals + HubSpot Tasks for anything clearly owed on our side + Company backfill on Lemlist-created deals), flag stale deals with a drafted follow-up, and update the SwimScore Notion board
---
Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Work the steps in order; open with a per-source preflight line
(HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion).

> **2026-07-25 — scope narrowed at Shaffy's request:** this sweep no longer
> creates HubSpot deals, and no longer advances stages freely. See step 6 for the
> one stage move still allowed, the Description/Next step refresh on early-funnel
> deals, and the (high-bar, deal-related-only) HubSpot Task creation.
>
> **2026-07-25 (2):** a separate automation now auto-creates a HubSpot deal for
> every Lemlist reply. This sweep still never creates deals — but for any deal in
> "In conversation (Lemlist)" it touches, it now verifies a Company is linked and
> creates+links one (by domain) if the external flow didn't. See step 6.
>
> **2026-08-11 — deal creation re-enabled, superseding the 2026-07-25 note
> above:** Shaffy asked for new HubSpot deals back, with one change from the old
> (pre-2026-07-25) behavior — a new deal never auto-lands past **In conversation
> (Lemlist)** or **Asked for information** (`hubspot.json →
> newDealDiscovery.stageAssignRule`), never further along the pipeline on
> autopilot. Step 1 now creates a deal (via W0, `new-deal-discovery.md`) for
> genuine new opportunities instead of just listing them.

1. **Lemlist — new & updated conversations.** Pull the full team inbox
   (`get_inbox_conversations listId=teamConversations`, paginated; read threads via
   `get_inbox_conversation`, note `aiLeadInterest`). Almost every reply now already
   has a deal (the external Lemlist-to-HubSpot flow creates one) — match to it and
   update (step 6). Genuine B2B interest with **no** deal (the external flow hasn't
   caught up, or it came via email/Shopify) → **create the deal** (W0,
   `new-deal-discovery.md`): find-or-create the company + contact, land the deal at
   **Asked for information** if the reply asks a question/requests info or
   pricing, else **In conversation (Lemlist)** (the default — never assign further
   automatically), and add a `[new-deal]` note summarizing the thread. Low-confidence
   intros (thin, no real service text) still just get listed in the digest for
   review rather than created. Negative replies are not deals.

2. **Shopify — check inbound contact-form messages FIRST** (before the email
   threads). Read **only** the website `"New customer message"` contact-form
   submissions (they arrive as emails to `info@myswimscore.com`). Keep only B2B
   clinic/partner intent → match to a deal or hand to W0. **Do NOT read Shopify
   orders or the customer list** — those are B2C and out of scope.

3. **Email — read ALL of Shaffy's threads** (`shaffy@myswimscore.com` is the source
   of truth after Lemlist). For every serious conversation (calls held, proposals,
   pricing, scheduling, commitments), find its deal and capture what moved. **Also
   read threads where Shaffy is only CC'd by a teammate** (`cc:shaffy@myswimscore.com`
   + threads from team senders: info@myswimscore.com, elara.k@maleswimscore.com,
   stewart.hill@checkswimscore.com, syb@myswimscore.com, other *swimscore* domains) —
   these often carry pipeline updates. A dated `newer_than:Nd` sweep can miss a
   thread's actual latest message — never report "no activity" from an impression;
   state the literal last-message date/sender for every deal touched.

4. **Slack — sweep the deal channels** in `hubspot.json.slackChannels`: outbound /
   lemlist-replies, pipeline-clients, business-strategy, clinic-portal-dev,
   wellness-portal-dev, legal, daily-status, + any other account channel. Read
   threads. Pull anything that changes a deal or is an internal action item.

5. **Calls — read all recent Granola notes** for decisions, pain points, next steps.

6. **Update HubSpot per client — only on a development.** Check the deal's existing
   notes first (no duplicates); if something moved, add/refresh a `[hubspot-ingest]`
   note (Background → timestamped Timeline → Latest → Next step). **Stage moves are
   narrow** (`hubspot.json → ingest.stageAdvanceRule`): only ever move a deal off
   **In conversation (Lemlist)** to **Asked for information** or **Demo scheduled**,
   and only when a Lemlist reply shows real interest. Every deal already at
   Contracting, Portal onboarding, First order placed, Actively ordering, No orders
   (L3M), or Closed Lost keeps its current stage no matter what happened — just log
   the note. Honor autoApply; never touch a close state.

   **On early-funnel deals only** (Inbound request / In conversation / Asked for
   info / Demo scheduled — `pipelines.clinicPartnerships.earlyFunnelStages`), also
   refresh the native `description` and `hs_next_step` fields to match — **max 2
   sentences each**, since they render on the HubSpot board cards. Skip this for
   Contracting-onward deals; Shaffy manages those fields by hand.

   **For any deal at "In conversation (Lemlist)" you touch, verify it has a
   linked Company** (`hubspot.json → orgLinking`) — the external automation that
   auto-creates these deals doesn't reliably attach one. If missing, match or
   create a company by the contact's email domain and associate it — the one
   narrow exception to never creating records (company-only, never a deal or
   contact).

   **Create a HubSpot Task linked to the deal only for clearly deal-related items
   where we clearly owe a reply, or something clearly important surfaced in
   email — high bar** (`hubspot.json → tasks`, pipeline-wide). Not a catch-all:
   skip minor or ambiguous items, and anything the stale-follow-up flow (step 7)
   already covers — a cluttered task list gets ignored. **Verify against the
   deal's existing notes/activity first that it isn't already done.** Dedupe on a
   `Ref:` line in the task body; complete an existing task if a fresh note shows
   its item got resolved.

7. **Stale check → flag + drafted follow-up** (per `hubspot.json.staleFollowUp`).
   **Before flagging or re-affirming any deal as stale, verify directly**: an undated
   `from:`/`to:` search on that contact's email, reading the actual last message's
   date — never carry forward a prior run's stale flag unchecked. For each open deal
   confirmed quiet, compute last contact date across email/Lemlist/Slack/calls. If
   >14d (high priority >21d) and a nudge is warranted (alive, not awaiting a booked
   call): add a To-Do to the SwimScore Notion board under **New sales**
   (`Follow up with <clinic> — quiet <N>d`), Owner Shaffy, Link the deal, dedup on
   `Ref hubspot:stale:<dealId>`, and **draft a short, specific follow-up message from
   the deal's HubSpot context** into the to-do (only if a real message fits).

8. **Update the SwimScore Notion board from all channels** (W2 `daily-open-items.md`):
   reconcile internal to-dos across the seven pillars (Onboarding, New sales,
   Marketing, Research, Clinic & patient portal, Support, Finance) from
   business-strategy, product/portal, legal, finance, and support signal — check
   what each item is for, mark Done / advance, dedup on `Ref`, add new ones in
   house style per `STYLE.md`. Also check goal coverage against the **Key Goals**
   database (linked from `SWIMSCORE_NOTION.md`) — a goal with no linked to-do, or
   only execution items and nothing that measures progress, needs a new to-do.

End with a digest: **new deals created** (deal — contact — company — stage —
why), deals updated with notes (+ the narrow In conversation → Asked
for information/Demo scheduled stage moves only, + which early-funnel deals had
their Description/Next step refreshed), low-confidence opportunities found but
NOT created (for manual review), **HubSpot Tasks created/completed** (deal —
subject — why), **stale deals flagged (with/without drafted message)**, and
Notion to-dos added/advanced/closed per pillar. Call out the top risks and
decisions for the CEO.

9. **Post the recap to `#daily-recap` in Slack**, in this order:

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
