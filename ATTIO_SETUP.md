# Attio write-back setup

The daily sweep can **write follow-up activity back to Attio** (TechTower
workspace): when it surfaces an open item tied to someone in your pipeline, it
logs a note and makes sure a follow-up task exists on the matching record. Notion
stays your to-do list; Attio gets the CRM trail.

> **Direction:** Tracker → Attio (write). The sweep does not pull deals *out* of
> Attio into the tracker — that's a separate mode you can enable later.

## What it writes (and what it won't)

| Action | Default | Notes |
|---|---|---|
| Add a **note** to the matched record | ✅ on | Summarizes the latest interaction + the open action. Tagged with the item `Ref` so it's never duplicated. |
| Ensure a **follow-up task** (assigned to you, due in N days) | ✅ on | Idempotent on `Ref`. |
| Set a custom **Next step / Last contacted** attribute | ⬜ only if you map it | Leave blank in `attio.json` to skip. |
| **Advance / change deal stage** | ⛔ off | Requires explicit approval during the run. Never auto-closes or auto-wins a deal. |
| Delete anything | ⛔ never | The sweep is strictly additive. |

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

## How matching works

For each open item, the sweep resolves **who** it's about (email address →
`people.email_addresses`; company domain → `companies.domains`) and finds the
record via `POST /v2/objects/{object}/records/query`. If no record matches, it
does nothing in Attio and notes `not in Attio` on the tracker row. It never
creates new companies/people from this sweep.

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
