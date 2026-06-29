# Scheduling the daily run (Claude Code on the web)

The sweep runs as a **recurring scheduled session** in the Claude Code web app,
pointed at this repo. It runs in your authenticated account, so the connected
integrations (HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion) are
available without extra tokens.

Docs: https://code.claude.com/docs/en/claude-code-on-the-web

> **Simplest: one routine.** Schedule **one** daily session with the prompt
> **`/daily-crm`** — it runs **W0 → W1 → W2 in order**, so each step reads what the
> previous wrote (no timing race). Enable `newDealDiscovery.enabled` and
> `ingest.enabled` in `hubspot.json` (both already true).
>
> **Or three staggered sessions** (if you want them on separate cadences):
> 1. **W0 — New-deal discovery** (`new-deal-discovery.md`) → creates new deals —
>    **06:50**.
> 2. **W1 — HubSpot deal sync** (`hubspot-sync.md`) → notes + stages + stale flags — **07:00**.
> 3. **W2 — SwimScore Notion to-dos** (`daily-open-items.md`) — **07:15**.

## Routines = slash commands (committed)

The workflows are committed as slash commands in `.claude/commands/`, so a routine
prompt is a single line:

| Command | Runs |
|---|---|
| `/daily-crm` | **W0 → W1 → W2 in order** (recommended — one routine, guaranteed ordering) |
| `/new-deals` | W0 only — discover + create new deals in HubSpot |
| `/hubspot-sync` | W1 only — update deals (notes + stages) + stale-deal flags |
| `/notion-todos` | W2 only — SwimScore Notion board to-dos |

**Recommended:** one daily routine with the prompt **`/daily-crm`** — it runs
discovery → deal-sync → to-dos in sequence, so each reads the prior step's fresh
output. Use the split commands only if you want them on separate cadences.

## 1. Create the scheduled session (the "routine")

In the Claude Code web app:
1. Open this repo's environment (`shaffyroell/tasks`, branch
   `claude/client-pipeline-email-sync-jfk7m0` or wherever this is merged).
2. Create a new **scheduled / recurring task** (look for **Schedule /
   Automations / Routines**).
3. **Cadence:** daily, **07:00 Europe/Amsterdam**. If the scheduler is UTC-only,
   use **05:00 UTC** (= 07:00 CEST summer; it's 06:00 CET in winter — adjust if
   you care about the winter hour).
4. **Prompt:** `/daily-crm`
   (or, if the scheduler doesn't expand slash commands, paste the body of
   `.claude/commands/daily-crm.md`).
5. Enablement: already on in the committed `hubspot.json`
   (`newDealDiscovery.enabled`, `ingest.enabled`, `staleFollowUp.enabled` = true).
   Nothing to set — flip any to `false` and commit to pause that stage.

If you'd rather run W1 and W2 as **two** staggered routines instead of one:
W1 `/hubspot-sync` at 07:00, W2 `/notion-todos` at 07:15.

### Model / cost

Set the scheduled session to run on **Sonnet**, not Opus. This workflow is mostly
retrieval + structured writes (snippets/metadata, not full bodies; call
action-items, not full transcripts; scoped lookbacks), so it doesn't need
Opus-level reasoning. Sonnet runs it at roughly 1/5 the cost for the same quality
here — ballpark low-cents to ~$0.50 per daily run vs. ~$1–3 on Opus. Note that
model/runtime cost is the same regardless of *where* it's scheduled — hosting it
elsewhere doesn't reduce token usage, only changes where scheduling lives.

## 2. Environment secrets — usually NONE needed

The routine reads its config straight from git: **`hubspot.json`** (pipeline IDs +
enable flags + Slack channel map + stale-follow-up settings) is **committed** (no
credentials), so there is **nothing to paste** for the core pipeline. All sources
(HubSpot, Gmail, Slack, Granola, Lemlist, Shopify, Notion) run through your
**connected account** in the scheduled session.

Add a secret **only if** the first run's preflight shows a source ⚠️ unavailable
headless — then add just that one:

| Secret | Only if… |
|---|---|
| `HUBSPOT_TOKEN` | HubSpot shows ⚠️ in a scheduled run (connector not available headless) |
| `EMAIL_ACCOUNTS_JSON` | you add a mailbox beyond the connected inbox |
| `SLACK_WORKSPACES_JSON` | you add a Slack workspace beyond the connected one |

Real tokens go in env secrets (or the gitignored `hubspot.local.json`) — **never**
in the committed `hubspot.json`.

## 3. Network policy

The run makes outbound calls to the connector/API endpoints. Pick an environment
network policy that allows that outbound access (see the docs link above).

## 4. Verify after the first run

- Check the run's summary opens with a **Preflight** line and that each source
  you expect shows ✅ (not ⚠️).
- Confirm deals updated in HubSpot (notes + stage moves) and new/updated rows on
  the SwimScore Notion board (https://app.notion.com/p/bf5ee5540bd04fd69f10a2336656fd70).
- If a connector shows ⚠️ "not available" in a scheduled (headless) run, that
  source needs token-based access instead — add its token as a secret (HubSpot via
  `HUBSPOT_TOKEN`, Gmail via `EMAIL_ACCOUNTS_JSON`, Slack via
  `SLACK_WORKSPACES_JSON`). The preflight tells you exactly which, so you only fix
  what actually breaks.

## Fallback

If you ever want a non-Claude-hosted schedule (GitHub Actions cron or a
self-hosted cron), the workflow is already token-portable for HubSpot, Gmail,
Slack, Lemlist, and Notion. Ask and
this runbook can be extended with that path.
