# Multi-Slate Regime Coverage v1 -- Coverage Matrix

**status:** active doctrine (coverage matrix built from real data + multi-slate discovery; allowlist NOT widened)
**date:** 2026-06-28

## purpose

Record which of the 9 MLB prompt regimes have real-data evidence, at what level (canary-authoritative confirmed,
real-soak-clean, representative-only, or unavailable), so a later allowlist-widening slice has an evidence base.
This slice expands evidence; it does NOT widen the registry-authoritative allowlist.

## problem it solves

The canary allowlist has two runtime-confirmed regimes; the other 7 were "fixture-only" with no statement about
whether they even occur in real MLB inputs. This matrix answers that with real captured data + non-invasive
multi-slate discovery.

## what it is not

- NOT an allowlist widening (the two-regime allowlist is unchanged). NOT broad routing. NOT prompt tuning. NOT a
  buyer change. NO paid model calls were run this slice.

## method (zero new paid calls)

1. Re-soaked the EXISTING real captured inputs from prior slices (15 unique real matchups from 2026-06-28,
   captured via the default-off request capture) through `run_shadow_cohort_soak.py --requests`.
2. Multi-slate discovery via statsapi `schedule?hydrate=probablePitcher` (free, no model call) across
   2026-06-28/29/30 to estimate starter-side availability (both-announced -> enriched/named; one-announced /
   none -> starter_missing candidate).

No regime was fabricated. Regimes not observed in real data are marked representative-only or unavailable.

## real-cohort soak result (existing captures, no paid calls)

`run_shadow_cohort_soak.py --requests <scratch>/coverage_cohort.jsonl --sink <scratch>/coverage_soak.jsonl`:
cohort_size 15, attempted 15, matched 15, captured 15, mismatched 0, assembly_failed 0, sink_failed 0,
errored 0, partial_evidence_unrepresentable 0, persisted_provenance_lines 15, clean true, GO. Real regimes:
`starter_enriched_market_backed_depth` 1, `starter_enriched_market_missing` 14.

## multi-slate starter discovery (statsapi, free)

| date | games | both-announced | one-announced | none-announced |
|---|---|---|---|---|
| 2026-06-28 | 15 | 15 | 0 | 0 |
| 2026-06-29 | 13 | 11 | 2 | 0 |
| 2026-06-30 | 15 | 14 | 1 | 0 |

Observation: real slates are overwhelmingly both-announced (enriched). A few one-announced games exist (the only
realistic path to a `starter_missing` regime, since `MlbStarterContext` requires both starter names). No
"announced-but-no-season-stats" signal was observed (the `named` state), and prior real markets were always
multi-book (`backed_depth`) or absent (`missing`) -- never single-book (`backed`).

## coverage matrix (all 9 regimes)

| regime | source | real count | matched | captured | mismatch | assembly fail | partial-evidence | allowlist | recommendation |
|---|---|---|---|---|---|---|---|---|---|
| starter_enriched_market_missing | real canary + real soak | 14 | 14 | 14 | 0 | 0 | 0 | ALLOWLISTED | keep |
| starter_enriched_market_backed_depth | real canary + real soak | 1 (+confirmation runs) | 1 | 1 | 0 | 0 | 0 | ALLOWLISTED | keep |
| starter_enriched_market_backed | representative-only | 0 | - | - | - | - | - | off | single-book market not observed in real data; candidate for a targeted capture |
| starter_named_market_missing | representative-only | 0 | - | - | - | - | - | off | "named" (announced, no season stats) not observed; likely rare |
| starter_named_market_backed | representative-only | 0 | - | - | - | - | - | off | not observed |
| starter_named_market_backed_depth | representative-only | 0 | - | - | - | - | - | off | not observed |
| starter_missing_market_missing | representative-only (real-available) | 0 | - | - | - | - | - | off | reachable via one-announced games (discovery); needs a paid capture |
| starter_missing_market_backed | representative-only | 0 | - | - | - | - | - | off | needs paid capture + single-book market (unlikely) |
| starter_missing_market_backed_depth | representative-only (real-available) | 0 | - | - | - | - | - | off | reachable via one-announced + multi-book game; needs a paid capture |

Legend: "real canary" = runtime canary confirmation ([[registry-canary-real-confirmation-v1]],
[[registry-canary-backed-depth-confirmation-v1]]); "real soak" = clean shadow soak on real captured input;
"representative-only" = proven only by the 9-regime fixture equivalence tests; "real-available" = real inputs of
this regime exist on observed slates but have not been captured (would require a paid analyzer run).

## which regimes are runtime-confirmed canary-authoritative

`starter_enriched_market_missing`, `starter_enriched_market_backed_depth` -- the two allowlisted regimes, each
with a real canary confirmation AND a clean real soak.

## which regimes are real-soak-clean only

None beyond the two allowlisted (they are both real-soak-clean AND canary-confirmed). No additional regime has a
real soak yet.

## which regimes remain representative-only

All 7 unconfirmed regimes. Of these, `starter_missing_market_missing` and `starter_missing_market_backed_depth`
are REAL-AVAILABLE (one-announced games appear on other slates) but uncaptured; the `named` regimes and the
single-book `backed` regimes were not observed in real data on the discovered slates.

## which regimes were unavailable

On the fully-captured slate (2026-06-28): all 7 unconfirmed regimes -- every game was enriched + (depth or
missing) market. Across discovery (6/28-30): `named` and single-book `backed` regimes were not observed.

## paid call count

0. No new paid model calls this slice (re-soak used existing real captures; discovery was statsapi-only).

## why the allowlist was not widened

(1) The slice explicitly forbids widening. (2) No new regime earned a clean real soak this slice -- the only
real-soak-clean regimes remain the two already allowlisted. Widening requires capturing real inputs of the
target regimes (paid) and a clean soak, which is the next slice.

## exact commands

```
# re-soak existing real captures (no paid calls):
python scripts/run_shadow_cohort_soak.py --requests <scratch>/coverage_cohort.jsonl --sink <scratch>/coverage_soak.jsonl
# multi-slate starter discovery (free):
curl "https://statsapi.mlb.com/api/v1/schedule?sportId=1&date=<YYYY-MM-DD>&hydrate=probablePitcher"
```

## exact tests run

- `pytest -q` (full agent-service suite) -> **330 passed, 0 failed**.
- `pytest tests/test_registry_prompt_canary.py -q` -> 10 passed.
- `pytest tests/test_mlb_prompt_equivalence.py tests/test_mlb_branch_overlays.py -q` -> 30 passed.
- `python scripts/check_prompt_manifest.py` -> OK (8 templates, 9 recipes), exit 0.
- No code changed this slice.

## remaining risks and gaps

- **7 regimes lack real-soak evidence.** Two of them (`starter_missing_market_missing`,
  `starter_missing_market_backed_depth`) are real-available but uncaptured; the rest were not observed.
- **`named` and single-book `backed` may be hard to source.** They did not appear on three discovered slates;
  earning real evidence may require targeted opponents/dates or may be genuinely rare.
- **Modest sample.** One slate fully captured; discovery over three.

## deferred decisions

- A paid `starter_missing` capture slice (one-announced games) to earn real-soak evidence for the missing-starter
  regimes. Allowlist widening (a later slice, using accumulated evidence). DB persistence; `.NET sourceDepth ->
  dataRegime` ownership; caching -- unchanged deferrals.

## related docs

- [[registry-authoritative-prompt-canary-v1]], [[registry-canary-real-confirmation-v1]],
  [[registry-canary-backed-depth-confirmation-v1]], [[real-cohort-live-soak-v1]], [[live-data-shadow-soak-v1]].

## recommended next slice

**Starter-Missing Regime Capture v1 (operator-approved paid):** target the one-announced games discovered on a
real slate (e.g. 2026-06-29) to capture `starter_missing_market_*` real inputs, run the shadow soak, and record
whether they are real-soak-clean -- the only honest path to evidence for those regimes. Only after a regime has a
clean real soak should a separate allowlist-widening slice consider adding it to the canary. The `named` and
single-book `backed` regimes may require dedicated sourcing or may remain representative-only.
