# Writing style — client-facing to-dos & dashboard text

**Every to-do written to a client dashboard (and the master tracker) must follow
this.** It's inferred from TechTower's existing dashboards — keep it consistent.
W2 (`daily-open-items.md`) loads this before writing any Notion row; W0/W1 notes
in Attio should match the same voice.

## Voice in one line
**Imperative, verb-first, concise, specific — what to do, for whom, and the
outcome — no fluff, no internal commercials on client boards.**

## Rules

1. **Start with an action verb.** Get, Sign off, Send, Build, Stand up, Share,
   Confirm, Schedule, Set up, Decide, Launch, Fix, Review. Never start with a noun
   or "Need to…".
2. **One line, ~6–12 words core.** No trailing period. Sentence case (capitalize
   first word + proper nouns only).
3. **Name the owner / context in parentheses.** `(Hugo)`, `(Lucy's request)`,
   `(with Syb)`, `(Freddie)`.
4. **Show the downstream impact** with `→` (chain) or an em-dash `—` (risk/why):
   `… → unblock Google Maps → campaign launch`, `… — mis-send risk until live`.
5. **Be concrete** — real dates/times with tz, tools, and names:
   `Fri Jul 3, 12:00–12:30 CET`, `in Lemlist`, `Claude skill file`.
6. **Accepted shorthand:** LI (LinkedIn), V2, DB, MCP, CRM, ICP, tz codes (CET/ET),
   client initials (NJ). `&` and `+` for brevity. Don't over-abbreviate names.
7. **Client-safe on client dashboards:** no pricing, margins, internal commercials,
   or other-client references. (The master/internal tracker may carry internal
   detail; client boards never do.)
8. **Match the client's working language.** Default English; write in Dutch if the
   client's thread/relationship is in Dutch — don't mix within one item.
9. **Flag dependencies inline:** `(needs doc sources + vault keys, then review)`.
10. **No hedging / filler:** drop "please", "maybe", "we should", "follow up on"
    (say the actual next action instead).

## Worked examples (from live dashboards — match these)

- `Get competitor blocklist signed off (Hugo) — TerraFirma mis-send risk until live`
- `Sign off Rev Architect <8/>8 + V2 copy → unblock Google Maps → campaign launch`
- `Stand up V2 retargeting flow (accepted LI → dedicated follow-up)`
- `Send next-phase proposal (differentiate setup vs follow-on)`
- `Confirm next sync — Fri Jul 3, 12:00–12:30 CET + send invite`
- `Build & launch UK London fund-lawyers campaign in Lemlist`
- `Build compliance/culture Claude skill file (needs doc sources + vault keys, then review)`
- `Set up Auto Slide Maker for new hire Zafirah (Lucy's request)`
- `Schedule Justin catch-up call (next Wed ~9am ET)`

## Anti-patterns (rewrite these)

| ✗ Don't | ✓ Do |
|---|---|
| `Follow up with Hugo about the blocklist` | `Get competitor blocklist signed off (Hugo)` |
| `Need to send the proposal to Hassans` | `Send next-phase proposal (setup vs follow-on)` |
| `There is a call to confirm.` | `Confirm Jul 3 sync — 12:00 CET + send invite` |
| `Work on the retargeting stuff for V2` | `Stand up V2 retargeting flow (accepted LI → follow-up)` |

## Field conventions (To-Dos DB row)

- **Task** (title) — the line above, in this style.
- **For** — `Needed from client` | `TechTower`.
- **Status**, **Source** (Email/Slack/Meeting), **Link**, and a stable **Ref**
  (for dedup/upsert — re-runs never duplicate).
