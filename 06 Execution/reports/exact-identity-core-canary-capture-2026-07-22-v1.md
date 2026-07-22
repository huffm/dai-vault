---
title: "Exact-Identity Core Canary Capture 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "captured -- one-run technical core canary; migration applied; NOT a measurement-grade cohort and NOT settled evidence"
project: "DAI"
slice: "WI-0034 / WI-0035 / WI-0036 composition (conditional core canary)"
repos:
  dai: "source unchanged; local database schema advanced by the one authorized migration"
  dai-vault: "docs only; local branch ops/2026-07-22-planner-pass1-free-preflight, NOT pushed"
tags:
  - evidence-operations
  - sports-v1
  - capture
  - wi-0036
related:
  - "06 Execution/patterns/daily-evidence-acquisition-operating-workflow-v1.md"
  - "06 Execution/reports/market-contrast-final-free-identity-gate-2026-07-22-v1.md"
  - "02 Platform/system-development/work-items/WI-0036-wildcard-evidence-discovery-loop-v1.md"
---

# exact-identity core canary capture 2026-07-22 v1

## objective override -- read this first

This was a **narrowly bounded one-run technical core canary**, executed under an explicit
operator authorization that deliberately changed the immediate objective away from a
measurement-grade flight.

It is **NOT**:

- evidence that the earlier four-run gate passed. That gate remains **NO-GO**:
  `exact_current_screenable_join_count = 1 < 4`, and this capture does not change it;
- a wildcard test. Zero wildcards were scheduled (`floor(1/4) = 0`, mode `disabled`);
- a measurement-grade cohort;
- settled evidence. Settlement and reconciliation were **not authorized** and did not occur.

It **is** a technical proof that the integrated vertical -- migration, paid screen, Planner
Pass 2, WI-0036 freeze, realization, provenance export, controller trust boundary, and
realized-position writeback -- works end to end on one exactly-identified candidate.

## timeline (live clocks)

| moment | UTC | ET |
|---|---|---|
| entry | `14:54:10Z` | 10:54:10 |
| migration applied | `15:01:43Z` -> `15:01:46Z` | 11:01 |
| paid screen | `15:02:48Z` -> `15:02:52Z` | 11:02 |
| flight frozen | `15:05:00Z` | 11:05 |
| AgentRun | `15:15:26Z` -> `15:15:38Z` | 11:15 |
| first pitch | `22:40:00Z` | 18:40 |

Execution occurred **444 minutes** before first pitch, far above the 60-minute floor.

## Phase A -- migration (the one authorized durable schema change)

Pre-state: migration count 23, tip `20260629174632_AddAgentRunPromptRouteProvenance`, target
**unapplied**, `AgentRuns` rows **302**, flight columns **0**, active runs for 823438 **0**,
any runs for 823438 **0**.

Shape re-verified before mutation: exactly 9 `AddColumn` / 9 `DropColumn`, 9 `nullable: true`,
**zero** `defaultValue`, **zero** raw `.Sql(`, **zero** `UpdateData/InsertData/DeleteData`,
**zero** `DropTable/DropIndex/DropForeignKey/DropPrimaryKey/AlterColumn/CreateTable`, and the
only table referenced is `AgentRuns`. Exactly one migration was pending and it was the
authorized one.

Applied `20260722100648_AddAgentRunFlightSelectionWriteback`.

Post-state: migration count **24**, target applied **1**, `AgentRuns` rows **302
(unchanged)**, flight columns **9**, rows with any non-null flight value **0** (no backfill),
runs for 823438 still **0**. Column types as designed: `FlightId` nvarchar(128),
`FlightPlanFreezeFingerprint` nvarchar(64), `FlightSelectionLane` nvarchar(16),
`FlightSelectionWildcardPlanRole` nvarchar(32), `FlightSelectionRealizedVia` nvarchar(32),
`FlightSelectionSubstitutedForSourceProvider` nvarchar(64),
`FlightSelectionSubstitutedForExternalEventId` nvarchar(128), both positions `int` -- all
nullable.

## Phase B -- one bounded paid market screen

One `/v4/sports/baseball_mlb/odds` request, markets `h2h,spreads`, region `us`, zero
retries. **Exactly 2 credits** (`usage_last = 2`; used 282 -> 284; remaining 218 -> 216),
matching the authorized ceiling. Database reads 1, writes 0. Authority ledger all false.
Terminal `completed`, mode `paid`, schema `market-contrast-screen-bundle/1.3`. Bundle sha
`d45b974eef6779e397afe6efac6a15c6133cce15bd5a60dafad623bfb63de379`.

**gamePk 823438 re-verified in the fresh paid response** (the morning `/events` binding was
not trusted on its own):

- no preblock; status `evaluated`
- classification **`includable`**, screen tier `primary`
- market join **`matched`**: `teams+start exact; books=9; fresh=9; spread_context=present`
- provider event id exactly **`111a955795876d50988b15c219ce0796`**
- `exact_match_count = 1`, `nearest_start_delta_seconds = 0`,
  `reversed_orientation_count = 0`
- **`same_orientation_team_pair_count = 1`** -- the mandatory multiplicity restriction is
  satisfied: exactly one provider event for that team pair in the full date bracket
- disagreement known, range `0.0169`

823438 was the **only** matched candidate in the whole slate. The other twelve
start-mismatched candidates and BAL@BOS were not used.

## Phase C -- Planner Pass 2 and one-core freeze

Pass-2 request derived by the integrated planner CLI `replay` from the exact frozen Pass-1
request plus the canonical paid bundle (no hand-composed evidence); request sha
`b622579c8cfedd78bde14653df9d62a3f683b6043ac8034dc5f30c74002b830f`.

Planner Pass 2 board: **`COHORT_PROPOSED_FOR_OPERATOR_REVIEW`**, board
`daily-evidence-board/2.2`, planner `daily-evidence-planner/2.2`, primary pool **`[823438]`**,
reserve pool empty, pools disjoint, authority all false. The planner selected 823438 itself
-- nothing was hand-promoted and the board was not modified.

WI-0036 flight plan frozen from that board:

| fact | value |
|---|---|
| flight id | `flight-2026-07-22-exact-core-canary-1` |
| freeze fingerprint | `0d44530e2985ab0d4d6d460263306ef113f32171a73c1ecc5803a3151bf8f954` |
| total scheduled runs | 1 |
| scheduled core | 1 |
| scheduled wildcard | **0** |
| wildcard scheduled max | **0** (`floor(1/4)`) |
| wildcard mode | `disabled` |
| core reserve / wildcard reserve | 0 / 0 |
| scheduled entry | 823438, lane `core`, position 1 |
| authority ledger | all false |

Producer-replay validation returned `valid -- exact re-production from the producer-verified
request`. All-available realization produced `realized_via = scheduled`, lane `core`, no
`substituted_for`, 0 substitutions, 0 unfilled, counts core 1 / wildcard 0 / total 1, and
validated as `canonically identical to the deterministic re-derivation from plan +
availability`.

Provenance export (`flight-selection-provenance/1.0`, plan `wildcard-flight-plan/1.1`, lane
vocabulary `wi0036-candidate-lane/1.0`): lane `core`, wildcard plan role null, scheduled
position 1, realized position 1, `realizedVia = scheduled`, `substitutedFor` null,
`substitutionEligible` **false** (correct for a core lane), authority 8 fields all false.
Sha `44c7ba64f5fb39db994f1f35bf1ed3c99fb28f6a238d49e6119701cf129e7284`.

## Phase D -- exactly one AgentRun

Submitted one `sports.matchup.analysis` run for gamePk 823438 carrying the producer-exported
provenance. The controller validated the block before the run row (its documented
fail-closed boundary).

**Run `a9b0433e-f36b-1410-8191-00373db4b724`, status `completed`**, tenant 1, provider
`mlb_statsapi`, lean `away` (Dodgers), confidence 0.75, posture `monitor`, evidence richness
2, duration 10.7s.

Writeback columns equal the exported provenance exactly:

```
flightId  = flight-2026-07-22-exact-core-canary-1
fp        = 0d44530e2985ab0d4d6d460263306ef113f32171a73c1ecc5803a3151bf8f954
lane      = core          role      = NULL
scheduled = 1             realized  = 1
via       = scheduled
substitutedForSourceProvider = NULL   substitutedForExternalEventId = NULL
```

`AgentRuns` rows **302 -> 303 (delta exactly +1)**; rows for 823438 = 1; **zero** other rows
carry any flight value. Market snapshot linked to the run with provider event
`111a955795876d50988b15c219ce0796` -- the same provider event as the exact join, so market
identity and candidate identity are separately correct and mutually consistent.

## cost and call ledger

| item | observed | ceiling |
|---|---|---|
| AgentRuns | **1** | 1 |
| model calls | **1** (`gpt-4o-mini`, 2936 in / 462 out) | 1 |
| model spend | **$0.0007176** | $0.01 |
| Odds `/odds` screen | 1 request, **2 credits** | 2 |
| Odds run market retrieval | 1 request, **3 credits** (`h2h,spreads,totals` x `us`) | -- |
| **total Odds credits** | **5** | **5** |
| Odds `/events` | 0 | n/a |
| migrations applied | 1 | 1 |
| settlement / reconciliation | 0 | not authorized |

The run's 3 credits are derived from `markets=h2h,spreads,totals` at one region under the
provider's per-market pricing; the screen's 2 were read directly from `x-requests-last`.
Confirming the run's figure from a response header would itself have cost a credit, so it is
recorded as a derivation rather than an observed header.

## what did not happen

Zero wildcards scheduled. Zero substitutions. Zero candidates added. No reserve promoted. No
non-exact or duplicate-team-pair binding used. No filter, tolerance, prompt, routing,
confidence, lean, posture, buyer, matching, or recipe change. No `/events` call. No second
attempt or retry. No settlement or reconciliation. No source or test change. No remote
mutation. Protected files byte-identical throughout; `dai` worktree clean.

## settlement status

This run **may enter later settlement/reconciliation** under its own separate authorization.
It is **not** counted as settled evidence now, and it does not contribute to calibration
until it has been settled through the governed path.

## services and gates

Started only what was required, in order: the `devcore-sql` container for Phase A, then the
agent-service and platform API for Phase D. Opening state (Docker engine down, container
stopped, no listeners on 5007/8000) was recorded and restored at close; both ports are down
and the container and engine are stopped. No configuration was persisted or modified -- the
existing development bypass and AI service settings were used as-is.

## artifacts (outside both repositories)

`<DAI_WORKSPACE_ROOT>/daily-evidence-capture-2026-07-22/exact-core-canary-1/` -- 23 files
including `paid-screen-bundle.json` (`d45b974e…`), `planner-pass2-board.json`,
`flight-plan.json` (`0d44530e…` fingerprint), `flight-realization.json`,
`flight-provenance.json` (`44c7ba64…`), `run-request.json`, `run-response.json`, bounded
logs, and `attempt-manifest.json` (`428d1f63…`). No secret, API key, credential, or raw
provider payload is committed to either repository.

## recommended next action (proposal only)

Two independent decisions, neither authorized here: (1) whether to settle this run once the
game completes, through the governed settlement path; and (2) whether the stable `+60s`
start-instant population is a real identity-join defect worth a designed fix -- still the
binding constraint on ever reaching a four-run measurement-grade flight.
