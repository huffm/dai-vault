# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-29 14:21 |
| competition | MLB |
| take | 8 |
| days | 2 |
| api base url | http://localhost:5007 |
| runs created | 8 |
| artifacts fetched | 8 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Cincinnati Reds | 2026-05-29 | 068e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.6 | 1 | none | 3 |
| Minnesota Twins at Pittsburgh Pirates | 2026-05-29 | 098e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.5 | 1 | none | 3 |
| San Diego Padres at Washington Nationals | 2026-05-29 | 0c8e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.75 | 1 | none | 2 |
| Toronto Blue Jays at Baltimore Orioles | 2026-05-29 | 0f8e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Boston Red Sox at Cleveland Guardians | 2026-05-29 | 158e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.75 | 1 | none | 2 |
| Los Angeles Angels at Tampa Bay Rays | 2026-05-29 | 178e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.65 | 1 | none | 3 |
| Miami Marlins at New York Mets | 2026-05-29 | 198e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Chicago Cubs at St. Louis Cardinals | 2026-05-29 | 1c8e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.75 | 1 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Atlanta Braves at Cincinnati Reds | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Minnesota Twins at Pittsburgh Pirates | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| San Diego Padres at Washington Nationals | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Toronto Blue Jays at Baltimore Orioles | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Boston Red Sox at Cleveland Guardians | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Los Angeles Angels at Tampa Bay Rays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Miami Marlins at New York Mets | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Chicago Cubs at St. Louis Cardinals | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| - | - | - | - | - | - | no follow-up diagnostics recorded (older artifact or no follow-up signals) |

## Quality Warnings

### signal / grounding

- **Atlanta Braves at Cincinnati Reds (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Atlanta Braves at Cincinnati Reds (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Atlanta Braves at Cincinnati Reds (2026-05-29)**: signals_used contains ballpark but ballpark was not grounded
- **Minnesota Twins at Pittsburgh Pirates (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Minnesota Twins at Pittsburgh Pirates (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Minnesota Twins at Pittsburgh Pirates (2026-05-29)**: signals_used contains ballpark but ballpark was not grounded
- **San Diego Padres at Washington Nationals (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **San Diego Padres at Washington Nationals (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Toronto Blue Jays at Baltimore Orioles (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Toronto Blue Jays at Baltimore Orioles (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Toronto Blue Jays at Baltimore Orioles (2026-05-29)**: signals_used contains ballpark but ballpark was not grounded
- **Boston Red Sox at Cleveland Guardians (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Boston Red Sox at Cleveland Guardians (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Los Angeles Angels at Tampa Bay Rays (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Los Angeles Angels at Tampa Bay Rays (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Los Angeles Angels at Tampa Bay Rays (2026-05-29)**: signals_used contains ballpark but ballpark was not grounded
- **Miami Marlins at New York Mets (2026-05-29)**: signals_used contains bullpen but bullpen was not grounded
- **Miami Marlins at New York Mets (2026-05-29)**: signals_used contains lineup_form but lineup_form was not grounded
- **Miami Marlins at New York Mets (2026-05-29)**: signals_used contains ballpark but ballpark was not grounded

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Cincinnati Reds | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Minnesota Twins at Pittsburgh Pirates | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| San Diego Padres at Washington Nationals | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Toronto Blue Jays at Baltimore Orioles | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Boston Red Sox at Cleveland Guardians | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Los Angeles Angels at Tampa Bay Rays | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Miami Marlins at New York Mets | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Chicago Cubs at St. Louis Cardinals | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 8 completed artifact(s) out of 8 assessed. Evidence richness was 1 on all runs. Posture: 8x monitor. Main flags: confidence high for partial evidence on 5 run(s); protocol had possibly unsupported claims on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Atlanta Braves at Cincinnati Reds | monitor | 1 | none | none | none |
| Minnesota Twins at Pittsburgh Pirates | monitor | 1 | none | none | none |
| San Diego Padres at Washington Nationals | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Toronto Blue Jays at Baltimore Orioles | monitor | 1 | none | what_would_change_contains_filler, confidence_high_for_partial_evidence | what_would_change quality, confidence calibration |
| Boston Red Sox at Cleveland Guardians | monitor | 1 | none | what_would_change_contains_filler, protocol_unsupported_claim, confidence_high_for_partial_evidence | protocol quality, what_would_change quality, confidence calibration |
| Los Angeles Angels at Tampa Bay Rays | monitor | 1 | none | none | none |
| Miami Marlins at New York Mets | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Chicago Cubs at St. Louis Cardinals | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Atlanta Braves at Cincinnati Reds | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Minnesota Twins at Pittsburgh Pirates | good | good | good | good | not recorded | good | good | not recorded | good | good | good | good |
| San Diego Padres at Washington Nationals | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Toronto Blue Jays at Baltimore Orioles | good | good | good | generic | not recorded | good | good | good | good | good | good | good |
| Boston Red Sox at Cleveland Guardians | good | good | good | generic | not recorded | good | good | good | good | good | good | good |
| Los Angeles Angels at Tampa Bay Rays | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Miami Marlins at New York Mets | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Chicago Cubs at St. Louis Cardinals | good | good | good | good | not recorded | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| what_would_change_contains_filler | 2 | 8 |
| protocol_unsupported_claim | 1 | 8 |
| confidence_high_for_partial_evidence | 5 | 8 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Confidence Calibration Rules v1 -- confidence was high relative to evidence richness on 5 of 8 run(s).

Basis:
- confidence_high_for_partial_evidence appeared on 5 of 8 run(s)
- no dominant prompt-quality or signal-quality scenario was triggered

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



