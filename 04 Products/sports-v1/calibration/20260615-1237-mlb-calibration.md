# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-15 12:37 |
| competition | MLB |
| take | 10 |
| days | 3 |
| api base url | http://localhost:5007 |
| runs created | 10 |
| artifacts fetched | 10 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Miami Marlins at Philadelphia Phillies | 2026-06-15 | 2303433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 2 |
| Kansas City Royals at Washington Nationals | 2026-06-15 | 2803433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 2 |
| New York Mets at Cincinnati Reds | 2026-06-15 | 2a03433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| San Diego Padres at St. Louis Cardinals | 2026-06-15 | 2e03433e-f36b-1410-8163-00373db4b724 | completed | wait | 0.38 | 0 | starting_pitching | 0 |
| Colorado Rockies at Chicago Cubs | 2026-06-15 | 3403433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Minnesota Twins at Texas Rangers | 2026-06-15 | 3603433e-f36b-1410-8163-00373db4b724 | completed | wait | 0.38 | 0 | starting_pitching | 0 |
| Detroit Tigers at Houston Astros | 2026-06-15 | 3d03433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Los Angeles Angels at Arizona Diamondbacks | 2026-06-15 | 4103433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |
| Pittsburgh Pirates at Athletics | 2026-06-15 | 4203433e-f36b-1410-8163-00373db4b724 | completed | wait | 0.5 | 1 | none | 0 |
| Tampa Bay Rays at Los Angeles Dodgers | 2026-06-15 | 4903433e-f36b-1410-8163-00373db4b724 | completed | monitor | 0.75 | 1 | none | 3 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Miami Marlins at Philadelphia Phillies | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Kansas City Royals at Washington Nationals | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| New York Mets at Cincinnati Reds | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| San Diego Padres at St. Louis Cardinals | starting_pitching | missing | mlb_statsapi | unavailable | unavailable | neutral | - |
| Colorado Rockies at Chicago Cubs | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Minnesota Twins at Texas Rangers | starting_pitching | missing | mlb_statsapi | unavailable | unavailable | neutral | - |
| Detroit Tigers at Houston Astros | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Los Angeles Angels at Arizona Diamondbacks | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Pittsburgh Pirates at Athletics | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Tampa Bay Rays at Los Angeles Dodgers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| - | - | - | - | - | - | no follow-up diagnostics recorded (older artifact or no follow-up signals) |

## Quality Warnings

### signal / grounding

- **Miami Marlins at Philadelphia Phillies (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Miami Marlins at Philadelphia Phillies (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **Kansas City Royals at Washington Nationals (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Kansas City Royals at Washington Nationals (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **New York Mets at Cincinnati Reds (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **New York Mets at Cincinnati Reds (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **New York Mets at Cincinnati Reds (2026-06-15)**: signals_used contains ballpark but ballpark was not grounded
- **Colorado Rockies at Chicago Cubs (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Colorado Rockies at Chicago Cubs (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **Colorado Rockies at Chicago Cubs (2026-06-15)**: signals_used contains ballpark but ballpark was not grounded
- **Detroit Tigers at Houston Astros (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Detroit Tigers at Houston Astros (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **Detroit Tigers at Houston Astros (2026-06-15)**: signals_used contains ballpark but ballpark was not grounded
- **Los Angeles Angels at Arizona Diamondbacks (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Los Angeles Angels at Arizona Diamondbacks (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **Los Angeles Angels at Arizona Diamondbacks (2026-06-15)**: signals_used contains ballpark but ballpark was not grounded
- **Tampa Bay Rays at Los Angeles Dodgers (2026-06-15)**: signals_used contains bullpen but bullpen was not grounded
- **Tampa Bay Rays at Los Angeles Dodgers (2026-06-15)**: signals_used contains lineup_form but lineup_form was not grounded
- **Tampa Bay Rays at Los Angeles Dodgers (2026-06-15)**: signals_used contains ballpark but ballpark was not grounded

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Miami Marlins at Philadelphia Phillies | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Kansas City Royals at Washington Nationals | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| New York Mets at Cincinnati Reds | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| San Diego Padres at St. Louis Cardinals | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| Colorado Rockies at Chicago Cubs | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Minnesota Twins at Texas Rangers | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| Detroit Tigers at Houston Astros | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Los Angeles Angels at Arizona Diamondbacks | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Pittsburgh Pirates at Athletics | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded |
| Tampa Bay Rays at Los Angeles Dodgers | recorded | recorded | recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 10 completed artifact(s) out of 10 assessed. 2 run(s) had missing signals. Evidence richness ranged from 0 to 1. Posture: 7x monitor, 3x wait. Main flags: counter_case was generic on 4 run(s); confidence high for partial evidence on 7 run(s); protocol had possibly unsupported claims on 3 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Miami Marlins at Philadelphia Phillies | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |
| Kansas City Royals at Washington Nationals | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |
| New York Mets at Cincinnati Reds | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| San Diego Padres at St. Louis Cardinals | wait | 0 | starting_pitching | protocol_unsupported_claim | protocol quality |
| Colorado Rockies at Chicago Cubs | monitor | 1 | none | protocol_unsupported_claim, confidence_high_for_partial_evidence | protocol quality, confidence calibration |
| Minnesota Twins at Texas Rangers | wait | 0 | starting_pitching | none | none |
| Detroit Tigers at Houston Astros | monitor | 1 | none | confidence_high_for_partial_evidence | confidence calibration |
| Los Angeles Angels at Arizona Diamondbacks | monitor | 1 | none | counter_case_generic, what_would_change_contains_filler, confidence_high_for_partial_evidence | interrogate quality, what_would_change quality, confidence calibration |
| Pittsburgh Pirates at Athletics | wait | 1 | none | protocol_unsupported_claim | protocol quality |
| Tampa Bay Rays at Los Angeles Dodgers | monitor | 1 | none | counter_case_generic, confidence_high_for_partial_evidence | interrogate quality, confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Miami Marlins at Philadelphia Phillies | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Kansas City Royals at Washington Nationals | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| New York Mets at Cincinnati Reds | good | good | good | good | not recorded | good | possibly unsupported | good | good | good | good | good |
| San Diego Padres at St. Louis Cardinals | good | good | good | good | not recorded | generic | good | not recorded | good | good | good | good |
| Colorado Rockies at Chicago Cubs | good | good | good | good | not recorded | generic | good | good | good | good | good | good |
| Minnesota Twins at Texas Rangers | good | good | good | good | not recorded | good | good | not recorded | good | good | good | good |
| Detroit Tigers at Houston Astros | good | good | good | good | not recorded | good | good | good | good | good | good | good |
| Los Angeles Angels at Arizona Diamondbacks | good | good | good | good | not recorded | good | generic | good | good | good | good | good |
| Pittsburgh Pirates at Athletics | good | good | good | good | not recorded | possibly unsupported | good | not recorded | good | good | good | good |
| Tampa Bay Rays at Los Angeles Dodgers | good | good | good | good | not recorded | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 4 | 10 |
| what_would_change_contains_filler | 1 | 10 |
| protocol_unsupported_claim | 3 | 10 |
| confidence_high_for_partial_evidence | 7 | 10 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- prompt-quality flags appeared on 7 of 10 run(s).

Basis:
- prompt-quality flags (counter_case, watch_for, what_would_change, contrast, protocol) appeared on 7 of 10 run(s)
- this dominates over structural signal/posture flags

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



