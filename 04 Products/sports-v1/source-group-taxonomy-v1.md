# Source Group Taxonomy v1

**date:** 2026-06-15
**status:** implemented. narrow backend domain component + read-time projection + tests in `dai`; report/ledger in `dai-vault`. no model spend, no generation, no reconciliation, no outcomes/evaluations written (12/12), no migration, no analyzer/prompt/matcher/confidence/lean/buyer change. no Perceive gate (deferred).
**classification:** taxonomy + derived read-only metadata.

## Problem statement

Artifacts already carry named `groundedSignals`/`missingSignals` and per-signal `signalAvailability`, but there was no source-group vocabulary, no critical/supporting tiers, no canonical server-side sufficiency band, and no typed null-reason. Source insufficiency was structurally observable but low directional separation was only legible in model prose. This slice makes sufficiency legible deterministically, without gating anything.

## Existing signal catalog (audit)

Named signals come from `_KNOWN_SIGNAL_CATEGORIES` (`services/agent-service/app/services/sports_analyzer.py`): market, injury_report, situational, weather, rest_fatigue, matchup_style, home_court, starting_pitching, bullpen, lineup_form, ballpark, sharp_public. `signalAvailability` status values seen: grounded / missing; quality: usable / unavailable; decisionUse: supporting_context / not_usable; confidenceEffect: support_cautiously / neutral. `evidenceRichness` is a persisted count. No DTO carried a sufficiency-like field; the only band logic lived in the Angular buyer projection (now mirrored canonically server-side, read-only).

**Reality that shaped the rules:** MLB runs ground only `starting_pitching` (market/odds is not tracked for MLB). So a rule like "market_odds missing -> insufficient" would wrongly flag every MLB run. The band is therefore based on grounded decision-useful count + the sport-critical group, not on market_odds presence.

## Source groups

`identity_schedule`, `market_odds`, `starting_pitching`, `bullpen_availability`, `lineup_injury`, `team_form`, `rest_travel`, `weather_park`, `market_movement`, `fallback_proxy`. (Some have no current grounded signal mapping; they exist for stable vocabulary as coverage grows.)

## Tiers

- **Critical:** identity_schedule, market_odds, starting_pitching.
- **Supporting:** bullpen_availability, lineup_injury, rest_travel, market_movement.
- **Contextual:** team_form, weather_park, fallback_proxy.

The **sport-critical** group (the one whose grounding most determines a directional read) is per-sport: MLB -> `starting_pitching`; odds-sourced (NBA) -> `market_odds`.

## Known signal -> group / tier mapping

| signal | group | tier |
|---|---|---|
| market | market_odds | Critical |
| starting_pitching | starting_pitching | Critical |
| bullpen | bullpen_availability | Supporting |
| injury_report | lineup_injury | Supporting |
| rest_fatigue | rest_travel | Supporting |
| sharp_public | market_movement | Supporting |
| lineup_form | team_form | Contextual |
| matchup_style | team_form | Contextual |
| situational | team_form | Contextual |
| home_court | team_form | Contextual |
| weather | weather_park | Contextual |
| ballpark | weather_park | Contextual |
| (unknown) | fallback_proxy | Contextual |

`identity_schedule` is sourced from the run row (`SourceProvider`+`ExternalGameId` present), not from a signal.

## Sufficiency band rules (deterministic)

Let `g` = count of grounded decision-useful signals.

- **insufficient:** `g == 0`.
- **thin:** `g == 1`.
- **moderate:** `g >= 2` with identity present and the sport-critical group grounded (but not enough supporting/contextual for rich).
- **rich:** `g >= 2`, identity present, sport-critical grounded, and >= 2 supporting/contextual groups grounded.

These are intentionally simple and revisable. For current buyer-ready MLB (max one grounded signal), runs land in `insufficient` (priors-only) or `thin` (starter grounded) -- honest, matching the "MLB single-signal thin" doctrine.

## Null-reason codes (derived only for a null lean; structural only)

Vocabulary: `SourceInsufficient_MissingCriticalGroup`, `SourceInsufficient_NoGroundedSignals`, `SourceInsufficient_OnlyPriors`, `NoDirectionalSeparation`, `ModelAbstention`, `UnsupportedMarket`, `SourceUnavailable`, `Unknown`.

Derivation (no prose parsing):

- present lean -> `null` (not applicable).
- `g == 0` and the sport-critical group is unavailable -> `SourceInsufficient_MissingCriticalGroup`.
- `g == 0` and no critical unavailable -> `SourceInsufficient_NoGroundedSignals`.
- `g >= 1` and sport-critical unavailable -> `SourceInsufficient_MissingCriticalGroup`.
- `g >= 1` and no critical unavailable, yet null lean -> `NoDirectionalSeparation`.

`ModelAbstention` / `UnsupportedMarket` / `SourceUnavailable` exist in the vocabulary but are not inferred here (they would need prose or a typed model field); `Unknown` is the safe default elsewhere.

## How this explains the prior MLB null/rerun cases (verified live)

Read-time projection on the real runs (no regeneration):

| run | gamePk | leanSide | band | nullReason | matches audit |
|---|---|---|---|---|---|
| 2e03433e (old) | 823046 | null | insufficient | SourceInsufficient_MissingCriticalGroup | yes -- starter unavailable |
| 3603433e (old) | 822887 | null | insufficient | SourceInsufficient_MissingCriticalGroup | yes -- starter unavailable |
| 4203433e (old) | 824993 | null | thin | NoDirectionalSeparation | yes -- grounded, no edge |
| 5003433e (rerun) | 823046 | home | thin | (none) | yes -- starter now grounded |
| 5703433e (rerun) | 824993 | null | thin | NoDirectionalSeparation | yes -- stable abstention |
| 2303433e (usable) | 823452 | home | thin | (none) | yes |

The taxonomy reproduces the Perceive Signal Sufficiency Audit's narrative deterministically and **server-side**: source insufficiency (`MissingCriticalGroup`) is now distinguished from low directional separation (`NoDirectionalSeparation`) without reading prose.

## Code changes

- `platform/dotnet/DevCore.Api/AgentRuns/SourceSignalTaxonomy.cs` (new): `SourceGroups`, `SourceTier`, `SourceSufficiencyBands`, `SourceNullReasons`, `SourceGroupCoverage`, `SourceSufficiency`, `SourceSignalTaxonomy.Classify`/`SportCriticalGroup`, `SourceSufficiencyBuilder.Build` -- all pure.
- `AgentRunContracts.cs`: `AgentRunArtifactDto` += `SourceSufficiency? SourceSufficiency` (read-only).
- `AgentRunsController.cs`: `GetArtifact` derives and projects `SourceSufficiency` from the deserialized artifact + run identity. No persistence, no migration.

## Tests

- `SourceSignalTaxonomyTests.cs` (new, 15): group/tier mapping; unknown/null -> fallback_proxy safely; sport-critical group per sport; zero-grounded -> insufficient; starter grounded -> thin; null + missing critical -> MissingCriticalGroup; zero-grounded no-critical-unavailable -> NoGroundedSignals; null + starter grounded -> NoDirectionalSeparation and not falsely missing; present lean -> no null-reason.
- artifact endpoint test extended: DTO surfaces `SourceSufficiency` (band thin, null reason null for the fixture).
- full `DevCore.Api.Tests`: 680/680.

## Verification

- TDD: pure taxonomy tests written and green (15/15); full suite 680/680 (matcher/identity/reconcile/eligibility unaffected).
- no-spend live smoke on real runs reproduces the table above exactly.
- outcomes/evaluations unchanged at 12/12; no model calls; no migration.

## What this enables next

- **Perceive Sufficiency Gate Contract v1**: the band + critical-group coverage + null-reason are now the inputs a gate needs (block if identity_schedule missing; downgrade if sport-critical missing; allow thin if a critical group + supporting coverage exist). This slice does not gate.
- **Probe Fallback Catalog v1**: `fallback_proxy` group + per-group coverage give a place to attach acceptable proxies and fallback attempts.
- **phase-level contribution scoring** later: group coverage is the substrate for attributing confidence/lean to source groups.

## Deferred

- live fallback execution / Probe activation; model/prompt changes; calibration threshold changes; buyer-facing source display; WNBA support; persisting the band (kept derive-on-read like ProtocolView); promoting/retiring the frontend band derivation (a later consolidation, not taken here); the non-structural null-reason codes (ModelAbstention/UnsupportedMarket/SourceUnavailable) until a typed source exists.

## Recommended next slice

**Perceive Sufficiency Gate Contract v1** (contract/spec, possibly dormant code) -- define how the now-legible band/critical-coverage/null-reason would gate or downgrade Perceive, without activating it. In parallel, reconciliation of the 9 active usable MLB runs after settlement remains the calibration-evidence priority and is independent of this work. WNBA remains deferred.
