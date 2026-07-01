---
title: "Asymmetric-Enriched Recipe + Regime Split v1"
type: "evidence-report"
date: "2026-06-30"
status: "complete"
project: "DAI"
slice: "Asymmetric-Enriched Recipe + Regime Split v1"
repos:
  dai: "code+docs"
  dai-vault: "docs-only"
tags:
  - prompt-registry
  - source-depth
  - asymmetric
related:
  - "06 Execution/reports/asymmetric-overlay-byte-alignment-v1.md"
  - "06 Execution/reports/asymmetric-registry-canary-v1.md"
---

# Asymmetric-Enriched Recipe + Regime Split v1 -- Evidence Report

**status:** complete (code + tests shipped; full suite green; shadow_only, not allowlisted; not pushed)
**date:** 2026-06-30

## purpose

Implement the first targeted taxonomy refinement: split the overloaded `starter_enriched` behavior into
complete/symmetric quality vs one-sided/asymmetric quality, and add a shadow-only registry recipe for the
asymmetric case so the deterministic `assembly_error` class (run 260018) routes to its own regime instead of
breaking the symmetric enriched recipe. Forward-only/additive; no allowlist widening; no paid calls.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `44c0373` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). Manifest started at 8 templates / 9 recipes. No paid calls.

## root evidence

The classifier `_starter_state` returned `enriched` whenever *either* side had season-form quality
(`format(home) OR format(away)`), conflating two shapes: **complete** (both sides quality -> symmetric enriched
recipe assembles) and **asymmetric** (one side quality -> the enriched recipe's both-sided form slots are
un-representable -> `PromptAssemblyError` -> safe live fallback `fallbackReason=assembly_error`). Run 260018
(Tigers@Yankees) is the one observed live instance.

## taxonomy decision

Split `enriched` into `enriched` (complete/symmetric) and a new `asymmetric` starter state. `named` and
`missing` are unchanged. `enriched` now means symmetric quality for forward runs; legacy persisted
`starter_enriched_*` rows keep their string (no rewrite). This is the single refinement the taxonomy plan
accepted; `market_stale`/`unstable` and a parallel decision-readiness axis were rejected there and are not
implemented.

## regime definition

`starter_asymmetric` = both probable starters announced, season-form quality present for **exactly one** side.
Canonical regime name: `starter_asymmetric_market_{market_state}`. First implemented variant:
`starter_asymmetric_market_backed_depth` (the market shape of the observed 260018 failure).

Classifier (`mlb_starter_state_split(present, home_quality, away_quality)`):
both -> enriched; exactly one -> asymmetric; neither -> named; not present -> missing.

## recipe / template changes (additive)

- NEW template `mlb.overlay.starter.asymmetric.v1` (+`.txt`): requires `home_starter_name, home_starter_hand,
  away_starter_name, away_starter_hand, enriched_side, enriched_starter_form` -- both names/handedness plus a
  single side-agnostic form slot, so a one-sided shape assembles without demanding a both-sided form. Buyer-safe
  language: "use the provided season stats only for the side where they are given ... do not fabricate." sha256
  in manifest; `check_prompt_manifest.py` revalidates.
- NEW recipe `mlb.pregame.analysis.starter_asymmetric_market_backed_depth.v1`: pieces = base + asymmetric starter
  overlay + market backed_depth overlay; `dataRegime=starter_asymmetric_market_backed_depth`;
  `lifecycle=shadow_only`.
- Slot derivation: `_starter_slots` gains an `asymmetric` branch emitting `enriched_side` (the quality-bearing
  side, "home"/"away") + `enriched_starter_form` (that side's phrase from the SAME `format_pitcher_quality`).
- Manifest now reports **9 templates, 10 recipes**.

No change to the base piece, market overlays, existing enriched/named/missing overlays, or any complete-enriched
recipe.

## allowlist status

**DEFAULT_ALLOWLIST unchanged -- exactly four regimes; `starter_asymmetric_*` is NOT included.** A test asserts
the exact 4-tuple and that no allowlist entry contains "asymmetric". Effect in the canary: an asymmetric game
now classifies as `starter_asymmetric_*`, which (not being allowlisted) returns a clean
`fallbackReason=regime_not_allowlisted` live fallback -- a policy decision -- instead of the old
`assembly_error` (a recipe failure). The asymmetric assembly_error class is resolved. The recipe is ready for a
future shadow/canary promotion if/when evidence justifies it.

## behavioral change summary

| asymmetric game | before | after |
|---|---|---|
| selectedDataRegime | starter_enriched_market_backed_depth | **starter_asymmetric_market_backed_depth** |
| canary fallbackReason | assembly_error (recipe failure) | **regime_not_allowlisted** (policy) |
| recipe assembles? | no (PromptAssemblyError) | **yes** (dedicated overlay) |
| model input | live bytes (unchanged, safe) | live bytes (unchanged, safe) |

## tests

TDD. NEW `tests/test_asymmetric_regime_split.py` (12 tests): split classification (enriched/asymmetric/named/
missing); end-to-end `mlb_route_and_slots` regime + slots (asymmetric emits `enriched_side`/`enriched_starter_
form`, not both-sided form); asymmetric backed_depth recipe **assembles cleanly**; symmetric enriched + missing
unchanged; DEFAULT_ALLOWLIST unchanged + asymmetric excluded; asymmetric recipe shadow_only. Updated existing
tests to the new behavior:
- `test_prompt_route_decision.py`: replaced the obsolete `test_decision_assembly_failure_falls_back` with
  `test_decision_asymmetric_routes_to_asymmetric_regime_not_assembly_error` (asymmetric -> regime_not_allowlisted)
  and `test_decision_partial_market_depth_still_assembly_errors` (genuine assembly_error preserved via a
  backed_depth market missing a depth metric -- a non-starter shape).
- `test_shadow_validation.py`: asymmetric now ASSEMBLES but its shadow prompt diverges from live -> a safe
  MISMATCH (still not captured/used); updated the log assertion from "could not assemble" to "MISMATCH".
- `test_shadow_cohort_soak.py`: one-sided quality still auto-detected as partial_evidence_unrepresentable; the
  un-representable reason for asymmetric+market_missing is now "no selectable recipe" (no recipe for that market
  yet), updated the detail assertion.
- `test_manifest_integrity.py`: expected counts 8/9 -> 9/10.

## verification

- `pytest` full agent-service suite -> **397 passed**.
- `check_prompt_manifest.py` -> OK (9 templates, 10 recipes).
- DEFAULT_ALLOWLIST unchanged (asserted). Asymmetric recipe lifecycle shadow_only (asserted). Complete-enriched
  routes unchanged (asserted). Asymmetric backed_depth assembles in a non-paid fixture (asserted). No buyer
  contract touched (no .NET / SportsAnalysisResponse change). No paid calls.

## buyer-facing impact

**None.** Change is confined to the FastAPI prompt routing layer (classifier + shadow recipe/template + tests).
`SportsAnalysisResponse` and the .NET platform are untouched; no chain-of-thought or prompt text exposed. The
asymmetric prompt is shadow_only and never reaches the model (not allowlisted, and diverges from live).

## paid-call status

**None.** Unit/fixture assembly + a non-paid manifest check only.

## migration / compatibility notes

Forward-only and additive -- **no DB migration, no row rewrite**:
- Persisted `PromptRouteProvenanceJson` rows keep their original `starter_enriched_*` strings (historical mixed
  complete+asymmetric). Only NEW runs emit `starter_asymmetric_*`.
- `selectedDataRegime` is free-text in the provenance contract; the .NET `RouteKey` derives keys from arbitrary
  regime strings, so NO contract and NO route-key change. The new regime appears as its own metrics group going
  forward; the pre-split legacy enriched bucket is read as "pre-2026-06-30 mixed."
- `STARTER_STATES` is now a 4-tuple `(enriched, asymmetric, named, missing)`; `mlb_regime`/`split_mlb_regime`
  accept the new state.

## risks / deferred items

- Only `starter_asymmetric_market_backed_depth` has a recipe; asymmetric + market_missing/backed have no recipe
  yet -> they classify distinctly but fail closed (no recipe) on direct assembly and fall back
  regime_not_allowlisted in the canary. Add those recipes only if asymmetric games recur in those markets.
- The asymmetric recipe is not byte-equal to the live prompt (it is a new shape), so it cannot be allowlisted
  for registry-authoritative use until a shadow/canary equivalence target is defined -- intentional, deferred.
- The split creates a legacy/new boundary in metrics at 2026-06-30; documented here, not rewritten.
- Justification rests on n=1 observed assembly_error, but the failure is deterministic from the recipe, so the
  code path + classifier change are sufficient.

## next recommended slice

**Asymmetric Registry Canary v1** -- define a shadow/canary equivalence target for the asymmetric recipe (prove
its rendered bytes against a chosen authoritative shape) so it can be promoted to registry-authoritative under
an explicit override, closing the loop from "classified + assembles" to "usable." Alternatively, **Outcome
Reconciliation Follow-up v1** once the 20-run backlog games reach Final (still the prerequisite for
allowlist-promotion evidence).
