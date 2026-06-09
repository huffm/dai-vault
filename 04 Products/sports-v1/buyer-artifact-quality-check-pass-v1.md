# Buyer Artifact Quality Check Pass v1

**date:** 2026-06-09
**status:** quality pass. docs only. no prompt change, parser change, Angular change, .NET change, source addition, probe refresh activation, schema change, confidence/posture/lean mutation engine, tenant/auth/billing work, dashboard work, or deployment work.
**scope:** inspect the sports buyer artifact as a packaged product unit after buyer-copy safety and polish work.

## Purpose

DAI is a decision artifact factory. This slice inspected whether the sports assembly line is producing buyer-credible packaged artifacts after the recent buyer-copy safety and polish loop.

Primary question:

Does the current buyer artifact sound credible, non-repetitive enough, appropriately cautious, and buyer-useful without becoming a tout sheet, prediction engine, debug report, or vague corporate summary?

## Scope

Reviewed:

- buyer-facing lean/read language
- confidence posture
- Signal Summary wording
- artifact completeness as a packaged buyer unit
- repeated phrases
- safety/toutiness
- internal diagnostic visibility

## Non-goals

Not changed:

- FastAPI prompts
- parser sanitation
- .NET runtime logic
- confidence, posture, lean, or decision logic
- sources or Sharp/Public expansion
- probe refresh activation
- schemas
- tenant/auth/billing/Stripe
- dashboards or deployment
- Jera
- phrase bank

## Fresh generation status

Fresh artifacts were not generated.

The local calibration harness exists, but fresh generation requires the sports dev stack and model calls. During this pass:

- platform API on `localhost:5007` did not respond within the timeout
- FastAPI agent service on `127.0.0.1:8000` refused connection

Because the stack was not available, this report uses the latest committed artifact samples and projects them through the current buyer safety/polish boundary. This is a valid quality pass over available packaged artifacts, but it is not a fresh post-safety generation batch.

## Sample set reviewed

Ten committed artifacts were reviewed. The sample includes NBA, MLB, legacy/coarse artifacts, rich v2/v3 `signalAvailability` artifacts, clear leans, mixed reads, measured-confirmation cases, and weak/thin-signal cases.

| # | league | matchup / focus | artifact | shape | projected buyer lean | buyer rows | quality note |
|---|---|---|---|---|---|---|---|
| 1 | NBA | Cavaliers at Pistons | `calibration/artifacts/20260507-1001-nba-ccbc433e.json` | legacy/coarse | Slight lean toward Detroit Pistons based on market favoring them at home. | Rest & Schedule; Market / Spread; Confirmation strength | buyer credible |
| 2 | NBA | Cavaliers at Thunder | `calibration/artifacts/20260507-1001-nba-cdbc433e.json` | legacy/coarse | Slight lean toward Oklahoma City Thunder based on significant market favor. | Rest & Schedule; Market / Spread; Confirmation strength | buyer credible |
| 3 | NBA | Cavaliers at Knicks | `calibration/artifacts/20260518-1001-nba-5f16433e.json` | rich v2 | Slight lean toward New York Knicks based on significant rest advantage. | Rest & Schedule; Market / Spread; Confirmation strength | buyer credible |
| 4 | NBA | Knicks at Thunder | `calibration/artifacts/20260528-0931-nba-fa8d433e.json` | rich v2 | Slight lean toward Oklahoma City Thunder based on home court advantage and rest differential. | Rest & Schedule; Market / Spread; Confirmation strength | buyer credible |
| 5 | NBA | Spurs sample | `calibration/artifacts/20260528-0931-nba-f48d433e.json` | rich v2 | Slight lean toward San Antonio Spurs based on market favoring them by 3.5 points. | Rest & Schedule; Market / Spread; Confirmation strength | buyer credible |
| 6 | MLB | Braves at Marlins | `calibration/artifacts/20260518-1010-mlb-6416433e.json` | rich v2 | Slight lean toward Braves based on starting pitching advantage. | Starting Pitching | credible but thin/high-confidence watch |
| 7 | MLB | Orioles at Rays mixed read | `calibration/artifacts/20260518-1010-mlb-6816433e.json` | rich v2 | Signals are split. | Starting Pitching | buyer usable; internal warnings remain |
| 8 | MLB | Twins at Pirates mixed read | `calibration/artifacts/20260529-1421-mlb-098e433e.json` | rich v3 | Signals are split. | Starting Pitching | buyer usable; internal warnings remain |
| 9 | MLB | Padres at Nationals | `calibration/artifacts/20260529-1421-mlb-0c8e433e.json` | rich v3 | Slight lean toward Padres based on starting pitching advantage. | Starting Pitching | credible but thin/high-confidence watch |
| 10 | MLB | Cubs at Cardinals | `calibration/artifacts/20260529-1421-mlb-1c8e433e.json` | rich v3 | Slight lean toward Cardinals based on starting pitching advantage with Pallante's handedness against Cubs lineup. | Starting Pitching | credible but thin/high-confidence watch |

Note: raw historical artifact JSON still contains old pre-safety `Edge toward` text in 8 of 10 samples. The buyer-facing quality judgment uses the current safety boundary projection, because historical artifacts were not rewritten.

## Method

For each sample:

- reviewed projected buyer lean
- reviewed confidence band and posture
- reviewed buyer Signal Summary rows
- scanned projected buyer copy for unsafe betting/tout/internal terms
- checked repeated phrase counts
- checked artifact quality warnings
- checked whether dev/internal diagnostics remain explicit

Unsafe buyer terms checked included:

- `edge toward`
- `strong edge`
- `betting edge`
- `best bet`
- `lock`
- `sharp side`
- `value play`
- `prediction`
- `expected to cover`
- `should win`
- `play X`
- `take X`
- `hammer X`
- `free money`
- `sharp_public`
- `missing signal`
- `unavailable signal`
- `source unavailable`
- `fallback failed`
- `probe required`
- `confidence capped`
- `not available in this run`
- `Sharp vs Public unavailable`

## Summary judgment

**Pass with minor follow-up.**

The packaged buyer artifact is credible enough to keep hardening toward paid validation. It reads like decision support, not a tout sheet. It keeps direction visible, avoids unsafe buyer language, and hides internal factory/source gaps from the buyer while preserving diagnostics internally.

The remaining quality concerns are not prompt emergencies:

- fresh post-safety generation still needs to be tested when the local stack is available
- three MLB clear-lean samples are thin/high-confidence watches because they rely on one signal while confidence is high
- the static buyer note `Current read from the available evidence` is repeated by design and is acceptable product consistency, not harmful sameness
- `Confirmation strength - Measured` remains acceptable; it appears in measured-confirmation NBA cases and does not flatten the grounded rows

No prompt phrase bank is warranted from this pass.

## Credibility findings

Pass.

The artifact sounds like a serious decision-support product:

- Direction appears in plain language: `Slight lean toward...`
- Mixed cases remain clear: `Signals are split.`
- Buyer Signal Summary shows what supported the read before deeper factors.
- Measured confirmation language explains caution without exposing source failure.
- No salesy or pressure language appears in projected buyer copy.

The strongest buyer examples:

- `Slight lean toward New York Knicks based on significant rest advantage.`
- `Slight lean toward Oklahoma City Thunder based on home court advantage and rest differential.`
- `Slight lean toward Cardinals based on starting pitching advantage with Pallante's handedness against Cubs lineup.`

The artifact is careful, but not evasive.

## Repetition findings

Phrase counts in the 10-sample projected buyer package:

| phrase | count | judgment |
|---|---:|---|
| `Slight lean toward` | 8/10 | expected because 8 raw historical leans began with `Edge toward`; not enough to justify phrase-bank work |
| `Signals are split` | 2/10 | useful consistency for mixed reads |
| `Current read from the available evidence` | 10/10 | static UI note; acceptable product consistency |
| `Confirmation strength` | 5/10 | repeated only where measured-confirmation gap exists; acceptable |
| `Measured` | 5/10 | acceptable confidence posture language |
| `Available evidence` | 10/10 | mostly from static note plus confirmation language; watch fresh runs but not harmful |
| `market confidence` | 3/10 | acceptable in NBA market-supported reads |
| `starting pitching advantage` | 3/10 | acceptable in MLB one-signal reads, but watch for model laziness in fresh generation |

Controlled consistency is present. Harmful sameness is not proven because this is a projection over historical artifacts. Do not create a phrase bank yet.

## Safety / toutiness findings

Pass.

Projected buyer copy had 0 unsafe hits across the 10 samples.

No projected buyer copy contained:

- lock/guarantee/free-money language
- pressure language
- betting-adjacent `Edge toward` posture
- raw `sharp_public`
- missing source/fallback/probe language
- internal factory terms

The buyer posture remains appropriately restrained.

## Signal explanation quality

Pass with watch items.

What works:

- NBA artifacts show Market / Spread, Rest & Schedule, and Confirmation strength.
- MLB artifacts show Starting Pitching and do not invent unavailable extra context.
- The buyer can understand why the read exists without seeing raw diagnostic fields.
- Internal warnings remain internal.

Watch items:

- MLB single-signal artifacts can look too thin for `High` confidence, even when the buyer copy itself is safe.
- Warning samples still have internal `signals_used` quality warnings for ungrounded domains. These stay out of buyer copy, but they remain useful diagnostic evidence.

This does not require runtime mutation work in this slice. It is a quality-watch item for fresh artifact generation.

## Artifact completeness and packaging

Pass.

The packaged unit is complete enough:

- decision/read appears first
- supporting Signal Summary follows
- uncertainty is visible without overwhelming the buyer
- internal diagnostics are not exposed in buyer copy
- legacy/coarse artifacts remain acceptable through coarse support rows plus Confirmation strength
- rich v2/v3 artifacts render cleanly in the projected buyer package

No dangling-separator or awkward empty-row issue was found in this pass that warranted code changes.

## Specific repeated or risky phrases

Repeated but acceptable:

- `Slight lean toward`
- `Current read from the available evidence`
- `Confirmation strength - Measured`
- `Available evidence is not strong enough for a firmer stance`

Risky watch phrases:

- `starting pitching advantage` in MLB clear-lean samples can sound templated if repeated in fresh runs.
- `market confidence` is acceptable, but should not become fake certainty when confirmation is measured.

No phrase was risky enough to justify prompt changes now.

## Recommended next slice

**Fresh Buyer Artifact Generation Calibration v1**, only when the local sports stack is available and model calls are worth spending.

That slice should generate a small fresh batch after Buyer Copy Safety v1 and Buyer Copy Polish Review v1, then decide whether:

- phrase-bank prompt guidance is needed
- MLB single-signal high-confidence cases need product handling
- NFL artifacts need separate buyer calibration

## Deferred decisions / ledger

No ledger update was needed.

No new runtime, prompt, source, or artifact-contract decision was discovered. Prompt phrase-bank work remains only a recommendation candidate, not a deferred decision. Entry 19 remains deferred; this pass does not reopen Sharp/Public source expansion.

## What was not changed

- no code
- no prompt
- no parser
- no .NET
- no source expansion
- no probe refresh
- no schema
- no confidence/posture/lean mutation
- no tenant/auth/billing/Stripe
- no dashboard/deployment work
- no Jera work
- no phrase bank

## Verification plan

Docs-only checks:

- `git diff --check`
- added-line exact-path scan
- vault non-ASCII added-line scan

No npm, FastAPI, or .NET tests are required because no code changed.
