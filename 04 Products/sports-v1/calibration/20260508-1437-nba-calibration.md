# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-08 14:37 |
| competition | NBA |
| take | 3 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 3 |
| artifacts fetched | 3 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | 2026-05-08 | e9bc433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.63 | 2 | sharp_public | 0 |
| San Antonio Spurs at Minnesota Timberwolves | 2026-05-08 | ecbc433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |
| Detroit Pistons at Cleveland Cavaliers | 2026-05-09 | f0bc433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| San Antonio Spurs at Minnesota Timberwolves | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Detroit Pistons at Cleveland Cavaliers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 3 completed artifact(s) out of 3 assessed. 3 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 3x monitor. No major quality flags detected.

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | monitor | 2 | sharp_public | frame_missing_rest_context, missing_sharp_public, posture_aligned_with_partial_evidence | frame completeness |
| San Antonio Spurs at Minnesota Timberwolves | monitor | 2 | sharp_public | missing_sharp_public, posture_aligned_with_partial_evidence | none |
| Detroit Pistons at Cleveland Cavaliers | monitor | 2 | sharp_public | frame_missing_rest_context, missing_sharp_public, posture_aligned_with_partial_evidence | frame completeness |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | good | good | good | good | good | good | good | good | good | good | good | good |
| San Antonio Spurs at Minnesota Timberwolves | good | good | good | good | good | good | good | good | good | good | good | good |
| Detroit Pistons at Cleveland Cavaliers | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| frame_missing_rest_context | 2 | 3 |
| missing_sharp_public | 3 | 3 |
| posture_aligned_with_partial_evidence | 3 | 3 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Signal Source Review v1 -- sharp_public was missing from all 3 run(s). check provider coverage for this competition.

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



