# Roadmap & related workflows

Things this sweep depends on or could grow into — tracked separately so the daily
workflow stays focused.

## Done: HubSpot pipeline backfill + stage automation
The one-off pipeline cleanup is **done**:
- Backfilled the genuine B2B clinic leads from the full Lemlist team inbox into
  HubSpot as deals (deal named after the clinic + linked contact + company + domain),
  each with a conversation note.
- Enriched every deal note from Shaffy's Gmail threads (source of truth after
  Lemlist) + Slack + Granola: Background → timestamped Timeline → Latest → Next step.
- Set stages from real evidence (call held → Discovery Completed, proposal sent →
  Proposal Sent, onboarding booked → Pilot Discussion, etc.).

Keeping it current is now a **daily workflow** — see [`hubspot-sync.md`](./hubspot-sync.md):
it checks Lemlist, Shopify contact-form inbound, Gmail, Slack, and Granola, moves
deal stages on fresh evidence, and appends a dated note to each deal so **HubSpot is
the source of truth**.

## Future
- **Stale read-in:** the daily run already flags deals with no contact in 14+ days
  and drafts a follow-up to the SwimScore Notion board (New sales pillar). Could
  extend to a weekly digest of all open/stalled deals.
- **Auto-create:** done — daily new-inbound deal creation (create + link company +
  contact + note) is its own front workflow, `new-deal-discovery.md` (W0).
  Low-confidence intros are listed for review rather than created.
- **Closed-lost hygiene:** auto-propose Closed Lost for cold-list hard rejections
  (currently left out of the pipeline rather than created).
- **Onboarding template:** Mirva, Epoch Health, and Epic Fertility are converging on
  Jul 10 onboarding — a one-click onboarding checklist (sample kit + portal setup +
  one-pager) per new clinic would help.

## Out of scope
- **B2C Shopify orders/customers** — individual patient purchases are not deals; the
  sweep only reads B2B website contact-form messages.
- **Microsoft Teams** — see `CONNECTIONS.md`. Not pursued.
