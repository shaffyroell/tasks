# W2 — Daily Internal To-Dos (SwimScore Notion)

**Run this every morning (~7:15 Europe/Amsterdam), right after W1
(`hubspot-sync.md`).** It produces the **internal SwimScore To-Dos in Notion** —
what still needs to be acted on internally (product, team, hiring, finance,
roadmap, ops), plus any owner follow-ups the account work surfaced. Target the
SwimScore Notion tracker in `hubspot.json` → `internalTracker`.

> **Notion connection:** if `internalTracker.dataSourceId` is still a placeholder
> (Notion not connected via MCP yet), **don't fail** — gather the items and list
> them in the digest for manual handling. Once the SwimScore Notion data-source id
> is set, this writes/reconciles there.

**Hybrid inputs (read in this order):**
1. **HubSpot (primary, accounts)** — the layer W1 just wrote: latest deal
   **stage** and the newest `[hubspot-ingest …]` notes. This is the authoritative
   "what's happening with whom" for the pipeline.
2. **Fresh sources** — also read **Gmail**, **Slack** (client / Connect + internal
   channels), **Granola** call notes, **Lemlist** replies, and **B2B Shopify
   website inquiries** directly, to catch internal action items and owner
   follow-ups not captured in HubSpot. (B2C patient orders are out of scope.)
3. **Existing Notion to-dos** — read the current SwimScore tracker and
   **reconcile**: **check what each item is for**, mark items **Done** when
   evidence shows they were handled, **advance** ones that moved forward, then add
   genuinely new to-dos (dedup on `Ref`).

It reconciles the SwimScore internal Notion tracker so it always reflects open
actions. (Architecture: W1 `hubspot-sync.md` writes all account comms to HubSpot
deals; this W2 reads HubSpot + fresh sources and derives the internal to-dos. See
`README.md`.)

**Client-specific values come from the client config** (`CLIENT_CONFIG_JSON` env
secret, else `client.json` / `clients/<client>.json`) — so this workflow is the
same for every client. See `ONBOARDING.md`. The values below are the **TechTower
"client zero"** instance, shown as a worked example:

**Capability-based:** the config declares which **provider** each capability uses,
and the sweep uses whatever that company has connected in *their* Claude
workspace. One company = one config = one Claude workspace = one tracker.

- **Owner:** `owner.primaryEmail` (TechTower: shaffy@techtower.ai), tz `owner.timezone`.
- **Tracker:** `stack.tracker` (TechTower: Notion, data source
  `45a1139d-c188-4a05-936d-adbea5d6715e`,
  page https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144).
- **Email:** `stack.email.provider` (gmail | outlook), scoped to
  `stack.email.domains` (TechTower: techtower.ai, techtowerops.com).
- **Chat:** `stack.chat.provider` (slack | google_chat; Teams out of scope),
  identity `stack.chat.userId` (TechTower: `U07CJK9H78A`).
- **Meetings:** `stack.meetings.provider` (fireflies | otter | gemini | granola).
- **CRM:** HubSpot (always on — the canonical deal/contact list, single source of truth).
  Used both to **recognize** which emails/chats are client/pipeline and as the
  **write-back** target (step 6).
- **Lookbacks:** `lookbackDays` (defaults 21d email / 7d chat / 7d meetings).

> **Scope to one business.** This instance is **TechTower-only**. A connected
> inbox may aggregate other brands (e.g. SwimScore mail forwards in) — ignore
> anything outside `stack.email.domains`. Other businesses run as their **own**
> duplicated workflow in a separate Claude account/tracker.

- **Schema:** `Item` (title), `Source` (Email/Slack/Meeting), `Status`
  (Needs Response / Follow Up / To Do / Waiting / Done), `Priority`
  (High/Medium/Low), `Who`, `Action Needed`, `Link`, `Ref`, `First Seen`,
  `Last Updated`, `HubSpot` (the linked deal, if any),
  `Handled By` (AI / AI + Review / Human), `AI Can Do`, `Needs from You`.

---

## What counts as an open item (inclusion criteria)

Only add things that genuinely need Shaffy's input or action:

1. **Account follow-ups owed by the owner** — a real person (prospect, client, or
   partner) is waiting on a reply, or the owner owes a follow-up. **Use HubSpot as
   the signal for who counts:** clients/prospects live in HubSpot deals, so a
   counterparty matching a HubSpot contact/company is a strong pipeline signal.
   Treat a HubSpot match as a **positive signal, not a gate** — never drop an item
   just because it isn't matched; fall back to email-domain + conversational cues.
   Genuine new inbound with no deal yet is handed to **W0** to create the deal.
2. **Internal SwimScore items** — product, team, hiring, finance, roadmap, ops:
   anything internal that needs action, routed to the SwimScore Notion board under
   the right pillar (see the Kanban pillars in `SWIMSCORE_NOTION.md`).
3. **Slack messages the owner should weigh in on** — @mentions, DMs, or threads
   where a question is open and unanswered.
4. **Meeting next-steps** — action items committed to in recent calls (Granola
   notes; Lemlist replies that imply an owner action).

**Exclude:** newsletters, promotions, automated/no-reply mail, calendar
accept/decline notifications, n8n/Make/workflow error alerts, system notices,
and anything already handled (the owner replied and nothing is outstanding).

> **HubSpot is the account signal**; the **SwimScore Notion board** is where
> internal to-dos live. Account notes + stages are W1's job in HubSpot; W2 only
> reconciles the internal Notion board. Load the relevant HubSpot deals early so
> email/chat counterparties can be matched — best-effort, never a hard gate.

---

## Procedure

### 0. Preflight — verify everything's in place
Before gathering, check each dependency and build a readiness list. **Degrade
gracefully**: if a source isn't ready, skip just that source and flag it in the
summary — never abort the whole run.

Check and record ✅ / ⚠️ for each:
1. **Notion tracker** reachable (data source `45a1139d-c188-4a05-936d-adbea5d6715e`
   responds). If not → stop and report (there's nowhere to write).
2. **Gmail** — connected inbox responds. For each extra account in the email
   config (`EMAIL_ACCOUNTS_JSON` env secret, else `email-accounts.json`):
   confirm it has a non-placeholder `refreshToken` and that a 1-message test
   fetch succeeds. List which accounts loaded and which failed/expired.
3. **Slack** — connected workspace responds. For each workspace in
   `SLACK_WORKSPACES_JSON` / `slack-workspaces.json`: confirm a non-placeholder
   token and a test `search` call succeeds.
4. **Granola** — recent meeting-notes call responds. **Lemlist** —
   `get_campaigns` responds.
5. **HubSpot** (account signal) — `get_user_details` responds. **Notion**
   (internal tracker) — only if `internalTracker.dataSourceId` is set (not a
   placeholder); if it's a placeholder, note "Notion: not connected — internal
   items listed for manual handling" (not an error).

Open the morning summary with one readiness line, e.g.:
`Preflight: HubSpot ✅ · Gmail ✅ · Slack ✅ · Granola ✅ · Lemlist ✅ · Notion ⚠️ not connected`.
A source marked ⚠️ is simply not swept this run — say so explicitly so a missing
or expired credential surfaces loudly instead of silently dropping coverage.

### 1. Load current tracker state
Query the data source (`45a1139d-c188-4a05-936d-adbea5d6715e`) for all rows that
are **not** `Done`. Build a lookup by `Ref` so you can update instead of
duplicate. `Ref` formats:
- Email → `gmail:<threadId>` (or `outlook:<id>`)
- Chat → `chat:<provider>:<workspace>/<channel>/<ts>`
- Meeting → `ff:<transcriptId>` (or `<provider>:<id>`)

### 2. Gather email — provider per `stack.email.provider` (last ~21 days)
Use the company's email provider (Gmail or Outlook) and every mailbox it can
reach: the connected inbox plus any account in the email config
(`EMAIL_ACCOUNTS_JSON` env secret, else `email-accounts.json`). See
`EMAIL_SETUP.md`.

> **Domain scope (important for shared inboxes):** only consider threads to/from
> `stack.email.domains`. If the connected inbox also receives other brands (e.g.
> SwimScore), those are **out of scope for this instance** — they belong to their
> own duplicated workflow. If `domains` is empty, include everything.

> **Reply-detection needs Sent mail.** The connected inbox *receives* alias mail
> (`myswimscore`, `techtowerops`) but does **not** contain replies sent from
> those separate accounts. Any address Shaffy sends replies from must be its own
> connected account, or threads he already answered there will be wrongly flagged
> as open. When in doubt for a thread whose last visible message is inbound but
> belongs to an unconnected account, mark it `Follow Up` (Low), not
> `Needs Response`.

In each mailbox search:
`in:inbox newer_than:21d -category:promotions -category:social -category:updates -category:forums`
For each business thread, look at the **last message**:
- Last message is **from someone else** and it asks/expects something → `Needs Response`.
- Last message is **from Shaffy** but it's an open deal/proposal/quote → `Follow Up` (or `Waiting`).
- Thread is purely informational, resolved, or automated → skip.
Capture sender/company in `Who` (prefix with the mailbox name when it's not the
default, e.g. `personal-gmail · John D.`), a one-line `Action Needed`, and
`Link = https://mail.google.com/mail/u/0/#all/<threadId>`.

> Email thread IDs are globally unique, so `Ref = gmail:<threadId>` stays stable
> across mailboxes (no need to qualify by account).

### 3. Gather chat — provider per `stack.chat.provider` (last ~7 days)
Use the company's chat tool. Items here get `Source = Chat`.
- **Slack** — cover every workspace it can reach: the connected Slack connector
  (user = `stack.chat.userId`; TechTower `U07CJK9H78A`), plus any workspace in
  `slack-workspaces.json` / `SLACK_WORKSPACES_JSON`, each via its own `xoxp-`
  user token (`search:read`). See `SLACK_SETUP.md`.
- **Google Chat** — use that connector instead. (Microsoft Teams is out of scope.)

For each workspace/space, search the user's mentions and DMs. Read enough thread
context to tell if it's still open; if the owner already answered or someone else
resolved it, skip. Otherwise add a row, prefix `Who` with the workspace name
(e.g. `Crewline · @jane`), and set `Link =` the message permalink. Use a
provider-qualified `Ref` so the same message never collides across
workspaces/tools: `chat:<provider>:<workspace>/<channel>/<ts>`.

> A user token only sees what that user can already access (their DMs, mentions,
> and member channels). Workspaces where no token can be minted (no admin
> approval) are out of automated scope — note them as a manual check, don't fail.

### 4. Gather meeting next-steps — provider per `stack.meetings.provider` (last ~7 days)
List recent transcripts from the company's meeting-notes tool (Fireflies, Otter,
Gemini, Granola, …). For each, pull the action items and keep only those assigned
to the **owner** (or "owner + team"). Consolidate per meeting into one row where
sensible. For Fireflies, `Link = https://app.fireflies.ai/view/<transcriptId>`.

### 5. Reconcile the tracker
- **New** (Ref not present) → create a row. Set `First Seen` and `Last Updated`
  to today.
- **Existing & still open** → update `Status` / `Action Needed` if it changed,
  set `Last Updated` to today.
- **Resolved** (Shaffy has since replied, or the meeting action is done) →
  set `Status = Done`, `Last Updated` today. Do not delete.
- Keep `First Seen` unchanged on updates.

**Dev/product items resolve conversationally — go looking, don't wait to notice.**
Dev completions (`clinic-portal-dev`, `wellness-portal-dev`, `daily-status-report`)
get reported as a reply buried in a thread or a line in a daily "Work Done" post,
not as an explicit "closing to-do X." For every currently-open **Clinic & patient
portal** / **Support** item, actively search the relevant dev channel(s) for a
resolution signal referencing that specific task before carrying it forward as
still open — the same discipline as the email-recency check in `hubspot-sync.md`
§3a: don't conclude an item is still open just because you didn't happen to notice
it get closed.

### 5b. Triage each item — what AI can do vs what needs the human
For every open (non-`Done`) item, set:
- **`Handled By`** — `AI` (AI can complete it end-to-end, low-risk), `AI + Review`
  (AI prepares/drafts; the owner approves before anything external goes out), or
  `Human` (judgment, relationship, signature, or access only the owner has).
- **`AI Can Do`** — the concrete slice AI can execute now (draft this reply,
  compile these materials, build this draft, log to CRM).
- **`Needs from You`** — the specific decision / approval / access only the owner
  can give.

Default client-facing sends and commercial decisions to `AI + Review`. Promote an
item to `AI` only for a class of action the owner has pre-approved.

**Optional execution pass** (only if `execution.enabled` in the client config):
after triage, AI acts on the AI-doable slices — create Gmail **drafts** (never
auto-send unless `execution.autoSend` is true **and** the item is
`Handled By = AI`), draft docs, or stage CRM updates — and records what it
prepared in `AI Can Do` / the page body. It never flips an item to `Done` on the
owner's behalf. This is what lets the owner open the tracker and **only
review/approve**, while AI clears the prep.

### 5c. Route each to-do to its destination (agency mode)
If an agency registry is present (`clients.json` / `CLIENTS_JSON`), write each
open to-do to the right Notion destination as well as the master tracker:

1. **Classify** the item to a client by matching the counterparty's **email
   domain** or a **name alias** in the registry.
2. **Route:**
   - Matches a `type:client` entry → write to that client's **dashboard page**
     (`dashboardPageId`).
   - Matches a `type:pipeline` entry, or is internal/hiring/ops, or matches no
     client → write to the **TechTower internal board** (`internalBoard`).
3. **Write target:**
   - **Title field differs by board** — write the to-do text into the target's
     `titleField` from the registry: **`Task`** on the client v2 dashboards,
     **`Item`** on the master/internal tracker. Using the wrong field name fails
     the create.
   - **Client dashboards (v2 template) →** write rows into that client's
     **"✅ To Dos" database** (on its "6. To Dos" sub-page;
     `todoDataSourceId` in the registry). Set the title field, `For`
     (`Needed from client` | `TechTower`), `Status`, `Source`, `Link`, and a
     stable `Ref` for dedup. **Write every to-do in the house style — load and
     follow [`STYLE.md`](./STYLE.md)** (verb-first, concise, specific,
     client-safe: no pricing / internal commercials / other-client references;
     **never use arrows**). Upsert on `Ref` so re-runs never duplicate.
   - **Internal board →** the TechTower internal "Daily To-Do List" / managed
     section, with full internal detail (`Needs from You` / `AI Can Do`).
   - **Completion = set `Status` to Done — never delete.** The Notion connector
     has no archive/trash capability, so reconciliation only updates `Status`
     (and advances items); it must not rely on deleting rows.
4. If a client has no dashboard page yet, flag it (don't fail); a page can be
   created from the client-dashboard template.

> Single-tenant client instances skip this — they just use their own tracker.
> This routing is the **agency** feature for TechTower fanning out across many
> client dashboards.

### 6. Account follow-ups live in HubSpot (handled by W1)
Account-side activity — notes on the deal and stage moves — is written by **W1
(`hubspot-sync.md`)**, since HubSpot is the single source of truth. W2 does **not**
re-write the CRM; for any account item it surfaces here, point the tracker row's
`Link` / context at the matched HubSpot deal and let W1 own the note + stage. If an
account item has no deal yet, hand it to **W0 (`new-deal-discovery.md`)** rather
than creating records from W2. W2's own writes are limited to the **internal
SwimScore Notion** tracker.

### 6b. Stale-deal follow-ups → SwimScore Notion (New sales pillar)
W1 computes each open deal's **last contact date** and hands W2 the deals that have
gone quiet (per `hubspot.json.staleFollowUp`: >14d flag, >21d high priority; skip
Closed Lost, converted-client stages (First order placed onward), and dead leads).
**Don't take a "stale" hand-off on faith — before
upserting or re-affirming a stale To-Do, do the undated `from:`/`to:` contact search
described in `hubspot-sync.md` §3a/§7 yourself.** A deal flagged stale on a prior run
can have gone active since; carrying an old flag forward without re-checking is how
a live negotiation gets reported as abandoned. For each flagged deal (verified),
**upsert a To-Do on the SwimScore Notion board** under **New sales** (dedup on
`Ref = hubspot:stale:<dealId>`):
- **Task** (house style): `Follow up with <clinic> — quiet <N>d (<contact>)`.
- **Owner** Shaffy · **Priority** High if >21d else Medium · **Link** the HubSpot deal.
- **Suggested message** (if `draftSuggestedMessage` and a nudge actually fits): drop a
  short, specific draft into the page body, written from the deal's HubSpot context
  (who they are, last touch + date, their open question / pain point, the agreed next
  step). Keep it ready-to-send and personal — not a generic "just checking in." If no
  genuine message fits, add the task without a draft (flag only). Mark the To-Do
  **Done** once W1 sees a fresh contact on that deal.

### 7. Report
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
