# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-10 19:13 |
| competition | NBA |
| take | 1 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 1 |
| artifacts fetched | 1 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | 2026-05-10 | 04bd433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| San Antonio Spurs at Minnesota Timberwolves | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| San Antonio Spurs at Minnesota Timberwolves | sharp_public | missing | actionnetwork | provider_returned_empty_odds | unavailable | block_aggressive_posture | line_movement, market |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| San Antonio Spurs at Minnesota Timberwolves | market | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement may help proxy market confirmation when sharp/public splits are unavailable, but it is not implemented yet |
| San Antonio Spurs at Minnesota Timberwolves | sharp_public | line_movement | not_implemented | future_signal_candidate | proxy_candidate | line movement may help proxy market confirmation when sharp/public splits are unavailable, but it is not implemented yet |
| San Antonio Spurs at Minnesota Timberwolves | sharp_public | market | grounded | already_grounded | directional_context | market can provide direction but not sharp/public confirmation |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 1 completed artifact(s) out of 1 assessed. 1 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 1x monitor. No major quality flags detected.

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | monitor | 2 | sharp_public | frame_missing_spread_context, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | frame completeness, posture alignment |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| San Antonio Spurs at Minnesota Timberwolves | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| frame_missing_spread_context | 1 | 1 |
| missing_sharp_public | 1 | 1 |
| posture_aligned_with_partial_evidence | 1 | 1 |
| signal_quality_blocks_aggressive_posture | 1 | 1 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Signal Follow-Up Diagnostics v1 -- the dominant issue is signal follow-up/coverage, not prompt quality.

Basis:
- cognitive phase quality was good across all assessed runs
- sharp_public was missing on 1 of 1 run(s) and blocks aggressive posture
- follow-up path exists but line_movement is not implemented yet (diagnosed by SignalFollowUpEvaluator)
- posture is aligned with partial evidence (monitor/pass) on 1 of 1 run(s)

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



