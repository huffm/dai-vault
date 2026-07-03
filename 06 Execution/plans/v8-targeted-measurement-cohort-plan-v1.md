---
title: "v8 Targeted Measurement Cohort Plan"
type: "plan"
date: "2026-07-03"
status: "approval-gated (no spend authorized)"
project: "DAI"
slice: "v8 Targeted Measurement Cohort Plan"
repos:
  dai: "unchanged (beed3fc, residue contract already committed)"
  dai-vault: "docs-only"
tags:
  - calibration
  - cohort
  - measurement
  - approval-gate
related:
  - "06 Execution/plans/live-calibration-cohort-planning-v1.md"
  - "02 Platform/decisions/0006-canonical-reconciliation-residue-contract.md"
  - "06 Execution/reconciliations/outcome-reconciliation-follow-up-v7c.md"
---

# v8 Targeted Measurement Cohort Plan

**slice:** design the next approval-gated paid measurement cohort; do NOT execute it. planning + approval
packet only. supersedes the illustrative tables in `live-calibration-cohort-planning-v1.md` with a
concrete, dated candidate slate now that the residue contract guarantees clean settlement.

## measurement need (pooled state after v7c)

104 reconciled / 87 directional / 17 no-decision / 10 slates. **conclusionsAllowed = FALSE.**
- routes: enriched_backed_depth 0.625 (n=16); enriched_market_missing 1/3 (n=3); unknown 0.603 (n=68).
- confidence buckets: 0.75 = 0.607 (n=61, ONLY bucket >= the n>=15 gate); **0.80 = 0.500 (n=12)**;
  0.63(1) 0.68(5) 0.70(6) 0.72(2).
- home 0.5714 (n=63) vs away 0.6667 (n=24); market-agree 0.634 (n=41) vs disagree 0.500 (n=4).

**Closed gate / blocking evidence gap:** the binding pooled gate is `confidenceBucketN >= 15` on a
bucket **beyond 0.75**. Only 0.75 qualifies; the next-closest is **0.80 at n=12 (needs +3)**. Secondary
thin spots: enriched_market_missing (n=3), market-disagreement (n=4), away-lean (n=24). **Calibration is
not justified** because every actionable bucket except 0.75 is under-powered and non-monotonic -- tuning
now would fit noise. **Prompt-selection hardening is not justified** -- no prompt-selection defect exists;
the only recent defect (thin residue) was settlement-writer-path and is now fixed (ADR 0006).

## candidate discovery (free sources only)

- **Source:** StatsAPI `schedule?sportId=1&hydrate=probablePitcher` (free; no paid call, no run).
- **Window:** today = 2026-07-03. 07-04 slate = 15 games, **14 both-probable** (~24h out = the correct
  selection window per the day-before doctrine). 07-05 = 15 games, 13 both-probable (48h; will firm up).
- **Starter dimension (predictable):** both probables announced -> `starter_enriched`. one probable ->
  asymmetric (fails closed to live per the assembly_error diagnostic). none -> `starter_missing`
  (no-decision by design).
- **Market dimension (NOT schedule-predictable):** regime = starter_state x market_state; market_state is
  `backed_depth` (multi-book de-vig consensus), `backed` (single run line), or `missing` (no odds), decided
  by odds DEPTH captured at generation (`dataregime.py`). The 07-02 backlog came out
  `enriched_market_missing` -> market was NOT captured for those runs. **So whether v8 lands backed_depth
  depends entirely on market capture being active at generation -- this is the cohort's central
  uncertainty and stop-condition.**
- **Paid unit + cost:** the analyzer is **gpt-4o-mini** (`sports_analyzer.py:647`), list price
  **$0.15/1M input, $0.60/1M output** (`model_metering.py:20`). A sports.matchup.analysis run is small
  (~4-10k in / ~1-2k out) => **~$0.002/call**. Dollar cost is negligible for any option below; the real
  constraint is the OpenAI quota (429 risk) and the discipline of moving a NAMED gate, not chasing vibes.
- **Generation tooling:** `run-artifact-calibration.ps1 -Competition mlb -Days 1` (fetches
  `/api/competitions/mlb/upcoming`, POSTs `/api/agent-runs`). **Settlement tooling:**
  `reconcile-calibration-outcomes.ps1` (now residue-enforced) or per-game `POST /reconcile`.

## cohort options (NONE authorized)

Candidate gamePks are the 07-04 both-probable slate (externalGameId = gamePk, provider = mlb_statsapi;
available now). Lean side is NOT knowable pre-generation.

### Option A -- minimal enriched_market_missing top-up (5 games, ~5 calls, ~$0.01)
Goal: cheaply grow the thinnest directional route (n=3). Generate WITHOUT odds capture.

| gamePk | game | starters (A/H) | exp. starter | exp. market | exp. route | usefulness |
|---|---|---|---|---|---|---|
| 822716 | PIT@WSH | Ashcraft/Littell | enriched | missing | enriched_market_missing | route n=3->4 |
| 824499 | BAL@CIN | Young/Greene | enriched | missing | enriched_market_missing | route n |
| 824658 | STL@CHC | Leahy/Imanaga | enriched | missing | enriched_market_missing | route n |
| 824012 | BOS@LAA | Gray/Aldegheri | enriched | missing | enriched_market_missing | route n |
| 824983 | MIA@ATH | Alcantara/Civale | enriched | missing | enriched_market_missing | route n |

Settlement: 07-04 finals (~07-05 06:00Z), StatsAPI + canonical residue. **Does NOT move the blocking
0.80 gate** (market_missing runs trend lower-confidence / abstain); narrow route top-up only.

### Option B -- stronger backed_depth measurement slate (10 games, ~10 calls, ~$0.02) [RECOMMENDED]
Goal: attack the blocking 0.80 bucket AND grow backed_depth + home/away balance + market-agreement.
Generate WITH market capture (near-close odds) so runs land `backed_depth`.

| gamePk | game | starters (A/H) | exp. starter | exp. market | exp. route |
|---|---|---|---|---|---|
| 823526 | MIN@NYY | Matthews/Rodon | enriched | backed_depth* | enriched_market_backed_depth |
| 822882 | DET@TEX | Flaherty/Rocker | enriched | backed_depth* | enriched_market_backed_depth |
| 823118 | TOR@SEA | Bieber/Gilbert | enriched | backed_depth* | enriched_market_backed_depth |
| 824171 | TB@HOU | Rasmussen/Brown | enriched | backed_depth* | enriched_market_backed_depth |
| 824903 | NYM@ATL | Manaea/Sale | enriched | backed_depth* | enriched_market_backed_depth |
| 824092 | PHI@KC | Luzardo/Wacha | enriched | backed_depth* | enriched_market_backed_depth |
| 825063 | MIL@AZ | Woodruff/Kelly | enriched | backed_depth* | enriched_market_backed_depth |
| 823932 | SD@LAD | Canning/Yamamoto | enriched | backed_depth* | enriched_market_backed_depth |
| 824415 | CWS@CLE | Burke/Messick | enriched | backed_depth* | enriched_market_backed_depth |
| 824012 | BOS@LAA | Gray/Aldegheri | enriched | backed_depth* | enriched_market_backed_depth |

`*` = market_state CONTINGENT on odds depth captured at generation (see stop-condition). Settlement:
07-04 finals (~07-05 06:00Z), StatsAPI + canonical residue. **Honest gate math:** ~10 backed_depth runs
distribute across 0.75/0.80; expect +2-4 to the 0.80 bucket -> 0.80 plausibly reaches ~14-16 (right AT
the gate; may need a second slate). Also grows backed_depth route (16->~26), 0.75 bucket, away-lean n,
and DAI-vs-market disagreement samples regardless.

### Option C -- parallel control/candidate arm: NOT USEFUL YET
A control/candidate split is only meaningful when a candidate CHANGES something (prompt text, selection,
decisioning, confidence). **Nothing is changing this slice**, so a candidate arm has nothing to compare
against and would only double spend for zero discriminative value. **Deferred until a concrete,
non-behavioral hypothesis exists.** (A future legitimate use: same games, market-captured vs
market-withheld, to A/B the backed_depth-vs-missing route effect -- but that is a data-capture toggle,
not a prompt/decision change, and belongs in its own slice.)

## recommended approval packet

- **Recommended:** **Option B** (10 games, ~10 gpt-4o-mini calls, ~$0.02).
- **Exact games:** 823526, 822882, 823118, 824171, 824903, 824092, 825063, 823932, 824415, 824012
  (mlb_statsapi, 07-04).
- **Exact max paid calls:** 10 (1 per game; hard cap -- no retries beyond the 10 without re-approval).
- **Estimated cost:** ~$0.02 (negligible); binding limit is the 10-call cap + OpenAI quota.
- **Expected measurement value:** +~10 backed_depth directional rows; 0.80 bucket 12 -> ~14-16;
  backed_depth route 16 -> ~26; grows away-lean + market-agreement/disagreement samples; another slate
  toward pooled breadth.
- **Gates it CAN move:** confidence-bucket n (0.80 toward >=15), backed_depth route n, market-disagreement
  n, home/away balance.
- **Gates it probably will NOT move alone:** `conclusionsAllowed` may stay FALSE if 0.80 lands at 13-14
  (needs a second slate); the `analyzerConfidence==confidence` registry question (a separate read-only
  diagnostic); evidenceRichness-vs-outcome (needs richness variance).
- **Must stay unchanged during execution:** prompt text, prompt selection, analyzer decisioning,
  confidence formula, advertised strength, evidence sufficiency, market agreement, buyer copy, calibration
  + metrics denominators, schema. No backfill of the 7 thin rows. No CaptureMode column.
- **Reconciliation:** after all 10 are Final (StatsAPI), settle each via the canonical residue contract --
  `reconcile-precheck` per game, then `POST /reconcile` (SingleMatch) or `reconcile-calibration-outcomes.ps1`
  with `source=statsapi_final`, `sourceRef="gamePk <pk>"`, `notes="v8 cohort settle via <writer>"`.
- **Residue verification after settlement:** GET `/rows`, confirm each of the 10 new outcomes has non-null
  `settlementSource`/`settlementSourceRef`/`settlementNotes` (contract guarantees it; verify 0 thin rows
  added). Confirm `/metrics` denominator moved only by the expected +10.
- **Stop condition (canary-first):** generate **1-2 canary games first**, inspect
  `GET /{id}/artifact.promptRouteProvenance.selectedDataRegime` (or `/rows`). If the canaries come back
  `enriched_market_missing` instead of `enriched_market_backed_depth`, **STOP** -- market capture is not
  producing depth. Do NOT spend the remaining calls; either fix market capture in a separate slice or fall
  back to Option A intent. Also stop if a candidate's identity/probables changed (scratch/rainout).

## execution prompt draft (next slice, AFTER explicit approval)

See the fenced block returned in the handoff; also reproduced here for the vault record. The next slice
must not run a paid call until the operator replies "approved" with the final gamePks.

## safety ledger (this planning slice)

paid calls 0; new game runs 0; reconciliation writes 0; DB migrations 0; prompt / prompt-selection /
decision / buyer changes none; metrics denominator unchanged; historical rows not backfilled. residue
contract (beed3fc / fbec358) already committed; DevCore.Api.Tests 1043/1043 stands (no code changed this
slice). free StatsAPI schedule reads only.
