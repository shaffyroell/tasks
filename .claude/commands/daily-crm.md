---
description: Daily SwimScore CEO sweep — Lemlist + email + Slack + calls → update HubSpot deals, flag stale deals with a drafted follow-up, and update the SwimScore Notion board
---
Run the full daily sweep. HubSpot is the single source of truth for the pipeline;
the SwimScore Notion board holds internal + follow-up to-dos. Config: committed
`hubspot.json`. Work the steps in order; open with a per-source preflight line
(HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion).

1. **Lemlist — new & updated conversations.** Pull the full team inbox
   (`get_inbox_conversations listId=teamConversations`, paginated; read threads via
   `get_inbox_conversation`, note `aiLeadInterest`). New genuine B2B interest with no
   deal → create it (W0 `new-deal-discovery.md`: deal named after the clinic + linked
   contact + company + domain). Existing deal with new content → update it (step 5).
   Negative replies are not deals.

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
   note (Background → timestamped Timeline → Latest → Next step) and advance the stage
   on clear evidence. Honor autoApply; never auto-close Won/Lost (list for approval);
   never silently demote an active deal.

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
   reconcile internal to-dos across the six pillars (Onboarding, New sales, Research,
   Clinic & patient portal, Support, Finance) from business-strategy, product/portal,
   legal, finance, and support signal — check what each item is for, mark Done /
   advance, dedup on `Ref`, add new ones in house style per `STYLE.md`.

End with a digest: deals updated (notes + stage moves, any Closed awaiting approval),
new deals created, **stale deals flagged (with/without drafted message)**, and Notion
to-dos added/advanced/closed per pillar. Call out the top risks and decisions for the CEO.

9. **Post the short recap to `#daily-recap` in Slack** — *Short-term goals* (by
   category: Pipeline, Website conversion, Ads, Product/Onboarding, Support, ...)
   and *To dos (specific)*, plus a link to the Notion board. **Filter the to-dos by
   business judgment, not by mechanically listing every open/High-priority Notion
   item** — the full board is one click away via the link, so this list should only
   carry what actually moves the needle this week: revenue/pipeline decisions,
   live customer-facing issues, and anything genuinely blocked or needing Shaffy's
   input. Two defaults that follow from this:
   - **Don't list every stale deal individually** — name the highest-value one(s),
     batch the rest as a cleanup pass.
   - **Suppress routine dev-team-execution items** (most of Clinic & patient
     portal / Support) — Dmytro and Harsh close these out themselves in their own
     channels without needing Shaffy's attention. Only surface a dev item if it's
     (a) blocked or stalled past a normal turnaround, (b) needs a decision only
     Shaffy can make, or (c) is a live patient/customer-facing issue right now
     (e.g. a data bug affecting current users) — not just "high priority" in
     Notion. Everything else stays tracked in Notion, just not in the Slack recap.
