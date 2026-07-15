---
title: "AI Engineering Fitness Checks v1.1 (2026-07-15; WI-0020 corrections)"
type: "plan"
date: "2026-07-15"
status: "active"
project: "DAI"
slice: "DAI AI Engineering Hardening Catalog and Protocol Ready Queue v1"
repos:
  dai: "unchanged (evidence at 85a8831)"
  dai-vault: "docs-only"
tags:
  - architecture
  - fitness
  - governance
  - planning
related:
  - "06 Execution/plans/cloud-and-multisport-runway-v1.md"
  - "06 Execution/plans/ai-engineering-hardening-catalog-v1.md"
  - "06 Execution/plans/hardening-ready-queue-v1.md"
---

# ai engineering fitness checks v1.1

**CANONICAL COUNT (v1.1, WI-0020): Fourteen architecture fitness checks are currently
adopted: eleven existing checks and three newly added checks (FC-1, FC-4, FC-14).
The three conditional checks in section 2 (FC-C1 replicas-equals-one assertion,
FC-C2 reproducible deployment and rollback, FC-C3 CI enforcement) are defined but
CONDITIONAL: not adopted, not active, and NOT included in the adopted count of
fourteen. They remain gated by the cloud stage-2 deployment boundary / WI-0014.**

Minimal high-value checks. Extends (does not duplicate) the runway doc's fitness table
(cloud-and-multisport-runway-v1.md section 6); rows marked EXISTS cite their proof and
change nothing. Checks are rejected when maintenance burden is disproportionate to
CURRENT risk. Trigger vocabulary: per-suite-run (local full suite), per-slice (before
any integration), per-event (operational days), per-gate (cloud/sport gates).

## 1. adopted checks (14 total: 11 EXISTS + 3 ADD; conditional checks NOT counted here)

| # | check | status | type | location | trigger | failure action | maint. cost | risk controlled |
|---|---|---|---|---|---|---|---|---|
| FC-1 | canonical protocol vocabulary stable (retired names banned: interrogate.stress, discern.test, decide.choose; alias scaffolding) | ADD (queue G-05) | static test | DevCore.Api.Tests (or script) | per-suite-run | fail the suite; name the file | trivial | silent doctrine drift |
| FC-2 | buyer projections free of numeric confidence | EXISTS (WI-0011/12 claim-safety + sentinel suites) | automated tests | BuyerDecisionBriefTests / BuyerSettledRecapTests | per-suite-run | fail | none new | unsupported buyer claims |
| FC-3 | every configured model has pricing coverage | EXISTS (WI-0013; test_model_metering) | automated test | agent-service tests | per-suite-run | fail; runbook R3 stops unmetered days operationally | none new | silent unmetered spend |
| FC-4 | every tool declares authorization + cost class | ADD (queue G-09) | static test + doc table | ToolRegistry test + vault table | per-suite-run | fail; unclassified tool cannot ship | low | undeclared spend/permission surface |
| FC-5 | every persisted decision carries traceable source identity | EXISTS (gamePk propagation tests; settlement residue 422 ADR 0006) | automated + write-guard | GamePkPropagationTests; SettlementProvenance | per-suite-run + per write | 422 refusal at write time | none new | orphaned/unattributable decisions |
| FC-6 | final buyer result uses persisted evaluation, never recomputation | EXISTS (recap renders from persisted evaluation; recap suites) | automated tests | BuyerSettledRecap tests | per-suite-run | fail | none new | result drift vs settled truth |
| FC-7 | tenant-scoped routes cannot leak existence | EXISTS (404 matrices; guard non-disclosure) | integration tests | AgentRunsAuthBoundary / MarketSnapshotTenantScoping | per-suite-run | fail | none new | cross-tenant existence leak |
| FC-8 | distinct gamePks never collide | EXISTS (DH resolution 20 tests + guard suite) | automated tests | MlbDoubleheaderResolutionTests, DuplicateRunGuardTests | per-suite-run | fail | none new | identity corruption |
| FC-9 | unknown evidence states fail closed | EXISTS (readiness eligibility + recap honest states) | automated tests | SourceReadiness + recap state tests | per-suite-run | fail | none new | optimistic behavior on missing data |
| FC-10 | every prompt recipe versioned + manifest-valid | EXISTS (manifest hash verify + 57 governance tests) | automated tests + load-time verify | test_manifest_integrity etc.; registry load | per-suite-run + process start | registry load refuses; fallback named | none new | silent prompt drift |
| FC-11 | deterministic exports byte-stable for fixed residue | EXISTS (brief 25 + recap 23 determinism tests); EXTEND edge states (CAT-SYN-C-1) | automated tests | export test suites | per-suite-run | fail | low (extension) | nondeterministic buyer artifacts |
| FC-12 | runtime returns to cold through the runbook | EXISTS operationally (Gate 0 rehearsal PASS 2026-07-15) | operational check | runbook sections 1+8 | per-event (every operating day) | R10 recovery; day stops if unrecoverable | low (procedural) | stale runtime state, port squatters |
| FC-13 | new competition passes the qualification ladder | EXISTS as process (runway ladder stages 1-12) | gate checklist (NOT automated) | runway doc; per-sport readouts | per-gate (second sport+) | ladder stage refuses entry | none until sport 2 | unproven sport reaching buyers |
| FC-14 | protocol completion invariant (successful compose => completed protocol; failed => null) | ADD (CAT-SYN-I-1) | automated test | SportsComposer tests | per-suite-run | fail | trivial | partially-completed artifacts |

## 2. conditional checks (CONDITIONAL -- not adopted, not active, NOT included in the
adopted-check total of fourteen; each gated by the cloud stage-2 deployment boundary /
WI-0014)

- **FC-C1 (conditional, not adopted):** duplicate protection matches deployed topology
  (assert replicas==1 until WI-0015) -- pipeline check, adopt at cloud stage 2
  (already in the runway table).
- **FC-C2 (conditional, not adopted):** deployment artifact reproducible / env
  rebuildable + rollback tested -- stage-2 pipeline checks (runway table).
- **FC-C3 (conditional, not adopted):** CI running the full suites -- adopt with
  WI-0014 (no pipeline exists to host checks today; local discipline + snapshot is the
  current control).

## 3. rejected as excessive today (burden > current risk)

- mutation testing; dependency-graph linting; runtime architecture monitors
  (re-affirmed from the runway rejections; nothing changed the risk).
- continuous replay of the full corpus in CI -- no CI exists; adopt as a local suite
  member with A-03 instead.
- automated secret-scanning pipeline -- single-repo, single-operator; G-10 checklist +
  rotation (R-05) addresses the observed instance; revisit at cloud gate.
- Prometheus-style /metrics + alerting -- no consumer; single host; stage-3 item.
- protocol-content model-graded evaluation (LLM-as-judge) -- adds paid calls and a new
  moving part to evaluate prose whose buyer exposure is already fail-closed; containment
  heuristics (G-03) capture most value at zero spend.
- multi-instance concurrency harness -- meaningless before WI-0015's topology exists.
