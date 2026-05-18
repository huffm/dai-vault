# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-18 10:10 |
| competition | MLB |
| take | 6 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 6 |
| artifacts fetched | 6 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Miami Marlins | 2026-05-18 | 6416433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.7 | 1 | none | 0 |
| Baltimore Orioles at Tampa Bay Rays | 2026-05-18 | 6816433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.65 | 1 | none | 3 |
| Cincinnati Reds at Philadelphia Phillies | 2026-05-18 | 6f16433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.65 | 1 | none | 3 |
| Cleveland Guardians at Detroit Tigers | 2026-05-18 | 7016433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |
| New York Mets at Washington Nationals | 2026-05-18 | 7516433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.5 | 1 | none | 0 |
| Toronto Blue Jays at New York Yankees | 2026-05-18 | 7b16433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.7 | 1 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Atlanta Braves at Miami Marlins | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |
| Baltimore Orioles at Tampa Bay Rays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |
| Cincinnati Reds at Philadelphia Phillies | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |
| Cleveland Guardians at Detroit Tigers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |
| New York Mets at Washington Nationals | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |
| Toronto Blue Jays at New York Yankees | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | — |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | no follow-up diagnostics recorded (older artifact or no follow-up signals) |

## Quality Warnings

### signal / grounding

- **Baltimore Orioles at Tampa Bay Rays (2026-05-18)**: signals_used contains bullpen but bullpen was not grounded
- **Baltimore Orioles at Tampa Bay Rays (2026-05-18)**: signals_used contains lineup_form but lineup_form was not grounded
- **Baltimore Orioles at Tampa Bay Rays (2026-05-18)**: signals_used contains ballpark but ballpark was not grounded
- **Cincinnati Reds at Philadelphia Phillies (2026-05-18)**: signals_used contains bullpen but bullpen was not grounded
- **Cincinnati Reds at Philadelphia Phillies (2026-05-18)**: signals_used contains lineup_form but lineup_form was not grounded
- **Cincinnati Reds at Philadelphia Phillies (2026-05-18)**: signals_used contains ballpark but ballpark was not grounded

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Miami Marlins | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Baltimore Orioles at Tampa Bay Rays | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Cincinnati Reds at Philadelphia Phillies | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Cleveland Guardians at Detroit Tigers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| New York Mets at Washington Nationals | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Toronto Blue Jays at New York Yankees | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 6 completed artifact(s) out of 6 assessed. Evidence richness was 1 on all runs. Posture: 6x monitor. Main flags: counter_case was generic on 1 run(s); confidence high for partial evidence on 3 run(s); interrogate had possibly unsupported claims on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Atlanta Braves at Miami Marlins | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |
| Baltimore Orioles at Tampa Bay Rays | monitor | 1 | none | none | none |
| Cincinnati Reds at Philadelphia Phillies | monitor | 1 | none | none | none |
| Cleveland Guardians at Detroit Tigers | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| New York Mets at Washington Nationals | monitor | 1 | none | none | none |
| Toronto Blue Jays at New York Yankees | monitor | 1 | none | what_would_change_contains_filler, interrogate_unsupported_claim, confidence_high_for_partial_evidence | interrogate quality, what_would_change quality, confidence calibration |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Miami Marlins | good | good | good | good | good | good | good | good | generic | good | good | good |
| Baltimore Orioles at Tampa Bay Rays | good | good | good | good | good | good | good | good | possibly unsupported | good | good | good |
| Cincinnati Reds at Philadelphia Phillies | good | good | good | good | good | good | good | good | possibly unsupported | good | good | good |
| Cleveland Guardians at Detroit Tigers | good | good | good | good | good | good | good | good | good | good | good | good |
| New York Mets at Washington Nationals | good | good | good | good | good | good | good | good | good | good | good | good |
| Toronto Blue Jays at New York Yankees | good | good | good | good | possibly unsupported | good | good | good | generic | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 1 | 6 |
| what_would_change_contains_filler | 1 | 6 |
| interrogate_unsupported_claim | 1 | 6 |
| confidence_high_for_partial_evidence | 3 | 6 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Confidence Calibration Rules v1 -- confidence was high relative to evidence richness on 3 of 6 run(s).

Basis:
- confidence_high_for_partial_evidence appeared on 3 of 6 run(s)
- no dominant prompt-quality or signal-quality scenario was triggered

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



