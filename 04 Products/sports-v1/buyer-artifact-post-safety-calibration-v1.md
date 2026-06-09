# Buyer Artifact Post-Safety Calibration v1

**date:** 2026-06-09
**status:** calibration slice. docs only. no runtime code, prompt change, parser change, Angular change, source addition, probe refresh activation, schema change, confidence/posture/lean mutation engine, or tenant/auth/billing work.
**scope:** judge whether Buyer Copy Safety v1 kept the buyer artifact commercially useful while removing betting-edge posture and internal diagnostic leak-through.

## Purpose

Buyer Copy Safety v1 changed the buyer-copy boundary. This slice checks whether that boundary works in practice:

- directional usefulness is preserved
- betting/tout language is removed
- internal signal/source gaps stay out of buyer copy
- uncertainty reads as confidence posture, not application incompleteness
- the artifact does not become too vague to be useful

## Background

Before Buyer Copy Safety v1, calibration found the buyer route structurally useful, but the top buyer-visible `lean` often said `Edge toward <team>`. The safety slice changed the FastAPI prompt, added deterministic parser sanitation for buyer-facing top-level fields, and changed the Angular buyer signal mapper so missing/unavailable/proxy/unknown diagnostic rows collapse into one buyer row: `Confirmation strength - Measured`.

This calibration did not generate live post-safety runs. Live generation requires the local sports dev stack and model calls; for this docs-first slice, the cheaper path was to inspect existing committed artifacts through the current safety boundary:

- top-level buyer fields were reviewed after applying the same sanitation rules as `_sanitize_buyer_copy`
- Signal Summary rows were reviewed through the current `buildBuyerSignalSummary` behavior
- dev diagnostics were checked from the existing artifact fields and dev signal-table behavior

Persisted pre-safety artifact JSON still contains the old `lean` text. The judgment below is about how the current safety boundary handles equivalent generated content going forward, not about rewriting historical artifacts.

## Sample set

Eight committed artifacts were reviewed. The sample includes NBA and MLB, legacy/coarse and rich `signalAvailability` paths, clear directional leans, mixed reads, and multiple measured-confirmation cases.

| # | league | matchup | artifact | shape | post-safety lean | signal summary shape |
|---|---|---|---|---|---|---|
| 1 | NBA | Cleveland Cavaliers at Detroit Pistons | `calibration/artifacts/20260507-1001-nba-ccbc433e.json` | legacy/coarse | Slight lean toward Detroit Pistons based on market favoring them at home. | Rest & Schedule, Market / Spread, Confirmation strength |
| 2 | NBA | Cleveland Cavaliers at New York Knicks | `calibration/artifacts/20260518-1001-nba-5f16433e.json` | rich v2 | Slight lean toward New York Knicks based on significant rest advantage. | Rest & Schedule, Market / Spread, Confirmation strength |
| 3 | NBA | New York Knicks at Oklahoma City Thunder | `calibration/artifacts/20260528-0931-nba-fa8d433e.json` | rich v2 | Slight lean toward Oklahoma City Thunder based on home court advantage and rest differential. | Rest & Schedule, Market / Spread, Confirmation strength |
| 4 | MLB | Atlanta Braves at Miami Marlins | `calibration/artifacts/20260518-1010-mlb-6416433e.json` | rich v2 | Slight lean toward Braves based on starting pitching advantage. | Starting Pitching |
| 5 | MLB | Baltimore Orioles at Tampa Bay Rays | `calibration/artifacts/20260518-1010-mlb-6816433e.json` | rich v2 | Signals are split. | Starting Pitching |
| 6 | MLB | Minnesota Twins at Pittsburgh Pirates | `calibration/artifacts/20260529-1421-mlb-098e433e.json` | rich v3 | Signals are split. | Starting Pitching |
| 7 | MLB | San Diego Padres at Washington Nationals | `calibration/artifacts/20260529-1421-mlb-0c8e433e.json` | rich v3 | Slight lean toward Padres based on starting pitching advantage. | Starting Pitching |
| 8 | MLB | Chicago Cubs at St. Louis Cardinals | `calibration/artifacts/20260529-1421-mlb-1c8e433e.json` | rich v3 | Slight lean toward Cardinals based on starting pitching advantage with Pallante's handedness against Cubs lineup. | Starting Pitching |

## Calibration method

For each sample, checked:

- league, matchup, artifact path, and artifact shape
- post-safety lean text
- whether direction remained clear
- whether confidence posture remained clear
- unsafe buyer-copy phrase hits
- internal signal/source gap leakage into buyer copy
- usefulness for a buyer
- repetition or generic phrasing
- buyer Signal Summary safety/usefulness
- dev/internal diagnostic explicitness

Unsafe buyer-copy phrases checked included: `edge toward`, `strong edge`, `betting edge`, `best bet`, `lock`, `sharp side`, `value play`, `prediction`, `expected to cover`, `should win`, `play X`, `take X`, `hammer X`, `free money`, `sharp_public`, `missing signal`, `unavailable signal`, `source unavailable`, `fallback failed`, `probe required`, `confidence capped`, `not available in this run`, and `Sharp vs Public unavailable`.

## Summary judgment

**Buyer Copy Safety v1 passed with minor polish follow-up.**

The safety boundary removes the unsafe posture without killing the product. Clear-lean artifacts still name the side and the reason. Mixed artifacts still say the read is split. The buyer Signal Summary no longer exposes `sharp_public`, missing source state, fallback failure, or probe language. Dev diagnostics remain explicit.

This did not overcorrect into useless neutrality. The lean still tells the buyer where the artifact points when the artifact has direction.

The main follow-up is polish, not repair: the safe phrasing is serviceable but somewhat repetitive, especially `Slight lean toward...` and `Confirmation strength - Measured`.

## Buyer-copy safety findings

Pass.

- 8/8 samples had a clear post-safety lean or a clear mixed-read statement.
- 0/8 projected buyer surfaces contained the unsafe phrase list.
- 0/8 projected buyer surfaces exposed raw `sharp_public`, `missing signal`, `fallback failed`, `probe required`, or source-unavailable language.
- Parser-level sanitation preserves direction while changing posture: `Edge toward Cardinals...` becomes `Slight lean toward Cardinals...`.
- The buyer route still avoids raw run ids, artifact version, protocol labels, source kinds, and follow-up diagnostics.

No small safety correction is warranted from this sample.

## Directional usefulness findings

Pass.

Clear-lean samples remain commercially useful:

- NBA: `Slight lean toward New York Knicks based on significant rest advantage.`
- NBA: `Slight lean toward Oklahoma City Thunder based on home court advantage and rest differential.`
- MLB: `Slight lean toward Cardinals based on starting pitching advantage with Pallante's handedness against Cubs lineup.`

Mixed samples remain understandable:

- `Signals are split.`

The artifact does not hide direction behind generic confidence language. The buyer can still tell whether the read points to a side or stays mixed.

Limit: every clear directional sample now starts with `Slight lean toward`. That is safe and clear, but it may feel formulaic across a list of runs. This belongs in Buyer Copy Polish Review v1, not a safety fix.

## Signal Summary findings

Verdict: buyer-safe and useful, with minor polish needed later.

What works:

- Grounded context still appears: Market / Spread, Rest & Schedule, Starting Pitching.
- NBA missing-confirmation cases now avoid raw `Sharp vs Public - Not available`.
- `Confirmation strength - Measured` is understandable next to grounded rows; it says why confidence stays conservative without exposing the missing provider/story.
- MLB samples with only starting-pitching evidence remain concise and do not invent confirmation rows.

What is weaker:

- In NBA samples, `Confirmation strength - Measured` repeats the same evidence phrase and what-would-change phrase each time.
- The aggregate row hides which specific confirmation source is absent. That is the intended safety tradeoff, and it did not flatten the table too much because Market / Spread and Rest & Schedule still remain visible.
- Legacy/coarse rows still lack impact metadata, so the UI can render placeholder-like flag text beside otherwise useful rows. This is a legacy presentation polish item, not a safety defect.

Overall: the aggregate row helps more than it hurts. It prevents internal gap leakage and still leaves enough grounded signal context to explain the read.

## Dev/internal diagnostic findings

Pass.

Internal truth remains available where reviewers need it:

- legacy/coarse samples still expose `groundedSignals` and `missingSignals`
- rich NBA samples still expose `signalAvailability.sharp_public`
- rich NBA samples still expose `signalFollowUps`
- `protocolView.interrogate.probe` still names the sharp/public gap internally
- MLB warning samples still expose `artifactQualityWarnings`, including ungrounded `signals_used` claims
- dev Brief Signal Table still shows source type, fallback need, probe eligibility, evidence, and what-would-change

Buyer safety did not erase diagnostic evidence. It only moved implementation gaps out of buyer copy.

## Defects found

None.

No code change was made. The calibration found no safety regression, no direction loss, and no internal diagnostic loss.

## Non-defect polish observations

- `Slight lean toward...` is clear but could become repetitive across multiple artifacts.
- `Confirmation strength - Measured` is safe but may need tone/cadence tuning after real post-safety generated runs.
- The buyer note `Structured lean from the current read` is serviceable but slightly implementation-flavored.
- Legacy/coarse Signal Summary rows can still show placeholder-like impact text because those artifacts lack impact metadata.
- This sample did not include NFL. NFL remains commercially important and should be reviewed once cheap NFL artifacts exist.

## Recommended next slice

**Buyer Copy Polish Review v1** remains the right next follow-up, but it should stay a polish review, not a repair. Scope it to:

- lean phrase variety
- confirmation-language cadence
- whether `Confirmation strength` should be relabeled
- whether `Structured lean from the current read` should become plainer buyer copy
- prompt efficiency and repeated phrasing after fresh generated artifacts

Do not move to source expansion first. Entry 19 remains valid, but source absence is not commercially blocking in this sample because the buyer surface handles it without leaking implementation gaps.

## Deferred items

- Entry 21, Buyer Copy Polish Review v1: remains Deferred, now confirmed by post-safety calibration as polish-only.
- Entry 19, Sports signal source and fallback catalog: remains Deferred; source expansion and Sharp/Public coverage are not resolved.
- NFL buyer artifact calibration: still needed when NFL artifacts are available or cheap to generate. No new ledger entry added because this is a sample limitation already carried by prior calibration.
- No probe refresh activation, source/fallback catalog, confidence/posture/lean mutation engine, schema work, tenant/auth/billing, or .NET work was opened.

## References reviewed

- `<DAI_VAULT_ROOT>/04 Products/sports-v1/buyer-copy-safety-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/buyer-artifact-ux-calibration-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/buyer-signal-summary.ts`
- `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/analyzer.component.html`
- `<DAI_REPO_ROOT>/apps/sports-app/src/app/dev-artifact-review/signal-table.ts`
- `<DAI_REPO_ROOT>/services/agent-service/app/services/sports_analyzer.py`
- the 8 artifact JSON files named in the sample table

## Verification plan

Docs-only verification:

- `git diff --check` for changed repos
- added-line exact-path scan
- vault non-ASCII added-line scan

No FastAPI, Angular, sports-app build, or .NET tests are required because no code changed.
