# Adding Slack workspaces to the daily sweep

The daily workflow reads **multiple Slack workspaces** by using one **user token**
(`xoxp-`) per workspace. A user token acts as *you*, so it can only search what
you already have access to (your DMs, mentions, and the channels you're in) —
nothing more.

> You only need to do this for workspaces **beyond** TechTower. TechTower already
> works through the connected Slack connector.

## What you can and can't add

- ✅ Workspaces where you're an **admin/owner**, or where members are allowed to
  install apps → you can mint a token yourself.
- ⚠️ Workspaces where you're a **guest/member with no install rights** → an admin
  there has to approve the app first. If they won't, there's no API access to
  that workspace and it stays a manual check.

## Mint a user token (per workspace) — ~3 minutes

1. Go to <https://api.slack.com/apps> → **Create New App** → *From scratch*.
   Name it e.g. `Open-Items Sweep`, pick the target workspace.
2. **OAuth & Permissions** → under **User Token Scopes** add: `search:read`
   (optionally `channels:history`, `groups:history`, `im:history`, `mpim:history`
   for deeper thread reads).
3. **Install to Workspace** → approve. (If you're not admin, this sends an
   approval request to a workspace admin.)
4. Copy the **User OAuth Token** — it starts with `xoxp-`.

## Wire it up

1. `cp slack-workspaces.example.json slack-workspaces.json` (gitignored — never
   committed).
2. Add one entry per workspace with its `name`, `domain`, your `userId` in that
   workspace, and the `xoxp-` `userToken`.
3. That's it — the daily run loops over every workspace listed and merges the
   results into the same Notion tracker.

## Security

- `slack-workspaces.json` is gitignored alongside `accounts.json`. Never commit
  tokens.
- Use **user** tokens (`xoxp-`), not bot tokens — they scope access to exactly
  what you can already see.
- Revoke a token anytime from the app's **OAuth & Permissions** page.
