---
title: "Competition Capability Matrix v1 (evidence-classified, 2026-07-14)"
type: "plan"
date: "2026-07-14"
status: "active"
project: "DAI"
slice: "Release Timeline, Architecture Runway, and Multisport Sequence Review v1"
repos:
  dai: "unchanged (evidence at 85a8831)"
  dai-vault: "docs-only"
tags:
  - planning
  - multisport
  - architecture
related:
  - "06 Execution/plans/cloud-and-multisport-runway-v1.md"
  - "06 Execution/reports/release-architecture-review-2026-07-14-v1.md"
---

# competition capability matrix v1

Every cell is repository- or corpus-evidenced at dai `85a8831`; `unknown` means no
evidence exists -- absence of evidence is never converted into confidence. Proof levels:
implemented -> fixture-proven -> live-read-proven -> paid-run-proven -> settlement-proven
-> buyer-delivery-proven -> pilot-proven.

## proof-level summary

| capability | MLB | NBA | NFL | NCAAMB | NCAAF |
|---|---|---|---|---|---|
| competition code + catalog entry | implemented (buyer-ready) | implemented (buyer-ready flag) | implemented (smoke) | implemented (smoke) | implemented (smoke) |
| analyzer path | paid-run-proven (285 v1 + 16 v2 runs) | paid-run-proven (legacy smoke runs, May) | paid-run-proven (legacy smoke) | live-routable, unproven | live-routable, unproven |
| stable event identity | settlement-proven (statsapi gamePk, DH-safe WI-0006/0009) | NOT implemented (odds event id only) | NOT implemented | NOT implemented | NOT implemented |
| source readiness | settlement-proven (regime classifier) | NOT implemented (MLB-only classifier) | NOT implemented | NOT implemented | NOT implemented |
| grounded evidence depth | paid-run-proven (starters + 9-book depth) | thin (market + sharp_public + ESPN rest) | thin (market + sharp_public) | thin, unproven | thin, unproven |
| prompt recipe (registry v2) | paid-run-proven (mlb.* recipes only) | none | none | none | none |
| buyer brief/recap contract | buyer-delivery-ready (WI-0011/12; live-verified) | frame-compatible, content thin | frame-compatible, content thin | unknown | unknown |
| outcome + evaluation semantics | settlement-proven | compatible (no ties) | compatible via draw (ties) | COLL? tournaments break two-side model | COLL? same |
| reconciliation | settlement-proven (provider+externalId) | blocked on identity provider | blocked | blocked | blocked |
| operator runbook coverage | complete (WI-0013) | none | none | none | none |
| pilot | in progress (V1.0) | none | none | none | none |

## per-competition assessments (evidence-classified)

**MLB** -- schedule: statsapi (settlement-proven). Identity: gamePk incl. same-day
repeat handling (adversarially proven). Readiness: starter x market regimes. Markets:
moneyline consensus, 9-book de-vigged depth. Starters/lineups: probable pitchers +
quality enrichment (WI-0005-hardened cache). Recipe: mlb.pregame.*.v2 registry route.
Outcomes: full status set incl. postponement doctrine (823357). Cost: ~$0.0007/run
measured. Fixtures: extensive (1235-test suite). THE reference implementation.

**NBA** -- schedule: odds-api (generic) + ESPN rest context. Identity: NO stable
provider id path for settlement (odds event id is capture-side; no statsapi-equivalent
wired) -- THE gating gap. Readiness: none (classifier is MLB-only). Evidence: market +
sharp_public (ActionNetwork, pro-only) + rest; no lineup/injury depth. Recipe: none
(live prompt path only). Outcomes: two-side, no ties -- clean. Buyer-ready flag already
set (frontend-exposable). Corpus: legacy May smoke runs exist, never settled at
identity grade. Season tips late Oct. **Best-positioned second sport.**

**NFL** -- schedule: odds-api. Identity: none stable. Evidence: market + sharp_public;
injuries/depth charts (the sport's dominant evidence) NOT implemented -- the heaviest
evidence-source build of the pro sports. Outcomes: ties map to draw (works). Cadence:
weekly -- slow qualification-ladder learning. Season: Sept (earlier than NBA but the
evidence build + slow cadence outweigh the calendar). **Third.**

**NCAAMB / NCAAF** -- catalog + seed data + analyzer routing exist (smoke). Structural
risks beyond "another code": ~360/130 team identity normalization; provider coverage
variance (unknown); market depth much thinner (unknown); roster/lineup volatility;
neutral-site + tournament/advancement semantics BREAK the two-team home/away
single-winner stack (RunEvaluator + recap winner model); thin-data/no-position expected
dominant. **Feasibility spike required before any ladder entry; own release.**

## seam classification (from the 85a8831 inventory)

GENUINELY GENERIC: competition catalog + /api/competitions + buyer-ready gating;
Angular selection (fully data-driven); odds retrieval keyed by catalog OddsApiKey;
reconciliation matcher (provider + externalGameId exact); outcome status vocabulary;
buyer stance vocabulary + brief/recap frames + claim-safety; metering; run/outcome/
evaluation/tenancy/auth core; run-artifact-calibration.ps1 (-Competition).

GENERIC WITH SPORTS ASSUMPTIONS: CompetitionMatchupInput (two-team scheduled event);
SportsRetriever (generic orchestration, per-sport if/else branches); signal vocabulary
(fixed enumerated set); evaluation (two-side winner); brief/recap identity framing
(away@home).

MLB-SPECIFIC: MlbStarterClient/MlbEventResolver (statsapi identity + DH resolution);
SourceReadinessClassifier regimes; dataregime.py + all registry recipes;
check-settlement-finals.ps1 + preflight default provider; starter/bullpen evidence.

PROFESSIONAL-SPORTS-SPECIFIC: ActionNetwork sharp_public (football+basketball wiring);
GameIdentity season logic (hardcoded family windows).

LIKELY COLLEGIATE-INCOMPATIBLE: the outcome/evaluation/recap two-team single-winner
model vs tournaments and neutral sites; identity at 360-team scale.

"GENERIC IN NAME ONLY" (recorded pressure, no refactor now): SportsRetriever's if/else
dispatch (a capability contract would replace it -- WI-0016 seam); the fixed signal
vocabulary; SourceReadiness as a concept (currently an MLB class, should be a
per-competition contract).
