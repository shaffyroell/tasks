# Roadmap & scope notes

Things this sweep deliberately doesn't do, and things it could grow into —
tracked separately so the workflow itself stays focused.

## Where it stands

The pipeline backfill is done, and keeping it current is the daily workflow — see
[`.claude/skills/daily-crm/SKILL.md`](./.claude/skills/daily-crm/SKILL.md). It
reads Lemlist, Front, Gmail, and the Shopify contact form, writes each
conversation onto its deal, and posts a stale brief to `#daily-recap`.

As of the 2026-08-19 rewrite it is **enrichment-only**: it adds context and never
moves a deal between stages. Stage movement, and the judgement that goes with it,
is Shaffy's.

## Could grow into

- **Weekly digest** — the stale brief covers 4+ days quiet, split by owner. A
  weekly roll-up of everything open and stalled would complement it.
- **Closed-lost hygiene** — hard rejections are currently *suggested* for Closed
  Lost in the digest. Acting on them stays manual by design; a batched
  "approve these 8" flow would keep that judgement with a human while cutting the
  clicking.
- **Onboarding template** — a one-click checklist per new clinic (sample kit +
  portal setup + one-pager) once a deal reaches Portal onboarding.

## Out of scope, on purpose

- **Moving deal stages.** Removed entirely in the 2026-08-19 rewrite. Every
  earlier version of this repo that automated stage movement got it wrong often
  enough to be worth not doing.
- **Creating deals from Lemlist / Front / Gmail.** A separate automation creates a
  deal on every Lemlist reply. Only the Shopify contact form — which nothing else
  watches — can create here.
- **B2C Shopify orders and customers.** Individual patient purchases are not
  deals; only B2B website contact-form messages are read.
- **HubSpot Tasks.** Dropped in the rewrite — what's owed surfaces in the Slack
  stale brief and the digest's reply-owed list instead of a second queue.
- **Notion and Granola.** Removed as sources; the internal to-do board is no
  longer part of this workflow.
- **Microsoft Teams** — see `CONNECTIONS.md`. Not pursued.
