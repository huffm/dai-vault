---
title: "Daily Evidence Second Screen and Capture 2026-07-23 v1"
type: "evidence-report"
date: "2026-07-23"
status: "complete -- one paid screen (2 credits); pass-2 pivoted to 822785 after the anchor was preblocked; ONE capture frozen 4.8 min before cutoff; unsettled"
project: "DAI"
slice: "daily evidence operation 2026-07-23 (second-screen paid phase; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs and API invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-late-slate-reevaluation, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - capture
related:
  - "06 Execution/reports/daily-evidence-late-slate-reevaluation-2026-07-23-v1.md"
  - "06 Execution/reports/daily-evidence-paid-screen-pass2-capture-blocked-2026-07-23-v1.md"
---

# daily evidence second screen and capture 2026-07-23 v1

## opening and corrected status discipline

Entry 17:55:13Z. dai `85af96d`==origin clean; vault main `5bf9b91`==origin; branch
`ops/2026-07-23-late-slate-reevaluation` tip `03a576c`, worktree clean; preserved WI-0035
worktree untouched (six paths). No second screen or capture existed (only `paid-cohort-1/`);
DB baseline 303/138; quota baseline exact (289/211). **Every game-status check in this
operation used the date-bucket-constrained query (`date=2026-07-23` + gamePk, exactly one
match required); no positional `dates[0]` access was used.**

## freshness and free gates (attempt-4, $0)

Attempt-3 artifacts would have exceeded the 15-minute rule at screen time, so exactly one
replacement free preflight ran (bundle `ef71e1c7...`) plus one zero-quota `/events`
re-observation (artifact `40b98143...`; audit passed, `x-requests-last=0`, 289/211).
Qualified bindings: 822785, 823042, 824247; both in-progress games preblocked. Pre-screen
date-bucketed statuses: 823042 `Pre-Game` (one match), 824247 `Scheduled` (one match).
822785 was treated as capture-expired at the pre-call ledger (cutoff 18:07Z).

## paid screen (exactly one)

17:57:15Z, one `/odds` call, bundle 1.4 `4e8f32df...` (attempt `4b7269d08807`):
`x-requests-last=2`, used 289->291, remaining 209 -- 2 credits, reconciled.

| gamePk | result |
|---|---|
| **822785 TB@TOR** | **includable, tier primary, disagreement 0.0360 (strongest today), 9 books** |
| 823042 ARI@STL | `skipped_preblocked caller_state_mismatch` -- frozen 17:44Z slate said `scheduled`, live state at screen was `pregame`; market NOT evaluated |
| 824247 KC@DET | excluded, `outside_market_contrast_range` (0.0190) |

**Operational finding (second demonstration):** the `caller_state_mismatch` fail-closed
rule has now cost two candidates in one day on routine Scheduled->Pre-Game drift --
TB@TOR at the 16:11Z gate and 823042 at this 17:57Z paid screen (it passed the
date-bucketed status gate minutes earlier). This is recorded as a confirmed
over-constraint candidate needing a narrowly scoped review slice. No policy was changed
or bypassed in this operation.

## deterministic pass-2 and capture

Replay context frozen from the receipt (pass-1 `cfd0bfd6...`, bundle `4e8f32df...`);
replay emitted pass-2 request `a7c79186...`; board (prefix `0b2b1a28`):
**COHORT_PROPOSED_FOR_OPERATOR_REVIEW, primary [822785], reserve empty.** 823042 was
`input_not_evaluated` (not board-proposed; not captured; no substitution performed);
824247 remained board-rejected.

Pre-capture gate at 17:59:10Z: 822785 `Pre-Game`, exactly one bucket match, cutoff
18:07:00Z NOT crossed (7m50s remaining), binding valid, no duplicate, quota within
ceiling. The full published path executed: flight `flight-2026-07-23-paid-cohort-2`
(freeze fingerprint `45796b4e...`, wildcard disabled, 1 core run), verbatim binding
(event `36ba7a8a8c46e8cc308c1dd037995889`, fingerprint `78d8fe3a...`), provenance 1.1
with replay reference, binding-aware run request, execution with Slice-D enforcement.

**Run frozen: `d329433e-f36b-1410-8196-00373db4b724`** created 18:02:04-18:02:13Z --
4.8 minutes before the cutoff, 64.8 minutes before first pitch. Lean `away` (Tampa Bay
Rays) @ 0.75, posture monitor. Row verified: market consensus `away` (9 books,
market-agreed), observed regime `starter_enriched_market_backed_depth`, stratum `core`,
writeback fingerprint == provenance freeze fingerprint, attribution fidelity Pass,
exclusion null, **outcome null (UNSETTLED)**. AgentRuns 303 -> 304.

## cost ledger (this session)

| operation | count | credits | model |
|---|---|---|---|
| freshness `/events` | 1 | 0 (audited) | $0 |
| paid screen | 1 | **2** (last=2, 289->291) | $0 |
| capture 822785 | 1 | ~3 (inferred; confirm at next zero-quota header read) | ~$0.0007 (one gpt-4o-mini call, under $0.01 ceiling) |
| **session total** | | **~5 of 8 ceiling** | **< $0.01** |

Whole-day provider spend: 2 (first screen) + 2 (second screen) + ~3 (run) = ~7 credits.

## unsettled reconciliation queue

| run | gamePk | game | first pitch | earliest settlement |
|---|---|---|---|---|
| `d329433e-f36b-1410-8196-00373db4b724` | 822785 | TB@TOR | 2026-07-23T19:07:00Z | after official Final (finals guard), likely late evening 07-23 ET |

Settlement is NOT authorized tonight under this operation.

## boundaries honored

One paid screen only; no retry; no substitution of board-rejected candidates; no
settlement; no policy/threshold/source change; no merge/push/remote branch/PR; preserved
worktree untouched; prior reports not rewritten. Artifacts under
`<DAI_WORKSPACE_ROOT>/daily-evidence-capture-2026-07-23/paid-cohort-2/` plus
`.../daily-evidence-flight-2026-07-23/attempt-4/` and `.../events-gate-2026-07-23/attempt-4/`.
