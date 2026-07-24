---
title: "WI-0037 Slice 1 Caller-State Progression 2026-07-24 v1"
type: "evidence-report"
date: "2026-07-24"
status: "implemented local -- RED-first correction complete, suites green; NOT reviewed, NOT integrated, NOT pushed"
project: "DAI"
slice: "WI-0037 Slice 1: monotonic caller-state progression"
repos:
  dai: "code+tests on local branch wi/0037-game-state-correctness-slice-1 (base 85af96d); NOT integrated"
  dai-vault: "docs on local branch wi/0037-game-state-correctness-slice-1 (base 51b64f4); NOT pushed"
tags:
  - system-development
  - sports-v1
  - correctness
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/daily-evidence-second-screen-capture-2026-07-23-v1.md"
---

# wi-0037 slice 1 caller-state progression 2026-07-24 v1

## opening state

dai main == origin/main == `85af96d` (clean, no implementation commits after it); vault
main == origin/main == `af69725` at gate, then fast-forward published to `51b64f4` (the
reviewed WI-0037 planning record) before any source work; planning commit `51b64f4`
published unchanged after independent review (docs-only, single registration, evidence
references resolve, WI-0035 untouched at complete). Slice branches:
`wi/0037-game-state-correctness-slice-1` in dai (base `85af96d`) and dai-vault (base
`51b64f4`). Preserved `wi/0035-provider-event-binding` worktree untouched (six dirty
paths, content hash verified unchanged).

## pre-edit contract map

- frozen/caller state field: `ScreenSlateCandidate.ScheduleState` (slate assertion)
- authoritative/live state field: statsapi `entry.DetailedState`, normalized
- normalized vocabulary (`NormalizeScheduleState`): scheduled, pregame, in_progress,
  final, postponed, cancelled, suspended, unknown ("warmup" normalizes to pregame)
- screenable authoritative states: scheduled, pregame (line 349); all others block as
  `schedule_state_{state}` BEFORE any caller-state comparison
- prior comparison predicate: ordinal equality of normalized caller vs authoritative
  state; any inequality returned `caller_state_mismatch`
- gate precedence (unchanged by this slice): statsapi source failure -> identity
  unresolved/ambiguous/unmatched -> `caller_team_mismatch` -> `caller_start_mismatch`
  (exact instant equality) -> `schedule_state_*` -> `caller_state_mismatch` ->
  `insufficient_start_margin` -> `starters_not_announced` -> active-run gates
- reason codes surface in the published screen bundle json (`preblock_reason`); no
  public contract shape involved; policy version `MarketContrastPolicy.Version`
  untouched

## red characterization (before any production edit)

Added to `platform/dotnet/DevCore.Api.Tests/AgentRuns/MarketContrastSourceAdapterTests.cs`:

- `caller_state_transition_matrix` -- 18-row table-driven theory over ordered
  (caller, authoritative) pairs: identical screenable pairs accepted; warmup/pregame
  normalization equivalence accepted; the one authorized progression
  (`Scheduled -> Pre-Game`); backward progression, every non-screenable caller state,
  unknown/empty caller states fail-closed as `caller_state_mismatch`; every
  non-screenable authoritative state blocks as its `schedule_state_*` reason with
  precedence over the caller comparison.
- `scheduled_candidate_survives_natural_pregame_progression` -- operationally realistic
  July 23 fixture: frozen `Scheduled`, identical identity, exact start instant,
  authoritative state `Pre-Game`, market inputs valid.
- `pregame_progression_does_not_weaken_start_instant_blocker` -- same progression with a
  moved authoritative start instant stays blocked by `caller_start_mismatch`.

RED proof (command: `dotnet test ... --filter FullyQualifiedName~MarketContrastSourceAdapterTests`):
exit 1, **Failed: 2, Passed: 45, Skipped: 0, Total: 47** -- the two failures were exactly
the progression rows, each failing `Expected: "completed" / Actual:
"completed_no_paid_call"` because current code returned `caller_state_mismatch`. No
unrelated failures.

## transition matrix (caller x authoritative -> reason)

| caller \ auth | scheduled | pregame | non-screenable X |
|---|---|---|---|
| scheduled | accept (unchanged) | **accept (NEW -- the only change)** | `schedule_state_X` (unchanged) |
| pregame | `caller_state_mismatch` (backward, unchanged) | accept (unchanged) | `schedule_state_X` (unchanged) |
| warmup (=pregame) | `caller_state_mismatch` | accept (normalization, unchanged) | `schedule_state_X` |
| in_progress / final / postponed / suspended / cancelled / unknown / empty | `caller_state_mismatch` (unchanged) | `caller_state_mismatch` (unchanged) | `schedule_state_X` |

## production correction

`platform/dotnet/DevCore.Api/AgentRuns/MarketContrastSourceAdapter.cs` lines 348-357:
the equality predicate now admits identical states PLUS exactly
`callerState == "scheduled" && state == "pregame"`. Explicit boolean, no state-ranking
abstraction, no normalization change, no reason-code string change,
`caller_state_mismatch` retained for every other unequal pair, no contract shape or
policy-version change, no authority change. 8 lines changed in the adapter; nothing
else touched.

## verification

- focused adapter suite: **47/47** (exit 0)
- related screen/binding/envelope suites (`MarketContrast|ProviderEventBinding|PlannerEnvelope`):
  **343/343**
- full suite `dotnet test platform/dotnet/DevCore.Api.Tests/DevCore.Api.Tests.csproj
  --no-restore -v minimal`: **1780/1780 passed, 0 failed, 0 skipped** (baseline 1760 +
  exactly the 20 new tests)
- no dependency installed or upgraded; no snapshot altered; scope audit: exactly two
  changed paths (adapter + its tests); `git diff --check` clean; no machine paths or
  secrets

## exclusions preserved

No change to: start-instant tolerance (sibling exact-equality branch proven intact),
date-bucket resolution (`MlbEventResolver`, `OddsScheduleClient` untouched --
WI-0037 Slice 2, NOT authorized), finals/settlement scripts, schemas, migrations,
planner, provider binding, thresholds, policy versions, paid-screen authority. Slice 2
remains unauthorized and unimplemented. WI-0037 is NOT complete; Slice 1 is NOT
integrated and NOT published.

## residual questions

- Whether a bounded start-instant tolerance beyond WI-0035 Slice A's
  `bounded_start_tolerance` qualification is wanted at the pre-elimination gate --
  named only; behavior unchanged.
- Whether `warmup` (normalizing to pregame) should also be admitted from frozen
  `scheduled` -- it already is, via normalization (`Scheduled -> Warmup` normalizes to
  scheduled->pregame and is covered by the new allowance); noted for reviewer
  attention.

## commits and branch state

- dai: `0a9129b` "fix(screen): allow scheduled to pregame progression" on
  `wi/0037-game-state-correctness-slice-1` (base `85af96d`), local only, NOT pushed,
  NOT integrated; working tree clean.
- dai-vault: one docs commit on `wi/0037-game-state-correctness-slice-1` (base
  `51b64f4`, the published planning tip), local only, NOT pushed -- this record, the
  WI-0037 spec slice-1 state note, and the current-slice handoff.

## next governed action

Independent adversarial review of the Slice 1 chain (both branches), then a separate
integration authorization. No push, merge, or integration is authorized by the
implementing slice.
