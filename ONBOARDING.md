# Onboard a new client (plug-and-play)

The sweep is config-driven: nothing client-specific lives in `daily-open-items.md`.
Onboarding a client = create their tracker, fill one config, collect tokens,
schedule. ~20 minutes, no code edits.

> TechTower is "client zero" — `client.json` is the worked example to copy.

## 1. Create the client's Notion tracker (~3 min)
Either duplicate the master tracker, or create a fresh one from this schema
(Notion → create database → these columns):

```
Item (title) · Source (Email/Slack/Meeting) · Status (Needs Response/Follow Up/
To Do/Waiting/Done) · Priority (High/Medium/Low) · Who (text) · Action Needed
(text) · Link (url) · Ref (text) · First Seen (date) · Last Updated (date) ·
Attio (text)
```
Share it with the client's Notion integration, then copy the **data source ID**
(the `collection://<id>` UUID).

## 2. Fill the client config (~5 min)
`cp client.example.json clients/<client>.json` and set:
- `owner.primaryEmail`, `owner.timezone`
- `notion.dataSourceId` + `trackerUrl`
- `slack.primaryUserId` (their Slack member ID)
- `sources` — toggle which of email / slack / fireflies / Attio write-back apply
- `schedule.cron` + `timezone` (their local 07:00)

## 3. Collect credentials per enabled source (~10 min)
Only for the sources you toggled on:
- **Email** — connector for the main inbox, or per-mailbox tokens → `EMAIL_SETUP.md`
- **Slack** — connector for one workspace, or `xoxp-` tokens → `SLACK_SETUP.md`
- **Fireflies** — the client's API key (if used)
- **Attio** — API token if write-back is on → `ATTIO_SETUP.md`

For a **managed** deployment, store each client's creds as that client's env
secrets (`CLIENT_CONFIG_JSON`, `EMAIL_ACCOUNTS_JSON`, `SLACK_WORKSPACES_JSON`,
`ATTIO_JSON`) — never mix tenants in one file.

## 4. Schedule it (~2 min)
Follow `SCHEDULING.md` — daily at the client's 07:00, on **Sonnet**, prompt:
*"Run the daily open-items sweep in `daily-open-items.md` for the client in
CLIENT_CONFIG_JSON."*

## 5. First run + verify
Run once manually. Confirm:
- The summary's **Preflight** line shows ✅ for each enabled source.
- The client's Notion tracker populated with real open items.
- (If Attio on) a note + follow-up task landed on a matched record.

## 6. Hand off
Send the client the tracker link and a one-line "what to expect each morning."

---

### Per-tenant isolation checklist
- [ ] Separate Notion tracker per client (never shared)
- [ ] Separate credential secrets per client (no shared `accounts.json`)
- [ ] `client` name set so logs/summaries are attributable
- [ ] Client can revoke any token without affecting other tenants
