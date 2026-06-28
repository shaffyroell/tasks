# Roadmap & related workflows

Things this sweep depends on or could grow into — tracked separately so the daily
open-items workflow stays focused.

## Done: Attio data hygiene & enrichment + stage automation
The pipeline cleanup that this sweep depended on is **done** (one-off pass):
- Backfilled deal stages from email/meeting evidence (23-status mess → clean set;
  stale → `Nurture / recontact`, explicit declines → `Lost`).
- Created genuine new inbound deals found in the last 4 months of mail.
- Linked **people → deals** and **companies → deals**, backfilled company
  `domains`, and set `value` + `type_of_client_8`.

Keeping it current is now a **daily workflow** — see
[`attio-ingest.md`](./attio-ingest.md): it reads Gmail/Slack/Granola/
Fireflies/calendar, moves deal stages on fresh evidence, and appends a dated
comms note to each deal (active clients get a running log) so **Attio is the
source of truth**. Matching still relies on links staying populated, so don't let
new records go unlinked.

## Future
- **Attio read-in:** pull open / stalled / overdue deals *out* of Attio into the
  open-items tracker as their own items (not just write-back).
- **Auto-create:** done — daily new-inbound deal creation (create + link company
  + link person + note) is its own front workflow, `new-deal-discovery.md` (W0).
  Low-confidence intros are listed for review rather than created.
- **Weekly deep re-sweep:** a wider-lookback variant of `attio-ingest.md` to
  catch deals that went quiet (vs. the daily incremental run).
- **Per-client onboarding template:** one-click Notion tracker duplication +
  guided token collection (see ONBOARDING.md for the manual version today).

## Out of scope
- **Microsoft Teams** — see `CONNECTIONS.md`. Reading client Teams needs
  per-tenant Graph apps + metered protected APIs; not pursued.
