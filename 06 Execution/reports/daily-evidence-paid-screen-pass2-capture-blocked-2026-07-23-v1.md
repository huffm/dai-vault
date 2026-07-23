---
title: "Daily Evidence Paid Screen, Pass-2, and Capture Disposition 2026-07-23 v1"
type: "evidence-report"
date: "2026-07-23"
status: "complete -- one paid screen (2 credits); pass-2 proposed exactly one candidate; capture BLOCKED at the pre-capture gate by a live weather postponement; zero captures, zero model calls"
project: "DAI"
slice: "daily evidence operation 2026-07-23 (paid phase; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs and commands invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-daily-evidence, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - capture
related:
  - "06 Execution/reports/daily-evidence-planner-free-preflight-2026-07-23-v1.md"
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
---

# daily evidence paid screen, pass-2, and capture disposition 2026-07-23 v1

## opening and duplicate-operation gate

Entry 2026-07-23T16:09:49Z (12:09 EDT). dai `85af96d` == origin, clean. Vault main
`533fd74` == origin; ops branch tip `6ef0415` (parent `533fd74`), clean ops worktree.
Duplicate checks: no July 23 screen/capture artifact roots existed; reconcile-precheck
returned 0 active runs for all five slate gamePks; quota trail (used 287 / remaining 213)
matched the morning observation exactly. Preserved WI-0035 worktree untouched throughout.

## freshness recheck (authorized replacement free calls)

Morning artifacts (15:10Z) were ~60 minutes old -- stale under the 15-minute reuse rule.
Executed exactly one replacement free preflight (bundle `356e553c...`,
`completed_preflight_no_paid_call`) and one zero-quota `/events` re-observation
(artifact `cb7dbbef...`, audit passed, `x-requests-last=0`, used 287 unchanged).

Live cutoff and state results at 16:11:51Z:

| gamePk | game | disposition |
|---|---|---|
| 823042 | ARI@STL 21:15Z | qualified (event `8e2411581a27c6dccd9eae0b233fe64c`) |
| 824247 | KC@DET 22:40Z | qualified (event `38d3052ec454022a14314b324e0ff5f3`) |
| 822785 | TB@TOR 19:07Z | dropped: `caller_state_mismatch` (gate fail-closed; not relaxed) |
| 824406 | MIN@CLE 17:10Z | dropped: `insufficient_start_margin` (cutoff 16:10Z) |
| 824893 | SD@ATL 16:15Z | dropped: `insufficient_start_margin` (stayed excluded) |

## paid screen (exactly one)

One `/odds` request at 16:14:51Z (h2h+spreads, us, eastern bracket 04:00Z->04:00Z),
bundle `market-contrast-screen-bundle/1.4` sha
`bc475cd0aada05ee0102daae1b6ee55e842d84aacaa2ff12ce22b4e7e6e4b7c5`, attempt id
`202d6ac01f6a`. Provider cost headers: `x-requests-last=2`, used 287->289, remaining
213->211 -- exactly the expected 2 credits, fully reconciled. 5 provider events received,
9 fresh books on both qualified candidates:

- **823042 ARI@STL: includable, tier primary** (disagreement range 0.0187; binding
  fingerprint `478cfe50fc534fd1b1efe3aef371736689971754825bc6e6bbaa530f0dcda268`).
- **824247 KC@DET: excluded, `outside_market_contrast_range`** (0.0158) -- canonical
  threshold, not relaxed.

## deterministic pass-2

Replay context frozen from the producer receipt (sha `2368bbca...`); replay validated the
bundle hash, pass-1 hash, attempt id, schema, and bracket, and emitted pass-2 request
`79c46816...`. Board `229ba2e0...`: **COHORT_PROPOSED_FOR_OPERATOR_REVIEW, primary pool
[823042], reserve empty.** No threshold, scoring, or board content was altered.

## flight freeze

`flight-2026-07-23-paid-cohort-1`, freeze fingerprint
`a3560dc883749f40e0109ed95ea16abf81454366384245e9aa17fc6b127ab671`, plan schema
`wildcard-flight-plan/1.2`, provenance `flight-selection-provenance/1.1` (sha
`77a70c55...`) carrying the verbatim provider-event binding and the replay reference;
1 scheduled core run, wildcard mode disabled, all authority ledgers false.

## capture disposition: BLOCKED (zero captures)

The immediate pre-capture gate at 16:17:00Z re-checked live StatsAPI status for 823042
and found **Postponed** -- `statusCode DI`, reason **Inclement Weather**,
`codedGameState D`, `abstractGameState Final`. The postponement landed in the minutes
after the paid screen (16:14:51Z still showed the game screenable). The game record shows
original gameDate 2026-06-25 with rescheduleDate 2026-07-23T21:15:00Z -- the June 25
makeup was postponed again. Capture of a postponed game is prohibited; it was not
attempted.

Substitution was not permitted: pass-2 proposed only 823042. 824247 was rejected by the
canonical screen threshold, and the standing ranking may not override a board rejection.
Captures attempted: 0. Runs created: 0. Database writes: 0. Model calls: 0.

The paid market evidence for the slate is preserved immutably in the frozen bundle for
audit; it grounds no capture and no settlement. The postponed game will reappear on a
future slate under whatever identity StatsAPI assigns; the next planner cycle must
re-derive it from authoritative sources, not from this record.

## cost ledger (this paid session)

| operation | count | expected credits | actual credits | model cost |
|---|---|---|---|---|
| freshness `/events` | 1 | 0 | 0 (audited, last=0) | $0 |
| paid market screen | 1 | 2 | **2** (last=2, 287->289) | $0 |
| capture 1 | 0 | ~3 | 0 | $0 |
| capture 2 | 0 | ~3 | 0 | $0 |
| **total** | | max 8 | **2** | **$0 / $0.01 ceiling** |

Header deltas reconcile exactly (287 + 2 = 289; 213 - 2 = 211).

## unsettled reconciliation queue

Empty for July 23: no capture exists, nothing awaits settlement. The July 22 canary was
already settled this morning. Gate 4 unchanged (FALSE); no calibration claim is made from
today's paid evidence.

## boundaries honored

Exactly one paid call; no second paid request; no retry; no capture; no settlement; no
threshold or policy change; no source/schema/test change; no merge, push, remote branch,
or PR; preserved WI-0035 worktree untouched; ceiling 8 credits not approached (2 used).

## artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/daily-evidence-capture-2026-07-23/paid-cohort-1/` (bundle, replay
context, pass-2 request/board, flight request/plan/realization/provenance, logs, attempt
manifest), `<DAI_WORKSPACE_ROOT>/daily-evidence-flight-2026-07-23/attempt-2/`,
`<DAI_WORKSPACE_ROOT>/events-gate-2026-07-23/attempt-2/`. Key hashes recorded in the
attempt manifest; no secrets or raw provider payloads persisted in either repository.
