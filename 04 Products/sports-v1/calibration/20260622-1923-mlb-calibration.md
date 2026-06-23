# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-22 19:23 |
| competition | MLB |
| take | 1 |
| days | 3 |
| api base url | http://localhost:5007 |
| runs created | 1 |
| artifacts fetched | 1 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | 2026-06-22 | e5b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Cleveland Guardians at Chicago White Sox | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Cleveland Guardians at Chicago White Sox | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 1 completed artifact(s) out of 1 assessed. Evidence richness was 2 on all runs. Posture: 1x monitor. Main flags: confidence high for partial evidence on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | monitor | 2 | none | what_would_change_contains_filler, confidence_high_for_partial_evidence | what_would_change quality, confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Cleveland Guardians at Chicago White Sox | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| what_would_change_contains_filler | 1 | 1 |
| confidence_high_for_partial_evidence | 1 | 1 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- prompt-quality flags appeared on 1 of 1 run(s).

Basis:
- prompt-quality flags (counter_case, watch_for, what_would_change, contrast, protocol) appeared on 1 of 1 run(s)
- this dominates over structural signal/posture flags

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



