# Scheduling the run (Claude Code on the web)

The enrichment sweep runs as a **recurring scheduled session** in the Claude Code
web app, pointed at this repo. It runs in your authenticated account, so the
connected integrations (HubSpot, Gmail, Slack, Lemlist, Front, Shopify) are
available without extra tokens.

Docs: https://code.claude.com/docs/en/claude-code-on-the-web

> **One routine, several times a day.** Schedule a single recurring session with
> the prompt **`/daily-crm`**. There is only one workflow now — it enriches the
> pipeline and posts the stale brief. Every write is idempotent, so running it
> again an hour later produces no duplicate notes and no churn.

## Cadence

Pick the rhythm that matches how fast replies land. A sensible default is
**three runs on weekdays** — early morning, midday, late afternoon
(Europe/Amsterdam) — so a lead who replies at 9am is enriched well before the
end of the day rather than the next morning.

The workflow is safe at any frequency. The things that make re-running cheap:

- Notes are keyed by thread. A thread whose note is already current is **skipped**,
  not rewritten.
- Touch-tracking fields are recomputed from the conversation each time, so a
  later run corrects an earlier one rather than compounding it.
- No stage is ever written, so no run can undo another's classification.

## The routine

The workflow is committed as a slash command in `.claude/commands/`, so the
routine prompt is a single line:

| Command | Runs |
|---|---|
| `/daily-crm` | The full sweep: enrich Reply-to-be-enriched → Gmail today+yesterday → Shopify contact form → Slack stale brief |

## 1. Create the scheduled session

In the Claude Code web app:

1. Open this repo's environment (`shaffyroell/tasks`, branch `swimscore`).
2. Create a new **scheduled / recurring task** (look for **Schedule /
   Automations / Routines**).
3. **Cadence:** your chosen times, Europe/Amsterdam. If the scheduler is UTC-only,
   remember 07:00 CEST = 05:00 UTC in summer and 06:00 UTC in winter.
4. **Prompt:** `/daily-crm`
   (or, if the scheduler doesn't expand slash commands, paste the body of
   `.claude/commands/daily-crm.md` — it points at the skill.)
5. Nothing to enable. The committed `hubspot.json` carries the settings; flip
   `staleFollowUp.enabled` or `shopifyIntake.enabled` to `false` and commit to
   pause that piece.

### Model / cost

**Sonnet**, not Opus. This is mostly retrieval plus structured writes against
scoped lookbacks, so it doesn't need Opus-level reasoning — roughly 1/5 the cost
for the same quality here. Model cost is the same wherever it's scheduled;
hosting it elsewhere changes where scheduling lives, not token usage.

## 2. Environment secrets — usually NONE needed

The routine reads its config straight from git: **`hubspot.json`** is committed
and holds no credentials, so there is nothing to paste. All sources run through
your **connected account** in the scheduled session.

Add a secret **only if** a run's preflight shows a source ⚠️ unavailable
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

- The summary opens with a **preflight** line; each source you expect shows ✅,
  not ⚠️.
- HubSpot shows notes added or updated on the deals the run touched, and the
  touch-tracking fields filled on Reply-to-be-enriched deals.
- The stale brief landed in `#daily-recap`, split by owner.
- **No deal changed stage.** The skill's end-of-run spot-check covers this; if a
  stage moved, that is a bug.
- If a connector shows ⚠️ in a scheduled (headless) run, that source needs
  token-based access instead — the preflight names which, so you only fix what
  actually breaks.
