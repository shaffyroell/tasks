# Roadmap & related workflows

Things this sweep depends on or could grow into — tracked separately so the daily
open-items workflow stays focused.

## Needed: Attio data hygiene & enrichment (separate workflow)
The daily sweep uses Attio as a *signal* for who's a client/prospect, but Attio is
currently **under-tagged**:
- Many **company domains** are missing → email senders can't be matched to companies.
- Many **deal names** are missing/inconsistent.
- **People are not linked to deals** → can't tie a contact to their pipeline.

Until this is fixed, the sweep treats an Attio match as a positive signal only
(never a gate) and does not rely on people↔deal links. A dedicated workflow
should:
- Backfill company `domains` from known contact emails / web lookup.
- Normalize and de-duplicate deal names.
- Link people → deals (and companies → deals) so pipeline is traversable.
- Flag records that still can't be auto-resolved for human review.

This is a prerequisite for two upgrades below.

## Future
- **Attio read-in:** once data is clean, pull open / stalled / overdue deals
  *out* of Attio into the tracker as their own open items (not just write-back).
- **Auto-create Attio records** for genuine new inbound not yet in the CRM
  (currently flag-only; would move to AI-create-with-review).
- **Per-client onboarding template:** one-click Notion tracker duplication +
  guided token collection (see ONBOARDING.md for the manual version today).

## Out of scope
- **Microsoft Teams** — see `CONNECTIONS.md`. Reading client Teams needs
  per-tenant Graph apps + metered protected APIs; not pursued.
