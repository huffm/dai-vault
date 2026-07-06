---
title: "Frozen Cohort Slate Template v1"
type: "template"
date: "2026-07-06"
status: "active"
project: "DAI"
tags:
  - calibration
  - cohort
  - capture
  - template
related:
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
---

# Frozen Cohort Slate Template v1

Copy into `06 Execution/reports/<objective>-candidate-slate-<date>-vN.md` and fill every
field BEFORE any paid generation. Fields mirror the capture-v2 slate that passed QA.

```markdown
---
title: "<objective> Candidate Slate <date> vN (FROZEN)"
type: "report"
date: "<date>"
status: "FROZEN -- slate frozen before any paid generation"
project: "DAI"
slice: "<slice name>"
---

# <objective> Candidate Slate <date> vN (FROZEN)

## freeze statement
Frozen at <local time + UTC>. No paid model call was made, and agent-service was not
started for generation, before this freeze. Screening used only <read-only sources>.
AgentRuns count at freeze = <N>. Outcomes = <N>, Evaluations = <N>.
After this freeze: no game is added because early outputs agree with market, no generated
game is removed because it is inconvenient, and all failures are recorded explicitly.

## measurement objective
<one sentence: the question this cohort answers>

## caps (binding)
- MAX_PAID_RUNS = <N>
- TOTAL_COST_CAP_USD = <N>
- MAX_MODEL_CALLS_PER_RUN = 1
- MODEL_EXPECTED = <model>
- SETTLEMENT_IN_THIS_SLICE = false

## run window
<target window + actual screening time + margin to first start>

## sport / competition
<sport, competition code, GAME_DATE>

## candidate universe
<how the universe was read: schedule source, count, statuses>

## screening timestamp(s)
<schedule read, odds read, source-readiness reads -- times + quota used>

## screening method
<1. schedule -> pre-game only; 2. one odds slate read -> de-vig implied probs + gap;
 3. source-readiness per candidate -> identity/regime/eligibility>

## candidate classification (all games considered)
| gamePk | matchup | start | status | pre-game | identity | starter | market | books | consensus | favorite | favP | gap | class | decision | reason |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
<one row per game in the universe; class = Primary / Secondary / Exclude / Blocker>

## selected cohort
<per selected game: why it serves the measurement objective>

## exclusions
<per excluded game: the reason, tied to the filter>

## stop conditions
<the guardrail breaches that stop generation, verbatim from the slice prompt>

## pre-generation baselines (for post-slice validation)
- AgentRuns = <N>
- AgentRunOutcomes = <N>
- AgentRunEvaluations = <N>
- existing active runs for selected gamePks = <N per pk, expect 0>
- external API quota remaining = <N>

## generation results (filled AFTER capture, never edited before)
| # | gamePk | agentRunId | lean | conf | market favorite | agreement | promptSource | regime | cost$ |
|---|---|---|---|---|---|---|---|---|---|

## settlement plan
<the settlement slice that will close the loop: identity /reconcile after official finals;
 preflight-settlement.ps1 manifest attached before settlement>
```
