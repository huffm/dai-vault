# Buyer Copy Polish Review v1

**date:** 2026-06-09
**status:** polish review with one small buyer-facing Angular copy change. no prompt change, parser change, source addition, probe refresh activation, schema change, confidence/posture/lean mutation engine, .NET change, or tenant/auth/billing work.
**scope:** review the already-safe buyer artifact for cadence, phrase repetition, buyer trust, and premium analyst feel.

## Purpose

Buyer Copy Safety v1 made the artifact safe. Buyer Artifact Post-Safety Calibration v1 confirmed the artifact stayed useful. This slice reviewed whether the safe copy now feels polished enough for a paid buyer.

The review asked:

- does the lean still tell the buyer where the read points?
- does confidence posture read clearly?
- does the copy sound composed rather than templated?
- does `Confirmation strength - Measured` help or feel abstract?
- are unsafe betting phrases still absent?
- do internal signal/source gaps stay out of buyer copy?
- are dev diagnostics still explicit?

## Background

The prior safety boundary is preserved:

- buyer copy may state direction and confidence posture
- buyer copy must not sound like a tout sheet, bet slip, prediction engine, debug report, or vague corporate summary
- internal diagnostic fields can still name raw signal keys, missing source state, fallback status, probe status, and quality warnings

This slice did not generate new model artifacts. It reviewed the same committed sample set used by the post-safety calibration and projected those artifacts through the current safety boundary.

## Sample set

Eight artifacts were reviewed:

| # | league | matchup | artifact | shape | cadence focus |
|---|---|---|---|---|---|
| 1 | NBA | Cleveland Cavaliers at Detroit Pistons | `calibration/artifacts/20260507-1001-nba-ccbc433e.json` | legacy/coarse | clear lean, measured confirmation |
| 2 | NBA | Cleveland Cavaliers at New York Knicks | `calibration/artifacts/20260518-1001-nba-5f16433e.json` | rich v2 | clear lean, rich follow-ups |
| 3 | NBA | New York Knicks at Oklahoma City Thunder | `calibration/artifacts/20260528-0931-nba-fa8d433e.json` | rich v2 | clear lean, measured confirmation |
| 4 | MLB | Atlanta Braves at Miami Marlins | `calibration/artifacts/20260518-1010-mlb-6416433e.json` | rich v2 | clear lean, one-signal artifact |
| 5 | MLB | Baltimore Orioles at Tampa Bay Rays | `calibration/artifacts/20260518-1010-mlb-6816433e.json` | rich v2 | mixed read, quality warnings |
| 6 | MLB | Minnesota Twins at Pittsburgh Pirates | `calibration/artifacts/20260529-1421-mlb-098e433e.json` | rich v3 | mixed read, quality warnings |
| 7 | MLB | San Diego Padres at Washington Nationals | `calibration/artifacts/20260529-1421-mlb-0c8e433e.json` | rich v3 | clear lean, quality warnings |
| 8 | MLB | Chicago Cubs at St. Louis Cardinals | `calibration/artifacts/20260529-1421-mlb-1c8e433e.json` | rich v3 | clear lean, clean diagnostics |

## Review method

For each sample, reviewed:

- post-safety lean
- summary/rationale cadence
- confidence posture
- repeated phrases
- buyer Signal Summary rows
- `Confirmation strength - Measured`
- buyer usefulness
- unsafe phrase hits
- internal diagnostic visibility

Phrase counts in the projected sample:

| phrase | count |
|---|---:|
| `Slight lean toward` | 6/8 |
| `Signals are split` | 2/8 |
| `Structured lean from the current read` | 8/8 |
| `Confirmation strength` | 3/8 |
| `Available evidence is not strong enough for a firmer stance` | 3/8 |
| `Keeps confidence measured` | 3/8 |

## Summary judgment

**Buyer Copy Polish Review v1 resolved with one small UI copy change.**

The artifact is safe and still useful. The read still gives direction when direction exists, and mixed cases are understandable. The main cadence issue was not the model prompt or the Signal Summary label; it was a static Angular note under the current lean: `Structured lean from the current read.`

That phrase is accurate but implementation-flavored. It appears on every buyer artifact with a lean, so it was changed to:

`Current read from the available evidence.`

No prompt phrase bank was added. The projected sample shows repetition, but it uses pre-safety artifacts passed through the sanitizer. A prompt phrase bank should wait for fresh post-safety generated artifacts so we do not tune against historical sanitizer output.

## Lean phrase findings

Clear leans stayed clear:

- `Slight lean toward New York Knicks based on significant rest advantage.`
- `Slight lean toward Oklahoma City Thunder based on home court advantage and rest differential.`
- `Slight lean toward Cardinals based on starting pitching advantage with Pallante's handedness against Cubs lineup.`

Mixed reads stayed clear:

- `Signals are split.`

The phrase `Slight lean toward` is safe and direct. It is also repetitive in this projected sample, but that repetition is mostly an artifact of deterministic sanitizer behavior applied to older `Edge toward` artifacts. Fresh post-safety generation is needed before changing the prompt or sanitizer.

Decision: no FastAPI prompt/code change in this slice.

## Summary/rationale cadence findings

The summary text remains useful. It generally explains the signal context in plain language.

Non-defect observations:

- Some MLB summaries still use `edge` as a matchup noun, for example `starting pitching edge`. This is not the forbidden `Edge toward` betting posture, but fresh artifacts should be watched for overuse.
- Some summaries mention ungrounded domains like bullpen or lineup form in warning samples. That remains an internal quality warning issue, not a buyer copy polish issue, because dev diagnostics already flag it.
- The UI note under the lean was the strongest buyer-facing cadence problem because it sounded like product plumbing.

Change made: `Structured lean from the current read.` -> `Current read from the available evidence.`

## Signal Summary wording findings

Decision: keep `Confirmation strength - Measured` for now.

Why:

- It explains confidence posture without naming `sharp_public`, missing source state, fallback status, or probe status.
- It appears only when internal gap rows exist.
- It sits beside grounded rows such as Market / Spread and Rest & Schedule, so it does not flatten the whole signal picture.
- Alternatives such as `Confidence posture`, `Evidence strength`, `Read strength`, and `Support level` were reviewed, but none is clearly better in the current UI.

The supporting phrases are safe:

- `Available evidence is not strong enough for a firmer stance`
- `Keeps confidence measured`
- `Stronger independent confirmation would firm up the read`

They are repetitive across the three NBA measured-confirmation samples, but the repetition is acceptable until fresh post-safety runs prove it is a real buyer cadence problem.

## Repetition and boilerplate findings

Material:

- `Structured lean from the current read` was static and appeared on every sample. Fixed.

Watchlist:

- `Slight lean toward` may become repetitive in fresh generated artifacts.
- `Confirmation strength - Measured` may become repetitive if many buyer runs have the same missing confirmation profile.
- `Available evidence is not strong enough for a firmer stance` is accurate but long.

No phrase bank was implemented because the sample is projected from historical artifacts, not fresh post-safety model output.

## Buyer usefulness findings

Pass.

The buyer can still answer:

1. what the artifact leans toward
2. whether the read is directional, mixed, or measured
3. why confidence is restrained
4. which grounded signals support the read

The artifact remains careful without becoming evasive.

## Safety regression check

Pass.

Projected buyer copy contained none of:

- `edge toward`
- `best bet`
- `lock`
- `prediction`
- `expected to cover`
- `should win`
- `sharp_public`
- `missing signal`
- `unavailable signal`
- `source unavailable`
- `fallback failed`
- `probe required`
- `confidence capped`
- `not available in this run`
- `Sharp vs Public unavailable`

Dev/internal diagnostics remain explicit in the artifact fields and dev review surfaces. No diagnostic data was removed.

## Recommended changes

Made now:

- Replace the current lean note with plainer buyer copy: `Current read from the available evidence.`

Not made now:

- No prompt phrase bank.
- No sanitizer variation.
- No Signal Summary label variants.
- No source expansion.
- No runtime or confidence logic change.

## Changes made

`dai`:

- `apps/sports-app/src/app/analyzer/analyzer.component.ts` - changed the buyer-facing lean note from `Structured lean from the current read.` to `Current read from the available evidence.`

`dai-vault`:

- this review note
- deferred ledger entry 21 resolved as a completed polish review
- current-slice handoff updated

## Deferred follow-ups

No new ledger entry was added.

Follow-up watchlist:

- Run fresh post-safety artifact generation and review whether `Slight lean toward` repeats too often.
- Revisit `Confirmation strength - Measured` only if fresh runs show it feels mechanical.
- Calibrate NFL buyer artifacts when cheap NFL samples exist.

Entry 19 remains deferred. This slice does not reopen Sharp/Public source expansion.

## Next recommended slice

**Fresh Buyer Artifact Generation Calibration v1** only when the local stack and model calls are worth spending. The goal would be to generate fresh post-safety artifacts and confirm whether the prompt naturally varies lean phrases.

If generation is not the next priority, return to product hardening with no source/runtime expansion unless a buyer-visible defect appears.

## Verification

- `npm test -- --watch=false`: 30 passed.
- `npm run build`: passed.
- final repo checks recorded in `06 Execution/handoffs/current-slice.md`.
