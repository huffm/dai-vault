# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-09 11:16 |
| competition | MLB |
| take | 4 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 4 |
| artifacts fetched | 4 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | 2026-06-09 | 9f6b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Arizona Diamondbacks at Miami Marlins | 2026-06-09 | a46b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |
| Boston Red Sox at Tampa Bay Rays | 2026-06-09 | a56b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| New York Yankees at Cleveland Guardians | 2026-06-09 | aa6b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Arizona Diamondbacks at Miami Marlins | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Boston Red Sox at Tampa Bay Rays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| New York Yankees at Cleveland Guardians | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| - | - | - | - | - | - | no follow-up diagnostics recorded (older artifact or no follow-up signals) |

## Quality Warnings

### signal / grounding

- **Seattle Mariners at Baltimore Orioles (2026-06-09)**: signals_used contains bullpen but bullpen was not grounded
- **Seattle Mariners at Baltimore Orioles (2026-06-09)**: signals_used contains lineup_form but lineup_form was not grounded
- **Seattle Mariners at Baltimore Orioles (2026-06-09)**: signals_used contains ballpark but ballpark was not grounded
- **Boston Red Sox at Tampa Bay Rays (2026-06-09)**: signals_used contains bullpen but bullpen was not grounded
- **Boston Red Sox at Tampa Bay Rays (2026-06-09)**: signals_used contains lineup_form but lineup_form was not grounded
- **Boston Red Sox at Tampa Bay Rays (2026-06-09)**: signals_used contains ballpark but ballpark was not grounded

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Arizona Diamondbacks at Miami Marlins | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| Boston Red Sox at Tampa Bay Rays | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| New York Yankees at Cleveland Guardians | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 4 completed artifact(s) out of 4 assessed. Evidence richness was 1 on all runs. Posture: 4x monitor. Main flags: counter_case was generic on 2 run(s); confidence high for partial evidence on 4 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Arizona Diamondbacks at Miami Marlins | monitor | 1 | none | counter_case_generic, what_would_change_contains_filler, confidence_high_for_partial_evidence | interrogate quality, what_would_change quality, confidence calibration |
| Boston Red Sox at Tampa Bay Rays | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |
| New York Yankees at Cleveland Guardians | monitor | 1 | none | what_would_change_contains_filler, confidence_high_for_partial_evidence | what_would_change quality, confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | good | good | good | generic | not recorded | good | good | good | good | good | good | good |
| Arizona Diamondbacks at Miami Marlins | good | good | good | generic | not recorded | good | good | not recorded | good | good | good | good |
| Boston Red Sox at Tampa Bay Rays | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| New York Yankees at Cleveland Guardians | good | good | good | good | not recorded | generic | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 2 | 4 |
| what_would_change_contains_filler | 2 | 4 |
| confidence_high_for_partial_evidence | 4 | 4 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- prompt-quality flags appeared on 3 of 4 run(s).

Basis:
- prompt-quality flags (counter_case, watch_for, what_would_change, contrast, protocol) appeared on 3 of 4 run(s)
- this dominates over structural signal/posture flags

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



