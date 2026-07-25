---
description: Daily key-account sweep — read all email/Slack/calls, log to the right Attio deals (if not already noted), keep stages honest, reconcile Notion action items
---
Act like the person who manages these key accounts. Work these steps in order:

1. **Read everything (last few days):** all Gmail threads (native MCP), all
   relevant Slack channels (client + internal), and all Granola/Fireflies call
   notes — the substance, decisions, and commitments, not just headlines.

2. **W0 first** (`new-deal-discovery.md`): for genuinely NEW opportunities not yet
   in Attio, create the deal + link company + person + add a `[new-deal]` note.

3. **W1 — link comms to deals** (`attio-ingest.md`): for every meaningful
   conversation — **including ones landing on a `Nurture / recontact` or
   advanced/frozen deal, not just fresh prospects** — find its Attio deal,
   **check the deal's existing notes**, and add a `[attio-ingest]` note **only
   if it isn't already there** (what happened, decisions/commitments, risks,
   next step). Refresh the deal's native `deal_description` + `next_step`
   fields on every deal touched, any stage (≤2 sentences each — no exclusion);
   find-or-create + link its Company/Person if missing (the one exception to
   never creating records); create an Attio Task when something is clearly
   owed on our side (high bar, deduped on its `Ref:` line, completed once
   resolved). Keep the stage honest per the live/frozen split: `1.  Reply`
   through `3. Demo planned` plus `Nurture / recontact` move per evidence
   (including reactivating a Nurture deal on genuine new activity, and
   reclassifying a plain opt-out to `Lost - not interested` as hygiene);
   `4. Scope agreed` onward is frozen — never auto-moved, Shaffy handles those
   by hand; never silently demote an active client. Close any gap where a real
   comm has no note.

4. **W2 — reconcile Notion action items** (`daily-open-items.md`): per client, see
   if the to-dos already exist — mark Done/advance, dedup on `Ref` — and add new
   ones in the house style per `STYLE.md` (verb-first, concise, no arrows).

End with a digest: accounts touched, notes added (gaps closed), stage moves, new
deals, and Notion to-dos added/updated/closed. Think critically per account
throughout (progressing, stalling, at risk, upsell/renewal). Config: committed
`attio.json` + `clients.json`.
