---
name: daily-crm
description: Daily CEO sweep. Reads ALL recent email, Slack, Granola and Lemlist; classifies each meaningful item as ACCOUNT or INTERNAL; routes account items to HubSpot deals (find-or-create + link people, note, stage) and internal items to SwimScore Notion (find the item, read what it's for, update status). HubSpot is the single source of truth for the pipeline. Use as the scheduled daily run.
---

**Think like the CEO of this business**, not a note-taker. Read everything, decide
what actually matters, and make the systems reflect reality: the **pipeline lives
in HubSpot**, **internal execution lives in SwimScore Notion**. Work the steps in
order. Config: committed `hubspot.json` (+ `clients.json` if agency routing is on).

## 1. Read everything from the last few days
- **All email** — every Gmail thread with a new inbound/outbound message (native
  Gmail MCP only). Read the substance, asks, and commitments.
- **All Slack** — every relevant channel: client / Slack-Connect channels **and**
  the internal channels where the business is run. Read threads, not just top
  messages.
- **All call notes** — every Granola note in the window; these hold the real
  decisions.
- **All Lemlist replies** — `get_inbox_conversations` → `get_inbox_conversation`;
  note the `aiLeadInterest` signal (positive/neutral/negative).

## 2. Classify each meaningful item — ACCOUNT or INTERNAL
For every item that matters, decide:
- **ACCOUNT** — an external party (prospect, client, partner, clinic) — anything
  about winning, running, or growing a deal. → goes to **HubSpot** (step 3).
- **INTERNAL** — SwimScore's own operations: product, team, hiring, finance,
  roadmap, internal decisions. → goes to **SwimScore Notion** (step 4).

When an item is genuinely both (e.g. a client request that spawns internal work),
log the account side to HubSpot **and** the internal task to Notion.

## 3. ACCOUNT items → HubSpot (single source of truth)
For each account item, resolve it to a **deal** (contact email → deal; company
domain → deal; or name match):
- **Deal exists?** Run W1 (`hubspot-sync.md`): check the deal's existing notes,
  and **if this conversation isn't already captured, add a consolidated
  `[hubspot-ingest]` note** (what happened per channel, decisions/commitments,
  risks, next step). Move `dealstage` when the evidence warrants; honor
  `autoApply` + `allowStageClose` (never auto-close Won/Lost); never silently
  demote an active deal.
- **No deal yet?** Run W0 (`new-deal-discovery.md`): **create the deal, link the
  company and the people** (find-or-create contact + company, associate both),
  and add a `[new-deal]` note. Don't force-fit new inbound onto an existing deal.
- Close any gap where a real account conversation has no HubSpot note.

## 4. INTERNAL items → SwimScore Notion
Run W2 (`daily-open-items.md`) against `internalTracker` in `hubspot.json`:
- **Check if the item already exists** in SwimScore Notion (dedup on `Ref` / a
  title match). **Read what it's for**, then **update it to the latest status**
  (advance it, or mark Done if the evidence shows it's handled).
- If it's genuinely new, add it in the house style per `STYLE.md` (verb-first,
  concise, no arrows).
- If `internalTracker.dataSourceId` is still a placeholder (Notion not connected
  yet), **list the internal items in the digest** for manual handling instead of
  failing.

## 5. Report
End with a CEO digest: **Accounts** — deals reviewed, notes added (gaps closed),
stage moves (+ any Closed Won/Lost awaiting approval), new deals created and who
was linked. **Internal** — Notion items updated/advanced/closed/added (or listed
for manual handling). Call out the top risks and decisions that need the CEO.

Open with a per-source preflight line (HubSpot, Gmail, Slack, Granola, Lemlist,
Notion); stop and report if a critical source is down.
