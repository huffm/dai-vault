# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-09 08:17 |
| competition | NBA |
| take | 1 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 1 |
| artifacts fetched | 1 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Detroit Pistons at Cleveland Cavaliers | 2026-05-09 | f5bc433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.68 | 2 | sharp_public | 0 |

## Signal Availability

| matchup | signal | status | source | reason |
|---|---|---|---|---|
| Detroit Pistons at Cleveland Cavaliers | rest_schedule | grounded | espn | grounded |
| Detroit Pistons at Cleveland Cavaliers | market | grounded | odds_api | grounded |
| Detroit Pistons at Cleveland Cavaliers | sharp_public | missing | actionnetwork | provider_returned_empty_odds |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Detroit Pistons at Cleveland Cavaliers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 1 completed artifact(s) out of 1 assessed. 1 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 1x monitor. No major quality flags detected.

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Detroit Pistons at Cleveland Cavaliers | monitor | 2 | sharp_public | missing_sharp_public, posture_aligned_with_partial_evidence | none |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Detroit Pistons at Cleveland Cavaliers | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| missing_sharp_public | 1 | 1 |
| posture_aligned_with_partial_evidence | 1 | 1 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Signal Source Review v1 -- sharp_public was missing from all 1 run(s). check provider coverage for this competition.

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



