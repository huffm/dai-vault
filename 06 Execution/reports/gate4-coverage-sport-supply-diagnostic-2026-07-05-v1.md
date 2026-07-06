---
title: "Gate-4 Coverage Diagnostic + Sport Supply Expansion Audit v1"
type: "report"
date: "2026-07-05"
status: "complete -- no-spend diagnostic; Gate 4 blocked by confidence-distribution/criterion, NOT sample supply"
project: "DAI"
slice: "Gate-4 Coverage Diagnostic + Sport Supply Expansion Audit"
repos:
  dai: "unchanged (c6d4f43) -- read-only"
  dai-vault: "docs-only (this report + handoff)"
tags:
  - calibration
  - gate4
  - sport-supply
  - measurement
  - planning
related:
  - "06 Execution/reports/highest-leverage-slice-discovery-interrogate-audit-2026-07-05-v1.md"
  - "02 Platform/architecture/governance/evidence-readiness-gates-v1.md"
  - "06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md"
---

# Gate-4 Coverage Diagnostic + Sport Supply Expansion Audit v1

## 1. objective

No-spend: find the fastest, safest path to Gate 4 (Calibration Sufficiency), and decide whether sport-supply
expansion (esp. WNBA) should be part of the measurement strategy. Real counts from the live `/rows` read-model
(273 rows) + code-cited sport-supply audit. No implementation.

## 2. repo state

dai `c6d4f43` (dirty only on pre-existing csproj phantom; intentionally untouched); dai-vault `38d77bf`, 2 ahead
(unpushed prior docs), untracked `system-state-synopsis-v1.md` (pre-existing). No unexpected state.

## 3-4. inspected

Docs: `evidence-readiness-gates-v1.md`, `confidence-calibration-rules-v1.md`, prior Interrogate audit, divergence
capture plan/blocked report. Code/data: `services/agent-service/app/services/pooled_calibration.py` (gate logic),
`scripts/pooled_calibration_report.py` (CLI), live `GET /prompt-route-calibration/rows` (273 rows) computed
locally; sport-supply code audit across `SportsRetriever.cs`, `CompetitionCatalog.cs`, `ToolRegistry.cs`,
`OddsMarketClient.cs`, `OutcomeReconciliationMatcher.cs`, `GameIdentity.cs`, `SourceReadiness.cs`, `SourceDepth.cs`,
`dataregime.py`.

## 5. Gate 4 criteria (from `pooled_calibration.py:85-101`)

`conclusionsAllowed = slatesMet AND enrichedMarketMissingMet AND (not below_n) AND marketDisagreementMet`, where
`below_n` = any confidence bucket with n < 15, buckets keyed by confidence rounded to 2 decimals
(`_conf_bucket`, `:45-47`); thresholds MIN_SLATES=3, MIN_CONF_BUCKET_N=15, MIN_MARKET_DISAGREEMENT_N=2 (`:14-16`).

## 6. Gate 4 distance table (live counts, active = ExclusionReason IS NULL, settled, directional; n=92)

Overall: 92 directional settled runs, **56 correct / 36 incorrect (60.9%)**.

| Gate 4 criterion | Current state | Required | Gap | Backfillable? | Action |
|---|---|---|---|---|---|
| slatesMet | >=3 slates | >=3 | MET | n/a | none |
| enrichedMarketMissingMet | n=3 directional | >=1 | MET | n/a | none |
| marketDisagreementMet | **n=4** (2c/2i) | >=2 | MET (arithmetic) | n/a | grow sample (quality thin) |
| **not below_n** | buckets 0.63(1), 0.68(5), 0.70(6), 0.72(2) all <15 | every bucket >=15 | **FAILS** | **NO** | criterion review |

Confidence buckets (2dp, active directional settled): **0.63 n=1 | 0.68 n=5 | 0.70 n=6 | 0.72 n=2 | 0.75 n=63
(39c/24i) | 0.80 n=15 (8c/7i)**. Only 0.75 and 0.80 clear n>=15. evidenceRichness uniform = 2 across the set.

**conclusionsAllowed = FALSE, and the ONLY failing sub-gate is `not below_n`.** The failing buckets
(0.63/0.68/0.70/0.72, ~14 runs) are sparse low/legacy-dampened confidence values.

**Critical, verified with the data: no supply-side action clears this gate.** New backed_depth capture lands at
0.75/0.80 (already >=15); it never fills the sub-15 buckets. Tested redefinitions all still FAIL:
- range buckets (0.60-0.69 / 0.70-0.79 / 0.80+): **0.60-0.69 = 6 (<15) -> FAIL** (DAI rarely emits a low-confidence directional lean).
- registry-attribution-complete corpus only (2dp): n=25, all buckets <15 -> FAIL.
- registry corpus + range buckets: 0.60-0.69=3, 0.80+=7 -> FAIL.

**Root cause = DAI's confidence DISTRIBUTION (leans cluster at 0.75/0.80; low-confidence directional leans are
structurally rare), plus sparse legacy exact-value buckets -- NOT sample supply and NOT slate timing.**

Latent infra note: `pooled_calibration.py` filters only `outcomeStatus` (`:24`), NOT `ExclusionReason`; the CLI
(`pooled_calibration_report.py`) passes raw `/rows`, so excluded runs can pollute the pooled buckets/denominator
(barely moves current numbers, but is a correctness gap to fix in any criterion-review slice).

## 7. market baseline coverage diagnostic (active directional settled; n=92)

Coverage: **52/92 have a market consensus side; 40 missing.**

| Group | Count | Why missing | Backfill source | Risk | Recommendation |
|---|---|---|---|---|---|
| legacy MLB live-path (regime null, promptSource null) | 37 | pre-attribution runs; no MarketSnapshotBatch captured at run time | none persisted; external read = wrong timing | high (measurement-invalid) | **NotBackfillable** |
| nba starter_enriched_market_missing | 3 | odds genuinely not posted at capture | none | n/a | **NotBackfillable** (no market existed) |

**Market baseline backfill is NOT a viable path** -- the uncovered runs are old runs with no persisted snapshot
(external backfill would fabricate a market that did not exist at decision time) or games whose market was
genuinely missing. Do not backfill. (The separate 59 `competition=None` rows are old 2026-03/04 runs, all
unsettled + no market + no provider -- not part of the calibration denominator.)

## 8. DAI-vs-market disagreement gap

- Directional settled with a market baseline: 52. **Agree 48, DISAGREE 4.**
- The 4 disagreements (all MLB, lean=home vs market=away): 824743 + 824662 correct (DAI right, market wrong);
  824500 + 823529 incorrect. **2 correct / 2 incorrect -> uninformative about edge.**
- backed_depth registry route: n=23, market-covered 23, **2 disagreements**, 15/23 correct.
- Correction to the prior audit's "n=0": disagreement is **n=4 overall (n=2 on backed_depth)**, so
  `marketDisagreementMet` is arithmetically satisfied -- but the sample is far too thin (2-2) to read an edge.

**Is disagreement still the core missing dimension? Yes -- in QUALITY, not gate-status.** The gate counts it as
met, but 4 disagreements split 2-2 tell you nothing about whether DAI beats the market. Growing this sample is the
substantive measurement need. The issue is **market dominance + slate selection** (the market-informed regime pulls
DAI to the favorite, so disagreements are rare), not prompt behavior -- do NOT tune to force disagreement. Morning
MLB capture with the divergence prefilter is the honest instrument to grow it; additional sports would grow volume
but in DIFFERENT (non-backed_depth) regimes, so they do not deepen this specific dimension.

## 9. sport supply expansion audit

"Sport" is a **code branch today, not a config dimension** (contradicts `niche-config-schema.md:15`):
`SportsRetriever.RetrieveAsync` is an if/else on competition (`:61-183`); `CompetitionCatalog.cs:53-129` is a
static array; market tools are deliberately per-sport (`ToolRegistry.cs:34-35`). Reconciliation IS league-agnostic
+ provider-identity-safe (`OutcomeReconciliationMatcher.cs:39-69`, keyed on provider+externalGameId only). Odds
fetch IS league-parameterized (`OddsMarketClient.cs:282`). The MLB-coupled part is the **regime/source-readiness/
depth stack**: `TargetRegime=starter_enriched_market_backed_depth` (`SourceReadiness.cs:59`), classifier reads only
MLB contexts, `SourceDepth.Evaluate` early-returns `[]` unless competition=="mlb" (`SourceDepth.cs:54`),
`dataregime.py` is pitcher/run-line specific. **`backed_depth` is meaningful for baseball only today**; basketball
market context drops the depth fields (`OddsMarketClient.cs:143-151`).

| Sport | Support | Missing | Sample value | Classification | Recommendation |
|---|---|---|---|---|---|
| MLB | full, buyer-ready | none | high (backed_depth, in-season) | **ReadyForCapture** | primary supply |
| **WNBA** | absent from catalog; odds/identity/reconcile generalize | CompetitionCodes const + catalog entry + widen basketball branch; season-label (cross-year family) caveat | spread-baseline only (NOT backed_depth), **in-season July** | **CaptureReadyWithSmallConfig** | feasibility doc, not enablement |
| NBA / NCAAMB | machinery-ready | source-readiness/depth (basketball) | **offseason -> no July supply** | ReadyForCapture (machinery) / Premature (supply) | defer |
| NFL | smoke-only | source-readiness/depth | offseason | Premature | defer |
| NHL | none (hockey deferred `GameIdentity.cs:79-80`) | retrieve branch + odds wrapper + context types | offseason | **NeedsSourceSupport** | defer |
| EPL | none | 3-way/draw model + reconciliation | offseason | **Premature** | defer |

Only **MLB and WNBA are in-season in July 2026.** WNBA is the only near-term expansion candidate, and it would
yield settlement-safe, identity-safe, **spread**-baselined samples (a different regime from MLB backed_depth).

## 10. factory architecture check

- Sport is currently a **code branch**, not config -- adding WNBA is config + one branch-widening (small), but NHL/
  EPL are new assembly lines.
- Reconciliation is league-agnostic / provider-identity-safe (factory-clean).
- Market-odds fetch + identity capture generalize; the **measurement-grade backed_depth regime does NOT** (pitcher/
  run-line specific). A naive WNBA add would silently degrade to `starter_missing_market_missing` or require per-
  sport regime code.
- Guardrails for any sport-expansion implementation: (a) add via `CompetitionCatalog`/`CompetitionCodes` + a
  basketball-branch widen, never a new bespoke pipeline; (b) key season labels by competition, not family; (c) do
  NOT claim backed_depth for a sport without a defined per-sport depth regime; (d) keep reconciliation on the
  generic (provider, externalGameId) contract.

## 11. ranked next options (Measurement / Revenue / Factory / Risk / Readiness / Cost)

| # | option | M | R | F | Risk | Rdy | Cost | total | paid | DB writes | runtime change |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **Calibration Sufficiency Criterion Review (no-spend, operator-gated)** | 5 | 2 | 3 | 4 | 5 | 5 | **24** | 0 | 0 | 0 (review; any change is approval-gated) |
| 2 | **Backed-Depth Divergence Capture v2 (paid, morning MLB)** | 4 | 3 | 2 | 4 | 4 | 4 | 21 | <=12 | 0 (capture only) | 0 |
| 3 | WNBA Capture Feasibility (no-spend doc) | 2 | 3 | 4 | 3 | 4 | 5 | 21 | 0 | 0 | 0 |
| 4 | WNBA Enablement (config + branch widen) | 2 | 3 | 4 | 2 | 3 | 4 | 18 | 0 | 0 | yes (code) |
| 5 | Market Baseline Backfill | 1 | 1 | 1 | 1 | 2 | 3 | 9 | 0 | many | yes -- **REJECT** (unbackfillable/invalid) |
| 6 | Buyer Trust-Surface Packaging | 2 | 5 | 3 | 4 | 4 | 5 | 23* | 0 | 0 | yes | (*premature until edge shown) |
| 7 | Tool Gateway durable invocation audit | 2 | 2 | 4 | 4 | 3 | 5 | 20 | 0 | schema? | yes |
| 8 | More Interrogate / ProbeRefresh | 1 | 1 | 2 | 2 | 2 | 4 | 12 | 0 | 0 | yes -- Gate-4-locked, defer |

## 12. recommended next slice

**#1 -- Calibration Sufficiency Criterion Review v1 (no-spend, operator-gated).** This is the only lever on the
actual Gate-4 blocker. The diagnostic proves 3 of 4 gate conditions are MET and the sole failure (`not below_n`) is
**unclearable by supply** -- more MLB games, WNBA, and market backfill all leave the sub-15 buckets untouched (they
land at 0.75/0.80 or in new regimes), and every tested redefinition still fails because DAI's confidence clusters
high and the low band is genuinely rare. So the honest path to Gate 4 is a principled, operator-approved decision
about the sufficiency CRITERION: (a) should n>=15 be required in every exact-2dp bucket given a naturally-clustered
confidence distribution, or assessed on populated buckets + a separately-tracked disagreement sample; (b) fix the
`ExclusionReason` filter gap in `pooled_calibration.py`; (c) decide range-vs-exact bucketing. **Hard caveat:** this
must be principled measurement hygiene, NOT goalpost-moving to force a pass -- it is a calibration-doctrine change
and therefore requires explicit operator approval; it changes no decision/confidence PRODUCTION logic.

Strategic note the review should confront: Gate 4 gates TUNING + buyer-performance-claims (Gate 5). The platform is
doing neither. So "clear Gate 4" may be the wrong near-term target; the substantive need is a richer DAI-vs-market
disagreement sample (below), independent of the formal gate.

## 13. runner-up

**#2 -- Backed-Depth Divergence Capture v2 (paid, morning MLB, 10:00-13:00 ET).** It does NOT clear Gate 4, but it
is the only honest way to grow the DAI-vs-market disagreement sample (currently 4, split 2-2, uninformative) --
the substantive evidence needed to ever read a market edge. Bounded ($0.05), planned, approval-gated. Do it
alongside #1 (which is free and clarifies whether Gate 4 is even the right target).

## 14. deferred

- **WNBA enablement** -- real factory value + only-other-in-season supply, but spread-baseline only (not
  backed_depth), sport=code-not-config, and it does NOT advance Gate 4. Do a no-spend WNBA **feasibility** doc
  first (option 3) if factory breadth is the goal; defer enablement.
- **Market baseline backfill** -- REJECTED (uncovered runs unbackfillable; external backfill = invalid timing).
- **Buyer packaging, Tool Gateway audit, more Interrogate/ProbeRefresh** -- per prior audit; Interrogate is
  Gate-4-locked.

**Condition that would change the recommendation:** if the operator's goal is factory breadth / near-term supply
rather than Gate-4 clearance -> WNBA feasibility rises to #1. If the goal is edge evidence -> morning capture rises
to #1. If Gate-4 clearance is required for a specific downstream (tuning or a buyer claim), #1 is unavoidable
because supply cannot clear it.

## 15. validation performed

Read-only. 273 `/rows` computed locally (no writes); gate logic read from source; 5th parallel code audit for sport
supply (file:line cited). No mutating endpoints, no DB writes, no services started (devcore-sql + DevCore.Api were
already up, read-only), no paid calls, no new AgentRuns.

## 16. what did not change

No runtime code, prompts, prompt registry recipes, routing, confidence logic, calibration rules, buyer copy,
schema/migrations, reconciliation. No new AgentRuns, no DB writes, no paid model calls, no services started, no
sport implemented. dai unchanged at `c6d4f43`.
