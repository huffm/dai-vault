# Starter-Missing Regime Capture v1 -- Evidence Report

**status:** active doctrine (2 starter_missing regimes now real-soak-clean; allowlist NOT widened)
**date:** 2026-06-28

## purpose

Record an operator-approved 4-game paid cohort that captured real MLB inputs for the starter_missing regimes
and ran the shadow soak, establishing which became real-soak-clean. The registry-authoritative allowlist is
UNCHANGED; this slice produces evidence only.

## problem it solves

[[multi-slate-regime-coverage-v1]] found the starter_missing regimes were real-available (one/none-announced
games) but uncaptured. This slice captures them and soaks them, moving two regimes from representative-only to
real-soak-clean.

## what it is not

- NOT an allowlist widening. NOT registry-authoritative routing for starter_missing (canary stayed OFF; the live
  prompt was the model authority on every run). NOT prompt tuning. NOT a buyer change. NO second model call per
  run.

## candidate discovery method

statsapi `schedule?hydrate=probablePitcher` (free, no model call) across upcoming slates, selecting games with a
TBD probable starter. Because `MlbStarterContext` requires BOTH starter names, a one/none-announced game yields
no starter context -> `starter_missing`. Four candidates were identified and approved.

## paid call approval

Operator approved all 4 candidate games (number=4, target games below). Capture was enabled only for this run;
the canary was OFF (live prompt authoritative).

## target games + outcome (all HTTP 200, input-only captures)

| game | date | TBD side | derived regime | market ctx |
|---|---|---|---|---|
| San Diego Padres @ Chicago Cubs | 2026-06-29 | away | `starter_missing_market_backed_depth` | present (multi-book) |
| Miami Marlins @ Colorado Rockies | 2026-06-29 | home | `starter_missing_market_backed_depth` | present (multi-book) |
| San Francisco Giants @ Arizona Diamondbacks | 2026-06-30 | home | `starter_missing_market_missing` | absent (no odds yet) |
| Chicago White Sox @ Baltimore Orioles | 2026-07-01 | both | `starter_missing_market_missing` | absent (no odds yet) |

All four captured records were verified input-only (no model output / artifact / confidence / buyer-copy keys;
`starter_ctx=False` as expected for TBD games).

## paid call count

**4** (one normal live analyze call per game; no second call per run, no retries).

## captured request count

4 input-only records (scratch jsonl, out-of-repo, NOT committed).

## regime distribution (captured)

- `starter_missing_market_backed_depth`: 2
- `starter_missing_market_missing`: 2

The third target, `starter_missing_market_backed` (single-book), was NOT naturally available -- the 6/29 games
had multi-book depth and the future games had no market, so single-book never appeared.

## shadow soak summary counts

`run_shadow_cohort_soak.py --requests <scratch>/sm_capture.jsonl --sink <scratch>/sm_soak.jsonl`:
cohort_size 4, attempted 4, matched 4, captured 4, mismatched 0, assembly_failed 0, sink_failed 0, errored 0,
partial_evidence_unrepresentable 0, persisted_provenance_lines 4, clean true, GO. Provenance: 4 lines, all
`mode=shadow`, all `lifecycle=shadow_only`, 4 distinct assembledHash, both regimes represented.

## which starter_missing regimes are now real-soak-clean

Both captured regimes meet the acceptance bar (>=1 real input, shadow prompt matched live byte-for-byte,
provenance captured after equality, zero mismatch/assembly/sink/error/partial-evidence):

- **`starter_missing_market_missing`** -- real-soak-clean (2 real inputs).
- **`starter_missing_market_backed_depth`** -- real-soak-clean (2 real inputs).

`starter_missing_market_backed` (single-book) remains representative-only (not naturally available).

## any mismatch / failure / partial-evidence

NONE. 4/4 matched, zero failures, zero partial-evidence. Shadow prompt == live prompt byte-for-byte on all four
real starter_missing inputs.

## updated 9-regime coverage (real evidence)

| regime | level | allowlist |
|---|---|---|
| starter_enriched_market_missing | canary-confirmed + real-soak-clean | ALLOWLISTED |
| starter_enriched_market_backed_depth | canary-confirmed + real-soak-clean | ALLOWLISTED |
| starter_missing_market_missing | **real-soak-clean (new)** | off |
| starter_missing_market_backed_depth | **real-soak-clean (new)** | off |
| starter_enriched_market_backed (single-book) | representative-only | off |
| starter_named_market_missing | representative-only | off |
| starter_named_market_backed | representative-only | off |
| starter_named_market_backed_depth | representative-only | off |
| starter_missing_market_backed (single-book) | representative-only | off |

4 of 9 regimes now have real evidence (2 canary-confirmed, 2 real-soak-clean); 5 remain representative-only (all
involve the `named` starter state or a single-book `backed` market, neither observed in real data).

## why the allowlist was not widened

The slice forbids it. The two new regimes are real-soak-clean (the prerequisite) but adding them to the
canary allowlist is a separate, explicit widening decision for a later slice. Real-soak-clean is necessary, not
sufficient.

## exact commands

```
# agent-service restarted with: DAI_MLB_REQUEST_CAPTURE=1, DAI_MLB_REQUEST_CAPTURE_PATH=<scratch>/sm_capture.jsonl
#   (canary OFF -- live prompt authoritative). 4 approved games triggered via POST /api/agent-runs.
python scripts/run_shadow_cohort_soak.py --requests <scratch>/sm_capture.jsonl --sink <scratch>/sm_soak.jsonl
# agent-service then restarted to DEFAULT (capture off, canary off).
```

## exact tests run

- `pytest -q` (full agent-service suite) -> **330 passed, 0 failed** (no code changed).
- `pytest tests/test_registry_prompt_canary.py tests/test_mlb_request_capture.py tests/test_shadow_validation.py
  tests/test_shadow_cohort_soak.py tests/test_mlb_soak_export.py tests/test_mlb_prompt_equivalence.py
  tests/test_mlb_branch_overlays.py -q` -> 98 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- live: 4 real paid agent-runs (HTTP 200) + 1 shadow soak (GO).

## service / env posture after slice

agent-service :8000 restarted to DEFAULT (canary OFF, capture OFF -- no DAI_MLB_* env); platform-api :5007;
devcore-sql up. Capture/soak/log files are scratch/out-of-repo; NONE committed.

## remaining risks and gaps

- **5 regimes still representative-only.** The three `named` regimes and the two single-book `backed` regimes
  have no real evidence; the `named` state and single-book markets were not observed across discovered slates and
  may be genuinely rare.
- **Modest sample.** 2 real inputs per new regime.
- **Allowlist still 2 regimes.** Widening to include the new real-soak-clean regimes is a deliberate future
  decision.

## deferred decisions

- An allowlist-widening slice to add `starter_missing_market_missing` and `starter_missing_market_backed_depth`
  to the canary (using this evidence), likely after runtime canary confirmations like the enriched regimes had.
- Sourcing `named` and single-book `backed` real evidence (may require targeted dates/opponents or may stay
  representative-only). DB persistence; `.NET sourceDepth -> dataRegime` ownership; caching -- unchanged.

## related docs

- [[multi-slate-regime-coverage-v1]] (the matrix this updates), [[registry-authoritative-prompt-canary-v1]],
  [[registry-canary-real-confirmation-v1]], [[registry-canary-backed-depth-confirmation-v1]],
  [[real-cohort-live-soak-v1]].

## recommended next slice

**Starter-Missing Canary Confirmation + Allowlist Widening v1 (operator-approved):** with both starter_missing
regimes now real-soak-clean, run one canary-authoritative confirmation per regime (as was done for the enriched
regimes), then -- in the same or a follow-up slice -- explicitly widen the canary allowlist to the four
runtime-confirmed regimes. The `named` and single-book `backed` regimes remain representative-only pending real
evidence.
