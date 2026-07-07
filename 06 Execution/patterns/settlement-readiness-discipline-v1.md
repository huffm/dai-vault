---
title: "PATTERN: Settlement Readiness Discipline v1"
type: "pattern"
date: "2026-07-07"
status: "ACTIVE"
project: "DAI"
related:
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
  - "06 Execution/patterns/gate4-evidence-readout-template-v1.md"
---

# settlement readiness discipline v1

purpose: stop settlement slices from starting while cohort games are still preview,
scheduled, live, or otherwise not final. the 2026-07-07 cohort settlement was invoked
three times before first pitch; the internal finals gate blocked correctly every time,
but only after a slice had spun up and re-verified everything. the readiness guard makes
"not time yet" a one-command answer.

## 1. when to run the readiness guard

BEFORE launching any settlement slice -- before preflight, before snapshots, before any
settlement phase:

```
pwsh dai/scripts/dev/sports/check-settlement-finals.ps1 -Competition mlb -GamePks <pks>
```

a settlement slice starts only on READY (exit 0). anything else stops the slice at
phase 0 with a one-line note; no preflight, no before snapshot, no /reconcile.

## 2. status meanings

- READY -- every target game is Final per the authoritative source (statsapi
  abstractGameState=Final AND codedGameState=F). settlement may proceed to preflight.
- BLOCKED -- no target game Final (or a mix of not-final states with none defective).
  do not run settlement phases. the output includes a recommended next check time.
- PARTIAL -- some Final, some not. never settlement-ready (partial settlement is
  forbidden unless a capture handoff explicitly allows it). -FailOnPartial only hardens
  the message; the exit code is 2 either way.
- DEFECT -- readiness cannot be determined safely: gamePk missing from or duplicated in
  the source, postponed/cancelled/suspended games (those need a human outcome decision,
  not a finals wait), unsupported competition, malformed manifest, source error, or
  (with -CheckLocalRows) missing/duplicate/already-reconciled local rows. human review.

## 3. exit codes

0 = READY, 1 = BLOCKED, 2 = PARTIAL, 3 = DEFECT. stable; safe to script against.

## 4. how it composes with the other gates

- `check-settlement-finals.ps1` -- EXTERNAL final-state readiness (this guard). "is it
  time?"
- `scripts/dev/sports/preflight-settlement.ps1` -- INTERNAL cohort/write-safety
  (identity, provenance, unreconciled, market baseline). "is the cohort safe to write?"
- the reconcile POST endpoint -- the settlement write itself.

run them in that order. the guard never replaces preflight; preflight never replaces the
guard. the guard is read-only by construction (its test suite asserts no reconcile
endpoint reference and no POST anywhere in its source).

## 5. what it does not do

- no settlement writes, no db writes, no agent-service, no model calls
- no scheduler, no background polling, no auto-settlement
- never infers finality from elapsed time -- only from the source's stated state
- "game over" (coded O) is not final; postponed/cancelled/suspended are never READY

## 6. scheduling language correction (process rule)

- do not promise "scheduled cloud settlement" or "I'll settle this tonight" unless an
  actual scheduler/runner exists with access to the repo, local services, credentials,
  the database, and standing approval. none exists today, by design.
- a chat reminder or calendar reminder is NOT autonomous settlement execution.
- the correct near-term pattern is: operator runs the readiness guard; on READY, the
  operator runs the settlement slice.
- repeated blocked attempts must not create duplicate vault handoffs unless state
  changed materially; a blocked guard run is a console result, not a report.

## 7. example -- the 2026-07-07 cohort

```
pwsh dai/scripts/dev/sports/check-settlement-finals.ps1 -Competition mlb `
  -GamePks 823687,824820,822956,822713,823280,824579 `
  -CheckLocalRows -RequireUnreconciled
```

validated live 2026-07-07 (~11:45 ET): BLOCKED exit 1, 6/6 not final, recommended next
check 2026-07-08T04:50:00Z (~00:50 ET; latest first pitch 01:40Z + typical duration).
add `-JsonOnly` and/or `-OutputPath <file>` for machine-readable output.

optional flags: `-ManifestPath <frozen slate json>` instead of -GamePks;
`-CheckLocalRows` verifies each gamePk has exactly one active /rows row;
`-RequireUnreconciled` additionally fails DEFECT if a target run already has an
outcome/evaluation. tests: `pwsh dai/scripts/dev/sports/test-check-settlement-finals.ps1`
(offline, sample payloads via -ScheduleJsonPath injection; 32 assertions).
