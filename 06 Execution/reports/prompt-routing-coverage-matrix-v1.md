---
title: "Prompt Routing Coverage Matrix v1"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Prompt Routing Coverage Matrix v1"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - provenance
  - metrics
related:
  - "06 Execution/plans/evidence-regime-taxonomy-recipe-expansion-plan-v1.md"
---

# Prompt Routing Coverage Matrix v1 -- Evidence Report

**status:** complete (analysis only; no code, no paid calls)
**date:** 2026-06-30

## purpose

Build a clear coverage matrix for the Regime-Aware Prompt Routing Layer -- which prompt regimes are implemented,
allowlisted, live-exercised, reconciled, fallback-prone, thin, or still unproven -- and recommend the next
prompt-routing improvement slice. Analysis only: no prompt/recipe change, no paid calls, no reconciliation.

## terminology

**Regime-Aware Prompt Routing Layer** = regime classification + prompt registry (shadow recipes) + prompt route
decision + default allowlist + safe fail-closed fallback + durable prompt-route provenance + tenant-scoped route
metrics + fallback-specific route attribution.

A **data regime** = (starter_state x market_state). Starter states: enriched (names + season form), named
(names + handedness only), missing (no starter data). Market states: backed (run-line single book), backed_depth
(multi-book depth: consensus/median/disagreement), missing (no odds). 3 x 3 = 9 regimes.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `2996c55` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged -- four regimes. Route fix v1 present (regime::fallbackReason, line 147).

## manifest / recipe inventory (source of truth)

`app/prompting/templates/manifest.json` (manifestVersion 2; `check_prompt_manifest.py` -> OK, 8 templates,
9 recipes). All recipes `lifecycle=shadow_only` (registry bytes reach the model only when the canary proves them
byte-identical to the live prompt).

**8 templates:** current_equivalent (fixture), base, 3 starter overlays (enriched / named / missing),
3 market overlays (backed / backed_depth / missing).

**9 recipes (the full 3x3 regime matrix):**

| recipe id | regime | pieces |
|---|---|---|
| ...starter_enriched_market_backed.v1 | starter_enriched_market_backed | base + starter.enriched + market.backed |
| ...starter_enriched_market_missing.v1 | starter_enriched_market_missing | base + starter.enriched + market.missing |
| ...starter_enriched_market_backed_depth.v1 | starter_enriched_market_backed_depth | base + starter.enriched + market.backed_depth |
| ...starter_named_market_backed.v1 | starter_named_market_backed | base + starter.named + market.backed |
| ...starter_named_market_missing.v1 | starter_named_market_missing | base + starter.named + market.missing |
| ...starter_named_market_backed_depth.v1 | starter_named_market_backed_depth | base + starter.named + market.backed_depth |
| ...starter_missing_market_backed.v1 | starter_missing_market_backed | base + starter.missing + market.backed |
| ...starter_missing_market_missing.v1 | starter_missing_market_missing | base + starter.missing + market.missing |
| ...starter_missing_market_backed_depth.v1 | starter_missing_market_backed_depth | base + starter.missing + market.backed_depth |

The expected 9-regime set matches the manifest exactly -- the taxonomy is complete at 9.

## default allowlist status (4 of 9)

Allowlisted (DEFAULT_ALLOWLIST, `registry_prompt_canary.py`):
- starter_enriched_market_backed_depth
- starter_enriched_market_missing
- starter_missing_market_missing
- starter_missing_market_backed_depth

Not allowlisted (5): starter_enriched_market_backed, starter_missing_market_backed (the two run-line/single-book
regimes), and all three named-tier regimes (starter_named_market_backed / _missing / _backed_depth). Per the
canary doctrine, named-starter and single-book regimes "stay off pending real evidence."

## route metrics snapshot (DB, tenant 1, 2026-06-30)

254 total AgentRuns: **19 provenance-bearing** (routing layer) + **235 legacy/unknown** (pre-provenance history,
no regime; 68 reconciled, 41 matched / 27 unmatched -- the historical calibration baseline, not part of the
regime matrix). Per-regime aggregation of the 19 provenance runs:

| regime | source | fallback | runs | reconciled | matched | unmatched |
|---|---|---|---|---|---|---|
| starter_enriched_market_backed_depth | registry | -- | 15 | 7 | 6 | 1 |
| starter_enriched_market_backed_depth | live | assembly_error | 1 | 1 | 0 | 1 |
| starter_missing_market_missing | registry | -- | 2 | 0 | 0 | 0 |
| starter_missing_market_backed_depth | registry | -- | 1 | 0 | 0 | 0 |

(Metrics endpoint splits the enriched_backed_depth regime into two route keys: the registry route
`...backed_depth.v1@v1::starter_enriched_market_backed_depth` and the fallback route
`starter_enriched_market_backed_depth::assembly_error` -- the latter correctly attributed, not `unknown`,
confirming Calibration Route Attribution Fix v1 holds.)

## coverage matrix (one row per regime)

| # | regime | allowlisted | reg live | fb live | reconciled | matched | unmatched | matchRate | fallback seen | evidence status |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | starter_enriched_market_backed_depth | YES | 15 | 1 | 8 | 6 | 2 | 0.75 (reg route 0.857) | assembly_error x1 | **proven_live_reconciled** (only regime with outcomes; n still modest) + fallback_observed |
| 2 | starter_enriched_market_missing | YES | 0 | 0 | 0 | 0 | 0 | n/a | none | **unexercised** + blocked_by_availability |
| 3 | starter_missing_market_missing | YES | 2 | 0 | 0 | 0 | 0 | n/a | none | **live_unreconciled** + live_thin |
| 4 | starter_missing_market_backed_depth | YES | 1 | 0 | 0 | 0 | 0 | n/a | none | **live_unreconciled** + live_thin |
| 5 | starter_enriched_market_backed | NO | 0 | 0 | 0 | 0 | 0 | n/a | none | **fixture_only** + not_allowlisted |
| 6 | starter_missing_market_backed | NO | 0 | 0 | 0 | 0 | 0 | n/a | none | **fixture_only** + not_allowlisted |
| 7 | starter_named_market_backed | NO | 0 | 0 | 0 | 0 | 0 | n/a | none | **fixture_only** + not_allowlisted |
| 8 | starter_named_market_missing | NO | 0 | 0 | 0 | 0 | 0 | n/a | none | **fixture_only** + not_allowlisted |
| 9 | starter_named_market_backed_depth | NO | 0 | 0 | 0 | 0 | 0 | n/a | none | **fixture_only** + not_allowlisted |

next proof needed per regime: (1) reconcile the 8 pending v2 runs to grow n; (2) a real game with announced
starters AND no odds market (rare); (3,4) settle the 3 pending soak games; (5-9) real-evidence soak + an
allowlist-promotion decision before they can route live.

## evidence status definitions

- **proven_live_reconciled** -- live exercised and reconciled with enough outcome evidence to discuss
  performance (used cautiously here: regime 1 has only 8 reconciled rows -- directional, not statistically
  significant).
- **live_unreconciled** -- live exercised, games not settled yet.
- **live_thin** -- live exercised but too few rows for a meaningful performance read.
- **fixture_only** -- confirmed only through fixture/canary byte-equivalence, not live scheduled games.
- **unexercised** -- recipe/regime exists but no known live proof.
- **fallback_observed** -- one or more safe fallback cases on the route/regime.
- **blocked_by_availability** -- not enough real schedule/source conditions to exercise naturally.
- **not_allowlisted** -- exists but intentionally not default eligible.

## findings

1. **Taxonomy is complete (9 regimes), allowlist covers 4.** The registry implements the full 3x3 matrix; 4
   regimes are default-eligible, 5 are intentionally off (named tier + the two run-line regimes).
2. **Live evidence is severely concentrated.** 16 of 19 provenance runs (84%) are
   starter_enriched_market_backed_depth. It is the ONLY regime with any reconciled outcomes (8: 6 correct /
   2 incorrect, 0.75 regime / 0.857 registry route). Allowlist policy is effectively being validated on one
   regime.
3. **The other 3 allowlisted regimes are thin or unexercised.** starter_missing_market_missing (2 live, 0
   reconciled) and starter_missing_market_backed_depth (1 live, 0 reconciled) are the unsettled soak games;
   starter_enriched_market_missing has NEVER run live -- blocked by availability (games with announced starters
   almost always carry an odds market, so this combination rarely occurs naturally).
4. **Reconciliation is the binding constraint.** 11 live provenance runs are pending settlement (3 soak + 8 v2
   batch). Until they reconcile, more capture only deepens the unreconciled pile rather than producing
   performance signal.
5. **Fallback attribution is healthy.** The single assembly_error fallback is correctly attributed to its own
   route key (not `unknown`); no other fallback reasons (mismatch, regime_not_allowlisted, disabled, error)
   appear among provenance-bearing live runs.
6. **The 5 non-allowlisted regimes are fixture-proven only.** They pass canary byte-equivalence in tests but
   have zero live proof; promoting any of them needs a real-evidence soak first.

## risks / deferred items

- Concentration risk: allowlist confidence rests on one regime with only 8 reconciled rows -- do not overclaim
  per-regime performance.
- Availability risk: starter_enriched_market_missing may be effectively unreachable from natural slates; proving
  it may require a deliberately constructed/source-throttled candidate rather than waiting on the schedule.
- Backlog risk: 11 pending live runs + 3 soak; the matrix cannot move regimes 1/3/4 toward proven until they
  settle.
- This is a point-in-time snapshot (2026-06-30); counts shift as games settle and new batches run.

## next recommended slice

**Regime Discovery + Candidate Selection v1** (non-paid design + dry-run). The matrix's central problem is that
natural daily slates keep producing the same regime (enriched_market_backed_depth), so allowlist policy is
under-validated on the other three allowlisted regimes and entirely unexercised on
starter_enriched_market_missing. A deliberate candidate-selection step -- scan the schedule + source conditions
(TBD starters, thin/absent markets) to *target* the thin/unexercised allowlisted regimes before spending paid
calls -- is the highest-leverage routing improvement and needs no paid calls to design and dry-run.

**Strong companion (run first/alongside, non-paid, time-gated):** Outcome Reconciliation Follow-up v1 -- settle
the 11 pending live runs so regimes 1/3/4 gain real reconciled evidence. Reconciliation converts existing
capture into proof at zero paid cost and is the prerequisite for any allowlist-promotion decision.
