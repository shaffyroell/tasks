# Adding email accounts to the daily sweep

The daily workflow reads **multiple mailboxes** by looping over the accounts in
`email-accounts.json` (gitignored). Each account is read with its own
credentials and merged into the same Notion tracker.

## Why every account you SEND from needs its own entry

The connected inbox **receives** mail for several addresses (`myswimscore`,
`techtowerops` forward into it). But receiving is only half the picture: the
sweep decides "does this need a response?" by checking whether **the last message
in the thread is from you**. To know that, it has to see your **Sent** mail.

Mail you send **from a separate account** (e.g. replying as
`shaffy@myswimscore.com` from that login) **does not** land in the connected
inbox's Sent folder. So without connecting that account, the sweep can't see your
reply and will keep flagging answered threads as open.

> ✅ **Connect every mailbox you actually send replies from** — that's what makes
> reply-detection correct. `shaffy@myswimscore.com` needs its own entry for this
> reason.
> ➖ The one exception: an address configured as a true Gmail **"Send as" alias
> inside the connected account** — those sent messages do show up here, so no
> separate entry is needed.

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

## Where credentials live at run time

The config is read from, in order:

1. **`EMAIL_ACCOUNTS_JSON` environment secret** (preferred) — the entire contents
   of `email-accounts.json` as one value.
2. **`email-accounts.json`** file on disk (local/manual runs).

> ⚠️ **For the scheduled web run this matters:** each scheduled session clones the
> repo **fresh**, and `email-accounts.json` is gitignored — so it won't be present.
> Put the JSON into the environment's secret store as **`EMAIL_ACCOUNTS_JSON`**
> (Claude Code on the web → environment settings → secrets / env vars). The file
> on disk is only for local testing.

The same pattern applies to the other sources: `SLACK_WORKSPACES_JSON` and
`ATTIO_JSON`.

## Security

- `email-accounts.json` is gitignored alongside `accounts.json` and
  `slack-workspaces.json`. Never commit tokens.
- Use read scopes only (`Mail.Read` / Gmail read). The sweep never sends mail.
- Revoke a mailbox anytime: Gmail → Google Account → Security → third-party
  access; Outlook → Azure app registration.
