# TechTower - GitHub Operating Rules

**Every Claude session and every person working in ANY `TechTower-AI` repo MUST
read and follow this file** before creating repos or folders, moving files, or
suggesting structure. These are the house rules. If a request conflicts with a
rule, say so and propose the compliant alternative.

---

## Core model (do not confuse these three)

- **Repos** = where code, skills, and configs live + who has access.
- **Environments** = a workspace ("desk") with one or more repos + connectors loaded.
- **Connectors (MCP)** = hosted tools, available in EVERY session automatically.

Two consequences:
- A repo is **not** an environment. One environment loads **many** repos at once.
- Tools reach a session as **connectors**, not as repo code. A tool's repo is just
  where its code is maintained.

---

## Repo naming (MANDATORY)

- **lowercase, hyphen-separated** (`client-crewline`, not `Client_Crewline`).
- **Prefix by type** so the flat repo list groups by kind:
  - `TechTower` -> the internal core (skills, tools, configs, workflows)
  - `client-<name>` -> one repo per client (this is the access boundary)
  - `mcp-<name>` -> an MCP server (standalone software)
  - `tool-<name>` -> a standalone tool
- **Name after the thing, never after a person** (no `shaffy-...`, no `joep-...`).
- **One clear purpose per repo**, stated in the first line of its README.

---

## When to create a NEW repo - ask in order, stop at the first "yes"

1. **Do different people need access?** (e.g. a new client) -> new `client-<name>`,
   cloned from `client-template`.
2. **Is it standalone software** (an MCP server, or a tool with its own runtime/
   dependencies)? -> new `mcp-<name>` or `tool-<name>`.
3. **Otherwise** (a skill, prompt, config, or doc) -> **NO new repo.** It goes in an
   existing repo (`TechTower`, or the relevant `client-<name>`).

**Default = no new repo.** New repos are for access boundaries and standalone
software only. When unsure, put it in the right existing repo and ask.

---

## Configure, don't copy

A tool = one **engine** (code, written once) + **settings** (a small config file,
one per client). A client's "own version" is almost always just their settings, not
forked code:
- The client's config lives in **their** `client-<name>` repo.
- The engine stays in its `tool-`/`mcp-` repo and reaches them as a **connector**.
- Only fork a tool into a new repo if the client must run the engine's code
  themselves - rare, and costly (every fix must be applied to every copy).

Never copy an engine to customize it. Add a config.

---

## Client isolation (non-negotiable)

- Each `client-<name>` repo is that client's **access boundary.**
- **Never** put two clients in one repo.
- **Never** give a client access to any repo but their own.
- The team sees everything via **one team environment**; a client sees only their
  own repo via their own narrow environment.

---

## Migration discipline - do NOT dump

Nothing is bulk-dumped into the org. Triage every item:
- **KEEP** - live and ours -> migrate it, cleaned, into the right repo.
- **ARCHIVE** - ours but dead / superseded / a different business -> leave it in the
  personal account and mark Archived.
- **DROP** - third-party fork, experiment, or junk -> leave it; don't bring it in.

A repo is only "done" when it has: a one-line-purpose README, the correct name, **no
secrets committed**, a named owner, and cruft removed. **Migrating is a cleanup, not
a copy.**

---

## Secrets

**Never commit API keys, tokens, or credentials.** Use connector auth / environment
variables. Check before every push.

---

## Working conventions

- **Branch + PR review** on shared repos; don't push straight to `main`.
- **Every meaningful folder gets a short README** (what's here, where new things go).
- **One config per client, one engine** (see "Configure, don't copy").
- **New client** = clone `client-template` -> `client-<name>` -> add to the team
  environment -> grant the client access only if they need it.

---

## If you are an AI assistant working in this org

1. **Read this file first** and follow it when proposing ANY structure.
2. If a request would break a rule (e.g. "copy the scraper into this client repo"),
   **say so** and propose the compliant alternative (config, not copy).
3. When unsure where something goes, **default to no-new-repo and ask.**
4. Keep proposals to the smallest change that fits the rules.

Fuller detail and rationale: see `GITHUB_PLAYBOOK.md` in this repo.
