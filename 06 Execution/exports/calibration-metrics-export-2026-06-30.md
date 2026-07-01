---
title: "Calibration Metrics Export -- 2026-06-30 (snapshot)"
type: "export"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Calibration Metrics Export Download v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - metrics
  - export
  - vault
related:
  - "06 Execution/prompt-routing-coverage-matrix-v1.md"
  - "06 Execution/outcome-reconciliation-follow-up-v3.md"
  - "06 Execution/okf-yaml-front-matter-pattern-v1.md"
---

# Calibration Metrics Export -- 2026-06-30 (snapshot)

**status:** complete (read-only export/report; no code, no paid calls, no reconciliation writes)
**snapshot timestamp:** 2026-06-30 (tenant 1, dev SQL `devcore`)
**source:** authoritative metrics from `GET /api/agent-runs/prompt-route-calibration/metrics` (tenant-scoped,
read-only, no body/buyer surface) + read-only DB queries against `devcore` for breakdowns the endpoint does not
aggregate.

## purpose

Non-time-gated read/report export of the current calibration and per-route metrics, so system state is easy to
inspect/compare/hand off without new settled outcomes or paid calls. Pure read-only; no runtime change.

## start state

- `dai`: clean, synced, `d1fbb33` (0/0). `dai-vault`: clean, synced, `635bbef` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). devcore-sql up. No paid calls. No reconciliation writes.

## method / source notes

- **Metrics source = the existing endpoint** `prompt-route-calibration/metrics` (composes
  `PromptRouteCalibrationExporter.ExportAsync(tenantKey)` -> `PromptRouteCalibrationMetricsCalculator.Calculate`).
  It is the canonical, tenant-scoped aggregation; it returns the summary + per-route rows incl. `averageConfidence`.
  There is **no separate CSV/rows download endpoint** -- only this aggregated summary.
- The API was brought up read-only solely to call this endpoint, then stopped; no model call.
- Directional vs no-decision split was taken from a read-only DB query on the durable `AgentRuns.LeanSide`
  column + `PromptRouteProvenanceJson` regime (reliable).
- **Limitation (honest):** confidence-BAND and advised-strength / evidence-sufficiency distributions are NOT
  cleanly available. The artifact `OutputJson` is nested/versioned (sports_decision_artifact v1/v2/v3), so a raw
  `JSON_VALUE('$.confidence')` query returns null for ~261/263 rows -- only the exporter's per-row extraction is
  correct, and it is exposed only as a per-route AVERAGE (not per-row). So this export reports authoritative
  per-route `averageConfidence` (from the endpoint) instead of bands. Per-row bands would require a rows-export
  endpoint (deferred -- see risks).

## summary (authoritative, from endpoint)

| metric | value |
|---|---|
| totalRows | 263 |
| reconciledRows | 76 |
| unreconciledRows | 179 |
| noDecisionRows | 8 |
| unknownRouteRows | 235 |
| matchedRows | 47 |
| unmatchedRows | 29 |
| matchRate | 0.6184 |
| registryRows | 27 |
| liveRows | 1 |
| fallbackRows | 1 |
| averageConfidence | 0.6782 |

## per-route / per-regime export (6 routes, authoritative)

| route key | total | recon | unrecon | noDec | matched | unmatched | matchRate | src | reg | live | fb | avgConf |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| unknown (legacy/pre-provenance) | 235 | 68 | 159 | 8 | 41 | 27 | 0.6029 | -- | 0 | 0 | 0 | 0.684 |
| ...starter_enriched_market_backed_depth.v1@v1::starter_enriched_market_backed_depth | 15 | 7 | 8 | 0 | 6 | 1 | 0.8571 | registry | 15 | 0 | 0 | 0.750 |
| ...starter_missing_market_missing.v1@v1::starter_missing_market_missing | 8 | 0 | 8 | 0 | 0 | 0 | n/a | registry | 8 | 0 | 0 | 0.375 |
| ...starter_enriched_market_missing.v1@v1::starter_enriched_market_missing | 3 | 0 | 3 | 0 | 0 | 0 | n/a | registry | 3 | 0 | 0 | 0.675 |
| ...starter_missing_market_backed_depth.v1@v1::starter_missing_market_backed_depth | 1 | 0 | 1 | 0 | 0 | 0 | n/a | registry | 1 | 0 | 0 | 0.585 |
| starter_enriched_market_backed_depth::assembly_error (fallback) | 1 | 1 | 0 | 0 | 0 | 1 | 0.0 | live | 0 | 1 | 1 | 0.750 |

Totals check: registry 15+8+3+1 = 27; live/fallback 1; unknown 235; sum 263. ✔

## directional vs no-decision coverage (provenance-bearing runs, DB-verified)

| regime | runs | directional (lean set) | no-decision (null lean) |
|---|---|---|---|
| starter_enriched_market_backed_depth | 16 | 16 | 0 |
| starter_enriched_market_missing | 3 | 3 | 0 |
| starter_missing_market_backed_depth | 1 | 0 | 1 |
| starter_missing_market_missing | 8 | 0 | 8 |

Confirms the structural pattern (from the taxonomy plan): **starter present -> always directional; starter
missing -> always no-decision** (28/28 provenance runs follow this with zero exceptions).

## current per-regime / per-route findings

- **Evidence-rich routes carry higher confidence:** averageConfidence tracks evidence richness --
  enriched_market_backed_depth 0.750 > enriched_market_missing 0.675 > starter_missing_market_backed_depth
  0.585 > starter_missing_market_missing 0.375. The lowest-evidence regime is the least confident, which is the
  desired calibration direction (no overconfidence on no-data regimes).
- **Only one provenance regime has reconciled outcomes:** enriched_market_backed_depth (7 registry reconciled,
  6/1 = 0.857 match) + its 1 assembly_error fallback (incorrect). Every other provenance regime is 0-reconciled
  (the pending backlog). The 235-row `unknown` bucket is legacy pre-provenance history (41/27, 0.603).
- **Fallback attribution healthy:** the single assembly_error is correctly keyed to
  `starter_enriched_market_backed_depth::assembly_error` (src=live, fb=1), not collapsed to `unknown`.

## current pending backlog shape (20 runs, unchanged since v3)

The 20 unreconciled provenance-bearing runs (= the v1/v2/v3 reconciliation backlog):

| group | runs | regime | directional? |
|---|---|---|---|
| v2 batch (keys 270013-270020) | 8 | starter_enriched_market_backed_depth | directional |
| targeted (270027-270029) | 3 | starter_enriched_market_missing | directional |
| targeted (270021-270026) | 6 | starter_missing_market_missing | no-decision |
| soak (260014-260015) | 2 | starter_missing_market_missing | no-decision |
| soak (260013) | 1 | starter_missing_market_backed_depth | no-decision |

= **11 directional + 9 no-decision; 0 Final as of Follow-up v3** (StatsAPI: 5 of the 06-30 slate Pre-Game, rest
Scheduled). On settlement, the 11 directional produce matched/unmatched (first real read on
enriched_market_backed_depth at scale + the first-ever enriched_market_missing reads); the 9 no-decision become
noDecisionRows (coverage, not win rate).

## tests / checks run (read-only)

- `GET prompt-route-calibration/metrics` -> summary + 6 routes captured (values above). Cross-checked totals
  (registry 27 / live 1 / unknown 235 = 263). 
- Read-only DB: directional-vs-no-decision by regime; totals (TotalRuns 263, TotalOutcomes 84,
  BacklogReconciled 0). No writes. No code changed -> no test suite required.

## paid-call status

**None.** Endpoint read + DB reads only.

## files changed

Vault only: this export + the handoff entry. dai HEAD unchanged at `d1fbb33`.

## confirmation -- untouched surfaces

Prompts / prompt registry: **untouched.** DEFAULT_ALLOWLIST: **unchanged** (4). Buyer copy / routes:
**untouched.** DB schema: **untouched** (no migration). Reconciliation state: **untouched** (no writes; outcomes
still 84, backlog still 0 reconciled).

## risks / deferred items

- This is a point-in-time snapshot (2026-06-30); values move as the backlog settles.
- **Confidence-band / advised-strength / evidence-sufficiency distributions are not exposed** -- the metrics
  endpoint only aggregates to per-route averages, and raw artifact JSON is nested/versioned. A small read-only
  **rows-export endpoint** (`GET .../prompt-route-calibration/rows` -> CSV/JSON of PromptRouteCalibrationRow,
  which already carries confidence/advertisedStrength/posture) would unlock band-level analysis. Deferred --
  not built here to keep the slice read-only/lowest-risk.
- Calibration confidence still rests on one reconciled provenance regime (enriched_market_backed_depth, n=7);
  the directional read broadens only when the backlog settles.

## recommended next slice

**Outcome Reconciliation Follow-up v4** -- once a StatsAPI probe shows >=1 backlog game `Final` (the 06-30 slate
is at first pitch and should settle soon), reconcile that subset for the first broad directional calibration
read. If still no Final, the small **Calibration Rows Export Endpoint v1** (read-only `GET .../rows` CSV/JSON) is
the natural follow-on to this export -- it unlocks the confidence-band/advised-strength breakdowns this snapshot
could not produce.
