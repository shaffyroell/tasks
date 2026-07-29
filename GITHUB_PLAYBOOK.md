# TechTower GitHub Playbook

How we run GitHub as a company so we can collaborate on automations, tools, and
client work — without recreating the mess we're migrating out of.

Read this before you move a repo, add a tool, or onboard a client.

---

## 1. The mental model (read this first)

Three separate layers. Most confusion comes from mixing them up.

| Layer | What it is | Who/what sees it |
|---|---|---|
| **Repos** | Where code, skills, and configs *live* (for editing + access control) | Whoever you grant access |
| **Environments** | A "desk" that has one or more repos + connectors loaded, ready to work | Whoever the environment is for |
| **Connectors (MCP)** | Hosted tools (scraper, Google Ads, Attio, Gmail…) | **Every session, automatically** |

Two rules that follow from this:

- **A repo is not an environment.** One environment can have *many* repos loaded at
  once. You don't connect to repos one-at-a-time — you lay them all on the desk.
- **Tools reach every session as connectors, not as repo code.** A tool's repo is
  just where its code lives for maintenance. At runtime it's a connector, available
  everywhere with no reconnecting.

---

## 2. Repo structure & naming

GitHub has **no folders for repos** — an org is a flat list. We fake folders with
**naming prefixes**, so the alphabetical list clusters by type.

```
TechTower-AI/  (organization — everything private by default)

  techtower-internal        ← ALL internal skills, configs, workflow docs
  techtower-<...>           ← (only if internal work needs its own repo later)

  mcp-google-ads            ← MCP servers: one repo each (standalone software)
  mcp-google-analytics
  mcp-<name>

  tool-lead-scraper         ← standalone tools: one repo each (make PRIVATE)
  tool-lemlist

  client-template           ← the template new client repos are cloned from
  client-crewline           ← client repos: one each (this is the access boundary)
  client-<name>
```

### Naming rules

- **All lowercase, hyphen-separated** (`client-crewline`, not `Client_Crewline`).
- **Prefix by type:** `techtower-` (internal) · `mcp-` · `tool-` · `client-`.
- **Name after the thing, not the person** — no `shaffy-...` or `joep-...`.
- **One clear purpose per repo.** If you can't say what a repo is for in one line,
  it's wrong.

---

## 3. When do I create a new repo?

Ask these **in order**. Stop at the first "yes."

1. **Does a different set of people need access?** (e.g. a new client)
   → **New repo.** `client-<name>`, cloned from `client-template`.
2. **Is it standalone software that runs on its own** (an MCP server, or a tool with
   its own dependencies / runtime)?
   → **New repo.** `mcp-<name>` or `tool-<name>`.
3. **Otherwise** — it's a skill, prompt, config, or doc.
   → **No new repo.** It goes in an existing repo (`techtower-internal`, or the
   relevant `client-<name>`).

> **Default answer is "no new repo."** New repos are for access boundaries and
> standalone software only. Everything else is a folder inside an existing repo.

---

## 4. Configure, don't copy (per-client tools)

A client wanting "their own version" of a tool almost **never** means a new repo.

- A tool has an **engine** (the code, written once) and **settings** (a small config
  file, one per client).
- Same engine, different settings = a client's "own version." You swap the settings,
  not the code.
- The client's settings file lives in **their** `client-<name>` repo (they can see and
  edit it); the engine stays in the **tool** repo (they never touch it).
- The engine reaches their session as a **connector**, so they get results without a
  copy of the code.

**You only fork a tool into a new repo if the client must run the engine's code
themselves, independently.** That's rare and costly (every fix must be applied in
every copy) — treat it as a last resort.

This is already how `techtower-internal` works: one `daily-crm` skill (engine) driven
by `clients.json` (settings). One engine, many clients.

---

## 5. Environments: how we avoid "reconnecting per workspace"

Isolation is for the **client**, not for **us**.

- **Team environment** = `techtower-internal` **+ every client repo** loaded at once,
  **+ all tool connectors**. Our whole team works here. Everything is already on the
  desk — no switching, no reconnecting. New client? Add their repo to this environment
  once; it stays.
- **Client environment** (only if a client uses Claude Code) = *only* their
  `client-<name>` repo + the connectors they're allowed. They see only their own stuff.

Same repos, two different desks. We get everything in one place; clients stay walled off.

**Tools = connectors.** Package reusable tools (scraper, Google Ads, GA, lemlist) as
MCP servers and connect them once. Then they're in every team session and every
routine automatically — the repo is only where the code is maintained.

---

## 6. Migration discipline — how we DON'T recreate the mess

The central org has **standards**. Nothing gets "dumped in." Every candidate from a
personal account (Shaffy's *or* Joep's) is triaged one at a time.

### Triage each item: KEEP / ARCHIVE / DROP

| Verdict | Meaning | Action |
|---|---|---|
| **KEEP** | Live, used, and it's TechTower's | Migrate into the **right** repo (per §3), cleaned |
| **ARCHIVE** | Ours but dead / superseded / a different business | Leave in personal account, mark **Archived** |
| **DROP** | Third-party fork, experiment, junk | Leave it; don't bring it in |

Questions that decide the verdict:

1. **Is it actually used?** (Has it run in the last few months? Is a routine pointing
   at it?) If no → ARCHIVE.
2. **Is it TechTower's?** Not SwimScore, not a personal experiment, not someone else's
   fork. If no → DROP or keep separate.
3. **Which bucket?** internal / mcp / tool / client (per §3).
4. **Does it duplicate something already migrated?** If yes → merge into that, don't
   add a second copy.

### Definition of "done" for a repo in the org

A repo is only allowed to be a first-class org repo once it has:

- [ ] A **README** — one paragraph: what it is, how to run it, where settings live.
- [ ] The **correct name** and prefix (§2).
- [ ] **No secrets committed** (API keys, tokens → env vars / connector auth, never in
      git). Check before pushing.
- [ ] A clear **owner** (who maintains it).
- [ ] Dead files removed — migrate the *current* thing, not years of cruft.

> Rule of thumb: **migrating is a cleanup, not a copy.** If you'd be embarrassed to
> show the repo to the other person, it's not done.

### The "don't dump" rule for Joep's + Shaffy's accounts

- We do **not** transfer whole messy repos wholesale. We migrate the **handful of KEEP
  items**, cleaned, into the right place.
- Most personal repos will be ARCHIVE or DROP. That's expected and healthy — a clean
  org of ~8–10 purposeful repos beats 40 dumped ones.
- Each person triages their **own** account (only the account owner can transfer/clean
  their repos).

---

## 7. Migration process (step by step)

For each **KEEP** item:

1. **Decide the destination** repo (§3) and whether it's new or folds into an existing
   one.
2. **Bring the code in** — copy the cleaned files into the destination repo and commit,
   *or* transfer the whole repo if it's already clean and standalone (GitHub → Settings
   → Danger Zone → Transfer to `TechTower-AI`). Transfers keep history; copies give a
   clean slate.
3. **Re-point any routine** that used the old location:
   - A routine clones a **source repo** every run. Moving the repo means updating the
     routine to the new `TechTower-AI/...` location.
   - If files were reorganized, update the **paths** referenced in the routine's prompt
     (e.g. `.claude/skills/daily-crm/`).
   - **Test-fire the routine** and confirm it runs green before archiving the old repo.
4. **Flip visibility** — proprietary tools (e.g. the scraper) must be **private**.
5. **Archive/clean** the old personal copy so there's one source of truth.

---

## 8. Working conventions (day to day)

- **Branch, don't push to main directly.** Turn on branch protection on shared repos
  so changes land via a quick review.
- **One config per client, one engine.** Never copy an engine to customize it (§4).
- **Every meaningful folder gets a short README** saying what's in it and where new
  things go. This is what keeps repos from turning into junk drawers.
- **Secrets live in connector auth / env vars, never in git.**
- **New client onboarding** = clone `client-template` → `client-<name>` → add to the
  team environment → grant the client access if they need it.

---

## Appendix A — Current triage (Shaffy's account, 21 repos)

Starting point, as an example of the discipline in §6. Joep runs the same pass on his.

**KEEP → migrate**
- `TechTower-automations` / `tasks` — internal core (reconcile the two; one source of
  truth). → `techtower-internal`
- `command-center` — confirm whether it's an app (own repo) or configs (fold in).
- `Maps_scrape` — the lead scraper. ⚠️ **currently public — flip to private.** →
  `tool-lead-scraper`
- `mcp-google-ads`, `google-analytics-mcp` — → `mcp-*` (bring in as standalone, drop
  the fork link).
- `lemlist` — → `tool-lemlist` (confirm it's a tool vs a fork).
- `carousel` — confirm what it is, then bucket.

**Separate business (not TechTower)**
- `swimscore-insight-dashboard`, `Swimscore_vid1`, `Swimscore_vid2` — SwimScore. Keep
  separate; its own grouping later.

**DROP / leave (third-party forks & junk)**
- `mj-mcp` — fork, **but load-bearing** (a live SwimScore routine runs from it): do not
  archive until that routine is re-pointed.
- `meta-ads-mcp`, `google_workspace_mcp`, `goose-skills`, `Flowise` — forks; leave.
- `test`, `MLiFC_invoice_classifier` — junk; archive.
- `brandcat`, `video`, `dashboard` — inspect, then bucket or archive.

**Note on skills not in any repo:** ~16 client/business skills currently live only in
the local `~/.claude/skills/` folder (Blume, SHIFT, SwimScore, Crewline, Affinity).
These are unversioned and unshareable. Blume/SHIFT stay local for now (by decision);
**Crewline** gets imported to `client-crewline`.

---

## Appendix B — Cheat sheet

- **New client?** → clone `client-template` → `client-<name>` → add to team environment.
- **New MCP/tool?** → `mcp-<name>` / `tool-<name>`, connect it as a connector.
- **New skill/prompt/config?** → into an existing repo (internal or the client's). No
  new repo.
- **Client wants a tool "their way"?** → new **config** in their repo, not a new repo.
- **Where do tools come from at runtime?** → connectors (everywhere), not repos.
- **Access everything without reconnecting?** → one team environment, all repos loaded.
- **Migrating something?** → it must pass KEEP triage + the "done" checklist first.
