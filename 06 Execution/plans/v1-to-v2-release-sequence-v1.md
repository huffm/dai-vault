---
title: "V1-to-V2 Release Sequence, Feature Hierarchy, and Gates v1"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "Release Timeline, Architecture Runway, and Multisport Sequence Review v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - planning
  - release
  - roadmap
related:
  - "06 Execution/reports/release-architecture-review-2026-07-14-v1.md"
  - "06 Execution/plans/cloud-and-multisport-runway-v1.md"
  - "06 Execution/plans/competition-capability-matrix-v1.md"
---

# v1-to-v2 release sequence, feature hierarchy, and gates v1

Verdicts live in the review report; this document carries the ladder, decomposition,
dates, and gates. Sizes are relative (XS/S/M/L/XL) with confidence high/med/low -- no
hour estimates. Nothing here authorizes work; every paid/write/implementation step keeps
its own gate.

## 1. release ladder

| release | promise | boundary evidence |
|---|---|---|
| **V1.0 MLB concierge private pilot** (now -> 08-21) | one operator host, manual selection/generation/email/Stripe/settlement, canonical markdown brief + recap, no hosted app, no multisport promise | ALL code integrated at `85a8831`; remaining = RC drill + commercial validation |
| **V1.1 repeatable MLB pilot ops** (post-08-21 gate) | proven repeat use, operator minutes trending down, refined buyer copy (WI-0018), sturdier recovery, better delivery/entitlement workflow; NO platform expansion | requires >= 1 retained paying buyer |
| **V1.2 cloud-operated single-sport** (gated, ~Sept-Oct) | stage-2 topology (runway doc): deployed API+agent service, managed SQL, env separation, secrets, CI/CD, observability, rollback; db-coordinated duplicate enforcement only if multi-instance; possibly still MLB-only | requires the cloud gate (below) |
| **V1.3 second professional sport = NBA** (gated, ~Oct-Nov) | competition capability contract + NBA adapter through the full qualification ladder (runway doc) to a bounded NBA pilot | requires second-sport gate; season tips late Oct |
| **V2.0 collegiate feasibility -> support** (option, Dec+) | its OWN release: ~360-team identity normalization, provider-coverage audit, thin-data-dominant posture, tournament semantics; never "another competition code" | requires NBA ladder proven + collegiate gate |

## 2. feature hierarchy (release -> epic -> feature -> slice)

Legend per row: value | owner (platform/niche/frontend/ops) | prereq | feedback | size/conf
| P = required before payment, C = before cloud, S = before second sport.

### V1.0 epic: commercial validation  (owner: ops/commercial -- NO code)
- **Target-buyer confirmation** -- validates segment | ops | none | conversation signal | XS/high | P
- **Outreach wave 1 (>= 10 genuine contacts)** -- starts loop 2 | ops | send-ready copy (exists) | interest/objections | S/high | P
- **Sample delivery (existing 823845 brief+recap pair)** -- proof artifact, zero spend | ops | none | comprehension feedback | XS/high | P
- **Fresh prospect-chosen samples (<= 4, 1 paid call each)** | ops | RC gate 1 | conversion signal | S/high | P
- **Stripe payment link (live) + entitlement verification** | ops (Stripe=truth) | test-mode dry-run done | payment = validation | XS/high | P
- **First paid delivery** | ops | RC verdict + paying buyer | THE gate evidence | S/high | P
- **Feedback interview + repeat-use tracking + attention accounting** | ops | ledger (exists) | retention data | S/high | P(ongoing)
- **Retention decision at 08-21** | ops | 2 weeks pilot data | gate input | XS/high

### V1.0 epic: release-candidate proof  (owner: ops; all four drill gates separate)
- **RC gate 1 pregame drill on TB@BOS DH 824766/824737** (opening checks, payment
  test-mode dry-run, screening, canary + duplicate-409 proof, brief render+"delivery",
  forced source-outage recovery, shutdown) | drill package caps 2 calls/2 runs | S/high | P
- **RC gate 2 settlement + recap + idempotency proof** | day after finals | S/high | P
- **RC record + final verdict** | both gates | XS/high | P
This single drill also retires the dormant doubleheader evidence packet (distinct-gamePk
creation + per-game identity) at no extra spend.

### V1.2 epic: cloud readiness (ALL gated by the cloud gate; C)
- topology decision (stage 2, runway doc) XS/high | container packaging (Dockerfiles
  exist; verify + pin) S/high | env config + secrets (Key Vault-class) M/med | managed
  SQL + EXPLICIT MIGRATION PATH (repo currently dev-bootstraps; must be proven) M/med |
  CI (build+test) M/high | deployment pipeline + smoke M/med | health/readiness probes
  (health exists; readiness to add) S/high | structured logs + metrics + alerting M/med |
  cost controls XS/med | rollback + backup/restore M/med | domain+TLS S/high | prod
  Entra values + auth verification S/high | tenant onboarding (manual, documented)
  S/med | multi-instance duplicate enforcement (WI-0015 -- ONLY if topology needs it;
  migration) M/med | background-job ownership decision (generation stays synchronous
  single-instance until evidence says otherwise) XS/high | operational support model
  (runbook cloud addendum) S/high

### V1.3 epic: competition portability (gated by second-sport gate; S)
- **competition capability descriptor** (platform-owned declaration: markets, evidence
  inputs, identity provider, outcome semantics, buyer-readiness) M/med | schedule
  adapter contract S/med | canonical event identity + same-day repeat handling (exists
  for MLB; contract-ize) M/med | team normalization S/med | game-state vocabulary audit
  (ties/OT per sport) S/high | source-readiness contract (generalize regimes) M/med |
  market-type contract S/med | evidence-source contract (starters -> sport-equivalent:
  NBA lineups/rest, NFL depth/injuries) L/low | prompt recipe per sport (registry
  pattern exists) M/med | evidence-sufficiency + no-position policy per sport S/high |
  outcome adapter + evaluation semantics S/high | reconciliation (already keyed
  provider+externalId -- verify per provider) S/high | brief/recap presentation deltas
  S/med | cost profile per sport XS/high | sport qualification tests (ladder stages 1-3)
  M/med

### V1.3 epic: NBA onboarding (after contract; S) -- ladder stages in the runway doc
adapter impl M/med -> fixtures + adversarial identity M/med -> live read-only S/high ->
bounded paid generation S/high (gated) -> identity/buyer-contract verification S/high ->
settlement verification S/high -> repeated cohort M/med (gated) -> runbook addendum
S/high -> buyer sample + pilot (commercial gates) S/med.
NFL: same ladder, later; weekly cadence slows every stage; injuries/depth-chart evidence
is a heavier evidence-source slice (L/low).
Collegiate (NCAAB then NCAAF): feasibility SPIKE first (provider coverage + identity
scale + market depth on real data, read-only, XS-S) before any ladder entry.

## 3. dependency + parallel map (committed horizon)

```
07-15..16  OUTREACH wave 1 (parallel, no deps)     STRIPE test-mode link created
07-17      RC GATE 1 drill (TB@BOS DH)  ||  outreach continues
07-18      RC GATE 2 settlement + recap -> RC VERDICT (13 days early)
07-18..08-06  samples (dep: gate 1) -> conversion -> live Stripe link (dep: none)
<= 08-07   FIRST PAID DELIVERY (dep: RC verdict + paying buyer)  [earlier allowed]
08-07..21  pilot ops >= 3 delivery days/week; ledger; feedback interview (~08-14)
08-21      PILOT EVALUATION GATE
```
Truly sequential: gate1 -> fresh samples; RC verdict + payment -> paid delivery;
finals -> gate 2. Everything commercial runs parallel to everything operational.
Optional: extra samples beyond conversion. Creates-no-buyer-feedback (deprioritized):
any further internal tooling, refactors, docs beyond gate records.

## 4. forecast horizon (2026-08-24 -> ~2026-10-30, CONDITIONAL on the 08-21 gate)

| window | work | dependency | confidence |
|---|---|---|---|
| 08-24..09-05 | V1.1 pilot hardening: WI-0018 buyer copy polish (uses real feedback), ops tuning, second buyer outreach | retained buyer | med |
| 09-08..10-10 | V1.2 cloud stage 2 (WI-0014): ACA + managed SQL + CI/CD + observability + rollback | CLOUD GATE passed | med |
| 09-29..10-17 | WI-0016 competition capability contract (design + extraction, no new sport code) | SECOND-SPORT GATE passed | med |
| 10-13..10-30 | WI-0017 NBA qualification stages 1-5 (through live read-only; paid stages gated) | WI-0016; NBA preseason data | low |

## 5. option horizon (Nov-Dec 2026 -- options, NOT commitments)

NBA bounded pilot (ladder stages 6-12) | collegiate feasibility spike (NCAAB in season)
| tenant onboarding / stronger packaging (only if >= 3 buyers) | automation slices
justified line-by-line by the pilot ledger (WI-0019) | multi-instance enforcement
(WI-0015) only if topology actually demands it.

## 6. gates

| gate | evidence required | threshold | decision space | must NOT start before | exceptions |
|---|---|---|---|---|---|
| first paid delivery | RC verdict + Stripe receipt + entitlement ledger row | 1 paying buyer | proceed/hold | real buyer delivery | none |
| continued MLB investment (08-21) | ledger: payment, repeat use, minutes, feedback | >= 1 retained payer AND >= 3 reads/wk | proceed / simplify / stop | V1.1 work | strong qualitative signal w/ operator judgment |
| cloud investment | retention + operator-minutes trend + hosting need | >= 2 payers retained OR > 45 min/day sustained OR buyer requires hosted | proceed/hold | any WI-0014 build | long-lead design-only spike allowed |
| second-sport investment | MLB pilot economics + NBA season proximity | MLB gate green + < 30 min/day | proceed/hold | WI-0016 implementation | contract DESIGN doc allowed early |
| collegiate investment | NBA ladder complete + feasibility spike verdict | spike shows identity+coverage workable | proceed/hold/stop | any collegiate code | none |
| automation | ledger: same manual step painful >= 3x/wk for 2 wks | documented repetition | proceed/hold | building the automation | operator-safety fixes |
| multi-tenant self-service | >= 3 tenants demanding onboarding | demand, not forecast | proceed/hold | onboarding code | none |
| multi-instance deployment | topology evidence stage-2 insufficient | measured, not anticipated | proceed/hold | WI-0015 | none |
Payment and repeat use ALWAYS outrank internal technical completion as evidence.

## 7. proposed next work items (<= 6, NOT minted, each gated)

| id (proposed) | title | problem | scope sketch | excl. | acceptance sketch | owner | release | target | size | dep |
|---|---|---|---|---|---|---|---|---|---|---|
| WI-0014 | Cloud Deployment Stage 2 v1 | local-only host blocks hosted access + durable ops | ACA + Azure SQL + migrations + CI/CD + secrets + observability + rollback per runway stage 2 | multi-instance; autoscale; new features | env rebuildable from declared config; smoke + rollback proven; suites green in CI | platform | V1.2 | Sept (gated) | L | cloud gate |
| WI-0015 | DB-Coordinated Duplicate Enforcement v1 | process-local gate unsafe multi-instance | filtered unique index on active-run identity (or applock) + migration + concurrency tests | any other schema change | two instances cannot double-create; guard tests pass against db mechanism | platform | V1.2+ | when multi-instance gate opens | M | WI-0014 |
| WI-0016 | Competition Capability Contract v1 | sport logic hardwired as MLB classes; second sport would fork the pipeline | capability descriptor + identity/schedule/evidence/outcome adapter contracts; MLB re-homed as first adapter, byte-identical behavior | any NBA code; prompt changes | MLB regression suites green; descriptor drives buyer-ready list + readiness | platform/niche seam | V1.3 | late Sept (gated) | L | second-sport gate |
| WI-0017 | NBA Qualification Execution v1 | second sport unproven | ladder stages 1-8 for NBA per runway doc | collegiate; NFL | each ladder stage's exit criteria met w/ evidence artifacts | niche | V1.3 | Oct-Nov (gated, staged spend) | XL | WI-0016 |
| WI-0018 | Buyer Copy Polish v1 | ledger entry 21 + real buyer feedback | tone/cadence/label polish on brief+recap copy only | scoring/confidence/claims changes | claim-safety suites green; buyer feedback addressed | niche/frontend | V1.1 | late Aug (gated) | S | retained buyer |
| WI-0019 | Evidence-Justified Ops Automation v1 | repeated manual pain (if ledger proves it) | automate ONLY the specific ledger-proven steps | speculative automation | minutes/delivery drop measurably; no new paid paths | ops/platform | V1.1+ | Sept+ (gated) | M | automation gate |

## 8. immediate non-code commercial actions (no WI needed, start 07-15)

1. Select the single outreach venue (per buyer-validation-brief section 6).
2. Send >= 3 first-contact messages 07-15/16 using the approved copy; >= 10 by 07-25.
3. Prepare the demonstration sample from the settled 823845 brief + recap markdown.
4. Create the Stripe payment link and complete the TEST-MODE dry-run inside RC gate 1.
5. Start the delivery ledger + operator-time log files from the templates.
6. Schedule the ~08-14 buyer feedback interview slot in advance.
