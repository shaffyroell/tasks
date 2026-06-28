# W0 — Daily New-Deal Discovery (create + link + note in Attio)

**Run this first each morning (~06:50 Europe/Amsterdam), before W1.** It reads the
last day of inbound mail, finds **genuine new opportunities that aren't in Attio
yet**, and for each one **creates the deal, links the company and person
(creating them if needed), and adds a note** summarizing the thread — so every
real opportunity is captured in the CRM the moment it appears.

> **Pipeline:** **W0 (this) → W1 `attio-ingest.md` → W2 `daily-open-items.md`.**
> W0 *creates* new deals; W1 *updates* existing deals (notes + stages); W2 derives
> per-client To-Dos in Notion. W0 runs first so W1 then treats anything W0 created
> as a normal existing deal.

**Config:** `attio.json` → `newDealDiscovery` block (see `attio.example.json`) +
`client.json`. Gated by `newDealDiscovery.enabled`.

---

## 0. Preflight
Attio (required, write), Gmail (`stack.email`, scoped to `stack.email.domains`).
Slack optional (for warm intros that land in chat). Open with the readiness line;
skip a down source loudly rather than failing.

## 1. Pull yesterday's inbound
- Gmail: `in:inbox after:<lookbackDays ago>` (default **2 days**; widen on the
  first run after a gap), paginate fully. Native Gmail only.
- Optionally Slack: shared/Connect channels for "intro"/"connecting you" messages.

## 2. Keep only genuine new opportunities
**KEEP:** a real external party showing intent in TechTower's services (AI
automation, lead-gen / data enrichment, fundraising / investor sourcing,
exec-search / recruiting, AI agents) — an inbound inquiry, a warm intro
("introducing…/connecting you with…"), a booked intro/discovery call, or a
proposal/pricing discussion.

**EXCLUDE:** automated/SaaS notifications (Attio, Mercury, Stripe, Google,
LinkedIn, Calendly, Substack, billing/receipts), internal colleagues (TechTower's
own sending domains only: `@techtower.ai`, `@runtechtower`, `@gettechtower`,
`@techtowerops.com`, `@usetechtower.com`, …), vendors pitching **to** TechTower,
**recruiting/hiring** applicants and dev
shops, anything personal / SwimScore, and anyone **already a deal** — dedup the
sender's company/domain/person against existing Attio deals (and the people/
companies they link to).

**Confidence:** create automatically for clear opportunities (proposal/pricing,
booked call, explicit inquiry). For thin/low-confidence intros (personal-email
"introductions" with no service text), **list in the digest for review** instead
of creating — controlled by `newDealDiscovery.minConfidence`.

## 3. For each new opportunity — create + link + note
1. **Deal** — `create-record(object="deals", {name, stage, owner, comment})`.
   Stage from intent: proposal/pricing → `5. Proposal sent`; live scoping →
   `4. Scope agreed`; booked intro/discovery call → `2. Planning call`; positive
   inbound, no call yet → `1.  Positive reply`. `comment` = one-line source tag +
   contact email. Owner = `newDealDiscovery.ownerId`.
2. **Company** (skip for personal-domain emails) — find-or-create in `companies`:
   search by domain; reuse if it matches, else `create-record({name, domains:[…]})`.
   If a create returns a domain conflict, reuse the existing id from the error.
3. **Person** — find-or-create in `people`: search by email; reuse on exact match,
   else `create-record({name:[{first_name,last_name,full_name}], email_addresses:[…]})`.
4. **Link** — `update-record(deals, {associated_people:[…], associated_company:[…]})`
   (omit company for personal-domain contacts).
5. **Note** — `create-note(parent_object="deals", parent_record_id, title, content)`
   stamped `[new-deal YYYY-MM-DD]`, summarizing the thread: who reached out / how,
   the service in play, key points, pricing/proposal status, latest date, next step.

## 4. Idempotency / safety
- **Dedup hard** before creating: by contact email, company domain, and fuzzy
  name against existing deals — never create a duplicate of an existing deal.
- Use `allow_duplicates:false` semantics; if a person/company already exists,
  reuse it.
- Stamp notes `[new-deal …]`; if a deal for this contact/domain already exists,
  hand it to W1 instead (don't create).
- All creates are logged in the digest; nothing is deleted.

## 5. Output — discovery digest
**Preflight** ✅/⚠️ · **Created** (`Deal — contact — stage — why`, with person/
company created vs reused) · **Proposed (low-confidence, not created)** · **Threads
scanned** / **skipped buckets** (vendors, recruiting, already-deals).

## Scheduling
Claude Code web scheduled session, **daily ~06:50 Europe/Amsterdam**, **Sonnet**,
before W1. Enable via `newDealDiscovery.enabled`. Or run the whole chain with the
`/daily-crm` command (W0 → W1 → W2 in order).

**Prompt:**
> Run W0, the new-deal discovery defined in `new-deal-discovery.md` in this repo.
> Preflight, pull yesterday's inbound, keep only genuine new opportunities not
> already in Attio, then for each create the deal + link company + link person +
> add a thread-summary note. Finish with the discovery digest.
