# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-05-08 12:46 |
| competition | NBA |
| take | 1 |
| days | 7 |
| api base url | http://localhost:5007 |
| runs created | 1 |
| artifacts fetched | 1 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | 2026-05-08 | debc433e-f36b-1410-815b-00373db4b724 | completed | monitor | 0.63 | 2 | sharp_public | 0 |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Phase Completeness

Perceive: detect, frame, aim | Interrogate: balance, stress, reframe | Discern: test, listen, filter | Decide: calibrate, posture, voice

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 1 completed artifact(s) out of 1 assessed. 1 run(s) had missing signals. Evidence richness was 2 on all runs. Posture: 1x monitor. Main flags: counter_case was generic on 1 run(s); interrogate had possibly unsupported claims on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | monitor | 2 | sharp_public | counter_case_generic, frame_missing_rest_context, interrogate_unsupported_claim, missing_sharp_public, posture_aligned_with_partial_evidence | interrogate quality, frame completeness |

### Cognitive Phase Quality

| matchup | detect | frame | aim | balance | stress | reframe | test | listen | filter | calibrate | posture | voice |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Knicks at Philadelphia 76ers | good | good | good | possibly unsupported | good | good | generic | good | possibly unsupported | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| counter_case_generic | 1 | 1 |
| frame_missing_rest_context | 1 | 1 |
| interrogate_unsupported_claim | 1 | 1 |
| missing_sharp_public | 1 | 1 |
| posture_aligned_with_partial_evidence | 1 | 1 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Cognitive Prompt Tightening v1.5 -- generic or incomplete cognitive outputs appeared on most artifacts.

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



