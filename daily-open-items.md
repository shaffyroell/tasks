# W2 — Daily Open-Items / Per-Client To-Dos (Notion)

**Run this every morning (~7:15 Europe/Amsterdam), right after W1
(`attio-ingest.md`).** It produces the **per-client To-Dos in Notion** — what
Shaffy still needs to respond to or act on, routed to each client's board.

**Hybrid inputs (read in this order):**
1. **Attio (primary)** — the raw layer W1 just wrote: latest deal **stage**, the
   newest `[attio-ingest …]` comms notes, and any open follow-up tasks. This is
   the authoritative "what's happening with whom."
2. **Fresh sources** — also read **Gmail**, **Slack** (client / Connect
   channels), and **meeting notes (Granola + Fireflies)** directly, to catch
   action items not yet (or only thinly) captured in Attio.
3. **Existing Notion to-dos** — read the current tracker and **reconcile**: mark
   items **Done** when evidence shows they were handled, **advance** ones that
   moved forward, then add genuinely new to-dos (dedup on `Ref`).

It reconciles the **per-client** Notion tracker so it always reflects open
actions. (Architecture: W1 `attio-ingest.md` writes all raw comms to Attio; this
W2 reads Attio + fresh sources and derives the to-dos. See `README.md`.)

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
- **CRM:** `stack.crm` (attio, always on — the canonical client/prospect list).
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
  `Last Updated`, `Attio` (what was written back to the CRM, if anything),
  `Handled By` (AI / AI + Review / Human), `AI Can Do`, `Needs from You`.

---

## What counts as an open item (inclusion criteria)

Only add things that genuinely need Shaffy's input or action:

1. **Pipeline / client emails** — a real person (prospect, client, or partner)
   is waiting on the owner to reply, or the owner owes a follow-up. This includes
   threads where the owner sent the last message but the deal needs a nudge
   (→ `Follow Up` / `Waiting`). **Use Attio as a signal for who counts:**
   most clients/prospects live in Attio (`stack.crm`), so a counterparty matching
   an Attio person/company is a strong pipeline signal.
   ⚠️ **Attio data is currently incomplete** (missing company domains & deal
   names, people not linked to deals — a separate cleanup workflow, see
   `ROADMAP.md`). So treat an Attio match as a **positive signal, not a gate**:
   never drop an item just because it isn't matched, don't rely on people↔deal
   links, and fall back to email-domain + conversational cues. Genuine new inbound
   not yet in Attio still counts — flag it to be added.
2. **Slack messages Shaffy should weigh in on** — @mentions, DMs, or threads
   where a question is open and Shaffy hasn't answered.
3. **Meeting next-steps** — action items Shaffy committed to in recent calls
   (Fireflies `action_items` assigned to Shaffy / "Shaffy and team").

**Exclude:** newsletters, promotions, automated/no-reply mail, calendar
accept/decline notifications, n8n/Make/workflow error alerts, system notices,
and anything already handled (Shaffy replied and nothing is outstanding).

> **Attio is always on** for the standard client profile: it's the canonical
> client/prospect list used to recognize pipeline (step 1 criteria) **and** the
> write-back target for follow-up notes/tasks (step 6). Load the relevant Attio
> records early so email/chat counterparties can be matched against them — but
> matching is **best-effort** until the Attio hygiene workflow lands (`ROADMAP.md`):
> incomplete domains/deal names mean some clients won't match, so never gate on it.

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
4. **Fireflies** — recent-transcripts call responds.
5. **Attio** (write-back) — only if `ATTIO_JSON` / `attio.json` exists with a
   non-placeholder key; confirm a `GET /v2/objects` call succeeds. If absent,
   note "Attio write-back: off" (not an error).

Open the morning summary with one readiness line, e.g.:
`Preflight: Notion ✅ · Gmail techtower ✅ / myswimscore ⚠️ token missing · Slack TechTower ✅ · Fireflies ✅ · Attio off`.
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
   domain** or a **name alias** in the registry (not Attio — it's under-tagged).
2. **Route:**
   - Matches a `type:client` entry → write to that client's **dashboard page**
     (`dashboardPageId`).
   - Matches a `type:pipeline` entry, or is internal/hiring/ops, or matches no
     client → write to the **TechTower internal board** (`internalBoard`).
3. **Write target:**
   - **Client dashboards (v2 template) →** write rows into that client's
     **"✅ To Dos" database** (on its "6. To Dos" sub-page;
     `todoDataSourceId` in the registry). Set `Task`, `For`
     (`Needed from client` | `TechTower`), `Status`, `Source`, `Link`, and a
     stable `Ref` for dedup. Keep client-appropriate wording (no internal
     commercials). Upsert on `Ref` so re-runs never duplicate.
   - **Internal board →** the TechTower internal "Daily To-Do List" / managed
     section, with full internal detail (`Needs from You` / `AI Can Do`).
4. If a client has no dashboard page yet, flag it (don't fail); a page can be
   created from the client-dashboard template.

> Single-tenant client instances skip this — they just use their own tracker.
> This routing is the **agency** feature for TechTower fanning out across many
> client dashboards.

### 6. Write follow-up activity back to Attio (TechTower)
Only runs if `attio.json` is present (see `ATTIO_SETUP.md`). Direction is
**Tracker → Attio**: keep the CRM trail current for pipeline-related open items.
For each **open** item (skip `Done`) whose `Who` resolves to a real
person/company:
1. Match the record in Attio by email address (`people.email_addresses`) or
   company domain (`companies.domains`), checking the configured `objects` in
   order. If nothing matches → write nothing; set the tracker row's `Attio`
   column to `not in Attio`.
2. On the matched record:
   - **Add a note** summarizing the latest interaction + the open action.
   - **Ensure a follow-up task** assigned to Shaffy, due in `followUpDueDays`.
   - If `nextStepAttribute` / `lastContactedAttribute` are mapped, set them.
3. **Idempotency:** tag every note/task body with `[ref:<item Ref>]` and check
   for an existing one first — never create duplicate notes or tasks across runs.
4. **Guardrails:** never advance/close a deal stage unless `allowStageChange` is
   true **and** it's explicitly approved in this run. Never create new
   people/companies and never delete anything.
5. Record what was written in the tracker row's `Attio` column
   (e.g. `Note + task on "AgroCares" deal`).

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
