---
title: "Daily Evidence Late-Slate Reevaluation 2026-07-23 v1"
type: "evidence-report"
date: "2026-07-23"
status: "complete -- fresh free cycle re-qualified 3 candidates; midday postponement claim CORRECTED (date-bucket misread); second paid screen GO recommended"
project: "DAI"
slice: "daily evidence operation 2026-07-23 (late-slate reevaluation; no work item)"
repos:
  dai: "unchanged (read-only; integrated CLIs invoked from main 85af96d)"
  dai-vault: "docs only; local branch ops/2026-07-23-late-slate-reevaluation, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - reconciliation
related:
  - "06 Execution/reports/daily-evidence-paid-screen-pass2-capture-blocked-2026-07-23-v1.md"
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
---

# daily evidence late-slate reevaluation 2026-07-23 v1

## correction of record: the midday "postponement" was an operational misread

The published paid-phase report states the sole Pass-2 primary (gamePk 823042 ARI@STL)
was weather-postponed minutes after the 16:14:51Z paid screen. **That claim is wrong and
is corrected here** (the published report is not rewritten; this continuation record
supersedes its postponement claim).

What actually happened: StatsAPI returns gamePk 823042 under TWO date buckets --
2026-06-25 (the original game, `Final/Postponed`, statusCode `DI`, rescheduled to
2026-07-23T21:15:00Z) and 2026-07-23 (the makeup, which was and remains
`Preview/Scheduled`). The 16:17:00Z pre-capture check queried by bare gamePk and read
`dates[0]` -- the June-25 bucket -- and misattributed its postponed status to tonight's
makeup. Corroborating evidence that the makeup was never postponed today: the 16:11:51Z
events gate qualified it, the 16:14:51Z paid screen found 9 fresh books pricing it, and
the 17:43Z-17:45Z date-filtered queries show `dateBucket=2026-07-23, Scheduled,
gameDate 2026-07-23T21:15:00Z, rescheduledFrom 2026-06-25`.

**Consequence:** a deterministically proposed, fully bound capture was blocked on a
false premise. The 2-credit screen expired unused. Root cause: an ad-hoc status check
that was not date-bracket-aware, on exactly the makeup/doubleheader identity class the
runbook warns about. Operational lesson (no code change made or authorized): every
game-status recheck must filter to the frozen eastern date bucket (`date=` + gamePk, or
the canonical finals-guard path), never `dates[0]` of a bare gamePk query.

## prior paid-phase closure (re-proven live)

Database at 17:46Z: 303 total rows, 138 reconciled -- identical to post-settlement
state. No second paid screen, no capture, no model call, no new run appeared. Quota
exactly at the documented baseline: used 289 / remaining 211.

## remaining-game inventory (fresh, 17:44:56Z slate)

| gamePk | game | live status | start | cutoff | disposition |
|---|---|---|---|---|---|
| 824893 | SD@ATL | In Progress | 16:15Z | passed | excluded (in progress) |
| 824406 | MIN@CLE | In Progress | 17:10Z | passed | excluded (in progress) |
| 822785 | TB@TOR | Pre-Game | 19:07Z | 18:07Z | qualified; window ~20 min at observation -- operationally expiring |
| 823042 | ARI@STL | **Scheduled** (makeup of 06-25) | 21:15Z | 20:15Z | **qualified, ~2.5h runway** |
| 824247 | KC@DET | Scheduled | 22:40Z | 21:40Z | qualified; prior screen excluded it (`outside_market_contrast_range` 0.0158 at 16:14Z) |

## tb@tor caller-state analysis

Morning drop cause: the frozen 15:07Z slate recorded `Scheduled`; live state at the
16:11Z preflight was `Pre-Game`; the gate fail-closed with `caller_state_mismatch`.
Under the fresh 17:44:56Z slate (caller state `pregame`) the candidate **qualified
cleanly** (event `36ba7a8a8c46e8cc308c1dd037995889`, fingerprint `7588f8f0...`).
Classification: newly eligible under a fresh request; the transition was routine
game-day progression, so this is flagged as a **possible policy over-constraint**
recommending a narrowly scoped later review. The policy was not modified or bypassed.

## fresh free cycle (attempt-3)

- Pass-1: fresh request (as-of 17:44:56Z, current statuses incl. two in-progress and
  the makeup as `scheduled`), validated, board 2.3 published (expected pass-1 outcome
  `EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE`).
- Free preflight: `completed_preflight_no_paid_call`, bundle `d353607b...`, $0.
- Events gate (one call, 17:45:58Z): **zero-quota audit passed** (`x-requests-last=0`,
  used 289, remaining 211), artifact `1df430c4...`. Three qualified bindings, zero
  ambiguity, zero reversed orientation; in-progress games preblocked
  (`schedule_state_in_progress`).

| gamePk | provider event | fresh fingerprint |
|---|---|---|
| 822785 | `36ba7a8a8c46e8cc308c1dd037995889` | `7588f8f0...` |
| 823042 | `8e2411581a27c6dccd9eae0b233fe64c` | `a8b30b36...` |
| 824247 | `38d3052ec454022a14314b324e0ff5f3` | `2f301dce...` |

Prior bindings and the 16:14Z paid bundle are historical evidence only -- they carry no
current execution authority; the next paid screen re-freezes everything.

## second-screen decision: `JULY23_SECOND_SCREEN_GO_RECOMMENDED`

Grounds (not sunk-cost recovery):

1. **823042 ARI@STL** has ~2.5 hours of runway, a fresh qualified binding, and
   materially strong recent evidence: 77 minutes ago the paid screen classified it
   **includable / tier primary** (disagreement 0.0187, 9 books) and Pass-2 proposed it
   as sole primary. Its blocked capture was a false-premise block, not a market or
   identity defect. Principal risk: makeup-game context (original was weather-postponed
   06-25) -- recorded as risk evidence; no new exclusion rule invented.
2. Three candidates are technically valid (exceeds the two-candidate default), though
   TB@TOR's cutoff (18:07Z) makes it effectively decision-expired, and KC@DET's prior
   exclusion is unchanged by any new evidence -- it rides the screen only because the
   bracket-wide call re-prices it for free.
3. One screen (~2 credits) covers the whole remaining bracket; time suffices for
   screen, deterministic Pass-2, and date-bucket-correct pre-capture verification.

Latest safe operator decision: ~19:55Z for ARI@STL alone (~20:00Z conservative);
TB@TOR requires action before ~17:55Z and should be considered lapsed.

## boundaries honored

No paid call, no capture, no model call, no settlement, no policy/threshold change, no
source change, no merge/push/remote branch/PR; preserved WI-0035 worktree untouched;
prior published reports not rewritten. Artifacts under
`<DAI_WORKSPACE_ROOT>/daily-evidence-flight-2026-07-23/attempt-3/` and
`<DAI_WORKSPACE_ROOT>/events-gate-2026-07-23/attempt-3/`.
