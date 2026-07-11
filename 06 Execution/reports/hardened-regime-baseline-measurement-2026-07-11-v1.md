---
title: "Hardened-Regime Baseline Measurement v1 (2026-07-11)"
type: "evidence-report"
date: "2026-07-11"
status: "complete"
project: "DAI"
slice: "V2 Cadence Wrap 2026-07-11"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - calibration
  - attribution
  - prompt-hardening
related:
  - "04 Products/sports-v1/prompting/prompt-market-context-hardening-v1.md"
  - "06 Execution/reports/diagnostic-readout-failure-taxonomy-2026-07-07-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
  - "06 Execution/reconciliations/v2-day2-cohort-settlement-2026-07-11-v1.md"
---

# hardened-regime baseline measurement v1

The measurement the cadence was authorized to produce: do v2-era (hardened-prompt) runs show a
different market-attribution profile than the frozen v1 baseline? **Direction only** -- the v2
sample is small by design and these are rates, not significance tests.

## measurement rule (binding)

v2-route rows are a distinct regime era. Their attribution rates are **never pooled** with v1
rates. This note reports the two side by side, not combined.

## frozen v1 baseline (285 rows, established 2026-07-09 at v2 cutover)

```
Pass 72 | FAIL 10 | Unclear 203     (FAIL rate 10/285 = 3.51%)
```

## v2-era cohort (both cadence days; 15 active after the 823357 exclusion)

```
Pass 14 | FAIL 0 | Unclear 1        (FAIL rate 0/15 = 0.00%)
```

- All 16 captured rows (8 day-1 + 8 day-2) were generated on the v2 hardened prompt via the
  registry-authoritative route (`selectedPromptVersion=v2`, recipe `...backed_depth.v2`), proven
  byte-identical to live at generation.
- 823357 excluded (postponed non-event) -> 15 active v2 rows, all settled.
- The single Unclear is 822877 (`both_market_directions_asserted`) -- confirmed classifier
  opponent-as-object ambiguity, not a model attribution failure.

## corpus FAIL count across the cadence

```
day-1 pre-settle: FAIL 10   ->   day-1 post: FAIL 10   ->   day-2 post: FAIL 10
```

The corpus `FailMarketAttributionMismatch` count held at **10 throughout both cadence days**. No
v2-era run produced an attribution mismatch. Every one of the 6+ historical accidental-divergence
FAILs predates the hardening.

## reading (direction only)

- **v2 FAIL rate 0/15 vs v1 FAIL rate 10/285.** The hardening did not introduce attribution
  failures, and across 16 v2 captures none occurred. Consistent with the hardening's intent
  (team-named consensus + de-vigged both-side probabilities + required market-vs-lean
  acknowledgment removing the opponent-as-object and raw-median hazards that produced the v1
  accidental divergences).
- **The one deliberate divergence (823845) is the qualitative confirmation:** a v2 run that
  opposed the market and *named* the market-favored team while doing so -- the exact behavior the
  hardening was built to enable, absent from the entire v1 corpus.
- **What this does NOT establish:** n=15 cannot prove the FAIL rate improved (a 0/15 sample is
  consistent with a true rate well above zero); it cannot establish edge, discrimination, or buyer
  claims; and it changes no gate. Gate 4 stays `conclusionsAllowed=false`.

## conclusion

The hardening is **not contradicted** by the v2 cohort and is **qualitatively confirmed** by the
first deliberate divergence and the unbroken FAIL=0 record. It is not *statistically* validated --
that needs more v2-era rows than a two-day cadence produces, and would require a fresh capture
authorization. Recorded as a direction-only baseline; the frozen v1 numbers remain the comparison
anchor for any future v2 measurement.
