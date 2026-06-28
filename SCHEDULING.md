# Scheduling the daily run (Claude Code on the web)

The sweep runs as a **recurring scheduled session** in the Claude Code web app,
pointed at this repo. It runs in your authenticated account, so the connected
integrations (Gmail, Slack, Notion, Fireflies) are available without extra
tokens; separate accounts/workspaces come from environment secrets.

Docs: https://code.claude.com/docs/en/claude-code-on-the-web

> **Simplest: one routine.** Schedule **one** daily session with the prompt
> **`/daily-crm`** — it runs **W0 → W1 → W2 in order**, so each step reads what the
> previous wrote (no timing race). Enable `newDealDiscovery.enabled` and
> `attioIngest.enabled` in `attio.json`.
>
> **Or three staggered sessions** (if you want them on separate cadences):
> 1. **W0 — New-deal discovery** (`new-deal-discovery.md`) → creates new deals —
>    **06:50**.
> 2. **W1 — Attio ingest** (`attio-ingest.md`) → notes + stage moves — **07:00**.
> 3. **W2 — Per-client To-Dos** (`daily-open-items.md`) → Notion — **07:15**.

## Routines = slash commands (committed)

The workflows are committed as slash commands in `.claude/commands/`, so a routine
prompt is a single line:

| Command | Runs |
|---|---|
| `/daily-crm` | **W0 → W1 → W2 in order** (recommended — one routine, guaranteed ordering) |
| `/new-deals` | W0 only — discover + create new deals in Attio |
| `/attio-ingest` | W1 only — ingest comms → Attio |
| `/notion-todos` | W2 only — per-client To-Dos → Notion |

**Recommended:** one daily routine with the prompt **`/daily-crm`** — it runs
discovery → ingest → to-dos in sequence, so each reads the prior step's fresh
output. Use the split commands only if you want them on separate cadences.

## 1. Create the scheduled session (the "routine")

In the Claude Code web app:
1. Open this repo's environment (`shaffyroell/tasks`, branch
   `claude/deals-stage-attio-mapping-37w20d` or wherever this is merged).
2. Create a new **scheduled / recurring task** (look for **Schedule /
   Automations / Routines**).
3. **Cadence:** daily, **07:00 Europe/Amsterdam**. If the scheduler is UTC-only,
   use **05:00 UTC** (= 07:00 CEST summer; it's 06:00 CET in winter — adjust if
   you care about the winter hour).
4. **Prompt:** `/daily-crm`
   (or, if the scheduler doesn't expand slash commands, paste the body of
   `.claude/commands/daily-crm.md`).
5. Enable the workflows in `attio.json` → `newDealDiscovery.enabled: true` and
   `attioIngest.enabled: true`.

If you'd rather run W1 and W2 as **two** staggered routines instead of one:
W1 `/attio-ingest` at 07:00, W2 `/notion-todos` at 07:15.

### Model / cost

Set the scheduled session to run on **Sonnet**, not Opus. This workflow is mostly
retrieval + structured writes (snippets/metadata, not full bodies; Fireflies
action-items, not full transcripts; scoped 21d/7d lookbacks), so it doesn't need
Opus-level reasoning. Sonnet runs it at roughly 1/5 the cost for the same quality
here — ballpark low-cents to ~$0.50 per daily run vs. ~$1–3 on Opus. Note that
model/runtime cost is the same regardless of *where* it's scheduled — hosting it
elsewhere doesn't reduce token usage, only changes where scheduling lives.

## 2. Add environment secrets

In the environment's **secrets / env vars**, add only the ones you use (paste the
full JSON contents as the value):

| Secret | Needed for |
|---|---|
| `EMAIL_ACCOUNTS_JSON` | Extra mailboxes beyond the connected inbox (e.g. myswimscore) |
| `SLACK_WORKSPACES_JSON` | Extra Slack workspaces beyond the connected one |
| `ATTIO_JSON` | Attio CRM write-back |

The gitignored `*.json` files are **not** in the scheduled clone — secrets are the
only credential path for scheduled runs.

## 3. Network policy

The run makes outbound calls to the connector/API endpoints. Pick an environment
network policy that allows that outbound access (see the docs link above).

## 4. Verify after the first run

- Check the run's summary opens with a **Preflight** line and that each source
  you expect shows ✅ (not ⚠️).
- Confirm new rows appeared / updated in the Notion tracker
  (https://app.notion.com/p/92836c3b058e49fda9cbf9d5b956a144).
- If a connector shows ⚠️ "not available" in a scheduled (headless) run, that
  source needs token-based access instead — add its token as a secret (Gmail via
  `EMAIL_ACCOUNTS_JSON`, Slack via `SLACK_WORKSPACES_JSON`; Notion/Fireflies would
  need their own tokens). The preflight tells you exactly which, so you only fix
  what actually breaks.

## Fallback

If you ever want a non-Claude-hosted schedule (GitHub Actions cron or a
self-hosted cron + `gtm-multi-mcp`), the workflow is already token-portable for
email/Slack/Attio; you'd additionally provide Notion + Fireflies tokens. Ask and
this runbook can be extended with that path.
