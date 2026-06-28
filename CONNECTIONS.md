# Connections & supported tools

The sweep is **capability-based**: each company declares which provider it uses
per capability in `client.json` (`stack.*`), and the run uses whatever is
connected in *that company's* Claude workspace. One company = one config = one
Claude workspace = one tracker.

## Standard client profile (baseline assumptions)

Most clients fit this shape, so it's the default in `client.example.json`:

| Capability | Default | Notes |
|---|---|---|
| **Notion tracker** | always | Each client has a Notion page (their tracker output) |
| **Email** | always | Every client communicates via email |
| **Attio CRM** | always | Every client is in Attio — it's the canonical client/prospect list (used to recognize pipeline) **and** the write-back target |
| **Shared Slack** | some clients | Via a shared Slack Connect channel; `enabled:false` if none |
| **Meeting notes** | optional | Enable per client if they use a readable notes tool |

## Capability → provider matrix

| Capability | Providers | How it connects |
|---|---|---|
| Tracker (output) | Notion | Notion connector, or integration token |
| Email | Gmail, Outlook | Connector for the main inbox; extra mailboxes via OAuth refresh tokens (`EMAIL_SETUP.md`) |
| Chat | Slack, MS Teams, Google Chat | Connector for the primary workspace; extra Slack workspaces via `xoxp-` user tokens (`SLACK_SETUP.md`) |
| Meeting notes | Fireflies, Otter, Gemini, Granola | Provider connector / API key |
| CRM write-back | Attio, HubSpot, Pipedrive, Salesforce, Airtable | API token (`ATTIO_SETUP.md` for Attio) |

If a capability's tool isn't available in a given company's Claude, set
`stack.<capability>.enabled = false` — the preflight reports it and the run skips
it cleanly.

## Per-company model (assume each business uses Claude)

Each business runs its **own** instance: its own Claude workspace, its own
connected tools, its own `client.json`, its own Notion tracker. Onboarding a new
company = duplicate this workflow into their Claude, connect their tools, fill the
config. Nothing is shared across companies (no shared credentials, no shared
tracker).

## Accessing client Slack workspaces (long-term view)

Two different needs, don't conflate them:

1. **A client's own instance** sweeps the *client's* Slack — the client connects
   their workspace in their Claude. Not your problem to access.
2. **Your (TechTower) instance** wants client conversations where *you* are a
   participant. That's the case below.

For your own tracker to surface client chat, you need to reach those
conversations. Ranked best → worst for the long term:

- **✅ Best — Slack Connect into your own workspace.** Standardize external
  collaboration as **Slack Connect shared channels inside the TechTower Slack**.
  Then your sweep connects **one** Slack (yours), you own it, there are **zero
  per-client tokens or app approvals**, and it scales to any number of clients.
  Make "spin up a shared channel" a step in client onboarding.
  - *Limit:* only captures what flows through shared channels / Connect DMs. If a
    client @mentions you in their **internal** (non-shared) channels, Connect
    won't see it.
- **➕ Fallback — per-workspace user token.** For the few clients where you're
  genuinely **embedded as a guest in their internal workspace**, mint an `xoxp-`
  user token there (`SLACK_SETUP.md`) and add it to `slack-workspaces.json`.
  Needs their admin to approve an app, and it's one token per workspace to manage.
- **❌ Avoid — juggling many full workspace memberships** with no consolidation.
  That's the high-friction path you'd grow out of.

**Recommended default:** Slack Connect for all new client relationships (one
connection, you own it); per-workspace tokens only for embedded/guest cases.
That keeps your own instance on a single Slack connection long-term.

### Clients on Microsoft Teams

Same principle, harder mechanics. Teams has no user token like Slack's `xoxp-`;
reads go through a **Microsoft Graph app** (Entra/Azure AD registration).

- **Client-side (their own instance):** if the client is a Teams shop, they
  connect **their** Teams in **their** Claude — their tenant, their admin
  consents their own app. Easy, because it's their tenant.
- **Your instance reaching into a client's Teams:** much harder. Reading their
  Teams messages means an app registration **in their tenant** with admin consent
  (`Chat.Read` / `ChannelMessage.Read.All`), and Microsoft gates Teams message
  export behind "protected APIs" that can require approval and **per-message
  metered billing**. As an external guest you usually can't register apps there.
- **✅ Long-term analog to Slack Connect: Teams *shared channels* (Teams Connect)
  hosted in YOUR Microsoft 365 tenant.** Invite client users into a shared
  channel that lives in your tenant, so a **single Graph app in your tenant**
  (your admin consent) can read it — no per-client-tenant registrations. You
  already run a Graph app for Outlook, so adding Teams scopes to it is
  incremental.
- **Fallback:** if neither is possible, treat that client's Teams as a **manual
  check** (preflight reports `chat: manual`), rather than forcing fragile access.

**Cross-platform reality:** with some clients on Slack and some on Teams, your own
TechTower instance ends up owning **two chat hubs** — your Slack (with Connect)
and your M365/Teams (with shared channels). That's still *your* tenants and one
app each, not N client workspaces. Per-client instances stay single-provider
(whatever that client uses).
