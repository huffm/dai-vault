---
title: "PATTERN: Market Attribution Fidelity Guard v1"
type: "pattern"
date: "2026-07-07"
status: "ACTIVE"
project: "DAI"
related:
  - "06 Execution/reports/market-attribution-fidelity-debug-remediation-plan-2026-07-07-v1.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
---

# market attribution fidelity guard v1

## 1. purpose

deterministic, read-derived check of whether artifact PROSE faithfully represents the
STAGED market consensus, and how each persisted marketAgreement=false row should be
interpreted on the divergence ledger. first remediation for the systemic
accidental-divergence defect (all six audited persisted divergences were the model
inverting the market's direction in prose).

## 2. failure class

model generation misattributes the market side (staged `consensus away`, prose "the
market prefers <home team>"), enabled by the prompt's bare side label + home-only
raw rounded median (debug report, 2026-07-07). retrieval, staging, derivation, and the
read model are correct; only prose lies. the lean-vs-prose direction guard cannot see
this class -- it never compares prose to the staged MARKET.

## 3. inputs

persisted fields only, evaluated derive-on-read (no migration, no model call):
leanSide, marketConsensusSide (from the linked MarketSnapshotBatch), homeTeamRef,
awayTeamRef, and the artifact prose fields lean / summary / discern.contrast /
discern.weigh (claim sources) -- counterCase and whatWouldChangeTheRead are never claim
sources (counter-by-nature / hypothetical-by-nature). implementation:
`dai/platform/dotnet/DevCore.Api/AgentRuns/MarketAttributionFidelity.cs`, shared team
tokenization with the direction guard via TeamProseTokens.

## 4. outputs (additive /rows fields)

- `attributionFidelityStatus`: Pass | FailMarketAttributionMismatch |
  UnclearMarketAttribution
- `attributionFidelityReason`: bounded snake_case (e.g.
  `prose_claims_home_but_staged_consensus_is_away`, `no_market_consensus`,
  `both_market_directions_asserted`, `prose_omits_market_opposition`)
- `divergenceInterpretation`: MarketAligned | DeliberateDivergence |
  AccidentalDivergence | UnclearDivergence

the raw prose evidence clause stays OFF the row (prose-free row doctrine; the row test
asserts no prose token survives). auditors read the full prose via /artifact.

## 5. status semantics

- **Pass**: the prose market claim matches the staged consensus (or an agree row makes
  no market claim at all).
- **FailMarketAttributionMismatch**: prose claims a market side the staged consensus
  contradicts -- on ANY row, including agree rows (824011/824012 inversions).
- **UnclearMarketAttribution**: staged side or lean missing, both directions asserted
  across the factual prose, or a disagree row whose prose omits the market entirely.
  never a guess.

## 6. divergence interpretation semantics

computed from the SAME persisted lean/consensus values the row already carries:
- **MarketAligned**: lean == consensus. not a divergence, whatever the prose did.
- **DeliberateDivergence**: lean != consensus AND prose correctly acknowledges the
  opposing market (e.g. "the market favors the Dodgers, but the Padres' home
  advantage..." -- 823281). the ONLY class eligible for candidate-edge interpretation.
- **AccidentalDivergence**: lean != consensus AND prose claims the market supports the
  lean. a market-attribution failure, never candidate edge.
- **UnclearDivergence**: fidelity could not be established.

## 7. edge-ledger credit rules (deterministic, in code)

`MarketAttributionFidelity.CountsAsCandidateEdge(divergenceInterpretation)` returns true
ONLY for DeliberateDivergence. accidental and unclear divergences get zero edge-ledger
credit; aligned rows are not divergences. readouts and ledgers MUST use this rule (or
the /rows field) rather than raw marketAgreement=false counts when narrating edge.

## 8. what it does not change

persisted marketAgreement, gate-4 counts and criterion (pooled_calibration reads the
same fields it always did), settlement eligibility and the /reconcile path, prompts and
templates, confidence, models, buyer copy, /metrics (trailing fields are ignored by the
aggregate calculator). the guard adds interpretation, not evidence.

## 9. live baseline at ship time (2026-07-07, 285 rows)

Pass 72 / FAIL 10 / Unclear 203 (the unclear mass is legacy rows with no linked market
snapshot -> `no_market_consensus`). the 12-row marketAgreement=false ledger: 8 accidental
(incl. all 6 from the taxonomy audit + 824017 excluded + 824422 unsettled), 4 deliberate
(823284 + 823608, both excluded `invalid`; 823281 x2, active but unsettled and a
duplicate-active hazard). agree-row inversions: 824011, 824012. **the settled VALID
divergence ledger remains 100% accidental -- the taxonomy conclusion stands.** this
baseline is the measuring stick for Prompt Market Context Hardening v1.

## 10. how capture cadence resumes

the taxonomy pause condition is met: the guard exists, is tested (17 unit/exporter tests
inside the 1069-test suite), and is live on /rows. cadence may resume per the cadence
proposal ONCE (a) the 2026-07-07 cohort settles with a readout that consumes these
fields, and (b) the operator re-approves capture mornings. every future readout's
divergence rows must cite attributionFidelityStatus + divergenceInterpretation; only
CountsAsCandidateEdge rows may be narrated as candidate edge signals.
