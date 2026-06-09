# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-09 15:33 |
| competition | MLB |
| take | 5 |
| days | 3 |
| api base url | http://localhost:5007 |
| runs created | 5 |
| artifacts fetched | 5 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | 2026-06-09 | b96b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Arizona Diamondbacks at Miami Marlins | 2026-06-09 | bb6b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |
| Boston Red Sox at Tampa Bay Rays | 2026-06-09 | be6b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| New York Yankees at Cleveland Guardians | 2026-06-09 | c46b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |
| Minnesota Twins at Detroit Tigers | 2026-06-09 | c66b433e-f36b-1410-8161-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Arizona Diamondbacks at Miami Marlins | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Boston Red Sox at Tampa Bay Rays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| New York Yankees at Cleveland Guardians | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Minnesota Twins at Detroit Tigers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |

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
| Arizona Diamondbacks at Miami Marlins | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Boston Red Sox at Tampa Bay Rays | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| New York Yankees at Cleveland Guardians | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| Minnesota Twins at Detroit Tigers | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 5 completed artifact(s) out of 5 assessed. Evidence richness was 1 on all runs. Posture: 5x monitor. Main flags: counter_case was generic on 1 run(s); confidence high for partial evidence on 5 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Arizona Diamondbacks at Miami Marlins | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |
| Boston Red Sox at Tampa Bay Rays | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| New York Yankees at Cleveland Guardians | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Minnesota Twins at Detroit Tigers | monitor | 1 | none | what_would_change_contains_filler, confidence_high_for_partial_evidence | what_would_change quality, confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Seattle Mariners at Baltimore Orioles | good | good | good | generic | not recorded | good | good | good | good | good | good | good |
| Arizona Diamondbacks at Miami Marlins | good | good | good | generic | not recorded | good | good | good | good | good | good | good |
| Boston Red Sox at Tampa Bay Rays | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| New York Yankees at Cleveland Guardians | good | good | good | good | not recorded | good | good | not recorded | good | good | good | good |
| Minnesota Twins at Detroit Tigers | good | good | good | good | not recorded | good | good | not recorded | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 1 | 5 |
| what_would_change_contains_filler | 1 | 5 |
| confidence_high_for_partial_evidence | 5 | 5 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Confidence Calibration Rules v1 -- confidence was high relative to evidence richness on 5 of 5 run(s).

Basis:
- confidence_high_for_partial_evidence appeared on 5 of 5 run(s)
- no dominant prompt-quality or signal-quality scenario was triggered

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



