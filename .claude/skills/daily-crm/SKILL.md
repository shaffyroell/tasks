---
name: daily-crm
description: Enrich the existing HubSpot pipeline with conversation context, then post a stale-deal brief to Slack. Sweeps every deal at "Reply (to-be-enriched)" and fills its touch-tracking fields from the Lemlist inbox and Front (who spoke last, whether SwimScore has replied yet, first reply on each side, channel), then reads every Gmail message from today and yesterday and attaches each lead-related email thread to its deal as a single full-thread note that is updated in place as the thread grows, then checks the Shopify website contact form for new B2B enquiries. Ends by posting to #daily-recap the deals with no touchpoint in 4+ days, split by owner (Alex / Shaffy) and ranked by who to contact first. Runs several times a day; every write is idempotent. This workflow NEVER moves a deal between stages, and creates a deal in exactly one case — a Shopify contact-form enquiry with no existing deal; Lemlist replies get their deal from a separate automation.
---

Enrich the existing pipeline. Config: committed `hubspot.json`. Open with a
per-source preflight (HubSpot, Lemlist, Front, Gmail, Shopify, Slack), then work
the four steps in order.

## What this workflow is

Its job is to put **context** on deals that already exist, and then tell Slack
which ones have gone quiet. It adds notes and fills fields. The one thing it
creates is a deal for a website contact-form enquiry that has none (step 3),
because nothing else is watching that channel.

## Hard rules — no exceptions, ever

1. **Never set or change `dealstage`.** Not on a positive reply, not on a
   confirmed call, not on a signed agreement, not on a flat decline. A deal's
   stage is Shaffy's to move, by hand, always. If the conversation has clearly
   outgrown its stage, say so in the digest — never act on it.
2. **Never create a deal — with exactly one exception: a Shopify contact-form
   enquiry (step 3).** A separate automation already creates a deal the moment a
   Lemlist reply lands, so a Lemlist, Front, or Gmail lead with no deal is
   information for the digest, not a gap for this workflow to fill. The website
   contact form is the one channel no automation watches, so it is the one place
   this workflow creates.
3. **Never create or edit a contact.** Company backfill is the single narrow
   exception (step 1h) — company records only, search-before-create.
4. **Never set or clear a deal's owner.** Ownership drives the Slack split in
   step 3, so leave whatever is there alone.
5. **Safe to re-run.** This runs several times a day. Every write below is
   idempotent — dedup before you write, and a second run in the same hour must
   produce no duplicate notes and no churn.

## Step 1 — Sweep every deal at "Reply (to-be-enriched)"

Pull **all** deals at stage `4154315512` (`pipelines.clinicPartnerships.stages.replyToBeEnriched`),
not a sample. For each one, work it end to end before moving on.

**a. Read the actual conversation first.** Never derive a field from what
HubSpot already stores — the stored value is exactly what you are verifying, so
trusting it is circular and will carry an old error forward forever.

- **Lemlist** — `get_inbox_conversation` on the deal's contact, the *full*
  thread, both directions, not the `get_inbox_conversations` preview.
- **Front** — `search_conversations` for the contact's email and domain, then
  `read_conversation` on **every** result. A contact often has 2–3 threads, and
  the newest `updatedAt` is not reliably the one with the newest real message
  (a stale thread re-touched by system processing has sorted above a live reply
  in production). Confirm by content that each thread is actually that person.
- Filter Front noise before treating anything as a reply: subjects ending
  `- lemwarmup` are Lemlist's warm-up network, never a real lead; so are
  auto-replies (`Automatic reply:`, `Thank you for your email`,
  `We will get back to you shortly`, `Thank you for contacted...`).

**b. Decide direction from message content, never from metadata.** Both systems
lie about this, confirmed repeatedly in production:

- Front's `kind` / `origin.kind` routinely label SwimScore's own reply-in-thread
  as inbound-from-customer.
- Lemlist tags our own outbound as an inbound `emailsReplied` activity, sometimes
  with a positive `aiLeadInterest` score attached.

Read the body and the signature. Who actually typed this?

**c. Set the touch-tracking fields** (`touchTracking.fields`), all of them, in
one edit, recomputed from the read above:

| Field | What goes in it |
|---|---|
| `last_touch_date` | Date of the true last message from **any** source — Lemlist, Front, Gmail, or a Calendly booking/acceptance. |
| `last_touch_direction` | `Inbound` if the lead spoke last (a Calendly acceptance counts as Inbound), `Outbound` if we did. |
| `last_message` | Literal text of that last message, prefixed `Client: ` or `SwimScore: `. If the true last touch is a calendar acceptance with no text, describe *that* event — never leave an older message sitting here. |
| `initial_reply_lead` | The lead's chronologically **first** reply to our outreach, verbatim, **no prefix**. Even if it is one word. Do not skip a short reply to record a later, meatier one — that also corrupts `time_to_first_reply_hrs`. Set once. |
| `initial_reply_ss` | Our reply *to that first reply* — not our original cold email — verbatim, **no prefix**. **Leave empty if we genuinely have not answered yet.** Empty is a deliberate flag meaning a reply is owed; it drives the digest and the step-3 ranking. |
| `time_to_first_reply_hrs` | Hours between those two. Blank while `initial_reply_ss` is blank. |
| `reply_channel` | Acquisition channel (`email` / `linkedin` / `call`). Set once, at first enrichment. A later call never rewrites this to `call`. |
| `lemlist_campaign_reply` | Campaign name the reply came from, if traceable. |

The `Client:`/`SwimScore:` prefix belongs on `last_message` **only** — the
property name already tells you who sent the other two.

**d. Always check Calendly.** Search Gmail for `from:notifications@calendly.com`,
`from:teamcalendly@send.calendly.com`, or `subject:"Accepted:"` for this contact.
A booking is often the true last touch when no message has moved.

**e. Search Gmail domain-wide, not per-person** —
`(from:@theirdomain.com OR to:@theirdomain.com) in:anywhere`. Colleagues at the
same clinic correspond too, and a single-address search has missed real activity
in production. Fall back to a single address only on a personal domain
(gmail.com, yahoo.com, …). `in:anywhere` already covers Sent. Any SwimScore
sender counts as a real outbound touch even when Shaffy is only cc'd —
`info@myswimscore.com`, `elara.k@maleswimscore.com`,
`stewart.hill@checkswimscore.com`, `syb@myswimscore.com`, and the rotating
campaign aliases. Many Lemlist mailboxes never cc Shaffy at all, so zero Gmail
results for a Lemlist-sourced lead is a real answer, not a broken search.

**f. Attach the conversation as a note** — one note per thread, per the note
rule below (shared with step 2).

**g. Fill the qualifying fields**, unless the reply is a clean decline (then
just note it and move on — a dead lead is not worth enriching):

- `b2b_type` — what the practice actually is, from the reply or the company
  itself: IVF clinic, Acupuncture Fertility, TRT and men's health, Egg-freezing,
  Fertility guidance, Urologist, OB/GYN, Family Doctor. Leave unset only if
  genuinely unclear.
- `orders_pm` — closest bucket if the thread names a volume, else `1-5`. Never blank.
- `amount` — `2500` if not already set to something real. Never overwrite a real value.
- `description` — max 2 sentences, it renders on the board card.
- `hs_next_step` — a dated log, newest first: `M/D: <what happened>. Next step: <action>`
  (no year, no leading zeros). Prepend a new line; keep the 4 most recent; if the
  deal is touched twice in one day, replace that day's line rather than stacking.

**h. Backfill the Company if missing.** **Search by the contact's email domain
first** — a blind create made 5 duplicate companies in one live run. Reuse on
match; create and associate only if nothing exists. Skip entirely for personal
domains (`orgLinking.personalDomains`) and flag those in the digest instead.

**i. A clean decline** — "not interested", "unsubscribe", "no thank you",
"stop", "remove me" — gets a note explaining why it reads as a decline, and a
**suggestion** of Closed Lost in the note and the digest. Never set it. A soft
"not now" (service unavailable in their state, revisit next quarter) gets the
same treatment suggesting Interested (not now). Anything negative-but-ambiguous
— a vague brush-off, a deflection to a colleague — gets a `[flag-uncertain]`
note and its own line in the digest.

## Step 2 — Every Gmail message from today and yesterday

Read **all** of Shaffy's mail in the window, not a filtered subset:

- `in:inbox newer_than:2d` **and** `in:sent newer_than:2d` — sent matters on its
  own, because a follow-up he sent that has not been answered never appears in
  the inbox, and missing it produces a false stale flag in step 3.
- Threads where he is only cc'd (`cc:shaffy@myswimscore.com`) and threads from
  the team senders listed in step 1e — these carry pipeline news constantly.

**Never trust `search_threads`' inline message array.** It silently truncates
long threads — in production it returned 4 of a real 16-message thread, hiding
weeks of activity including the actual last touch, with no error. For any thread
that looks like a running exchange, call `get_thread` on the threadId. If the
response overflows to a file, read it with
`jq '[.messages[] | {date, sender, toRecipients, ccRecipients, subject, snippet}] | sort_by(.date)'`
and take the true last message. One extra call is cheap.

For every thread that concerns a lead: find its deal (contact email first, then
company domain — a deal is often linked to a colleague or to the company alone).
Attach it per the note rule, and recompute that deal's touch-tracking fields
(step 1c) from the thread's true latest message.

**If a lead's thread has no deal anywhere, do not create one** — unless it is a
contact-form enquiry, which step 3 handles. List it in the digest under "no
matching deal" so Shaffy can look. That is the whole response.

## The note rule (steps 1 and 2 both)

**One note per email thread, holding the full thread, updated in place.**

- Start the note body with a stable identity line so re-runs can find it:
  `[hubspot-ingest] Thread: <source>:<threadId>` — e.g. `gmail:1a018ad161ee`,
  `lemlist:ctc_xxx`, `front:cnv_xxx`. Follow it with
  `Latest: <M/D> <Client|SwimScore>` so you can tell at a glance whether the
  note is current.
- Below that, the full thread in chronological order, each message labelled with
  its date and sender.
- **Before writing, search the deal's existing notes for that identity line.**
  - No note for this thread → create it.
  - Note exists and its `Latest:` already matches the thread's newest message →
    **skip. Change nothing.** This is the common case on a re-run.
  - Note exists but the thread has newer messages → **update that note in place**
    so it carries the full current thread, and refresh the `Latest:` line.
- **Never create a second note for a thread that already has one.** Multiple
  distinct threads on one deal each keep their own note.

## Step 3 — Shopify contact form (the one place deals get created)

Read **only** the website `"New customer message"` contact-form submissions.
They arrive as emails to `info@myswimscore.com`, so the step-2 sweep will
surface them — but check them deliberately here, because they are the one
inbound channel no automation is watching.

**Do not read Shopify orders or customers.** Those are B2C patient purchases and
are out of scope for the pipeline entirely.

For each submission in the window:

1. **Keep only B2B intent** — a clinic, practice, or partner asking about
   offering testing to their patients. A patient asking about their own kit,
   their results, or an order is B2C: skip it, do not create anything.
2. **Search hard for an existing deal before creating** — by the sender's email
   on the contact record, and by their company domain, and by any deal linked to
   that company with no contact. Contact-form leads frequently already exist from
   an earlier campaign touch. The common case is that a deal is already there.
3. **Only if there is genuinely no deal anywhere, create one:**
   - pipeline `2314112732`, stage `4154315512` (Reply (to-be-enriched)) — never
     any further along, however warm the enquiry reads
   - `dealname` = the clinic or practice name
   - `amount` = `2500`
   - omit `hubspot_owner_id` entirely. HubSpot may auto-populate an owner anyway;
     per hard rule 4 leave whatever it sets, and name the new deal in the digest
     so Shaffy can reassign it.
4. **Then enrich it in the same pass** — link the company (step 1h), attach the
   enquiry as a note (the note rule), and set the touch-tracking fields with
   `reply_channel` = `email`, `initial_reply_lead` = their enquiry text, and
   `initial_reply_ss` left empty until we have actually answered.

Every deal created here goes in the digest: deal — company — "Shopify contact form".

## Step 4 — Post the stale brief to Slack

A deal is stale when its `last_touch_date` is more than **4 days** ago
(`staleFollowUp.thresholdDays`). Scope: open deals in the Clinic Partnerships
pipeline, excluding `staleFollowUp.excludeStages` (Closed Lost, and everything
from First order placed onward — those are customers, not open pipeline).

**Verify before you flag.** Re-check the contact directly with an undated
`from:`/`to:` Gmail search plus Lemlist and Front, and read the actual date of
the last message. Never carry a stale flag forward on the strength of a stored
field or a previous run — that stored field is what you are testing.

Post to `#daily-recap` (`slack.dailyRecapChannel`), **split by deal owner**:

- **Alex** — `hubspot_owner_id` `163314964`
- **Shaffy** — `hubspot_owner_id` `162479602`
- **Unassigned / other** — everything else, as a third group

Within each group, rank by who to contact first:

1. **We owe them a reply** — `initial_reply_ss` empty, or `last_touch_direction`
   is `Inbound`. They spoke last and we went silent. These go top, oldest first.
2. **Waiting on them** — we spoke last. Below the first group, oldest first.

One line per deal: name, days quiet, who spoke last, and the single most useful
detail for picking it back up (what they asked, what was promised). Keep it
scannable — if a group runs long, list the top handful and give a count for the
rest. Skip a group entirely if it is empty rather than printing a header with
nothing under it.

## Finish with a digest

- Deals enriched (count, plus anything notable)
- Notes created vs. notes updated in place
- **Deals created from the contact form** — deal — company — the enquiry, and
  whether HubSpot auto-assigned an owner that needs correcting
- **Reply owed** — every deal where `initial_reply_ss` is empty. This is the
  same-day follow-up list and it is the most actionable thing here.
- Declines with Closed Lost suggested, and `[flag-uncertain]` deals — each still
  sitting at its current stage, awaiting Shaffy
- **Deals whose conversation has clearly outgrown their stage** — call confirmed,
  agreement signed, portal live — named explicitly, since only Shaffy can move them
- Lead threads with no matching deal
- Companies linked or created
- Anything the run could not complete, named plainly

## Before you finish — spot-check all four

1. `last_touch_date NOT_HAS_PROPERTY` across Reply-to-be-enriched deals → should be empty.
2. `initial_reply_ss NOT_HAS_PROPERTY` → only deals where we genuinely have not
   replied, and every one of those belongs in the digest's reply-owed list.
3. No `initial_reply_lead` / `initial_reply_ss` value starts with `Client:` or `SwimScore:`.
4. No deal changed stage during this run. If one did, say so loudly — that is a bug.
