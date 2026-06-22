# Sports Artifact Calibration Report

| field | value |
|---|---|
| date | 2026-06-22 11:16 |
| competition | MLB |
| take | 10 |
| days | 5 |
| api base url | http://localhost:5007 |
| runs created | 10 |
| artifacts fetched | 10 |

## Summary

| matchup | game date | agentRunId | status | posture | confidence | evidence richness | missing signals | warnings |
|---|---|---|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | 2026-06-22 | a6b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Kansas City Royals at Tampa Bay Rays | 2026-06-22 | abb5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Texas Rangers at Miami Marlins | 2026-06-22 | b2b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Philadelphia Phillies at Washington Nationals | 2026-06-22 | b8b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.45 | 1 | starting_pitching | 0 |
| Houston Astros at Toronto Blue Jays | 2026-06-22 | b9b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Chicago Cubs at New York Mets | 2026-06-22 | beb5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Milwaukee Brewers at Cincinnati Reds | 2026-06-22 | c0b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.54 | 1 | starting_pitching | 1 |
| Cleveland Guardians at Chicago White Sox | 2026-06-22 | c7b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Los Angeles Dodgers at Minnesota Twins | 2026-06-22 | cab5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |
| Arizona Diamondbacks at St. Louis Cardinals | 2026-06-22 | d1b5433e-f36b-1410-8167-00373db4b724 | completed | monitor | 0.75 | 2 | none | 0 |

## Signal Availability

| matchup | signal | status | source | reason | quality | confidence effect | follow-up |
|---|---|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| New York Yankees at Detroit Tigers | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Kansas City Royals at Tampa Bay Rays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Kansas City Royals at Tampa Bay Rays | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Texas Rangers at Miami Marlins | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Texas Rangers at Miami Marlins | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Philadelphia Phillies at Washington Nationals | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Philadelphia Phillies at Washington Nationals | starting_pitching | missing | mlb_statsapi | unavailable | unavailable | neutral | - |
| Houston Astros at Toronto Blue Jays | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Houston Astros at Toronto Blue Jays | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Chicago Cubs at New York Mets | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Chicago Cubs at New York Mets | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Milwaukee Brewers at Cincinnati Reds | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Milwaukee Brewers at Cincinnati Reds | starting_pitching | missing | mlb_statsapi | unavailable | unavailable | neutral | - |
| Cleveland Guardians at Chicago White Sox | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Cleveland Guardians at Chicago White Sox | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Los Angeles Dodgers at Minnesota Twins | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Los Angeles Dodgers at Minnesota Twins | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |
| Arizona Diamondbacks at St. Louis Cardinals | starting_pitching | grounded | mlb_statsapi | grounded | usable | support_cautiously | - |
| Arizona Diamondbacks at St. Louis Cardinals | market | grounded | odds_api | grounded | usable | support_cautiously | sharp_public, line_movement |

## Signal Follow-Up Diagnostics

| matchup | triggered by | follow-up signal | status | reason | decision use | detail |
|---|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| New York Yankees at Detroit Tigers | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Kansas City Royals at Tampa Bay Rays | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Kansas City Royals at Tampa Bay Rays | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Texas Rangers at Miami Marlins | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Texas Rangers at Miami Marlins | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Philadelphia Phillies at Washington Nationals | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Philadelphia Phillies at Washington Nationals | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Houston Astros at Toronto Blue Jays | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Houston Astros at Toronto Blue Jays | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Chicago Cubs at New York Mets | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Chicago Cubs at New York Mets | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Milwaukee Brewers at Cincinnati Reds | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Milwaukee Brewers at Cincinnati Reds | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Cleveland Guardians at Chicago White Sox | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Cleveland Guardians at Chicago White Sox | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Los Angeles Dodgers at Minnesota Twins | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Los Angeles Dodgers at Minnesota Twins | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |
| Arizona Diamondbacks at St. Louis Cardinals | market | sharp_public | missing | primary_signal_missing | missing_confirmation | sharp_public was the recommended confirmation but is missing for this run; market remains directional only |
| Arizona Diamondbacks at St. Louis Cardinals | market | line_movement | candidate | future_signal_candidate | confirmation_candidate | line movement could confirm market direction in the future; not implemented yet and adjacent in equivalence |

## Quality Warnings

### signal / grounding

- **Milwaukee Brewers at Cincinnati Reds (2026-06-22)**: signals_used contains situational but situational was not grounded

## Cognitive Protocol Completeness

Perceive: detect, frame, aim | Interrogate: question, probe, verify | Discern: weigh, contrast, stress | Decide: resolve, position, justify

Read from the canonical persisted cognitiveProtocol (artifact v3). interrogate.probe is platform-completed (deterministic), not model-emitted. Synthesize (integrate, compose, deliver) is platform-owned and constant per run, so it is not shown as a per-run column.

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Kansas City Royals at Tampa Bay Rays | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Texas Rangers at Miami Marlins | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Philadelphia Phillies at Washington Nationals | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Houston Astros at Toronto Blue Jays | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Chicago Cubs at New York Mets | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Milwaukee Brewers at Cincinnati Reds | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Cleveland Guardians at Chicago White Sox | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Los Angeles Dodgers at Minnesota Twins | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |
| Arizona Diamondbacks at St. Louis Cardinals | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded | recorded |

## Automated Assessment

> deterministic checks only. no model calls. flags are signals, not verdicts.
> possibly unsupported: output mentions injuries, form, trends, or travel without a grounded signal for that category.
> not recorded*: field should have been filled given available signals.

### Overall Read

This batch produced 10 completed artifact(s) out of 10 assessed. 2 run(s) had missing signals. Evidence richness ranged from 1 to 2. Posture: 10x monitor. Main flags: confidence high for partial evidence on 8 run(s); protocol had possibly unsupported claims on 1 run(s).

### Per-Run Assessment

| matchup | posture | evidence | missing signals | flags | review focus |
|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | monitor | 2 | none | frame_missing_spread_context, confidence_high_for_partial_evidence | frame completeness, confidence calibration |
| Kansas City Royals at Tampa Bay Rays | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Texas Rangers at Miami Marlins | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Philadelphia Phillies at Washington Nationals | monitor | 1 | starting_pitching | protocol_unsupported_claim, posture_aligned_with_partial_evidence | protocol quality |
| Houston Astros at Toronto Blue Jays | monitor | 2 | none | what_would_change_contains_filler, frame_missing_spread_context, confidence_high_for_partial_evidence | what_would_change quality, frame completeness, confidence calibration |
| Chicago Cubs at New York Mets | monitor | 2 | none | what_would_change_contains_filler, confidence_high_for_partial_evidence | what_would_change quality, confidence calibration |
| Milwaukee Brewers at Cincinnati Reds | monitor | 1 | starting_pitching | frame_missing_spread_context, posture_aligned_with_partial_evidence | frame completeness |
| Cleveland Guardians at Chicago White Sox | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Los Angeles Dodgers at Minnesota Twins | monitor | 2 | none | confidence_high_for_partial_evidence | confidence calibration |
| Arizona Diamondbacks at St. Louis Cardinals | monitor | 2 | none | frame_missing_spread_context, confidence_high_for_partial_evidence | frame completeness, confidence calibration |

### Cognitive Protocol Quality

| matchup | detect | frame | aim | question | probe | verify | weigh | contrast | stress | resolve | position | justify |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| New York Yankees at Detroit Tigers | good | good | good | generic | good | good | generic | good | good | good | good | good |
| Kansas City Royals at Tampa Bay Rays | good | good | good | good | good | good | good | good | good | good | good | good |
| Texas Rangers at Miami Marlins | good | good | good | good | good | good | good | good | good | good | good | good |
| Philadelphia Phillies at Washington Nationals | good | good | good | good | good | good | good | good | possibly unsupported | good | good | good |
| Houston Astros at Toronto Blue Jays | good | good | good | good | good | good | good | good | good | good | good | good |
| Chicago Cubs at New York Mets | good | good | good | good | good | good | good | good | good | good | good | good |
| Milwaukee Brewers at Cincinnati Reds | good | good | good | good | good | good | good | good | good | good | good | good |
| Cleveland Guardians at Chicago White Sox | good | good | good | good | good | good | good | good | good | good | good | good |
| Los Angeles Dodgers at Minnesota Twins | good | good | good | good | good | good | good | good | good | good | good | good |
| Arizona Diamondbacks at St. Louis Cardinals | good | good | good | good | good | good | possibly unsupported | good | good | good | good | good |

## Pattern Summary

| flag | count | of |
|---|---|---|
| what_would_change_contains_filler | 2 | 10 |
| frame_missing_spread_context | 4 | 10 |
| protocol_unsupported_claim | 1 | 10 |
| confidence_high_for_partial_evidence | 8 | 10 |
| posture_aligned_with_partial_evidence | 2 | 10 |

## Recommended Next Slice

> this is a deterministic recommendation based on flags. it is not an automatic decision.

Confidence Calibration Rules v1 -- confidence was high relative to evidence richness on 8 of 10 run(s).

Basis:
- confidence_high_for_partial_evidence appeared on 8 of 10 run(s)
- no dominant prompt-quality or signal-quality scenario was triggered

## Manual Review Notes

### Good Patterns


### Bad Patterns


### False Warnings


### Missing Warnings


### Prompt Issues


### Quality Rule Issues


### Candidate Next Slice



