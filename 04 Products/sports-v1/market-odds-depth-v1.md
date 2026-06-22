# Market Odds Depth v1

**date:** 2026-06-22
**status:** IMPLEMENTED + live-verified. Code change (dai). TDD. No prompt-doctrine, deterministic
lean/confidence/posture-logic, reconciliation-logic, or buyer-UI-template change. No migration.
**slice type:** source/data improvement (deepens the market signal). Recommended by Sports Source
Strategy and Signal-Gap Review v1.

**Anchor:** Better inputs may change model output; that is intended. The constraint was: do not change
deterministic lean/confidence/posture logic, prompt doctrine, or buyer UI code. This slice creates real
source variance for later calibration -- it does NOT by itself calibrate confidence.

## What changed

The market signal was previously a single-book, current run-line snapshot (`SourceDepth.market_odds` was
hard-coded `shallow`). This slice fetches **multi-book moneyline + spread + total** from the existing
odds_api `/odds` endpoint and derives observed multi-book depth.

- **`OddsMarketClient`** now requests `markets=h2h,spreads,totals` across all US books. The single
  preferred-book run-line headline is unchanged (back-compat); a new per-book pass builds the depth
  summary. `platform/dotnet/DevCore.Api/Sports/OddsMarketClient.cs`.
- **New pure module `MarketDepth`** (`platform/dotnet/DevCore.Api/AgentRuns/MarketDepth.cs`): deterministic
  derivation of book count, consensus side (majority moneyline favorite), disagreement range (max-min of
  home implied probability), median home/away implied probability, median total. Plus `AgreementWithLean`
  (lean vs market consensus) and `IsEnriched`. No I/O; fully unit-tested.
- **`BaseballMarketContext`** gained nullable depth fields (`BookCount`, `ConsensusSide`,
  `MedianHomeImpliedProb`, `MedianAwayImpliedProb`, `DisagreementRange`, `MedianTotalPoint`) -- additive,
  backward compatible.
- **`SourceDepthEvaluator`** now classifies `market_odds` as **`enriched`** when >=2 books contributed a
  moneyline-implied consensus, else **`shallow`** (single book / run line only). Reuses the existing
  SourceDepth vocabulary -- no new tier (amendment 4). Detail carries the observed metrics.
- **Analyzer prompt-injection (Python)** appends a market-depth block (book count, consensus side, median
  implied probability, agreement strength, secondary total) when multi-book depth is present. Data
  injection only; no change to system prompt doctrine. `services/agent-service/app/{models/sports.py,
  services/sports_analyzer.py}`.

## Source-depth classification (reused vocabulary)

- `market_odds = enriched`: >=2 books with a moneyline-implied consensus.
- `market_odds = shallow`: single book / spread-only / run line only (prior behavior; the fail-soft path).
- omitted entirely when the market did not ground (breadth already reports the missing market signal).

## Before / After (live, 2026-06-22)

- **Before:** every MLB run carried `market_odds = shallow`, detail "run line only (no moneyline /
  consensus depth)". Confidence flat 0.75 on directional enriched runs.
- **After (live run d2b5433e, a real MLB game):** `market_odds = enriched`, detail "multi-book moneyline
  depth: 9 books, consensus away, median home win prob 48%, book disagreement 2%". The run leaned `away`
  (matching the market consensus) at confidence **0.8** -- the richer market input did move the model off
  the flat 0.75, exactly the source variance this slice was meant to create. (One data point; not a
  calibration claim.)

## Fail-soft behavior (amendment 6)

- If multi-book data is unavailable (single book, spread-only, or no h2h), depth stays `shallow` and the
  prior single-book run-line behavior is preserved.
- Missing depth fields never fail artifact generation; all depth fields are nullable and default to "no
  multi-book depth observed".
- Verified by unit tests (`single_book_moneyline_is_not_enriched`, `summarize_null_or_empty...`,
  `mlb_single_book_market_stays_shallow`).

## Not done this slice (deferred, documented)

- **Lean-vs-market agreement as a surfaced artifact field.** The pure helper
  `MarketDepth.AgreementWithLean` is implemented and unit-tested, and the consensus side is now persisted
  (in the `market_odds` SourceDepth detail) and injected to the model. Surfacing agreement as a
  *structured* artifact field needs a new persisted field + a derive-on-read projection (the lean is only
  known at compose time). Deferred to a small follow-up: **Market Agreement Projection v1** (read-time,
  observed-only, like ArtifactDirectionConsistency).
- **Historical line movement** (opening->current) remains deferred -- requires the paid odds-api snapshot
  endpoint.
- Football/basketball market contexts were left single-book (their records were not extended); the shared
  client fetch now pulls multi-book data but only the baseball context maps it. Extending NBA/NFL is a
  trivial follow-up once MLB proves the pattern.

## Verification

- **Unit (TDD):** `MarketDepthTests` 20/20 (implied probability, median, consensus majority/tie, spread
  fallback, disagreement, IsEnriched gating, agreement). `SourceDepthEvaluatorTests` +2 (enriched
  multi-book, single-book stays shallow).
- **Full suite:** `DevCore.Api.Tests` **773 passed, 0 failed**.
- **Python:** `py_compile` clean on `sports.py` and `sports_analyzer.py`.
- **Live end-to-end:** real MLB run d2b5433e -> `market_odds = enriched`, 9 books, consensus away, median
  home prob 48%, disagreement 2%; lean away @ 0.8. Confirms client fetch -> derivation -> context fields
  -> SourceDepth enriched -> persisted + projected, and richer market text reached the analyzer.
- **Unchanged:** no deterministic lean/confidence/posture-logic edit, no prompt-doctrine edit (system
  prompts untouched; only the factual market-data block was extended), no reconciliation-logic edit, no
  buyer-UI-template edit, no migration. dai test count rose only by the new market-depth tests.

## Next recommended slice

**Market Agreement Projection v1** -- surface the (already-tested) lean-vs-market agreement as a
read-time, observed-only artifact projection using the persisted consensus + structured LeanSide, so
"lean agrees/disagrees with the market" becomes a calibration dimension. Alternatively, once the
directional-contrast cohort (200013-200022) settles, fold the new enriched market depth into
**Reconciled-Outcome Calibration Read v2** to test whether multi-book agreement separates hits from
misses. Confidence tuning stays deferred until that settled read exists.
