---
title: "Game-Status Recheck Discipline v1"
type: "pattern"
date: "2026-07-26"
status: "ACTIVE"
project: "DAI"
slice: "WI-0037 Slice 2-i"
repos:
  dai: "consumes scripts/dev/sports/check-game-status.ps1 and the bracketed finals guard"
  dai-vault: "this doctrine"
tags:
  - evidence-operations
  - sports-v1
  - correctness
  - pattern
related:
  - "06 Execution/patterns/settlement-readiness-discipline-v1.md"
  - "02 Platform/system-development/work-items/WI-0037-game-state-correctness-v1.md"
  - "06 Execution/reports/daily-evidence-late-slate-reevaluation-2026-07-23-v1.md"
---

# game-status recheck discipline v1

## the rule

> Resolve the frozen Eastern date bracket first, then the exact gamePk within that
> bracket. Never select the first positional date bucket, never flatten all date
> buckets before authority resolution, and never treat an out-of-bracket reschedule
> record as an in-bracket duplicate.

This codifies (and supersedes as doctrine) the prose-only corrective rule recorded in
the 2026-07-23 late-slate reevaluation after the false-postponement misread: an ad-hoc
bare-gamePk StatsAPI query read the wrong date bucket for a postponed-original + makeup
pair and blocked a legitimate capture. The historical reports stand unaltered; this
pattern is the durable, executable form.

## supported operator command

`pwsh <DAI_REPO_ROOT>/scripts/dev/sports/check-game-status.ps1 -Competition mlb
-BracketDate <YYYY-MM-DD> -GamePk <pk> [-ExpectedAway <name> -ExpectedHome <name>]
[-JsonOnly] [-OutputPath <file>] [-ScheduleJsonPath <fixture>]`

Required inputs: the competition, the FROZEN Eastern bracket date (never inferred from
the system clock or the response), and the exact gamePk. Optional expected identity
adds a cross-check that refuses `identity_mismatch`. `-ScheduleJsonPath` replays a
stored payload offline with identical semantics (evidence preservation / test seam).
Machine output is `game-status-resolution/1.0` JSON on stdout with `-JsonOnly`;
diagnostics never contaminate it.

## typed refusals (closed vocabulary)

`bracket_missing` (requested bucket absent) | `game_not_in_bracket` |
`duplicate_in_bracket` (true identity hazard) | `bucket_malformed` |
`status_malformed` | `identity_mismatch`. A refusal exits 1 and is never success;
transport/parse failure exits 2 and is never a refusal; usage errors exit 3;
resolution exits 0.

## makeup, doubleheader, and duplicate behavior

- A gamePk appearing under BOTH a historical bucket (postponed original) and the
  bracket bucket (makeup) resolves to the in-bracket record; the other appearances are
  returned as `rescheduleContext` (bucket date, state, start) -- evidence, never
  authority, never a duplicate.
- A legitimate same-date doubleheader has DISTINCT gamePks; each resolves
  independently. Two records with the SAME gamePk inside one bracket are a true
  identity hazard: `duplicate_in_bracket`, fail closed.

## finals-guard invocation

`check-settlement-finals.ps1` accepts `-BracketDate <YYYY-MM-DD>`; supply it whenever a
target gamePk could have a reschedule history (postponed/makeup class). With the
bracket, the guard resolves authority within the bracket and reports
`rescheduleContextCount`; without it, single-bucket responses keep identical
classification, exit codes, and shared-field values (the two per-game fields above are
additive in all outputs)
and a multi-bucket gamePk is a DEFECT whose reason instructs the operator to supply
`-BracketDate` (fail closed, never a silent guess). Finality remains
`abstractGameState=Final` AND `codedGameState=F` per
settlement-readiness-discipline-v1; that doc's duplicate-DEFECT rule now means
IN-BRACKET duplication.

## evidence expectations

Preserve the raw payload (`-OutputPath`, or the fetched JSON) with the bracket date and
gamePk in the record whenever a status recheck feeds an operational decision; offline
replay of that payload through the same script must reproduce the decision.

## conformance

Contract and corpus: `<DAI_REPO_ROOT>/scripts/dev/sports/game-status-resolution-contract-v1.md`
and `scripts/dev/sports/fixtures/game-status-resolution-v1.json` (24 vectors; the
PowerShell harnesses consume them now; the C# Slice 2-ii runner consumes the SAME file
so the runtimes cannot drift).

## slice 2-ii-c hardening (2026-07-27)

- Live query is authority plus context: `check-game-status.ps1` fetches broadly by exact
  gamePk (`schedule?sportId=1&gamePks=<pk>`, no `date=` in the transport query) so a
  postponed original plus its makeup are both returned; the frozen `-BracketDate` remains
  the sole local bracket-selection authority. One GET, 30-second timeout, offline
  `-ScheduleJsonPath` unchanged, fail-closed exit codes unchanged.
- Normalization has one C# authority: `ScheduleStateNormalizer` (consumed by both
  `GameStatusResolver` and `MarketContrastSourceAdapter`); the PowerShell operator runner
  keeps its own `ConvertTo-NormalizedStatus` as a separate runtime implementation by design.
  Cross-runtime parity is pinned by the corpus normalization vectors, not by shared code.
- The corpus now carries a top-level `normalizationVectors` collection consumed generically
  by both runners (no-skip proven). The 25 scenario fixtures and six refusal reasons are
  unchanged; the contract stays `game-status-resolution/1.1`.
- The detail-status decision is a mandatory typed `GameStatusDetailRequirement`
  (Required / NotRequired) at every resolver call site; no defaulted boolean remains.
