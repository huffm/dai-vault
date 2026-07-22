---
title: "Daily Evidence Acquisition Operating Workflow v1"
type: "pattern"
date: "2026-07-22"
status: "active doctrine -- current implementation is MANUAL and operator-gated; automation is a described target, not built"
project: "DAI"
slice: "WI-0034 / WI-0035 / WI-0036 composition"
repos:
  dai: "unchanged by this record"
  dai-vault: "docs only"
tags:
  - evidence-operations
  - sports-v1
  - workflow
  - system-development
related:
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
  - "02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md"
---

# daily evidence acquisition operating workflow v1

## purpose

One record describing how a governed evidence day actually runs end to end: the stage
sequence, what each stage may observe or derive, what it may never authorize, and where the
human authorization gates sit. It exists so a day can be executed, paused, resumed, or
audited without rediscovering the order from handoffs.

**Read this as doctrine about sequence and authority, not as a claim of automation.** Every
stage below is executed today by an operator issuing a scoped prompt and running a one-shot
local command. There is no scheduler, no daemon, no endpoint, and no standing authority.

## truth hierarchy

1. Runtime source and tests in `<DAI_REPO_ROOT>` for every capability claim.
2. WI-0034 / WI-0035 / WI-0036 for state and gates.
3. This record for sequence, timing, and authority boundaries.
4. Slice handoffs and reports for what actually happened on a given date.

## current implementation vs target

| stage | today | target |
|---|---|---|
| state/calibration readiness | manual read of vault records | automated precondition check |
| Planner Pass 1 | integrated CLI, operator-run | scheduled prepare step |
| free preflight | integrated one-shot operator | automated readiness poll |
| zero-quota `/events` identity gate | integrated one-shot operator, optional | automated identity assertion |
| paid screen authorization | **human gate** | **stays human unless standing authority is separately designed** |
| paid market screen | integrated one-shot operator | automated within authorized budget |
| Planner Pass 2 | integrated CLI, operator-run | automated |
| wildcard flight plan + freeze | integrated WI-0036 core/CLI (offline, default off) | automated allocation + freeze |
| capture authorization | **human gate** | **stays human unless standing authority is separately designed** |
| availability realization + substitution | integrated WI-0036 realization | automated |
| AgentRun capture | existing runtime, operator-triggered | sequential automated capture |
| settlement | existing manual/settlement path | automated postgame |
| reconciliation + coverage refresh | existing manual path | automated next-morning |

Nothing in the "target" column is built. Do not cite this table as evidence that a stage is
automated.

## the primary morning sequence

1. **State / calibration readiness.** Confirm repository refs, protected state, and the
   current calibration posture. Produces no artifact and grants nothing.
2. **Planner Pass 1.** Deterministic planning over the target-date request. With no
   market-contrast evidence present, the expected board outcome is
   `EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE` -- Pass 1 is *supposed* to stop there.
   Zero network, database, model, or gateway calls.
3. **Free preflight.** One bounded StatsAPI schedule/hydration pass plus one tenant-scoped
   active-run database read. Terminal status `completed_preflight_no_paid_call`. This is
   where *current-day freshness* enters the workflow. Structurally cannot reach the Odds
   call site.
4. **Optional zero-quota `/events` identity gate.** At most one `/v4/.../events` request,
   zero retries, audited to `x-requests-last == 0`. Answers only "do our candidates join to
   provider events by identity and exact start instant?"
5. **Paid screen authorization -- HUMAN GATE.** Default off. Nothing upstream grants it.
6. **Paid market screen.** One `/odds` request per slate under explicit authorization,
   producing the market-contrast evidence Pass 1 lacked.
7. **Planner Pass 2.** Re-plan with the screen evidence present; this is the pass that can
   actually produce a cohort.
8. **WI-0036 wildcard flight-plan allocation and immutable freeze.** Core/reserve/wildcard
   allocation, cap arithmetic, bounded substitution reserve, then freeze. The frozen plan is
   immutable; a later change is a NEW flight, never a mutation.
9. **Capture authorization -- HUMAN GATE.** Default off, separate from the screen gate.
10. **Availability realization and reserve-first substitution.** Deterministic realization
    against observed availability; reserve-first precedence; one-core minimum.
11. **Sequential AgentRun capture.** Runs execute one at a time against the frozen plan.
12. **Postgame settlement.**
13. **Next-morning reconciliation and evidence-coverage refresh**, which feeds the next
    day's Pass 1.

## operating target times (America/New_York)

| window | stage |
|---|---|
| ~08:30-09:00 | planning begins (readiness + Planner Pass 1) |
| ~09:00-09:30 | free readiness preflight |
| ~09:30-10:15 | paid screen, Planner Pass 2, flight freeze |
| ~10:00-13:00 | primary capture |

**Hard admission rule:** every selected run must retain **at least 60 minutes before first
pitch**. A candidate whose `scheduled_start - 60 minutes` has passed is not admissible for
execution, regardless of how attractive it looks.

These are operating targets, not guarantees, and they never override an authorization gate.

## the two-pass planner distinction

- **Pass 1** plans against the target date *before* market evidence exists. Its correct
  outcome when market-contrast input is absent is `NOT_ADDRESSABLE`. It is a deterministic
  function of its frozen request; it does **not** refresh the schedule.
- **Pass 2** plans *after* the paid screen has produced market-contrast evidence, and is the
  pass capable of proposing a cohort.
- The **WI-0036 wildcard flight planner is a separate, WI-0036-owned contract** that
  *consumes* the WI-0034 board. It is not a third planner pass and it never changes
  WI-0034 ownership or versions.

## what each stage may do and may never do

| stage | may | may never |
|---|---|---|
| Pass 1 | derive addressability, name missing capabilities | call any source; authorize anything |
| free preflight | refresh schedule/starters; read active runs | reach `/odds`; write to the database |
| `/events` gate | observe provider identity at zero quota | consume quota; call `/odds`; grant screening |
| paid screen | fetch market contrast under authorization | exceed its authorized request budget |
| Pass 2 | propose a cohort | authorize capture |
| flight plan | allocate, rank, freeze | grant capture or execution authority |
| realization | substitute reserve-first | raise the run count or spend ceiling |
| capture | execute authorized runs | add candidates or sources |
| settlement/reconciliation | record outcomes, refresh coverage | retroactively authorize anything |

Every artifact in this chain carries a closed authority ledger whose fields are **all
false**. An artifact is evidence, never permission.

## execution truth

Retrieval may fetch fresh signals **from already configured and authorized sources only**.
It may **not** discover games, add candidates, add sources, or run the deferred Interrogate
probe-refresh loop. Expanding the candidate set or the source set is a separate governed
design decision, not an execution-time behavior.

## scheduled-wildcard count truth

A scheduled wildcard requires a flight of **at least four runs**:

```
wildcard_scheduled_max = floor(total_scheduled_runs / 4)
```

So 1-3 scheduled runs yield **zero** scheduled wildcards; 4-7 yield one; 8 yield two. A
bounded substitution reserve may still exist on smaller flights
(`max(0, scheduled_core_runs - 1)`), but a reserve is not a scheduled wildcard, and
substitution never raises the scheduled run count or the spend ceiling.

## migration prerequisite (deployment, not daily)

WI-0036 migration `20260722100648_AddAgentRunFlightSelectionWriteback` adds nine nullable
realized-selection columns to `AgentRuns`. It is **generated but not applied**.

Applying it is a **one-time deployment prerequisite** before any AgentRun-backed flight that
uses realized-position writeback. It is **never a daily workflow step**, and applying it
requires its own authorization. Until it is applied, flight-plan work remains offline and
AgentRun rows carry no realized-selection facts. The narrow preflight active-run read is
deliberately compatible with the unmigrated schema.

## optional afternoon / evening refresh

A later-day refresh is a **separate new flight**: new flight id, new authorization, new
artifacts, new immutable freeze. It is never a mutation of the morning flight, never an
edit to a frozen plan, and never a reuse of the morning flight's authorization.

## resumable automation states (target)

```
PREPARE           -> READY_FOR_SCREEN_AUTHORIZATION
SCREEN_AND_FREEZE -> READY_FOR_CAPTURE_AUTHORIZATION
EXECUTE           -> CAPTURE_COMPLETE
SETTLE            -> RECONCILIATION_READY
```

Each arrow is a resumable boundary: a day may stop at any state and resume from it with the
artifacts already on disk. The two `READY_FOR_*_AUTHORIZATION` states are exactly where the
human gates sit, and they are **default off**. Standing authority across those boundaries
does not exist and would need its own design.

## artifacts, idempotency, stop conditions, recovery

- **No-overwrite artifacts.** Every attempt claims a fresh destination
  (`<workspace>/<operation>-<date>/attempt-N/`) before the first live call. An existing
  destination is a stop condition, never something to delete or overwrite.
- **Idempotency key.** Operation + target date + attempt directory. Re-running an operation
  requires a new attempt directory, which makes accidental repetition visible.
- **Stop conditions.** Remote or protected-state drift; an existing attempt; missing or
  hash-mismatched frozen inputs; stale preflight (>5 minutes at consumption); insufficient
  time before the admission cutoff; missing credential; failed build or focused tests;
  schema incompatibility; any quota anomaly.
- **Recovery.** A writer-owned recovery artifact is preserved on publication failure and the
  operation stops; it is never retried in place.
- **Raw artifacts stay outside both repositories.** No credential, secret-bearing header,
  raw provider payload, or tenant data is ever committed.

## verification recorded 2026-07-22

Stages 2 and 3 were executed and evidenced on this date; see
[[daily-evidence-planner-pass1-free-preflight-2026-07-22-v1]]. Planner Pass 1 reproduced
its frozen canonical board exactly, and the free preflight completed with zero Odds
requests and zero database writes. Stages 5 onward were not executed and remain separate
decisions.

## related docs

- `04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md`
- `02 Platform/architecture/cognitive-factory/wildcard-evidence-discovery-loop-v1.md`
- `06 Execution/reports/market-contrast-events-gate-observation-2026-07-22-v1.md`
- `06 Execution/reports/market-contrast-start-instant-normalization-analysis-2026-07-22-v1.md`

## recommended next slice

Execute a full day through stage 8 (freeze) under explicit per-stage authorization, and
record where the manual sequence is slowest. That timing evidence is the honest input to any
later automation design. No automation should be built before one full governed day has been
run end to end.
