# Null-Lean MLB Rerun with Supersession v1

**date:** 2026-06-15
**status:** executed against local/dev. 3 eligible pre-start reruns generated (3 model calls), old null runs superseded via the new eligibility contract. no reconciliation, no outcomes/evaluations written (12/12 unchanged), no code change, no analyzer/prompt/matcher/confidence/buyer change.
**classification:** diagnostic rerun + supersession.

## Purpose

Determine whether the three original null-LeanSide MLB runs were stable model abstentions or timing/source-availability artifacts, and use Run Eligibility and Supersession Contract v1 to keep one active matcher candidate per game.

## Lookahead gate (passed)

Current UTC at execution: `2026-06-15 20:06Z`. StatsAPI state for all three target gamePks:

| gamePk | matchup | state | scheduled start |
|---|---|---|---|
| 823046 | SD @ STL | Preview / Pre-Game | 2026-06-15T23:45Z |
| 822887 | MIN @ TEX | Preview / Pre-Game | 2026-06-16T00:05Z |
| 824993 | PIT @ ATH | Preview / Scheduled | 2026-06-16T01:40Z |

All three pre-start (earliest first pitch ~3h39m away). None In Progress/Final. All eligible for predictive rerun.

## Old artifact trace summary (preserved)

The original runs' artifact JSONs remain in `04 Products/sports-v1/calibration/artifacts/20260615-1237-mlb-{2e03433e,3603433e,4203433e}.json` (untouched). Pre-slice eligibility: all three active (`ExclusionReason` null). Key fields:

| old run | gamePk | LeanSide | posture | conf | ev | grounded | note |
|---|---|---|---|---|---|---|---|
| 2e03433e | 823046 | null | wait | 0.375 | 0 | [] | priors-only; starting_pitching unavailable at 12:37Z |
| 3603433e | 822887 | null | wait | 0.375 | 0 | [] | priors-only; no grounded signal |
| 4203433e | 824993 | null | wait | 0.5 | 1 | [starting_pitching] | signal grounded but directionally empty (unknown handedness) |

## Rerun execution

Used the narrowest supported workflow -- the per-game create endpoint `POST /api/agent-runs` (`runType=sports.matchup.analysis`, explicit `{competition, homeTeam, awayTeam, gameDate}`) -- to target exactly the three matchups, not a broad batch. 3 runs, all `completed`, all identity-bearing (`mlb_statsapi` + correct gamePk). Model spend: **3 calls** (cap respected).

## Old-vs-new comparison

| matchup | gamePk | old run | new run | old Lean | new Lean | old grounded | new grounded | old ev | new ev | old conf | new conf | old posture | new posture | old status (post) | new status | classification |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Padres @ Cardinals | 823046 | 2e03433e | 5003433e | null | **home** | [] | [starting_pitching] | 0 | 1 | 0.375 | 0.75 | wait | monitor | superseded | active | **ImprovedSignalProducedLean** (ChangedDueToTiming -- starter announced between 12:37Z and 20:06Z) |
| Twins @ Rangers | 822887 | 3603433e | 5403433e | null | **home** | [] | [starting_pitching] | 0 | 1 | 0.375 | 0.75 | wait | monitor | superseded | active | **ImprovedSignalProducedLean** (ChangedDueToTiming) |
| Pirates @ Athletics | 824993 | 4203433e | 5703433e | null | null | [starting_pitching] | [starting_pitching] | 1 | 1 | 0.5 | 0.5 | wait | monitor | superseded | active | **StillNull_NoDirectionalSignal** (stable abstention -- signal grounded both times, model finds no directional edge) |

new lean prose: 823046 "Slight lean toward Cardinals based on starting pitching advantage"; 822887 "Slight lean toward Rangers based on starting pitching advantage with MacKenzie Gore's handedness"; 824993 "No clear lean based on the current signal picture".

## Finding

The original nulls were **not uniformly stable**. Two of three (823046, 822887) were **timing/source-availability artifacts**: probable starters were unannounced at the original 12:37Z generation, so the runs were priors-only (`grounded=[]`) and the model correctly abstained; by 20:06Z the starters were posted, `starting_pitching` grounded, and both produced a directional `home` lean. One (824993) is a **stable abstention**: the signal was grounded in both runs yet the model found no directional edge -- a genuine "no meaningful lean", not a data gap. This validates Directional Lean Contract v1: null is honest, and re-running with more signal can legitimately move null -> directional.

## New directionally usable count

- before slice: 7 active usable (original batch) + 3 active null.
- after slice: **9 active directionally usable** (7 original + 2 reruns that gained a `home` lean) + 1 active null (5703433e). The 3 old null runs are superseded (out of the active set).

## Supersession actions taken

`POST /api/agent-runs/{old}/exclude` with `reason=superseded`, `supersededByAgentRunId=<new>` for all three:

| old run | -> superseded by |
|---|---|
| 2e03433e | 5003433e |
| 3603433e | 5403433e |
| 4203433e | 5703433e |

The 7 directionally usable original runs were **not** touched (rule respected). Nothing hard-deleted; old `OutputJson`/trace preserved.

## Duplicate active key verification

Active run count per target gamePk after supersession: **823046 -> 1, 822887 -> 1, 824993 -> 1.** Each game has exactly one active run (the replacement); the matcher would auto-select only the active replacement. No duplicate active provider/game keys remain. The 7 usable runs remain active and singular per their games.

## Calibration inclusion recommendation

Treat the two reruns that gained a lean (5003433e, 5403433e) as **active eligible directional runs** -- they are legitimate pre-settlement reads on the same games and the active matcher will include them. Flag them in any calibration report as a **rerun cohort** (generated ~20:06Z with starter data, vs the 12:37Z priors-only originals) so the as-of/latency dimension stays honest; do not silently blend generation times. The superseded originals must be excluded from calibration (the contract already does this). 5703433e stays active but non-directional (no calibration contribution).

## Verification

- model calls: 3 (only the three target games; no broad generation).
- all 3 reruns identity-bearing (`mlb_statsapi` + matching gamePk), artifact v3, `completed`.
- outcomes/evaluations: 12/12 before and after -- none written; no reconciliation performed.
- one active run per target gamePk; old runs superseded with provenance link; new runs active.
- 7 usable original runs untouched (all active).
- old artifact traces preserved in the vault calibration folder.

## Recommended next slice

Unchanged primary action once tonight's games settle: **reconcile the now-9 active directionally usable MLB runs** (7 original + 2 reruns) via the proven statsapi `POST /api/agent-runs/reconcile` path; the superseded originals are correctly ignored. A small follow-up could formalize the "rerun cohort" tag in the calibration runbook so as-of grouping is explicit. WNBA remains deferred; auto-supersession-at-generation remains a deferred convenience.
