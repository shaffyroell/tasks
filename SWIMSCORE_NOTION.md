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

## The six pillars
| Pillar | What lives here |
|---|---|
| **Onboarding** | Bringing signed clinics live — slide deck + sample kits, onboarding calls, MSA/BAA, portal setup walkthroughs |
| **New sales** | Pipeline + outbound — clinic deck sends, Lemlist campaigns & list hygiene, retargeting, closing clinics like Onto Health |
| **Research** | Product/market questions — pricing ($325), HSA/FSA + Superbill, Tasso, repeat tests, white-label vs co-branded |
| **Clinic & patient portal** | Product/dev — clinic portal HIPAA compliance, status pull, deploys, patient-portal v2, patient-consent flow |
| **Support** | Patient/clinic support & comms — OTP/login, transactional emails, staging test logins |
| **Finance** | Billing & costs — clinic-pay model, MDI per-case rate, Stripe Patient Pay |

## Schema
`Task` (title) · `Pillar` (the 6 above) · `Status` (To Do / In Progress / Blocked /
Done) · `Priority` (High / Medium / Low) · `Owner` (Shaffy / Syb / Flo / Dmytro /
Harsh / Thijs / Team) · `Source` (Slack / Granola / Lemlist / Email / HubSpot) ·
`Link` (permalink) · `Ref` (stable dedup key) · `Created`.

## How W2 keeps it current
Each run (`daily-open-items.md`) reads HubSpot + fresh Gmail/Slack/Granola/Lemlist,
then for every internal item: **dedups on `Ref`**, **checks what each existing item
is for and updates its `Status`** (advance / mark Done), and adds genuinely new
items under the right pillar — every line in the house style per `STYLE.md`
(verb-first, concise, no arrows). It never deletes; completion = `Status = Done`.

> Seeded 2026-06-28 from Slack (#1-business-strategy, #clinic-portal-development,
> #12-pipeline-clients, #6-legal, #daily-status-report), Granola call notes, and
> Lemlist replies.
