# W0 — Daily New-Deal Discovery (create + link + note in HubSpot)

**Run this first each morning (~06:50 Europe/Amsterdam), before W1.** It reads the
last day of inbound mail **and fresh Lemlist replies**, finds **genuine new
opportunities that aren't in HubSpot yet**, and for each one **creates the deal,
links the contact and company (creating them if needed), and adds a note**
summarizing the thread — so every real opportunity is captured in HubSpot the
moment it appears.

> **Pipeline:** **W0 (this) → W1 `hubspot-sync.md` → W2 `daily-open-items.md`.**
> W0 *creates* new deals; W1 *updates* every existing deal (notes + stages from
> the full conversation); W2 derives per-client To-Dos in Notion. W0 runs first so
> W1 then treats anything W0 created as a normal existing deal.

**Config:** `hubspot.json` → `newDealDiscovery` block. Gated by
`newDealDiscovery.enabled`. New deals go into `defaultPipeline` at `defaultStage`,
owned by `defaultOwnerId` (all defined there).

---

## 0. Preflight
HubSpot (required, write — `get_user_details`), Gmail (native), Lemlist
(`get_campaigns`), Shopify (`get-shop-info`). Slack optional (warm intros that land
in chat). Open with the readiness line; skip a down source loudly rather than
failing.

## 1. Pull yesterday's new inbound
- **Gmail:** `in:inbox newer_than:<lookbackDays>d` (default **2 days**; widen on
  the first run after a gap), paginate fully. Native Gmail only. This also catches
  **Shopify website contact-form / wholesale inquiries** sent to
  `info@myswimscore.com` — those are often **B2B clinic leads**, keep them.
- **Lemlist (full inbox):** `get_inbox_conversations(listId="teamConversations")`,
  **paginated** (page/limit, max 50) so no reply is missed — campaigns send from
  multiple personas, so `teamConversations` (whole team) beats the user-scoped
  `myConversations`. Keep conversations with a recent `lastRepliedAt` in the
  window and/or `isYourTurn:true`. For each candidate, call
  `get_inbox_conversation(contactId)` to read the thread and the AI signal:
  `aiLeadInterest=positive` (a real reply showing interest) is a textbook new
  opportunity; `aiLeadInterest=negative` (e.g. "No thanks.") is **not** a new deal
  — skip it (and if it matches an existing deal, W1 proposes Closed Lost). Match
  the lead email / company domain against HubSpot before creating (dedup).
- **Shopify (B2B only):** website inquiries indicating clinic/wholesale interest.
  **Do not** create deals for individual B2C patient orders — B2C is out of scope.
- Optionally **Slack:** shared/Connect channels for "intro" / "connecting you"
  messages.

## 2. Keep only genuine new opportunities — SwimScore's ICP

**Who SwimScore sells to.** SwimScore (www.myswimscore.com) provides an **at-home
male-fertility testing kit** — semen analysis, DNA fragmentation, and a hormone
panel in one clinically-validated mail-in kit, with a **clinic dashboard** (real-time
patient status + results) and patient SMS updates, on a **clinic-pay model** (clinic
pays SwimScore, sets its own patient price). The target customer is a **clinic /
provider that wants to use SwimScore inside their patient workflow** — order kits for
patients, track status, review results, optionally mark up the price.

**KEEP** — a real clinic/provider showing intent to **offer SwimScore to their
patients**, e.g.:
- Fertility / IVF / reproductive-endocrinology clinics, urology & men's-health
  practices, OB/GYN, **fertility acupuncture / naturopathic** practices, cryobanks,
  and fertility-benefits platforms.
- Signals: wants at-home SA / DNA-frag / hormone testing for their male patients;
  wants visibility / follow-up on the male partner; wants to mark up price to
  patients; asks about onboarding, the clinic portal, pricing, or a pilot; a warm
  intro ("connecting you with…"); a booked discovery call; a positive Lemlist reply;
  a B2B website contact-form inquiry.

**EXCLUDE:** individual **B2C patients** buying a kit for themselves (out of scope —
B2B only); **research/academic labs** working on animal/preclinical samples (NIH,
university labs, etc.); automated/SaaS notifications (HubSpot, Stripe, Google,
LinkedIn, Calendly, billing/receipts); internal colleagues (your own sending
domains); vendors pitching **to** SwimScore; recruiting/hiring applicants and dev
shops; anything personal; and anyone **already a deal** — dedup the sender's email /
company domain / name against existing HubSpot deals (and their contacts/companies).
Clinics that do all testing in-house and explicitly decline are **not** new deals.

**Confidence:** create automatically for clear opportunities (proposal/pricing,
booked call, explicit inquiry, positive Lemlist reply). For thin/low-confidence
intros (personal-email "introductions" with no service text), **list in the digest
for review** instead of creating — controlled by `newDealDiscovery.minConfidence`.

## 3. For each new opportunity — create + link + note (HubSpot)
Use `manage_crm_objects`. Dedup before every create.
1. **Company** (skip for personal-domain emails) — `search_crm_objects(companies)`
   by domain; reuse on match, else create `{name, domain}`.
2. **Contact** — `search_crm_objects(contacts)` by email; reuse on exact match,
   else create `{email, firstname, lastname}`. Associate to the company.
3. **Deal** — create a `deals` object: `{dealname, pipeline:<defaultPipeline>,
   dealstage:<from intent>, hubspot_owner_id:<defaultOwnerId>}`, **associated** to
   the contact and company. Stage from intent (ids from `hubspot.json`):
   - proposal/pricing → `Proposal Sent`
   - booked intro/discovery call → `Discovery Scheduled`
   - positive reply / explicit inquiry, no call yet → `Outreach Sent`
   - thin warm intro only → `Target Identified` (or hold for review).
4. **Note** — create a `notes` object stamped `[new-deal YYYY-MM-DD]`, associated
   to the deal, summarizing the thread: who reached out / how (channel), the
   service in play, key points, pricing/proposal status, latest date, next step.

## 4. Idempotency / safety
- **Dedup hard** before creating: by contact email, company domain, and fuzzy
  name against existing deals — never create a duplicate of an existing deal.
- If a contact/company already exists, reuse its id.
- Stamp notes `[new-deal …]`; if a deal for this contact/domain already exists,
  hand it to W1 instead (don't create).
- All creates are logged in the digest; nothing is deleted.

## 5. Output — discovery digest
**Preflight** ✅/⚠️ · **Created** (`Deal — contact — stage — why`, with contact/
company created vs reused) · **Proposed (low-confidence, not created)** ·
**Sources scanned** / **skipped buckets** (SaaS/automated, internal, vendors,
recruiting, already-deals).

## Scheduling
Claude Code web scheduled session, **daily ~06:50 Europe/Amsterdam**, **Sonnet**,
before W1. Enable via `newDealDiscovery.enabled`. Or run the whole chain with the
`/daily-crm` command (W0 → W1 → W2 in order).

**Prompt:**
> Run W0, the new-deal discovery in `new-deal-discovery.md`. Preflight (HubSpot +
> Gmail + Lemlist), pull yesterday's inbound mail and fresh Lemlist replies, keep
> only genuine new opportunities not already in HubSpot, then for each create the
> deal + link company + link contact + add a thread-summary note. Finish with the
> discovery digest.
