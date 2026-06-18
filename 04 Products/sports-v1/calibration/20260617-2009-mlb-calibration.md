# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-17 20:09 |
| competition | MLB |
| take | 8 |
| days | 3 |
| api base url | http://localhost:5007 |
| runs created | 8 |
| artifacts fetched | 8 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | 2026-06-17 | b0de423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Baltimore Orioles at Seattle Mariners | 2026-06-17 | b4de423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Toronto Blue Jays at Boston Red Sox | 2026-06-18 | b8de423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Cleveland Guardians at Milwaukee Brewers | 2026-06-18 | bcde423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Minnesota Twins at Texas Rangers | 2026-06-18 | bdde423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Baltimore Orioles at Seattle Mariners | 2026-06-18 | bede423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| New York Mets at Philadelphia Phillies | 2026-06-18 | c2de423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Chicago White Sox at New York Yankees | 2026-06-18 | c4de423e-f36b-1410-8164-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Pittsburgh Pirates at Athletics | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Baltimore Orioles at Seattle Mariners | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Baltimore Orioles at Seattle Mariners | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Toronto Blue Jays at Boston Red Sox | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Toronto Blue Jays at Boston Red Sox | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Cleveland Guardians at Milwaukee Brewers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Cleveland Guardians at Milwaukee Brewers | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Minnesota Twins at Texas Rangers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Minnesota Twins at Texas Rangers | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Baltimore Orioles at Seattle Mariners | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Baltimore Orioles at Seattle Mariners | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| New York Mets at Philadelphia Phillies | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| New York Mets at Philadelphia Phillies | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Chicago White Sox at New York Yankees | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Chicago White Sox at New York Yankees | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Pittsburgh Pirates at Athletics | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Baltimore Orioles at Seattle Mariners | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Baltimore Orioles at Seattle Mariners | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Toronto Blue Jays at Boston Red Sox | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Toronto Blue Jays at Boston Red Sox | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Cleveland Guardians at Milwaukee Brewers | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Cleveland Guardians at Milwaukee Brewers | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Minnesota Twins at Texas Rangers | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Minnesota Twins at Texas Rangers | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Baltimore Orioles at Seattle Mariners | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Baltimore Orioles at Seattle Mariners | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| New York Mets at Philadelphia Phillies | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| New York Mets at Philadelphia Phillies | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Chicago White Sox at New York Yankees | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Chicago White Sox at New York Yankees | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |

## Quality Warnings

No quality warnings recorded across all runs.

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Baltimore Orioles at Seattle Mariners | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Toronto Blue Jays at Boston Red Sox | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Cleveland Guardians at Milwaukee Brewers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Minnesota Twins at Texas Rangers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Baltimore Orioles at Seattle Mariners | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| New York Mets at Philadelphia Phillies | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Chicago White Sox at New York Yankees | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 8 completed artifact(s) out of 8 assessed. Evidence richness was 2 on all runs. Posture: 8x monitor. Main flags: confidence high for partial evidence on 8 run(s); protocol had possibly unsupported claims on 2 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | monitor | 2 | none | frame_missing_spread_context, confidence_high_for_partial_evidence | frame completeness, confidence calibration |
| Baltimore Orioles at Seattle Mariners | monitor | 2 | none | what_would_change_contains_filler, protocol_unsupported_claim, confidence_high_for_partial_evidence | protocol quality, what_would_change quality, confidence calibration |
| Toronto Blue Jays at Boston Red Sox | monitor | 2 | none | protocol_unsupported_claim, confidence_high_for_partial_evidence | protocol quality, confidence calibration |
| Cleveland Guardians at Milwaukee Brewers | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Minnesota Twins at Texas Rangers | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Baltimore Orioles at Seattle Mariners | monitor | 2 | none | frame_missing_spread_context, confidence_high_for_partial_evidence | frame completeness, confidence calibration |
| New York Mets at Philadelphia Phillies | monitor | 2 | none | frame_missing_spread_context, confidence_high_for_partial_evidence | frame completeness, confidence calibration |
| Chicago White Sox at New York Yankees | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Pittsburgh Pirates at Athletics | good | good | good | generic | good | good | good | good | good | good | good | good |
| Baltimore Orioles at Seattle Mariners | good | good | good | generic | good | possibly unsupported | generic | good | good | good | good | good |
| Toronto Blue Jays at Boston Red Sox | good | good | good | possibly unsupported | good | good | generic | good | good | good | good | good |
| Cleveland Guardians at Milwaukee Brewers | good | good | good | good | good | good | generic | good | good | good | good | good |
| Minnesota Twins at Texas Rangers | good | good | good | good | good | good | good | good | generic | good | good | good |
| Baltimore Orioles at Seattle Mariners | good | good | good | good | good | good | good | good | good | good | good | good |
| New York Mets at Philadelphia Phillies | good | good | good | good | good | good | good | good | good | good | good | good |
| Chicago White Sox at New York Yankees | good | good | good | good | good | good | good | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| what_would_change_contains_filler | 1 | 8 |
| frame_missing_spread_context | 3 | 8 |
| protocol_unsupported_claim | 2 | 8 |
| confidence_high_for_partial_evidence | 8 | 8 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Confidence Calibration Rules v1 -- confidence was high relative to evidence richness on 8 of 8 run(s).

Basis:
- confidence_high_for_partial_evidence appeared on 8 of 8 run(s)
- no dominant prompt-quality or signal-quality scenario was triggered

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



