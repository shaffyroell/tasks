---
description: Enrich the existing HubSpot pipeline with conversation context (Reply-to-be-enriched sweep, then every Gmail message from the last 24 hours, then a pipeline-wide reply-owed audit against Front, then the Shopify contact form), and post a stale-deal brief to #daily-recap split by owner. Never moves a stage. Runs several times a day.
---

Run the pipeline enrichment sweep.

**The full workflow lives in the `daily-crm` skill** (`.claude/skills/daily-crm/SKILL.md`).
Invoke it and follow it exactly — it is the single source of truth for the steps,
and it is kept in sync with `hubspot.json`. This file deliberately does not
restate it; two copies of these instructions drifted apart once already.

The guardrails, restated here so they are visible without loading anything else:

1. **Never set or change `dealstage`.** Not on a positive reply, a confirmed
   call, a signed agreement, or a flat decline. Declines get Closed Lost
   *suggested* in the note and the digest. Shaffy moves every deal by hand.
2. **Never create a deal, except from a Shopify website contact-form enquiry
   that has none.** A separate automation already creates one on every Lemlist
   reply, so a Lemlist/Front/Gmail lead with no deal is a digest line, not a gap
   to fill.
3. **Never edit an existing contact's fields.** Linking a contact to its deal is
   fine, and creating a contact or company is fine too — but only search-first,
   reuse-on-match, create-only-if-nothing-exists. A blind create risks
   duplicates. Never set or clear a deal's owner — ownership is what the Slack
   brief splits on.
4. **Every write is idempotent.** This runs several times a day; a second run an
   hour later must produce no duplicate notes and no churn.

The five steps, in order:

1. Sweep every deal at **Reply (to-be-enriched)** and fill its touch-tracking
   fields from the real conversation — Lemlist's full thread plus every matching
   Front conversation — establishing who spoke last and whether SwimScore has
   actually replied yet, and link (or create if genuinely new) every thread
   participant's contact record on the deal.
2. Read **every Gmail message from the last 24 hours for shaffy@myswimscore.com**
   — inbox *and* sent — and attach each lead-related thread to its deal as one
   full-thread note, updated in place as the thread grows, linking/creating
   participant contact records along the way. Reconcile a thread count so
   nothing goes unlogged (see the skill) — this is exactly the step that missed
   8 real deals on 2026-08-19.
3. **Reply-owed audit (the key metric):** for every OPEN pipeline deal with
   `initial_reply_ss` blank — not just Reply-to-be-enriched — check Front for a
   reply that was sent but never logged, and correct it if found. Most replies
   go out immediately, so a blank field is usually a logging gap, not a real
   one; this is what keeps the reply-owed list trustworthy. Log every correction.
4. Check the **Shopify contact form** for new B2B enquiries — the one place this
   workflow creates a deal.
5. Post the **stale brief** (no touchpoint in 4+ days) to `#daily-recap`, split
   by owner — Alex, then Shaffy — and ranked by who to contact first.

End with the digest described in the skill.
