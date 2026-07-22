---
title: "Daily Evidence Planner Pass 1 and Free Preflight 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "complete -- Pass 1 reproduced byte-exact; one free preflight executed; stages 5+ not executed"
project: "DAI"
slice: "WI-0034 / WI-0035 / WI-0036 composition (morning preparation stages 2-3)"
repos:
  dai: "unchanged (read-only; integrated CLIs invoked from main 48a2931)"
  dai-vault: "docs only; local branch ops/2026-07-22-planner-pass1-free-preflight, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - planner
  - market-contrast
related:
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
  - "06 Execution/reports/market-contrast-events-gate-observation-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-start-instant-normalization-analysis-2026-07-22-v1.md"
---

# daily evidence planner pass 1 and free preflight 2026-07-22 v1

## purpose

Execute and evidence the first two stages of the daily evidence workflow for 2026-07-22:
deterministic Planner Pass 1 reproduction, and exactly one refreshed free market-contrast
preflight. Nothing beyond stage 3 was executed.

## two distinct timestamps -- do not conflate

- **Frozen Pass-1 planning timestamp:** `as_of_utc = 2026-07-20T06:35:11Z`. This lives
  inside the frozen canonical request and is intentionally unchanged. **Planner Pass 1 does
  not refresh the schedule and made no source call.**
- **Live preflight timestamps:** started `2026-07-22T14:05:22Z`, completed
  `2026-07-22T14:05:25Z`. This is where current-day freshness enters the workflow.

Any claim that "the planner refreshed today's schedule" would be false. The planner is a
deterministic function of its frozen input; the preflight is the refreshing stage.

## live clocks

| moment | UTC | America/New_York |
|---|---|---|
| entry | `2026-07-22T14:01:56Z` | 10:01:56 EDT |
| Planner Pass 1 | `2026-07-22T14:03:01Z` | 10:03:01 EDT |
| preflight completed | `2026-07-22T14:05:25Z` | 10:05:25 EDT |

Inside the morning window and ahead of the 10:30 EDT target. The operator-reported ~09:50
ET was not used as execution evidence.

## phase A -- Planner Pass 1 reproduction

Contract versions printed and verified before use: CLI `daily-evidence-planner-cli/2.5`,
request `daily-evidence-planner-request/2.1`, board `daily-evidence-board/2.2`, planner
`daily-evidence-planner/2.2`. The frozen request validated `status: valid`.

Frozen inputs verified by hash: `pass1-request.json` `0FCCAFB1…`, `pass1-board.json`
`6548CF09…`, `slate.json` `37EE908B…`; request target `2026-07-22` with 15 distinct
provider-scoped candidates.

**Result: the canonical board reproduced exactly.**

- reproduced canonical board sha-256:
  `3da03021d3ee6c21f4a1d18bc59735f5734e52a889073264e5d2fd919704156b`
- board outcome: `EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE` (expected -- Pass 1
  intentionally lacks market-contrast evidence)
- board/planner versions: `daily-evidence-board/2.2` / `daily-evidence-planner/2.2`
- authority ledger: 8 fields, all false
- zero network, database, model, gateway, or application-service calls

**One file-level artifact, stated plainly.** The reproduced file is 3473 bytes; the frozen
`pass1-board.json` is 3475 bytes. A byte-index scan found the **first differing byte index
is -1** -- i.e. all 3473 common bytes are identical -- and the frozen file's two extra
trailing bytes are `0D 0A`, a CRLF appended by the July-20 writer. Hashing the frozen file
with that trailing CRLF removed yields
`3da03021d3ee6c21f4a1d18bc59735f5734e52a889073264e5d2fd919704156b`, identical to the
reproduction, and the canonical JSON strings compare `-ceq` equal at 3473 characters each.

The planner's canonical output is therefore byte-identical; the delta is a trailing line
terminator outside the canonical document, introduced by how the earlier artifact was
written. This is recorded rather than silently normalized, because a whole-file hash
comparison alone would have read as a mismatch.

## phase B -- one refreshed free preflight

Pre-call verification from integrated source: `--preflight` cannot reach the Odds call site;
bundle schema `market-contrast-screen-bundle/1.3`; expected terminal
`completed_preflight_no_paid_call`; destination claimed before the live call; the database
operation is the tenant-1 provider-scoped active-run aggregate read that references none of
the nine unapplied WI-0036 flight-selection columns. Proportionate verification: no-restore
build clean, focused market-contrast tests 56/56, planner CLI tests 50/50. Full suites were
not re-run -- integrated refs were already verified this session.

Dependency: opening state recorded as Docker engine not running and `devcore-sql` stopped.
Only that dependency was started; the web application and agent-service were not started;
opening state restored at close.

**Result:** terminal `completed_preflight_no_paid_call`, bundle sha-256
`c58623e99cee85b3bd7a1cbac03514e3e3e644abcaa8bbd248539782bad16563`, schema
`market-contrast-screen-bundle/1.3`, mode `preflight`, target `2026-07-22`, exact Pass-1 SHA
recorded, 15 distinct identities, authority ledger all false, validated by two independent
JSON parsers.

### call ledger

| item | count |
|---|---|
| Odds `/events` | **0** |
| Odds `/odds` | **0** |
| StatsAPI schedule | 1 |
| StatsAPI starter attempts | 28 (0 failures) |
| database reads | 1 |
| database writes | **0** |
| model / Tool Gateway / AgentRun / capture / generation / scheduling / settlement / reconciliation | **0** |
| migrations applied | **0** |
| paid cost | **$0** |

No machine path, secret, API key, tenant payload, or raw provider response is persisted in
either repository.

## per-candidate readiness

Execution-admission cutoff is `scheduled_start - 60 minutes`. **No execution is granted.**

| gamePk | matchup | start (UTC) | state | starters | disposition | cutoff | vs 08:37 |
|---|---|---|---|---|---|---|---|
| 822784 | TB@TOR | 23:07:00Z | scheduled | yes | screenable | 22:07:00Z | same |
| 822873 | CWS@TEX | 00:05:00Z | scheduled | yes | screenable | 23:05:00Z | same |
| 823110 | CIN@SEA | 19:40:00Z | scheduled | yes | screenable | 18:40:00Z | same |
| **823438** | **LAD@PHI** | 22:40:00Z | scheduled | yes | **screenable** | 21:40:00Z | same |
| 823518 | PIT@NYY | 17:05:00Z | scheduled | yes | PREBLOCK `caller_start_mismatch` | 16:05:00Z | same |
| 823761 | NYM@MIL | 18:10:00Z | scheduled | yes | screenable | 17:10:00Z | same |
| 824004 | STL@LAA | 20:07:00Z | scheduled | yes | screenable | 19:07:00Z | same |
| 824083 | SF@KC | 18:10:00Z | scheduled | yes | screenable | 17:10:00Z | same |
| 824166 | MIA@HOU | 00:10:00Z | scheduled | yes | screenable | 23:10:00Z | same |
| 824327 | WSH@COL | 19:10:00Z | scheduled | yes | screenable | 18:10:00Z | same |
| 824408 | MIN@CLE | 22:40:00Z | scheduled | yes | screenable | 21:40:00Z | same |
| 824650 | DET@CHC | 00:10:00Z | scheduled | yes | screenable | 23:10:00Z | same |
| **824732** | **BAL@BOS** | 23:10:00Z | scheduled | **no** | **PREBLOCK `starters_not_announced`** | 22:10:00Z | same |
| 824896 | SD@ATL | 23:15:00Z | scheduled | yes | screenable | 22:15:00Z | same |
| 825055 | ATH@ARI | 19:40:00Z | scheduled | yes | screenable | 18:40:00Z | same |

Earliest scheduled start `2026-07-22T17:05:00Z` (823518, preblocked); earliest **screenable**
start `2026-07-22T18:10:00Z` with admission cutoff `2026-07-22T17:10:00Z`.

**Nothing changed versus the 08:37 preflight.** No start moved and no disposition flipped.

### the two called-out candidates

- **824732 BAL@BOS** -- still `starters_not_announced` at 10:05 EDT, exactly as at 08:37.
  The earlier analysis noted this candidate is already start-aligned with the provider
  (`23:10:00Z` on both sides) and would join exactly if it cleared this preblock. **It has
  not cleared it.** The hoped-for improvement has not materialized as of this preflight.
- **823438 LAD@PHI** -- screenable, and the one candidate that produced an exact provider
  join in the 08:38 observation.

## capacity facts, kept separate

1. **Current free-preflight screenable count: 13** (15 total minus 2 preblocked).
2. **Currently screenable identities that were exact in the 08:38 provider observation: 1**
   (823438 only). 824732 was start-aligned in that observation but remains preblocked, so it
   is not currently screenable.
3. **At least four candidates remain potentially screenable: yes** (13 >= 4).
4. **A four-run scheduled flight is possible in count terms only.** Under
   `floor(total_scheduled_runs / 4)`, a four-run flight admits at most one scheduled
   wildcard.
5. **Unknown until later separately governed stages:** actual market-screen eligibility,
   core/wildcard allocation, and capture authority. A screenable candidate is not an
   eligible candidate, and neither is an authorized one.

The intersection in (2) is derived from the **already captured 08:38 `/events` artifact**.
It is **not** a fresh provider join -- no `/events` call was made in this operation.

## what was not done

Planner Pass 2 was not constructed. No wildcard flight plan, allocation, or freeze. No paid
screen, `/events`, `/odds`, model call, Tool Gateway call, AgentRun, capture, database write,
migration application, or activation. WI-0036 Slices 4-6 remain deferred.

## artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/daily-evidence-flight-2026-07-22/attempt-1/`

| file | sha-256 |
|---|---|
| `planner-pass1-board.json` | `3da03021d3ee6c21f4a1d18bc59735f5734e52a889073264e5d2fd919704156b` |
| `planner-pass1-board.md` | `3e92f3e292fc9e30039d7053b636dfcfc5020aed6e13c85a0edb2de0b11988f1` |
| `preflight-bundle.json` | `c58623e99cee85b3bd7a1cbac03514e3e3e644abcaa8bbd248539782bad16563` |
| `attempt-manifest.json` | `ea1cdfac584c0e4faf8f6d029155fbb7f0e51597951bc19e4ca1b22e40f8aba5` |

Plus the four stdout/stderr logs (both stderr files empty). The destination was claimed
before the first live call and no earlier attempt was deleted or overwritten.

## recommended next action (proposal only)

If a paid screen is wanted today, the next decision is a **separately authorized paid market
screen** over the 13 screenable candidates, followed by Planner Pass 2. If instead the goal
is to raise the confirmed provider-join count first, a later free preflight plus zero-quota
`/events` re-observation once 824732's starters are announced would test that cheaply. Both
require their own authorization; neither is granted here.
