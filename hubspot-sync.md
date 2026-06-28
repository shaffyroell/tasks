# Daily HubSpot Deal Sync — keep every deal current from the full conversation

**Run every morning (~07:00 Europe/Amsterdam).** HubSpot is the **single source of
truth**. Your job is to act like the person who owns this pipeline: for **every
open deal**, read the **full recent conversation across all four channels —
Granola, Slack, Lemlist, and email — and make HubSpot reflect it**: a dated note
on the deal summarizing what happened, and the **deal stage** moved when the
evidence warrants.

> **Deal-first, evidence-from-everywhere.** Walk the open deals (don't wait for a
> source to surface them), but for each deal gather the latest from **all four
> sources** so nothing is missed. If a deal has fresh communication that isn't
> reflected in HubSpot, that's a **gap — always close it** with a note and, when
> warranted, a stage move.
>
> **Pipeline:** W0 (`new-deal-discovery.md`, creates new deals) → **W1 (this,
> updates every existing deal)** → W2 (`daily-open-items.md`, derives Notion
> to-dos from HubSpot).

**Config:** `hubspot.json`. Lookback = `ingest.lookbackDays` (default **3 days**;
widen after a gap). Pipelines, stages, and their internal IDs are all defined
there — never hard-code a stage; read it from the config.

---

## 0. Preflight
Open with a one-line readiness check for each source:
**HubSpot** (write — `get_user_details`), **Gmail** (native), **Slack**,
**Granola**, **Lemlist** (`get_team_info` / `get_campaigns`), **Shopify**
(`get-shop-info`). Mark each ✅/⚠️. If a **source is down, say so loudly** and never
move a stage on a partial view — note what you could see and flag the gap.

## 1. Load every open deal from HubSpot
- `search_crm_objects(objectType="deals")` filtered to **open** deals — exclude
  the closed stages (`closedWon` / `closedLost` ids in `hubspot.json`). Pull
  `dealname`, `dealstage`, `pipeline`, `hubspot_owner_id`, `amount`,
  `deal_currency_code`, `hs_lastmodifieddate`, `hs_object_id`.
- For each deal, pull its **associated contacts and company** (emails + domains) —
  these are the keys you'll match communications against.
- Note each deal's **current stage label** (resolve the id via the config). The
  stage tells you what to look for next (e.g. a deal in *Discovery Scheduled*
  should be checked for whether the call happened → *Discovery Completed*).

## 2. For each deal, gather the full recent conversation (all four sources)
Match on the deal's contact emails / company domain in each source:

- **Email (Gmail, native MCP only)** — every thread in the last `lookbackDays`
  with any of the deal's contacts. Read the substance: who said what, asks,
  commitments, dates. Reply-detection: is the ball in our court or theirs?
- **Slack** — search the deal's contacts/company across client & Connect channels
  **and** the internal channels where this account is discussed. Read threads,
  not just top messages.
- **Granola** — every meeting note in the window involving the deal's people/
  company. Calls carry the real decisions — read them in full.
- **Lemlist** — `get_inbox_conversations` (optionally `campaignFilter`) to find
  replies, then `get_inbox_conversation(contactId)` for the thread. Use the AI
  signal it carries: `aiLeadInterest` = **positive** (level ≥4) / neutral /
  **negative** (≤1). A positive reply is a buying signal; a negative reply is a
  churn/lost signal. Match the Lemlist lead email to the deal's contact.
- **Shopify (B2B inbound only)** — inbound **messages/inquiries via the website**
  (contact form, wholesale/clinic interest at www.myswimscore.com) are often
  **B2B clinic leads**. They typically arrive as emails to `info@myswimscore.com`
  (caught under Email above) and/or surface in Shopify. Keep only B2B clinic/
  partner intent: match it to an existing deal, or hand a genuinely new one to W0.
  **Ignore individual B2C patient orders** — B2C is out of scope; never open a deal
  for a B2C patient.

> If a meaningful communication belongs to a **prospect with no deal yet**, don't
> force-fit it — hand it to W0 (`new-deal-discovery.md`). B2C Shopify patients are
> **not** deals and are out of scope here — we focus on B2B.

## 3. Write one consolidated note to the deal — **if it isn't already there**
1. **Check first.** Read the deal's recent notes/engagements
   (`search_crm_objects(objectType="notes"` associated to the deal, or the deal
   timeline). If *this* conversation/call is already captured (match on
   thread/meeting + content, not just date), **skip it** — no duplicates.
2. **If it's missing, add a note.** Create a `notes` object
   (`hs_note_body`, `hs_timestamp` = now) and **associate it to the deal**
   (`manage_crm_objects` create `notes` with an association to the deal id).
   Stamp the body `[hubspot-ingest YYYY-MM-DD]` and structure it like an account
   manager's pre-call brief:
   - **What happened** — synthesize across email/Slack/Granola/Lemlist; the gist,
     not a dump. Name the channel each point came from.
   - **Decisions & commitments** — what was agreed, what *we* owe, what *they* owe,
     by when.
   - **Risks / blockers** — anything that could stall or lose the deal.
   - **Next step** — concrete action + owner + date.
3. **Think critically per deal:** is it progressing, stalling, or at risk? Is a
   commitment slipping? Is there a pilot-expansion or renewal cue? That judgement
   is the point.

## 4. Move the deal stage when the evidence warrants
Use the stage map in `hubspot.json` (`pipelines.<pipeline>.stages`). Update via
`manage_crm_objects` update on the deal, setting `dealstage` to the **stage id**
(and `pipeline` if it must change). Typical Clinic-Partnerships transitions:

| Evidence in the conversation | Move to |
|---|---|
| First outbound / Lemlist sequence sent, no reply yet | `Outreach Sent` |
| Positive reply / interest (email, Slack, or Lemlist `aiLeadInterest=positive`) & a call being booked | `Discovery Scheduled` |
| Discovery/intro call has happened (Granola note exists) | `Discovery Completed` |
| Proposal / pricing sent | `Proposal Sent` |
| Pilot terms being discussed | `Pilot Discussion` |
| Pilot live / kicked off | `Pilot Active` |
| Explicit win / signed | `Closed Won` *(approval required)* |
| Explicit no / Lemlist `aiLeadInterest=negative` / hard bounce-out | `Closed Lost` *(approval required)* |

**Rules:**
- Only advance on **clear** evidence; record `old → new — why` in the note.
- **Never silently demote an active deal** on a quiet day — only on an explicit
  pause/churn signal.
- Honor `ingest.autoApply` (false → propose moves in the digest instead of
  applying). Honor the guardrails: `allowStageAdvance` gates forward moves;
  `allowStageClose=false` means **never** auto-set Closed Won/Lost — list those
  for approval.

## 5. Idempotency & safety
- Dedup notes at the **content** level (step 3.1), not just the date marker.
- Dedup a call by its Granola meeting id; a Lemlist reply by its activity/contact
  id — never re-summarize the same conversation twice.
- Only write when there's genuinely new, un-noted content.
- Never delete anything; never create duplicate deals (that's W0's job).

## 6. Output — deal-sync digest
- **Preflight** ✅/⚠️ per source.
- **Deals reviewed** (count) · **Notes added** (deal — 1-line gist, channel(s)).
- **Stage moves** (`deal: old → new — why`) and **Proposed moves awaiting
  approval** (any Closed Won/Lost, or all moves if `autoApply=false`).
- **Gaps closed** — deals that had fresh comms with no HubSpot note before.
- **New opportunities handed to W0** (prospects with no deal).
- **Skipped / degraded** sources.

## Scheduling
Claude Code web routine, **daily ~07:00 Europe/Amsterdam**, **Sonnet**, after W0.
Enable via `ingest.enabled` in `hubspot.json`. Or run the whole chain with
`/daily-crm` (W0 → W1 → W2 in order).

**Prompt:**
> Run W1, the daily HubSpot deal sync in `hubspot-sync.md`. Preflight all sources;
> load every open deal; for each, read the full recent conversation across
> Granola, Slack, Lemlist, email, and B2B Shopify website inquiries; add a
> consolidated `[hubspot-ingest]` note where one is missing; and move the deal
> stage when the evidence warrants (honoring autoApply + close guardrails). B2C
> patients are out of scope (B2B only). Finish with the deal-sync digest.
