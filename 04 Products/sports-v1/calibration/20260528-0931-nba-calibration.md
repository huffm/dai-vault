# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-28 09:31 |
| competition | NBA |
| take | 5 |
| days | 14 |
| api base url | http://localhost:5007 |
| runs created | 3 |
| artifacts fetched | 3 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | 2026-05-28 | f48d433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |
| New York Knicks at Oklahoma City Thunder | 2026-06-03 | fa8d433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |
| New York Knicks at San Antonio Spurs | 2026-06-03 | 018e433e-f36b-1410-815f-00373db4b724 | completed | monitor | 0.63 | 2 | sharp_public | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| Oklahoma City Thunder at San Antonio Spurs | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Oklahoma City Thunder at San Antonio Spurs | sharp_public | missing | actionnetwork | provider_returned_empty_odds | unavailable | block_aggressive_posture | line_movement, market |
| New York Knicks at Oklahoma City Thunder | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| New York Knicks at Oklahoma City Thunder | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| New York Knicks at Oklahoma City Thunder | sharp_public | missing | actionnetwork | no_matching_game | unavailable | block_aggressive_posture | line_movement, market |
| New York Knicks at San Antonio Spurs | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| New York Knicks at San Antonio Spurs | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| New York Knicks at San Antonio Spurs | sharp_public | missing | actionnetwork | no_matching_game | unavailable | block_aggressive_posture | line_movement, market |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Oklahoma City Thunder at San Antonio Spurs | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to market direction; it is the recommended adjacent proxy when sharp_public is missing, but it is not implemented yet and cannot equivalently substitute |
| Oklahoma City Thunder at San Antonio Spurs | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to sharp_public (market pressure, not the public/sharp split); it cannot equivalently substitute and it is not implemented yet |
| Oklahoma City Thunder at San Antonio Spurs | sharp_public | market | grounded | already_grounded | directional_context | market provides direction but not the public/sharp split; it is a lateral_proxy and cannot lift confidence on its own |
| New York Knicks at Oklahoma City Thunder | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| New York Knicks at Oklahoma City Thunder | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to market direction; it is the recommended adjacent proxy when sharp_public is missing, but it is not implemented yet and cannot equivalently substitute |
| New York Knicks at Oklahoma City Thunder | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to sharp_public (market pressure, not the public/sharp split); it cannot equivalently substitute and it is not implemented yet |
| New York Knicks at Oklahoma City Thunder | sharp_public | market | grounded | already_grounded | directional_context | market provides direction but not the public/sharp split; it is a lateral_proxy and cannot lift confidence on its own |
| New York Knicks at San Antonio Spurs | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| New York Knicks at San Antonio Spurs | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to market direction; it is the recommended adjacent proxy when sharp_public is missing, but it is not implemented yet and cannot equivalently substitute |
| New York Knicks at San Antonio Spurs | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to sharp_public (market pressure, not the public/sharp split); it cannot equivalently substitute and it is not implemented yet |
| New York Knicks at San Antonio Spurs | sharp_public | market | grounded | already_grounded | directional_context | market provides direction but not the public/sharp split; it is a lateral_proxy and cannot lift confidence on its own |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | recorded | recorded | recorded | recorded | not recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded |
| New York Knicks at Oklahoma City Thunder | recorded | recorded | recorded | recorded | not recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded |
| New York Knicks at San Antonio Spurs | recorded | recorded | recorded | recorded | not recorded | recorded | not recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 3 completed artifact(s) out of 3 assessed. 3 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 3x monitor. Main flags: counter_case was generic on 2 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | monitor | 2 | sharp_public | missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | posture alignment |
| New York Knicks at Oklahoma City Thunder | monitor | 2 | sharp_public | counter_case_generic, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | interrogate quality, posture alignment |
| New York Knicks at San Antonio Spurs | monitor | 2 | sharp_public | counter_case_generic, what_would_change_contains_filler, frame_missing_rest_context, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | interrogate quality, what_would_change quality, frame completeness, posture alignment |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at San Antonio Spurs | good | good | good | good | not recorded | good | not recorded | good | good | good | good | good |
| New York Knicks at Oklahoma City Thunder | good | good | good | good | not recorded | good | not recorded | good | good | good | good | good |
| New York Knicks at San Antonio Spurs | good | good | good | good | not recorded | good | not recorded | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 2 | 3 |
| what_would_change_contains_filler | 1 | 3 |
| frame_missing_rest_context | 1 | 3 |
| missing_sharp_public | 3 | 3 |
| posture_aligned_with_partial_evidence | 3 | 3 |
| signal_quality_blocks_aggressive_posture | 3 | 3 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- prompt-quality flags appeared on 2 of 3 run(s).

Basis:
- prompt-quality flags (counter_case, watch_for, what_would_change, listen, interrogate) appeared on 2 of 3 run(s)
- this dominates over structural signal/posture flags

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



