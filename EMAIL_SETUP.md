# Adding email accounts to the daily sweep

The daily workflow reads **multiple mailboxes** by looping over the accounts in
`email-accounts.json` (gitignored). Each account is read with its own
credentials and merged into the same Notion tracker.

## First, check whether you even need an entry

The connected Gmail inbox **already aggregates** several addresses. The first run
pulled mail addressed to `shaffy@techtower.ai`, `shaffy@myswimscore.com`, and
`shaffy@techtowerops.com` — all from one inbox, because they're **send-as
aliases / forwards**.

> ✅ If an address already lands in your connected inbox, it's covered — do
> nothing.
> ➕ Only add a separate entry for a **genuinely separate mailbox**: a different
> Gmail login, or an Outlook/Microsoft 365 account.

## Gmail mailbox (separate account) — one-time OAuth

1. Google Cloud Console → a project with the **Gmail API** enabled → **OAuth
   client ID** → *Desktop app*. Put its `clientId` / `clientSecret` in the
   `gmailOAuth` block of `email-accounts.json` (shared by all Gmail mailboxes).
2. On the OAuth consent screen, add the mailbox address as a **test user**.
3. Run the one-time auth flow for that mailbox (same approach as the team's
   `gtm-multi-mcp` → `npm run gmail-auth`): approve in the browser, copy the
   printed **refresh token**.
4. Add an account entry with `provider: "gmail"`, the `email`, and that
   `refreshToken`.

## Outlook / Microsoft 365 mailbox

1. Azure Portal → **App registrations** → new registration (personal + work
   accounts).
2. API permissions → Microsoft Graph **delegated**: `Mail.Read`, `offline_access`.
3. Run the team's `scripts/outlook-connect.mjs` flow (in `gtm-multi-mcp`) to
   approve and capture a **refresh token**.
4. Add an account entry with `provider: "outlook"`, the `email`, and the
   `refreshToken`.

## Wire it up

1. `cp email-accounts.example.json email-accounts.json` (gitignored).
2. Add one entry per separate mailbox.
3. The daily run sweeps every listed account, tags each item with the mailbox
   name in `Who`, and dedupes on the email thread ID.

## Security

- `email-accounts.json` is gitignored alongside `accounts.json` and
  `slack-workspaces.json`. Never commit tokens.
- Use read scopes only (`Mail.Read` / Gmail read). The sweep never sends mail.
- Revoke a mailbox anytime: Gmail → Google Account → Security → third-party
  access; Outlook → Azure app registration.
