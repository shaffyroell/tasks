# Attio setup — primary CRM log

Attio is the **primary CRM destination** for the daily sweep (TechTower
workspace). When the sweep surfaces an open item tied to someone in your pipeline,
it ensures that **person exists** in Attio, that they're **linked to the right
deal**, and that the **activity trail** (note + follow-up task) is on the record.
Notion stays your human-facing to-do view; Attio is the system of record.

> **Direction:** Tracker → Attio (write). The sweep does not pull deals *out* of
> Attio into the tracker — that's a separate mode you can enable later.

## What it writes (and what it won't)

| Action | Default | Notes |
|---|---|---|
| **Create a person** if no email match exists | ✅ on (`ensurePersonExists`) | Name + email (+ company domain if known). Re-queries by email first so it never duplicates. Set false for legacy additive-only mode. |
| **Link a person to the right deal** if not already linked | ✅ on (`linkPeopleToDeals`) | Resolves the deal via the person's existing deal, else the matching company's open deal. Ambiguous/none → flags for review, never guesses. |
| Add a **note** to the deal/person | ✅ on | Summarizes the latest interaction + the open action. Tagged with the item `Ref` so it's never duplicated. |
| Ensure a **follow-up task** (assigned to you, due in N days) | ✅ on | Idempotent on `Ref`. |
| Set a custom **Next step / Last contacted** attribute | ⬜ only if you map it | Leave blank in `attio.json` to skip. |
| **Create a deal** for a new pipeline conversation | ✅ on (`createMissingDeals`) | Only when email (primary) + invites sent / follow-ups show real pipeline. **One deal per company, deduped by domain** — never a second. New deal gets a stage from `stageRules`. Weak signal → `no deal (review)`. |
| **Create a company** (to back a new deal) | ✅ on (`createMissingCompanies`) | Matched/deduped by domain first; created with name + domain only when needed for a new deal. |
| **Advance / change an _existing_ deal's stage** | ⛔ off | Requires explicit approval during the run. Setting the stage on a brand-new deal is allowed; advancing/closing/winning an existing one is not. Never auto-wins. |
| Delete anything | ⛔ never | The sweep never deletes. |

## Get an Attio API token (~2 min)

1. Attio → **Workspace settings → Developers → Access tokens → Create**
   (or build an OAuth integration if you prefer).
2. Grant scopes: `record:read-write`, `object_configuration:read`,
   `task:read-write`, `note:read-write`, `user_management:read`.
3. Copy the token into `attio.json` → `workspaces.techtower.apiKey`.

## Map your objects & attributes

Slugs differ per workspace, so confirm them before the first write:

- `GET https://api.attio.com/v2/objects` → confirm the object slugs
  (`deals`, `people`, `companies`).
- `GET https://api.attio.com/v2/objects/{object}/attributes` → find the slugs for
  any **Next step** / **Last contacted** / **Stage** attributes you want set, and
  put them in the `writeBack` block. Leave blank to skip.

## How matching & linking works

For each open item, the sweep resolves **who** it's about and reconciles them
against Attio via `POST /v2/objects/{object}/records/query`:

1. **Person** — match on email (`people.email_addresses`). No match +
   `ensurePersonExists` → create the person (re-querying first to avoid a
   duplicate). With `ensurePersonExists:false` it stays additive and writes
   `not in Attio` on the tracker row instead.
2. **Deal — one per company, keyed by domain.** The deal belongs to the company
   (matched on email domain → `companies.domains`), and there's at most one per
   company. Resolve in order: the deal already on that company → else a deal the
   person is already on → else none yet.
   - **Found** → if the person isn't linked, **add the person↔deal link**.
   - **None, but a new pipeline conversation** (email primary; corroborated by a
     calendar invite you *sent* and/or follow-up emails) → **create one deal** on
     the company (ensuring the company by domain first), set its **stage** from
     `stageRules`, and link the people.
   - **None, weak/ambiguous signal**, or **several plausible deals** → write
     nothing and flag `no deal (review)` / `multiple deals (review)` — never guess.
3. **Activity** — note + follow-up task land on the resolved/created deal
   (preferred) / person, idempotent on the item `Ref`.

This repairs the "people not linked to deals" gap noted in `ROADMAP.md` and opens
new inbound as deals at the right stage. It still **never** creates a second deal
for a company, never advances an existing deal's stage without approval, and never
deletes.

## Wire it up

1. `cp attio.example.json attio.json` (gitignored).
2. Paste the token; map any optional attributes.
3. The daily run writes back per the rules above and records what it did in the
   tracker's **Attio** column (e.g. `Note + task on "AgroCares" deal`).

## Security

- `attio.json` is gitignored alongside the other credential files. Never commit
  the token.
- Use the narrowest scopes that work. Rotate/revoke from Workspace settings →
  Developers anytime.
