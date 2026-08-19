# HubSpot pipeline setup — the single source of truth

The enrichment sweep keeps every HubSpot deal **in context** — it reads Lemlist,
Front, and Gmail, writes the conversation onto the right deal as a note, and
fills the touch-tracking fields that say where each conversation actually stands.
HubSpot holds everything; it's the canonical record.

**It never moves a deal between stages.** See "Stages are manual" below.

## What's connected

All run through the **connected account** in this environment (no tokens to paste
for the core pipeline):

| Capability | Provider | Used for |
|---|---|---|
| CRM (source of truth) | **HubSpot** (MCP) | Deals, notes, context fields — read + write |
| Outreach | **Lemlist** (MCP) | First replies, email **and** LinkedIn |
| Shared inbox | **Front** (MCP) | Email replies across the rotating sending mailboxes |
| Email | **Gmail** (native MCP) | Shaffy's full conversation per deal |
| Chat | **Slack** (MCP) | Output only — the stale brief to `#daily-recap` |
| Store | **Shopify** (MCP) | B2B website contact form only; orders/customers out of scope |

## The pipeline this is wired to

`hubspot.json` is committed (IDs + rules only, **no secrets**) and holds the live
pipeline pulled from this account.

**Primary pipeline: Clinic Partnerships** (`2314112732`). Stages, in order:

| Stage | ID |
|---|---|
| Reply (to-be-enriched) | `4154315512` |
| Inbound request | `3744632513` |
| Interested, send Information | `3744632514` |
| Interested, send follow-up | `3744632516` |
| Demo scheduled | `3744632517` |
| Contracting | `3744104176` |
| Portal onboarding | `3744104177` |
| First order placed | `3744632518` |
| Actively ordering (active L3M) | `3744632519` |
| No orders (L3M) | `4047247046` |
| Closed Lost | `3744104178` |
| Interested (not now) | `4065138365` |

**Reply (to-be-enriched)** is the intake bucket: a separate automation drops every
Lemlist reply here, and this workflow's step 1 sweeps it for context.

> Stage **internal ids** differ from their labels, and **labels get renamed in the
> HubSpot UI without notice** — this file has drifted before. The workflow always
> reads the id from `hubspot.json`, never hard-codes it, but the label prose in
> these docs can still go stale. If a name here looks off, re-run
> `get_properties(objectType="deals", propertyNames=["dealstage","pipeline"])` and
> update `hubspot.json` + this file to match.

## Owners

Verified live against `search_owners` on 2026-08-19. These two ids drive the
Slack stale-brief split:

| Owner | ID | Email |
|---|---|---|
| Alex from SwimScore | `163314964` | info@myswimscore.com |
| Shaffy from SwimScore | `162479602` | shaffy@myswimscore.com |

Syb Roell is `165632552` and owns no deals in this workflow. An earlier version
of `hubspot.json` mislabelled `163314964` as "Syb" — it is Alex.

**The workflow never sets and never clears `hubspot_owner_id`.** Ownership is what
the Slack brief groups by, so it is left exactly as found.

## Stages are manual

The workflow **never writes `dealstage`** — not on a positive reply, a confirmed
call, a signed agreement, or a flat decline. There is no config flag to turn this
on; the stage-advance rules were removed entirely.

What happens instead:

- A **clean decline** gets a note explaining why it reads that way, and Closed
  Lost is **suggested** in the digest.
- A **soft "not now"** gets Interested (not now) suggested, same treatment.
- A deal whose conversation has clearly **outgrown its stage** — call confirmed,
  agreement signed, portal live — is named explicitly in the digest so Shaffy can
  move it by hand.

If a run ever changes a stage, that is a bug. The skill's end-of-run spot-check
covers it.

## What the workflow writes

**Notes — one per email thread, updated in place.** The note body opens with an
identity line (`[hubspot-ingest] Thread: gmail:<threadId>`) and a `Latest:` line.
On a re-run, a thread whose note is already current is skipped; a thread that has
grown has its existing note **updated** to carry the full current thread. A deal
with three separate threads keeps three notes. This replaced the older
one-note-per-individual-message model.

**Context fields**, recomputed from a fresh read of the whole conversation rather
than trusted from what's already stored:

- `last_touch_date`, `last_touch_direction`, `last_message` — where the
  conversation actually stands, including a Calendly booking as a real touch.
- `initial_reply_lead` / `initial_reply_ss` — each side's first reply, verbatim,
  **no `Client:`/`SwimScore:` prefix** (that prefix belongs on `last_message`
  only). `initial_reply_ss` left **empty means a reply is owed** — a deliberate
  flag that drives the digest and ranks the deal to the top of the Slack brief.
- `time_to_first_reply_hrs`, `reply_channel`, `lemlist_campaign_reply`.
- `b2b_type`, `orders_pm` (default `1-5`), `amount` (`2500` when blank).

**Board-card fields:** `description` (max 2 sentences) and `hs_next_step` (a dated
log, newest line first, 4 lines max). Both render on the card, so both stay short.

## Deal creation — one channel only

A separate automation creates a deal the moment a **Lemlist reply** lands, so this
workflow does not. The single exception is the **Shopify website contact form**
(`"New customer message"` emails to `info@myswimscore.com`) — nothing else watches
that channel, so a genuine B2B enquiry with no existing deal gets one created at
**Reply (to-be-enriched)**, never further along.

Even there, it searches hard first — by contact email, by company domain, and for
any deal linked to that company with no contact — because contact-form leads
frequently already exist from an earlier campaign touch.

B2C patient orders and customers are **never** read.

## Company backfill

The narrow exception to "never creates records". Whenever the workflow touches a
deal with no associated Company, it takes the domain from the contact's email,
**searches for an existing company first**, and associates it; only if nothing
exists does it create one. Personal domains (gmail.com, yahoo.com, …) are flagged
in the digest rather than guessed.

Search-before-create matters: a blind create produced 5 duplicate companies in one
live run.

## Two traps that produced wrong data

- **Front and Lemlist both lie about direction.** Front's `kind`/`origin.kind`
  routinely label SwimScore's own reply-in-thread as inbound-from-customer;
  Lemlist tags our outbound as an inbound `emailsReplied` activity, sometimes with
  a positive `aiLeadInterest` score. Decide direction from the body and signature.
- **`search_threads` silently truncates.** It returned 4 messages of a real
  16-message thread in production, hiding the actual last touch with no error.
  Use `get_thread` on any running exchange.

## Scheduling

One recurring Claude Code web session, prompt **`/daily-crm`**, on **Sonnet**,
several times a day. Every write is idempotent. See `SCHEDULING.md`.

## Secrets — usually NONE

The pipeline reads `hubspot.json` from git and all sources from the connected
account. Add an env secret only if a scheduled (headless) run's preflight shows a
source ⚠️ unavailable — then add just that one (HubSpot via `HUBSPOT_TOKEN`, Gmail
via `EMAIL_ACCOUNTS_JSON`, Slack via `SLACK_WORKSPACES_JSON`). Real tokens go in
env secrets or a gitignored `hubspot.local.json` — **never** in the committed
`hubspot.json`.
