---
title: "V2 Day-2 Cohort Settlement -- 7 settled, 823357 excluded postponed (2026-07-11)"
type: "reconciliation"
date: "2026-07-11"
status: "complete"
project: "DAI"
slice: "V2 Cadence Wrap 2026-07-11"
repos:
  dai: "unchanged"
  dai-vault: "docs-only"
tags:
  - settlement
  - reconciliation
  - calibration
related:
  - "06 Execution/reports/v2-accelerated-capture-day2-2026-07-10-v1.md"
  - "06 Execution/reports/gate4-evidence-readout-v2-day2-2026-07-11-v1.md"
  - "06 Execution/patterns/capture-closeout-run-eligibility-rule-v1.md"
  - "06 Execution/reconciliations/canonical-reconciliation-residue-contract-v1.md"
---

# v2 day-2 cohort settlement -- 7 settled, 1 excluded

Final settlement pass of the 2-slate-day v2 cadence. 7 of 8 day-2 games settled; 823357
excluded as a postponed non-event. Zero paid calls, zero code change.

## 1. gates

| gate | result |
|---|---|
| `check-settlement-finals.ps1` (7 finals) | **READY 7/7**, exit 0 |
| `preflight-settlement.ps1` strict (7) | exit 0 -- 7 ready / 0 warnings / 0 blockers / agree 6 / disagree 1 |
| independent `feed/live` verify (7) | 7/7 Final (codedGameState=F), identity matches persisted refs, scores re-read before each write |

## 2. per-run results

| gamePk | run | matchup | final (a-h) | winner | lean | conf | market | agree | evaluation |
|---|---|---|---|---|---|---|---|---|---|
| 824493 | 549d433e | CHC@CIN | 0-4 | home | away | 0.75 | away | yes | **incorrect** |
| 822955 | 599d433e | SEA@TB | 2-7 | home | home | 0.75 | home | yes | **correct** |
| 823278 | 5d9d433e | TOR@SD | 5-3 | away | home | 0.75 | home | yes | **incorrect** |
| 823845 | 609d433e | CLE@MIA | 3-2 | away | away | 0.70 | home | **NO** | **correct** |
| 824252 | 659d433e | PHI@DET | 2-10 | home | home | 0.75 | home | yes | **correct** |
| 823685 | 709d433e | LAA@MIN | 4-3 | away | home | 0.75 | home | yes | **incorrect** |
| 823604 | 729d433e | BOS@NYM | 6-2 | away | home | 0.80 | home | yes | **incorrect** |

**Cohort record: 3 correct / 4 incorrect (0.4286).** The lone 0.80-confidence run (823604) lost.
Residue on all 7: `source=statsapi`, `sourceRef="gamePk <pk> final away <a> home <h>"`,
notes naming the 2026-07-11 wrap + verification method.

## 3. the deliberate-divergence row settled correct

**823845 CLE@MIA** -- the first deliberate divergence produced by a capture -- settled
**correct**: DAI leaned away (Cleveland, conf 0.70) against a 9-book home consensus (Miami);
Cleveland won 3-2. `divergenceInterpretation=DeliberateDivergence`,
`attributionFidelityStatus=Pass`, `attributionFidelityReason=prose_acknowledges_market_opposition`.

Discipline: this is one settled deliberate-divergence row, now correct. It moves the deliberate
ledger to **1 correct / 1 incorrect** (with 823281, which lost). **It is not an edge claim.**
One correct contrarian call at n=2 is a coin flip, not a demonstrated edge. Candidate-edge
language is reserved for the ledger's count, not for a performance claim.

## 4. 823357 MIL@PIT -- excluded, postponed non-event

823357 was **postponed on 2026-07-10** (statsapi codedGameState=D) and rescheduled to 2026-07-11
as a split doubleheader (a distinct event with its own start time, and potentially different
starters, lineups, market state, and game identity). The captured run 6c9d433e evaluated the
**07-10 scheduled event, which never occurred**.

Excluded via `POST /api/agent-runs/6c9d433e.../exclude` with
`reason="excluded"` (HTTP 200, `exclusionReason=excluded`, no supersession link).

**Reason-code choice.** The `RunExclusionReasons` enum is `{superseded, excluded, diagnostic,
invalid}`; there is no literal `postponed`. `invalid` mischaracterizes a sound decision (the run
is not malformed -- the event vanished). `superseded` requires a replacement run for the same
game (the 823613 precedent), and none was generated -- the operator's decision is that the replay
is a **separate event context**, not a supersession. `excluded` ("intentionally kept out of
automatic selection for an operator reason") is the exact fit. The postponement rationale is
documented here; the DB reason is a one-word enum.

**Rationale (operator, verbatim intent):** the captured decision applied to the July 10
scheduled event, which never occurred; today's doubleheader games are distinct decision contexts;
settling the original run against either replay would introduce post-hoc identity substitution and
contaminate calibration; exclusion preserves the rule that outcomes must correspond to the event
actually evaluated. The later replay is treated as a separate event context (no run was captured
for it; capture authorization ends with this wrap).

## 5. post-write verification (persisted)

- total rows 302 -> **302** (0 new runs)
- outcomes 133 -> **140** (+7); `settlementSource` present on all 140
- excluded 39 -> **40** (+1, = 823357)
- rows changed: **exactly 8** = the 7 settled + 823357; changed gamePk set == {7 finals} + {823357}
- 7 settled: outcome + full residue on all; **0 duplicate outcomes**
- 823357: `exclusionReason=excluded`, `outcomeStatus=None`
- duplicate-active gamePks after: **0**

Each write confirmed by reading back the run and `GET /{id}/evaluation`, and by diffing full
`/rows` snapshots before/after.

## 6. what did not change

No prompt, model, confidence, decision, source-readiness, reconciliation-semantics,
calibration-formula, Gate-4-threshold, schema, migration, buyer, or registry-allowlist change.
`agent-service` never started (settlement is free). 0 paid calls. dai unchanged (`bb10c3c`).
