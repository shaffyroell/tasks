# Daily Open-Items Sweep

**Run this every morning (~7:00 Europe/Amsterdam).** It scans Gmail, Slack, and
recent meeting notes (**Fireflies + Granola**), then (a) reconciles the Notion
tracker so it always reflects what Shaffy still needs to respond to or act on, and
(b) keeps **Attio current as the primary CRM log** — for every pipeline contact it
ensures the person exists, is linked to the right deal, and has a note + follow-up
task for the open action.

> **Two destinations, different jobs.** **Attio** is the canonical CRM record
> (who, which deal, the activity trail) — it's the primary write target. **Notion**
> stays the human-facing daily to-do view. The same swept items feed both.

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
- **Meetings:** `stack.meetings.providers` — a list, one or more of
  (fireflies | otter | gemini | granola). TechTower reads **both Fireflies and
  Granola**; the sweep iterates every configured provider. (Legacy single
  `stack.meetings.provider` is still accepted.)
- **CRM:** `stack.crm` (attio, always on — the canonical client/prospect list and
  the **primary write target**). Used both to **recognize** which emails/chats are
  client/pipeline and as the place every pipeline contact + activity is logged
  (step 6).
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
3. **Meeting next-steps** — action items Shaffy committed to in recent calls,
   from **every** configured meeting tool (Fireflies `action_items` and Granola
   notes/summaries) assigned to Shaffy / "Shaffy and team".

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
4. **Meeting notes** — for each provider in `stack.meetings.providers`:
   **Fireflies** recent-transcripts call responds; **Granola** `list_meetings`
   responds. List which loaded.
5. **Attio** (primary CRM log) — only if `ATTIO_JSON` / `attio.json` exists with a
   non-placeholder key; confirm a `GET /v2/objects` call succeeds and that the
   `deals` / `people` / `companies` object slugs resolve. If absent, note
   "Attio: off — CRM logging skipped" (not a hard error; Notion still updates).

Open the morning summary with one readiness line, e.g.:
`Preflight: Notion ✅ · Gmail techtower ✅ / myswimscore ⚠️ token missing · Slack TechTower ✅ · Fireflies ✅ · Granola ✅ · Attio ✅`.
A source marked ⚠️ is simply not swept this run — say so explicitly so a missing
or expired credential surfaces loudly instead of silently dropping coverage.

### 1. Load current tracker state
Query the data source (`45a1139d-c188-4a05-936d-adbea5d6715e`) for all rows that
are **not** `Done`. Build a lookup by `Ref` so you can update instead of
duplicate. `Ref` formats:
- Email → `gmail:<threadId>` (or `outlook:<id>`)
- Chat → `chat:<provider>:<workspace>/<channel>/<ts>`
- Meeting → `ff:<transcriptId>` (Fireflies) / `granola:<meetingId>` (Granola)
  / `<provider>:<id>` (other)

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

### 4. Gather meeting next-steps — every provider in `stack.meetings.providers` (last ~7 days)
Iterate **all** configured meeting tools (TechTower: Fireflies **and** Granola).
For each, pull recent meetings, extract the action items, and keep only those
assigned to the **owner** (or "owner + team"). Consolidate per meeting into one
row where sensible.

- **Fireflies** — `fireflies_get_transcripts` (last 7d) → for each, read its
  `action_items`. `Link = https://app.fireflies.ai/view/<transcriptId>`,
  `Ref = ff:<transcriptId>`.
- **Granola** — `list_meetings` (`time_range: this_week` / `last_week`, or a 7-day
  custom range) → `get_meetings` for the AI summary + notes, and pull owner
  action items / commitments from them (or use `query_granola_meetings` with a
  query like *"action items and follow-ups assigned to Shaffy in the last 7 days"*,
  preserving its citation links). `Ref = granola:<meetingId>`, and use the
  meeting's Granola URL as `Link` when available.

> **Dedup across tools:** the same call may exist in both Fireflies and Granola.
> If two meeting rows clearly describe the same meeting (same title/date/attendees),
> keep one row and note both sources; the `Ref` prefix (`ff:` vs `granola:`) keeps
> them distinct if you'd rather not merge.

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

### 6. Sync to Attio — the primary CRM log (TechTower)
Only runs if `attio.json` is present (see `ATTIO_SETUP.md`). Direction is
**Tracker → Attio**. Attio is the **primary CRM record**: every pipeline contact
should exist, be linked to the right deal, and carry the activity trail. Notion
remains the to-do view. Process each **open** item (skip `Done`) whose `Who`
resolves to a real person/company:

**a. Resolve the person.**
1. Match a person in Attio by email (`people.email_addresses`) via
   `POST /v2/objects/people/records/query`.
2. **If the person exists** → use it.
3. **If the person does not exist** and `ensurePersonExists` is true → **create**
   the person (name + email; company domain if known). If `ensurePersonExists` is
   false, write nothing and set the row's `Attio` column to `not in Attio` (legacy
   additive mode).

**b. Resolve the deal — one per company, keyed by domain.** (when `linkPeopleToDeals` is true)
The deal is **owned by the company**, deduped on the email **domain**
(`companies.domains`). There is at most **one deal per company**.
1. Find the company by the contact's email domain (`companies.domains`).
2. Determine the deal in priority order:
   - the deal **already on that company** (one-per-company → there should be at
     most one; if several exist, pick the open/most-recent and flag
     `multiple deals (review)`);
   - else a deal the **person** is already associated with;
   - else **no deal exists yet** → decide whether to create one (step b2).

**b2. Create a deal for a new pipeline conversation** (when `createMissingDeals` is true)
If no deal exists for the company **and** this is a genuine **new pipeline
conversation**, create exactly one deal. **Email is the primary signal**; confirm
it's real pipeline (not a one-off / vendor / newsletter) using corroborating cues:
   - a back-and-forth thread with a real prospect/partner, **and/or**
   - a **calendar invite the owner sent** to that counterparty (check Gmail/
     Calendar for a sent invite to the domain), **and/or**
   - **follow-up emails** in the thread.
   Then:
   1. **Ensure the company** exists (match by domain; create with name + domain if
      missing — `createMissingCompanies`). Re-query by domain first so you never
      create a second company for the same domain.
   2. **Create one deal** on that company (`nameTemplate`, `ownerEmail`), and set
      its **stage** from `newDealDefaults.stageRules` (first match wins) based on
      the same signals — e.g. invite sent → "Meeting booked"; proposal/quote in
      thread → "Proposal"; otherwise the `defaultStage` (e.g. "Lead").
   3. **Link the people** in the conversation to the new deal.
   If the signals are weak/ambiguous (could be a one-off), **don't** create — set
   `Attio` to `no deal (review)` and leave it for a human.

**b3. Link the person.** If the person is **not yet linked** to the resolved/created
deal, **add the link**. If already linked, leave it.

**c. Log the activity** on the resolved deal (preferred) and/or person:
   - **Add a note** summarizing the latest interaction + the open action.
   - **Ensure a follow-up task** assigned to `assigneeEmail`, due in
     `followUpDueDays`.
   - If `nextStepAttribute` / `lastContactedAttribute` are mapped, set them.

**d. Idempotency.** Tag every note/task body with `[ref:<item Ref>]` and check for
an existing one first — never create duplicate people, notes, tasks, or
person↔deal links across runs. Before creating a person, re-query by email to
avoid a race/duplicate.

**e. Guardrails.**
- **New deals** are created only for a genuine new pipeline conversation (step b2),
  **one per company** (deduped by domain) — never a second deal for a company that
  already has one.
- **Changing an existing deal's stage** is different from setting the stage on a
  brand-new deal: never advance/close/win an **existing** deal's stage unless
  `allowStageChange` is true **and** it's explicitly approved in this run.
- Companies/people are created only as needed to back a deal or resolve a
  contact (deduped by domain/email first). Never delete anything.

**f. Record** what was written in the tracker row's `Attio` column, e.g.
`Created "Acme" deal (stage: Meeting booked) + linked 2 people + note/task`,
`Linked to existing "AgroCares" deal + note`, or `no deal (review)`.

> **Attio data is incomplete today** (missing domains/deal names, people not linked
> to deals — `ROADMAP.md`). This step repairs it as it goes: one deal per company
> keyed by domain, the right people linked, new pipeline conversations opened as
> deals at the right stage. When a deal/link/stage is genuinely ambiguous, prefer
> flagging `(review)` over a wrong write.

### 7. Report
Post a short summary to Shaffy: counts by Source and Status, and call out the
top 3 `High` / `Needs Response` items. Include an **Attio line**: people created,
people newly linked to a deal, notes/tasks written, and anything flagged for
review (`no deal` / `multiple deals`). Keep it tight.

---

## Notes
- Dedup is driven entirely by `Ref`. Never create a second row for the same
  thread/mention/meeting.
- Be conservative: when unsure whether something needs a response, lean toward
  including it as `Follow Up` (Low) rather than spamming `Needs Response`.
- Credentials (`accounts.json` etc.) are never read or committed by this
  workflow.
