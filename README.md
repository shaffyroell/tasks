# Client Pipeline Enrichment — HubSpot (single source of truth)

One workflow, run **several times a day**, that keeps the HubSpot pipeline
honest. It reads the real conversation across Lemlist, Front, and Gmail, writes
that context onto the deals that already exist, and posts a stale-deal brief to
Slack.

It is deliberately narrow:

- **It adds context.** Notes and fields — who spoke last, whether SwimScore has
  replied yet, what each side first said, what the deal is worth doing next.
- **It never moves a deal between stages.** Not on a positive reply, a confirmed
  call, a signed agreement, or a flat decline. Declines get Closed Lost
  *suggested* in the digest. Shaffy moves every deal by hand.
- **It creates a deal in exactly one case:** a website contact-form enquiry that
  has no deal yet. Lemlist replies already get a deal from a separate automation
  the moment they land, so nothing else here creates.

```
Lemlist · Front · Gmail · Shopify contact form
        │
        ▼
   ┌─────────────────────────────────────────┐
   │ 1. Sweep "Reply (to-be-enriched)"       │
   │ 2. Every Gmail message, today+yesterday │──▶  HubSpot
   │ 3. Shopify contact form (creates)       │     (notes + context fields;
   └─────────────────────────────────────────┘      stages untouched)
        │
        ▼
   4. Stale brief ─────────────────────────────▶  Slack #daily-recap
      (4+ days quiet, split by owner: Alex / Shaffy)
```

## Where things live

- **The workflow:** [`.claude/skills/daily-crm/SKILL.md`](./.claude/skills/daily-crm/SKILL.md)
  — the single source of truth for the steps.
- **Slash command:** `.claude/commands/daily-crm.md` (`/daily-crm`) — a thin
  pointer to the skill, plus the guardrails.
- **Pipeline config (committed, IDs + rules, no secrets):** [`hubspot.json`](./hubspot.json)
- **HubSpot setup & pipeline map:** [`HUBSPOT_SETUP.md`](./HUBSPOT_SETUP.md)
- **Scheduling:** [`SCHEDULING.md`](./SCHEDULING.md)
- **Supported tools & connections:** [`CONNECTIONS.md`](./CONNECTIONS.md)
- **Adding more Slack workspaces:** [`SLACK_SETUP.md`](./SLACK_SETUP.md)
- **Adding more email accounts:** [`EMAIL_SETUP.md`](./EMAIL_SETUP.md)

## The sources

All read through the connected account in this environment:

- **Lemlist** — the primary source for a lead's first reply, on **both** channels
  a campaign runs: email *and* LinkedIn. Read the full thread via
  `get_inbox_conversation`, not the inbox preview.
- **Front** — the shared *Email Replies* inbox (`inb_nh7fs`), which aggregates
  replies across every rotating cold-outreach sending mailbox. Email only; its
  value is catching follow-ups that never resurface in Lemlist's inbox view.
- **Gmail** (`shaffy@myswimscore.com`) — every message from today and yesterday,
  inbox *and* sent, including threads where Shaffy is only cc'd.
- **Shopify — B2B contact form only.** The `"New customer message"` submissions
  from the website. **B2C patient orders and customers are out of scope** and are
  never read.

## Two traps worth knowing before you edit this

Both are documented in `hubspot.json` because both caused real, wrong data:

- **Front and Lemlist both lie about message direction.** Front's `kind` /
  `origin.kind` routinely label SwimScore's own reply-in-thread as
  inbound-from-customer; Lemlist tags our outbound as an inbound `emailsReplied`
  activity, sometimes with a positive interest score. Read the body and the
  signature to decide who actually sent something — never the metadata.
- **`search_threads` silently truncates long threads.** In production it returned
  4 messages of a real 16-message thread, hiding weeks of activity including the
  actual last touch, with no error. Use `get_thread` on anything that looks like
  a running exchange.

## Security

Credentials never belong in this repo. `hubspot.json` holds only IDs and rules
(no tokens) and is committed; real tokens go in env secrets or a gitignored
`hubspot.local.json`. `accounts.json`, `.env`, and key files are gitignored. The
workflow runs through connected MCP connectors (HubSpot, Gmail, Slack, Lemlist,
Front, Shopify) and does not read or store API keys.
