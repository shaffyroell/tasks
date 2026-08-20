---
name: daily-crm
description: Enrich the existing HubSpot pipeline with conversation context, then post a stale-deal brief to Slack. Sweeps every deal at "Reply (to-be-enriched)" and fills its touch-tracking fields from the Lemlist inbox and Front (who spoke last, whether SwimScore has replied yet, first reply on each side, channel), links every thread participant's contact record to the deal (creating one, search-first, only if genuinely new), then reads every Gmail message from today and yesterday — inbox and sent, for shaffy@myswimscore.com — reconciling a thread count so none go unlogged, and attaches each lead-related email thread to its deal as a single full-thread note that is updated in place as the thread grows, then checks the Shopify website contact form for new B2B enquiries. Ends by posting to #daily-recap the deals with no touchpoint in 4+ days, split by owner (Alex / Shaffy) and ranked by who to contact first. Runs several times a day; every write is idempotent. This workflow NEVER moves a deal between stages and NEVER edits an existing contact's fields, and creates a deal in exactly one case — a Shopify contact-form enquiry with no existing deal; Lemlist replies get their deal from a separate automation.
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
3. **Never edit an existing contact's fields.** Creating a contact or company is
   allowed, but only search-before-create, exactly like company backfill (step
   1i) — reuse on match, create only when nothing exists anywhere for that email/
   domain. See "Contact linking" below for the contact case. A blind create risks
   duplicates, confirmed in production for companies — hold contacts to the same
   discipline.
4. **Never set or clear a deal's owner.** Shaffy assigns ownership by hand while
   a deal sits at Reply (to-be-enriched); that is a human judgement about who
   picks the lead up. Leave whatever is there alone — including blank. **Report
   the blanks** rather than filling them: unassigned deals get their own section
   in the Slack brief (step 4) so they can be assigned.
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

**h. Note whether the deal has an owner.** Do not set one. Collect every deal at
this stage with `hubspot_owner_id` blank — they go in the "needs assigning"
section of the Slack brief so Shaffy or Alex can pick them up.

**i. Backfill the Company if missing.** **Search by the contact's email domain
first** — a blind create made 5 duplicate companies in one live run. Reuse on
match; create and associate only if nothing exists. Skip entirely for personal
domains (`orgLinking.personalDomains`) and flag those in the digest instead.

**j. A clean decline** — "not interested", "unsubscribe", "no thank you",
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

**Calendly and DocuSign notifications are touchpoints, not noise — never bucket
them as "system/calendar notification noise" and exclude them by sender pattern
without opening them.** A Calendly acceptance/booking notification or a DocuSign
signature-completed notification documents a real client action; it is exactly
the event `last_touch_date` / `last_touch_direction` / `last_message` exist to
capture (step 1c/1d, `calendarEventFormat`). Open every one, find the deal it
belongs to (contact email, then company domain), and recompute that deal's
touch-tracking from it — using the calendar-event description when there is no
message text. This is a hard requirement found violated in production on
2026-08-19: an entire run bucketed ~57 threads as "system/calendar/billing
notification noise" by sender pattern and excluded them wholesale, which meant
any Calendly booking or DocuSign signature in that batch never updated its
deal. The **only** administrative sends safe to exclude by sender pattern
without opening them are ones with no client on either end: SwimScore's own
internal team calendar invites, Mercury/banking notifications, HubSpot's own
system emails, and generic Google account/security notifications.

**Account for every thread the searches return — this is not optional.** Treat
the combined result of `in:inbox newer_than:2d`, `in:sent newer_than:2d`,
`cc:shaffy@myswimscore.com`, and the team-sender searches (1e) as a checklist,
not a sample. Every thread on it must end this step in one of three states:
touched (note attached/updated, touch-tracking recomputed), explicitly excluded
with a one-line reason (not lead-related, internal, warmup, duplicate of a
thread already handled), or listed under "no matching deal." Before moving to
step 3, count the threads the searches returned and the threads accounted for
in the digest (touched + excluded + no-matching-deal) — **the two counts must
match.** A silent gap between them is exactly how eight real client threads —
NP Modalities, OKC Fertility, Awakin Men's Health, Ares Men's Health, CycleScript,
Wellbar Co, Wellness Culture, Perrenia Wellness — went unlogged in production on
2026-08-19 despite matching this exact search, while the run's own digest
reported a clean spot-check. If the count doesn't match, go back and process
what's missing before finishing — do not report completion on a partial pass.

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
  This lookup is verified working — search `notes` with a filter of
  `hs_note_body CONTAINS_TOKEN "<source>:<threadId>"` (e.g. `gmail:1a01374593cf1472`);
  it returns the one matching note. Read its `Latest:` line to decide what to do:
  - No note for this thread → create it.
  - Note exists and its `Latest:` already matches the thread's newest message →
    **skip. Change nothing.** This is the common case on a re-run.
  - Note exists but the thread has newer messages → **update that note in place**
    so it carries the full current thread, and refresh the `Latest:` line.
- **Never create a second note for a thread that already has one.** Multiple
  distinct threads on one deal each keep their own note.

**Backfill `b2b_type` on any deal you touch, at any stage.** Not just intake
deals. Deals leave Reply (to-be-enriched) fast, so a type set only there means
most of the pipeline never gets one — a live check found it filled on 11 of 46
open deals, which makes the type split in step 4 mostly guesswork. If a deal you
are touching has `b2b_type` blank and the thread or company makes the answer
clear, set it. Leave it blank only when genuinely unclear; never guess to fill it.

## Contact linking (steps 1 and 2 both)

**Every real participant in a touched thread must be associated with the deal —
not just the one contact HubSpot happened to attach when the deal was created.**
A deal often carries only the first person who ever replied; a colleague who
joins the thread later (a practice manager, another provider at the same clinic)
frequently has no association to the deal at all, which then hides their
messages from a Front/Gmail domain search on a later run.

For every distinct external sender or addressee on a thread you touch (steps
1a/1e/2) — skip SwimScore's own addresses:

- **Search HubSpot contacts by email first.** Reuse on match.
- **Found, not yet associated with this deal** → associate the existing contact
  record with the deal. Never edit its existing fields (hard rule 3).
- **Found, already associated** → nothing to do.
- **No contact record exists anywhere for that email** → create one (firstname/
  lastname/email pulled from the thread signature, nothing invented) and
  associate it with the deal and, if one exists, the company. Search-before-
  create — the same discipline as company backfill (1i); a blind create risks
  duplicates.

Run this on every deal a run touches, in both step 1 and step 2 — not just
intake deals — same reasoning as the `b2b_type` backfill above: skipping it
anywhere outside step 1 means most of the pipeline never gets fully linked.

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

**A deal with a future meeting already booked is not stale.** Quiet for a week
because the call is next Tuesday is the system working, not a lead going cold.
Check `hs_next_step` and the thread for a confirmed upcoming date before listing
anything; if there is one, leave it out of the ranked lists and note it under a
short "already booked" line instead. Chasing these is exactly the noise that
makes the brief get ignored.

Post to `#daily-recap` (`slack.dailyRecapChannel`) — **that channel only, never
any other.**

**Group by type, and list every single one.** Two groups: **TRT & men's health**,
and **fertility, IVF & other**. Do not truncate to a top handful and do not
collapse the tail into a "+28 more" count — Shaffy wants the full list, so print
every deal in both groups, oldest first within each.

Type comes from `b2b_type`. It is blank on most of the pipeline, so where it is
unset, infer the group from the deal name and thread and **mark it as inferred**
(a `*` on the confirmed ones is enough) — never present a guess as a stored
value. Close with a one-line count of how many were confirmed vs inferred, so
the gap stays visible until the field is backfilled.

Owner rides on the line rather than splitting the message: mark Alex's deals
(`hubspot_owner_id` `163314964`) with `(A)`; everything else is Shaffy's
(`162479602`). Flag unassigned deals explicitly if any appear.

Also mark, per line, **who spoke last** — a `⬅` where `initial_reply_ss` is
empty or `last_touch_direction` is `Inbound`, meaning they spoke and we went
quiet. Those are the ones to pick up first.

One line per deal: name, days quiet, the markers above, and the single most
useful detail for picking it back up (what they asked, what was promised).

**Open with a "needs assigning" section**, from step 1h: every deal at
Reply (to-be-enriched) with no owner. These are freshly-arrived replies nobody
has picked up yet, so they belong at the top of the message, above the type
groups — a brand-new unassigned reply is more urgent than a deal that has been
quiet a week. Name the deal and the one-line reason it is worth someone's time
(what the lead actually said). Skip the section when the bucket is empty.

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
- **Contacts linked to a deal, and any contact created** for a previously-
  unrecorded thread participant
- **Step 2 thread count**: threads returned by the Gmail searches vs. threads
  accounted for (touched + explicitly excluded + no-matching-deal) — must match.
  Break out the excluded bucket: how many were Calendly/DocuSign notifications
  (each individually opened and attributed to a deal, per "Calendly and DocuSign
  notifications are touchpoints, not noise" above) vs. genuine administrative
  noise (internal team invites, banking, HubSpot/Google system mail). Never
  report a single unbroken "notification noise" number.
- Anything the run could not complete, named plainly

## Before you finish — spot-check all five

1. `last_touch_date NOT_HAS_PROPERTY` across Reply-to-be-enriched deals → should be empty.
2. `initial_reply_ss NOT_HAS_PROPERTY` → only deals where we genuinely have not
   replied, and every one of those belongs in the digest's reply-owed list.
3. No `initial_reply_lead` / `initial_reply_ss` value starts with a sender prefix
   (`Client:`, `SwimScore:`, or a rep's name like `Katelyn:`), and neither holds a
   placeholder like `"SwimScore replied"` instead of the literal message. Note
   there is no "starts-with" operator — a `CONTAINS_TOKEN` filter on these fields
   returns mostly false positives, so pull the values and eyeball the openings.
   Fix what you find; do not leave it for the next run.
4. No deal changed stage during this run. If one did, say so loudly — that is a bug.
5. The step 2 thread count reconciles (see "Account for every thread the
   searches return" in step 2) — Gmail threads returned equals threads touched
   plus explicitly excluded plus no-matching-deal. If it doesn't, the run is not
   done; go back and close the gap before reporting completion. Do not let a
   clean-looking digest substitute for this count actually matching — that is
   exactly what masked the 2026-08-19 gap. Within the excluded bucket, confirm
   every Calendly/DocuSign notification was individually opened and attributed
   to a deal rather than lumped into "notification noise" by sender pattern —
   that exact shortcut, on the very next run, hid an unknown number of real
   client touchpoints inside a ~57-thread bucket that was never opened.
