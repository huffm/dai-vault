# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-09 15:54 |
| competition | NBA |
| take | 1 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 1 |
| artifacts fetched | 1 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at Los Angeles Lakers | 2026-05-09 | 00bd433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.72 | 2 | sharp_public | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at Los Angeles Lakers | rest_schedule | grounded | espn | grounded | usable | support_cautiously | — |
| Oklahoma City Thunder at Los Angeles Lakers | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Oklahoma City Thunder at Los Angeles Lakers | sharp_public | missing | actionnetwork | provider_returned_empty_odds | unavailable | block_aggressive_posture | line_movement, market |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at Los Angeles Lakers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 1 completed artifact(s) out of 1 assessed. 1 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 1x monitor. Main flags: confidence high for partial evidence on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Oklahoma City Thunder at Los Angeles Lakers | monitor | 2 | sharp_public | frame_missing_spread_context, confidence_high_for_partial_evidence, missing_sharp_public, signal_quality_blocks_aggressive_posture, posture_aligned_with_partial_evidence | frame completeness, confidence calibration, posture alignment |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Oklahoma City Thunder at Los Angeles Lakers | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| frame_missing_spread_context | 1 | 1 |
| confidence_high_for_partial_evidence | 1 | 1 |
| missing_sharp_public | 1 | 1 |
| posture_aligned_with_partial_evidence | 1 | 1 |
| signal_quality_blocks_aggressive_posture | 1 | 1 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Signal Follow-Up Diagnostics v1 -- the dominant issue is signal follow-up/coverage, not prompt quality.

Basis:
- cognitive phase quality was good across all assessed runs
- sharp_public was missing on 1 of 1 run(s) and blocks aggressive posture
- follow-up signals are listed but not yet diagnosed as implemented or unavailable (e.g., sharp_public, line_movement, market)
- posture is aligned with partial evidence (monitor/pass) on 1 of 1 run(s)

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



