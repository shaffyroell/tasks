# SwimScore To-Dos — internal Kanban board

The internal execution board for SwimScore (at-home male-fertility testing: semen
analysis, DNA fragmentation, hormone panel; sold to fertility clinics on a
clinic-pay model + B2C patients). The **CEO sweep** (`/daily-crm`) routes every
**internal** item here; **account** items go to HubSpot deals instead.

- **Board:** https://app.notion.com/p/bf5ee5540bd04fd69f10a2336656fd70
- **Data source id:** `3b132f28-f83c-4966-ae75-3894cf9b76d2`
- Wired into `hubspot.json` → `internalTracker`.

## Views
- **Board by Pillar** (the Kanban — one column per pillar).
- **Board by Status** (To Do / In Progress / Blocked / Done).

## The seven pillars
| Pillar | What lives here |
|---|---|
| **Onboarding** | Bringing signed clinics live — slide deck + sample kits, onboarding calls, MSA/BAA, portal setup walkthroughs |
| **New sales** | B2B pipeline + outbound — clinic deck sends, Lemlist campaigns & list hygiene, closing clinics like Onto Health |
| **Marketing** | B2C growth + brand — ad spend/PDP/conversion, SEO blog cadence, Trustpilot/reviews, social content |
| **Research** | Product/market questions — pricing ($325), HSA/FSA + Superbill, Tasso, repeat tests, white-label vs co-branded, LTV upsell flows (medication/supplement), advisors, lab plan |
| **Clinic & patient portal** | Product/dev — clinic portal HIPAA compliance, status pull, deploys, patient-portal v2, patient-consent flow |
| **Support** | Patient/clinic support & comms — OTP/login, transactional emails, staging test logins, CSAT |
| **Finance** | Billing & costs — clinic-pay model, MDI per-case rate, Stripe Patient Pay |

## Key Goals table
A separate **Key Goals (Q3 2026)** database lives under the
[Goals for SwimScore](https://app.notion.com/p/38de4b44810180329d8eeebefad8a4a7)
page: https://app.notion.com/p/19ed2528d874454eb39a783ae94d42b1 — one row per Q3
goal (`Category`: Product / Customer experience / Growth model / Lifetime value /
Industry positioning / Lab outlook; `Status`: Not started / On track / At risk /
Blocked / Done). It has a two-way relation with the To-Dos board (`Goal` on a
to-do, `Sub-tasks` on a goal) so every to-do can trace to the goal it moves.
When sweeping, check goal coverage — a goal with no linked to-do, or only bug-fix/
execution items and no way to measure progress, needs a new to-do pointed at it.

## Schema
`Task` (title) · `Pillar` (the 7 above) · `Status` (To Do / In Progress / Blocked /
Done) · `Priority` (High / Medium / Low) · `Owner` (Shaffy / Syb / Flo / Dmytro /
Harsh / Thijs / Team) · `Source` (Slack / Granola / Lemlist / Email / HubSpot) ·
`Link` (permalink) · `Ref` (stable dedup key) · `Created` · `Goal` (relation to
Key Goals).

## How W2 keeps it current
Each run (`daily-open-items.md`) reads HubSpot + fresh Gmail/Slack/Granola/Lemlist,
then for every internal item: **dedups on `Ref`**, **checks what each existing item
is for and updates its `Status`** (advance / mark Done), and adds genuinely new
items under the right pillar — every line in the house style per `STYLE.md`
(verb-first, concise, no arrows). It never deletes; completion = `Status = Done`.

> Seeded 2026-06-28 from Slack (#1-business-strategy, #clinic-portal-development,
> #12-pipeline-clients, #6-legal, #daily-status-report), Granola call notes, and
> Lemlist replies.
