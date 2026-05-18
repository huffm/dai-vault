# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-18 10:01 |
| competition | NBA |
| take | 10 |
| days | 14 |
| api base url | http://localhost:5007 |
| runs created | 2 |
| artifacts fetched | 2 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | 2026-05-18 | 5816433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.72 | 2 | sharp_public | 0 |
| Cleveland Cavaliers at New York Knicks | 2026-05-19 | 5f16433e-f36b-1410-815c-00373db4b724 | completed | monitor | 0.72 | 2 | sharp_public | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| San Antonio Spurs at Oklahoma City Thunder | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| San Antonio Spurs at Oklahoma City Thunder | sharp_public | missing | actionnetwork | provider_returned_empty_odds | unavailable | block_aggressive_posture | line_movement, market |
| Cleveland Cavaliers at New York Knicks | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| Cleveland Cavaliers at New York Knicks | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Cleveland Cavaliers at New York Knicks | sharp_public | missing | actionnetwork | provider_returned_empty_odds | unavailable | block_aggressive_posture | line_movement, market |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| San Antonio Spurs at Oklahoma City Thunder | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to market direction; it is the recommended adjacent proxy when sharp_public is missing, but it is not implemented yet and cannot equivalently substitute |
| San Antonio Spurs at Oklahoma City Thunder | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to sharp_public (market pressure, not the public/sharp split); it cannot equivalently substitute and it is not implemented yet |
| San Antonio Spurs at Oklahoma City Thunder | sharp_public | market | grounded | already_grounded | directional_context | market provides direction but not the public/sharp split; it is a lateral_proxy and cannot lift confidence on its own |
| Cleveland Cavaliers at New York Knicks | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Cleveland Cavaliers at New York Knicks | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to market direction; it is the recommended adjacent proxy when sharp_public is missing, but it is not implemented yet and cannot equivalently substitute |
| Cleveland Cavaliers at New York Knicks | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement is adjacent to sharp_public (market pressure, not the public/sharp split); it cannot equivalently substitute and it is not implemented yet |
| Cleveland Cavaliers at New York Knicks | sharp_public | market | grounded | already_grounded | directional_context | market provides direction but not the public/sharp split; it is a lateral_proxy and cannot lift confidence on its own |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Cleveland Cavaliers at New York Knicks | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 2 completed artifact(s) out of 2 assessed. 2 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 2x monitor. Main flags: confidence high for partial evidence on 2 run(s); interrogate had possibly unsupported claims on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | monitor | 2 | sharp_public | frame_missing_rest_context, interrogate_unsupported_claim, confidence_high_for_partial_evidence, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | interrogate quality, frame completeness, confidence calibration, posture alignment |
| Cleveland Cavaliers at New York Knicks | monitor | 2 | sharp_public | frame_missing_rest_context, confidence_high_for_partial_evidence, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | frame completeness, confidence calibration, posture alignment |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Oklahoma City Thunder | good | good | good | good | possibly unsupported | good | good | good | good | good | good | good |
| Cleveland Cavaliers at New York Knicks | good | good | good | good | good | good | good | good | possibly unsupported | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| frame_missing_rest_context | 2 | 2 |
| interrogate_unsupported_claim | 1 | 2 |
| confidence_high_for_partial_evidence | 2 | 2 |
| missing_sharp_public | 2 | 2 |
| posture_aligned_with_partial_evidence | 2 | 2 |
| signal_quality_blocks_aggressive_posture | 2 | 2 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- prompt-quality flags appeared on 1 of 2 run(s).

Basis:
- prompt-quality flags (counter_case, watch_for, what_would_change, listen, interrogate) appeared on 1 of 2 run(s)
- this dominates over structural signal/posture flags

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



