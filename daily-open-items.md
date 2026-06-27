# Daily Open-Items Sweep

**Run this every morning (~7:00 Europe/Amsterdam).** It scans Gmail, Slack, and
recent meeting notes (Fireflies), then reconciles the Notion tracker so it always
reflects what Shaffy still needs to respond to or act on.

- **Owner:** Shaffy (shaffy@techtower.ai)
- **Destination:** Notion database **📥 Open Items — Daily Tracker**
  - Page: https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144
  - Data source ID: `45a1139d-c188-4a05-936d-adbea5d6715e`
- **Schema:** `Item` (title), `Source` (Email/Slack/Meeting), `Status`
  (Needs Response / Follow Up / To Do / Waiting / Done), `Priority`
  (High/Medium/Low), `Who`, `Action Needed`, `Link`, `Ref`, `First Seen`,
  `Last Updated`.

---

## What counts as an open item (inclusion criteria)

Only add things that genuinely need Shaffy's input or action:

1. **Pipeline / client emails** — a real person (prospect, client, or partner)
   is waiting on Shaffy to reply, or Shaffy owes a follow-up. This includes
   threads where Shaffy sent the last message but the deal needs a nudge
   (→ `Follow Up` / `Waiting`).
2. **Slack messages Shaffy should weigh in on** — @mentions, DMs, or threads
   where a question is open and Shaffy hasn't answered.
3. **Meeting next-steps** — action items Shaffy committed to in recent calls
   (Fireflies `action_items` assigned to Shaffy / "Shaffy and team").

**Exclude:** newsletters, promotions, automated/no-reply mail, calendar
accept/decline notifications, n8n/Make/workflow error alerts, system notices,
and anything already handled (Shaffy replied and nothing is outstanding).

> Pipeline source-of-truth also lives in **Attio** (techtower + crewline
> workspaces) and Lemlist. This session uses the hosted Gmail/Slack/Notion/
> Fireflies connectors only. If an Attio connector is added later, cross-check
> open deals there too.

---

## Procedure

### 1. Load current tracker state
Query the data source (`45a1139d-c188-4a05-936d-adbea5d6715e`) for all rows that
are **not** `Done`. Build a lookup by `Ref` so you can update instead of
duplicate. `Ref` formats:
- Email → `gmail:<threadId>`
- Slack → `slack:<workspaceName>:<channel>/<ts>`
- Meeting → `ff:<transcriptId>`

### 2. Gather Gmail (last ~21 days)
Search: `in:inbox newer_than:21d -category:promotions -category:social -category:updates -category:forums`
(across the connected mailboxes). For each business thread, look at the **last
message**:
- Last message is **from someone else** and it asks/expects something → `Needs Response`.
- Last message is **from Shaffy** but it's an open deal/proposal/quote → `Follow Up` (or `Waiting`).
- Thread is purely informational, resolved, or automated → skip.
Capture sender/company in `Who`, a one-line `Action Needed`, and
`Link = https://mail.google.com/mail/u/0/#all/<threadId>`.

### 3. Gather Slack — all configured workspaces (last ~7 days)
The sweep covers **every** Slack workspace it can reach:
- **TechTower** via the connected Slack connector (user `U07CJK9H78A`).
- **Any workspace listed in `slack-workspaces.json`** (gitignored), each via its
  own `xoxp-` user token with `search:read`. See `SLACK_SETUP.md`.

For each workspace, search the user's mentions (`<@userId>`) and DMs. Read enough
thread context to tell if it's still open. If Shaffy already answered or someone
else resolved it, skip. Otherwise add a row, prefix `Who` with the workspace name
(e.g. `Crewline · @jane`), and set `Link =` the message permalink. Use a
workspace-qualified `Ref` so the same message in different workspaces never
collides: `slack:<workspaceName>:<channel>/<ts>`.

> A user token only sees what that user can already access (their DMs, mentions,
> and member channels). Workspaces where no token can be minted (no admin
> approval) are out of automated scope — note them as a manual check, don't fail.

### 4. Gather meeting next-steps (last ~7 days)
List recent Fireflies transcripts (`mine: true`). For each, pull `action_items`
and keep only those assigned to **Shaffy** (or "Shaffy and team"). Consolidate
per meeting into one row where sensible. `Link = https://app.fireflies.ai/view/<transcriptId>`.

### 5. Reconcile the tracker
- **New** (Ref not present) → create a row. Set `First Seen` and `Last Updated`
  to today.
- **Existing & still open** → update `Status` / `Action Needed` if it changed,
  set `Last Updated` to today.
- **Resolved** (Shaffy has since replied, or the meeting action is done) →
  set `Status = Done`, `Last Updated` today. Do not delete.
- Keep `First Seen` unchanged on updates.

### 6. Report
Post a short summary to Shaffy: counts by Source and Status, and call out the
top 3 `High` / `Needs Response` items. Keep it tight.

---

## Notes
- Dedup is driven entirely by `Ref`. Never create a second row for the same
  thread/mention/meeting.
- Be conservative: when unsure whether something needs a response, lean toward
  including it as `Follow Up` (Low) rather than spamming `Needs Response`.
- Credentials (`accounts.json` etc.) are never read or committed by this
  workflow.
