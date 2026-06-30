# Evidence Regime Taxonomy + Recipe Expansion Plan v1

**status:** complete (design/planning only; no code, no paid calls, no recipe changes)
**date:** 2026-06-30

## purpose

Assess whether the current 9-regime prompt routing taxonomy should be refined or expanded, grounded in actual
live evidence (coverage matrix, targeted capture, assembly_error diagnostic, no-decision behavior). Define the
next recipe/regime expansion plan. Planning only -- no template/recipe/allowlist/route-key change.

## start state

- `dai`: clean, synced, `0f563d6` (0/0). `dai-vault`: clean, synced, `3808505` (0/0).
- Pre-existing untracked `06 Execution/system-state-synopsis-v1.md` left excluded.
- `DEFAULT_ALLOWLIST` unchanged (4). Manifest: 8 templates, 9 recipes (check OK). No paid calls.

## evidence reviewed

Recent execution docs (coverage-matrix-v1, regime-discovery-v1, targeted-live-batch-v1, controlled-live-batch-
v2, registry-assembly-error-diagnostic-v1, calibration-route-attribution-fix-v1, outcome-reconciliation-
readiness). Source: `migration_readiness._starter_state`/`_market_state` (classifier), manifest.json (recipes/
templates), `registry_prompt_canary.DEFAULT_ALLOWLIST`, `PromptRouteCalibrationExport.RouteKey`.

**Live per-regime evidence (28 provenance-bearing runs, tenant 1, 2026-06-30):**

| regime | source | runs | null-lean (no-decision) | reconciled |
|---|---|---|---|---|
| starter_enriched_market_backed_depth | registry | 15 | 0 | 7 |
| starter_enriched_market_backed_depth | live (assembly_error) | 1 | 0 | 1 |
| starter_enriched_market_missing | registry | 3 | 0 | 0 |
| starter_missing_market_backed_depth | registry | 1 | 1 | 0 |
| starter_missing_market_missing | registry | 8 | 8 | 0 |

(Plus 235 legacy/pre-provenance rows with no regime.)

## current taxonomy assessment

The 3x3 (starter enriched/named/missing x market backed/backed_depth/missing) is mostly sound, but **one cell is
overloaded**: `enriched`. The classifier (`_starter_state`) returns `enriched` when **either** side has
season-form quality (`format(home) OR format(away)`). This conflates two materially different evidence shapes:

- **complete** -- both starters have quality (symmetric). The enriched recipe (which requires both sides' form
  slots) assembles cleanly. 15/15 registry + the 3 enriched_market_missing were this shape.
- **asymmetric/partial** -- one starter has quality, the other does not. Still classified `enriched`, but the
  enriched recipe's required both-sided slots are un-representable -> `PromptAssemblyError` -> safe live fallback
  with `fallbackReason=assembly_error` (run 260018, the single observed fallback).

So `enriched` hides a deterministic failure mode. Every other cell behaves as one evidence condition.

## no-decision analysis

Decisive, clean signal: **starter presence drives the decision, market does not.**

- starter_enriched_* (19 runs): **0/19 null lean** -- every run produced a directional lean.
- starter_missing_* (9 runs): **9/9 null lean** -- every run produced no-decision (abstention).

With no starter data the model abstains regardless of market depth. starter_missing is therefore an
**abstention-expected** class: its calibration value is coverage + confidence/abstention behavior, NOT a
directional win rate. This is normal, correct artifact behavior -- not a defect.

How to treat no-decision (deliverable 5): **normal artifact outcome + an evidence-insufficiency signal already
captured by the existing `noDecisionRows` metric** -- NOT a separate route class. starter_missing already
predicts no-decision perfectly, so a parallel decision-readiness route axis (C below) would add zero information
while multiplying the matrix. The only change worth making is a **calibration label** marking starter_missing
regimes "abstention-expected" so their absent match rate is read as honest, not as missing data.

## questions answered

1. **Too coarse?** No, globally -- only `enriched` is too coarse (hides the asymmetric failure). 2. **Overloaded
regimes?** `starter_enriched_*` (complete vs asymmetric). 3. **Under-evidenced by timing/availability?**
starter_missing_market_backed_depth (1 row; needs the narrow L+1 odds-posted-but-starter-TBD window) and all 5
non-allowlisted regimes (0 rows). 4. **Does starter_missing_market_missing need a different recipe goal?** It
already behaves as abstention (8/8 no-decision); the behavior is correct, so the need is a *label/policy*
(abstention-expected), not a new recipe. 5. **Split starter_enriched?** Yes into **complete** + **asymmetric**;
`named` and `missing` already exist as their own states -- do not invent them. 6. **Split market further?** No --
no evidence that staleness/instability drives behavior; reject market_stale/market_unstable as speculative.
7. **Distinct no-decision route?** No -- starter_missing already encodes it; use a label + existing
noDecisionRows. 8. **Expand before or after backlog settles?** **After** -- 20+ rows unreconciled, only 8
reconciled total; adding recipes now creates more unvalidated routes. 9. **Minimum evidence threshold?** A new
recipe needs >=1 observed real failure/fallback OR a materially different evidence condition (the asymmetric
split clears this with 1 real assembly_error); allowlist promotion needs >=~10-15 reconciled directional rows
for a regime (abstention regimes gated on coverage + consistent abstention, not win rate). 10. **Next
implementation slice?** Asymmetric-Enriched Recipe + Regime Split -- but AFTER reconciliation.

## proposed regime refinements (evaluated against decision rules)

**ACCEPT -- A (starter quality), partial:** split today's `enriched` into:
- `starter_complete` (both sides quality) -- the current enriched recipe already serves this.
- `starter_asymmetric` (one side quality) -- NEW; needs a partial-enriched recipe.
Keep `starter_named` and `starter_missing` (already exist). Satisfies decision rules 1 (materially different
evidence), 2 (changes recipe/behavior), 3 (explains the repeated assembly_error), 6 (stops enriched hiding a
failure mode).

**REJECT -- B (market_stale / market_unstable):** no evidence any market instability drives behavior or fallback;
adding them only enlarges the matrix. Decision rules unmet.

**REJECT as routes / ACCEPT as label -- C (decision-readiness axis):** no_decision is fully predicted by
starter_missing; a parallel axis is redundant. Instead add an `abstention_expected` boolean/label on
starter_missing regimes for calibration honesty.

**ALREADY DONE -- D (fallback refinement):** `regime::fallbackReason` is live (Calibration Route Attribution Fix
v1) -- assembly_error / mismatch / not_allowlisted all already get distinct fallback route keys when they occur.
No taxonomy work needed.

## proposed recipe additions / deferrals

| recipe | decision | rationale |
|---|---|---|
| partial/asymmetric-enriched (one side form) | **ADD (deferred to impl)** | converts the assembly_error fallback into a clean registry route; pairs with starter_asymmetric. Start with the backed_depth market (where the fallback occurred), extend to other markets only if asymmetric games recur there. |
| complete-enriched | keep (= current enriched recipe) | already correct for symmetric quality |
| market-missing enriched | keep (= existing starter_enriched_market_missing) | now 3 live rows |
| missing-starter market-missing abstention | keep, optional prompt-policy review | already yields correct abstention (8/8); reframing the prompt as explicit abstention is low priority |
| missing-starter market-backed | reject add (exists, not allowlisted, 0 rows) | no evidence demand |
| named-tier recipes (x3) | keep deferred (exist, not allowlisted, 0 rows) | promote only on real evidence |

Net proposed change when implemented: **+1 starter overlay template** (asymmetric: requires only the
quality-bearing side's form slots; missing side degrades to name/handedness) and **+1-3 recipes** (asymmetric x
market states, starting with backed_depth). No change to base or market overlays.

## migration / compatibility plan (v1 -> v2 regime names)

Forward-only and additive -- **no DB migration, no row rewrite** (per non-goals):
- The classifier gains an `asymmetric` branch -> emits a new regime string (e.g. `starter_asymmetric_market_
  backed_depth`). `starter_enriched_*` narrows to mean "complete" for *new* runs only.
- Existing persisted `PromptRouteProvenanceJson` rows keep their original `starter_enriched_*` strings
  (historical mixed complete+asymmetric) -- not rewritten. Metrics will show both legacy `starter_enriched_*`
  and the new split regimes going forward; document the boundary date so the legacy enriched bucket is read as
  "pre-split."
- `selectedDataRegime` is free-text in the provenance contract, and `RouteKey` already derives keys from
  arbitrary regime strings, so **no contract change and no route-key change** are required.

## impact on prompt routing + calibration

- **prompt registry:** +1 template + 1-3 recipes (manifest version bump; `check_prompt_manifest.py` revalidates).
- **DEFAULT_ALLOWLIST:** unchanged in this plan; add `starter_asymmetric_*` only after it meets the evidence
  threshold. The asymmetric recipe should first run shadow/canary (byte-equivalence) before allowlisting.
- **route provenance:** unchanged contract; classifier emits new regime strings.
- **route metrics:** new route keys appear automatically; the assembly_error fallback route should shrink as
  asymmetric games route registry-clean instead of falling back.
- **calibration exports:** unchanged mechanism; new regime is its own group. Add the `abstention_expected` label
  so starter_missing regimes are not misread as having a missing/poor match rate.

## next implementation slice

**Outcome Reconciliation Follow-up v1** FIRST (non-paid, time-gated) -- settle the 20-run backlog so the enriched
regimes gain real reconciled performance before any taxonomy change. THEN
**Asymmetric-Enriched Recipe + Regime Split v1** (code): add the asymmetric starter overlay + recipe(s), the
classifier `asymmetric` branch, shadow/canary equivalence, and tests; verify the assembly_error fallback becomes
a clean registry route. Do not widen the allowlist until the new regime has shadow proof + the evidence
threshold.

## tests / verification run

`check_prompt_manifest.py` -> OK (8 templates, 9 recipes). Source-verified classifier signal defs + RouteKey;
DB per-regime evidence + no-decision query (28 provenance rows). Docs cross-checked vs recent handoffs. No code
changed -> no suite required.

## risks / deferred items

- The asymmetric split rests on a single observed assembly_error (n=1); a second asymmetric case before
  implementation would strengthen it -- but the failure mode is deterministic from the recipe, so n=1 plus the
  code path is sufficient justification.
- Splitting enriched creates a legacy/new boundary in metrics; without a documented boundary date the legacy
  enriched bucket could be misread. Mitigate with a dated note, not a rewrite.
- starter_missing_market_backed_depth stays thin until a TBD+odds-posted game is captured.
- Recommendation is point-in-time (2026-06-30, 28 provenance rows, 8 reconciled); thresholds should be revisited
  as the backlog settles.

## buyer-facing impact

**None.** Planning only. No template/recipe/allowlist/route-key/UX change.
