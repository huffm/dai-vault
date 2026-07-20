---
title: "Market-Contrast Events Gate (Zero-Quota Cross-Provider /events) v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 3 r7: zero-quota cross-provider events-gate operator (offline; local commits)"
repos:
  dai: "code+tests (DevCore.Api AgentRuns events gate; producer bundle 1.3; planner CLI 2.5; branch wi/0035-market-contrast-events-gate)"
  dai-vault: "docs + current-state reconciliation"
tags:
  - system-development
  - platform
  - market-contrast
  - sports-v1
  - cross-provider-identity
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "06 Execution/reports/market-contrast-live-screen-2026-07-22-v1.md"
  - "06 Execution/reports/market-contrast-join-diagnostics-2026-07-20-v1.md"
---

# market-contrast events gate v1

## purpose

Deliver the missing local-only operator surface for the time-gated 2026-07-22 cross-provider
identity/readiness gate. Offline implementation and testing only: no StatsAPI, no Odds
`/events` or `/odds`, no database, model, Tool Gateway, or application call; fixtures and an
injected http handler only. No push, no merge.

## what was built

A one-shot process command mode (never an http endpoint):

    DevCore.Api market-contrast-events-gate --preflight-bundle <file> --out <file> [--target-date <d>]

It observes the provider's ZERO-QUOTA `/events` endpoint at most once (never `/odds`), and
joins the returned events against the authoritative statsapi identities carried by a freshly
generated market-contrast preflight bundle. It has no StatsAPI, database, model, or
application path of its own; the only accepted flags are the bundle, the output path, and an
optional target date that must exactly equal the bundle's target date. There is no flag that
can widen sport, markets, regions, retries, caps, tenant, eligibility, aliases, tolerances,
authority, or overwrite.

Source files (dai, branch `wi/0035-market-contrast-events-gate`):

- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastEventsGate.cs` (new) -- the operator,
  the `market-contrast-events-gate/1.0` artifact renderer, the strict-json duplicate-key/
  non-finite parser, and the `PreflightBundleAdmission` validator.
- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastEventsGateCommand.cs` (new) -- the
  local-only one-shot command mode.
- `platform/dotnet/DevCore.Api/Program.cs` -- dispatch before the web host.
- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastSourceAdapter.cs` -- producer parity:
  bundle `market-contrast-screen-bundle/1.3` / adapter `market-contrast-source/1.3` add a
  per-candidate `authoritative_identity` (normalized home/away refs, exact scheduled utc,
  schedule state) sourced from the SAME authoritative statsapi facts the adapter already
  computes; additive, observation-only, no matching/classification/authority change.
- `services/agent-service/app/services/daily_evidence_planner_cli.py` -- CLI 2.5 accepts
  `market-contrast-screen-bundle/1.3` (1.1/1.2/1.3 replay-identical; authoritative_identity is
  replay-inert, verified by a new equivalence test).

## source reuse (no second implementations)

The gate reuses the integrated authorities only: `OddsMarketClient.EasternDayBracket`
(dst-aware date window), the existing strict `OddsApiEvent` `/events` DTO (projected to
`OddsApiOddsEvent` with empty bookmakers to feed the shared diagnostic),
`GameIdentityDerivation.NormalizeTeamRef`, `MarketJoinDiagnostics` (the exact integrated
predicate verbatim), and `ScreenBundlePublisher` (claim-based no-overwrite admission,
writer-unique staging, atomic replace, preserved recovery artifact). It does NOT use the
lossy `MatchupEventDto` projection, and introduces no second client stack, identity
normalizer, date-bracketing, or atomic publisher.

## exact-match authority (unchanged)

An event matches only when: the normalized provider home ref equals the authoritative home
ref, the normalized away ref equals the authoritative away ref, orientation is identical, the
utc commence time equals the authoritative scheduled utc exactly, and exactly one event
satisfies all predicates. Aliases, shortened names, reversed orientation, and any nonzero
start delta are recorded for humans and never count as a match. A fixed-seed adversarial
corpus (40 iterations) proves the gate's exact-match set is byte-equivalent to a direct
`MarketJoinDiagnostics.Analyze` over the same events.

## artifact, quota, and secret contract

Artifact `market-contrast-events-gate/1.0` (sole authoritative output, canonical json):
schema/operator versions, unique attempt id, target date, input preflight-bundle sha-256,
pass-1 request sha-256, observation start/completion, requested utc bracket, redacted request
description, http attempt count, zero retries, response status + `response_parsed`,
`events_received` (null on transport/http/parse failure, 0 only for a parsed empty array),
exact received quota headers, zero-quota audit result, normalized provider events (event id,
home, away, explicit utc commence), every admitted candidate with its `MarketJoinDiagnostics`
result, evaluated-candidate exact-match count, unique matched provider event id only when
globally unique, same/reversed-orientation counts, nearest signed start delta, an explicit
statement that diagnostic aliases/deltas never changed matching, and an authority ledger of
booleans only -- every one false (including `odds_request_authorized` and
`paid_attempt_authorized`). No api key, no tenant data, no generation/capture recipe, no
planner envelope. The zero-quota audit requires all three usage headers present, finite,
nonnegative, and `x-requests-last` exactly zero; a missing/malformed/negative/non-finite/
nonzero last cost is `quota_audit_failed` and cannot support a paid-attempt proposal.

Terminal statuses (closed set; none authorizes `/odds`):
`exact_match_ready_for_separate_operator_decision`, `no_events_returned`, `no_exact_matches`,
`alias_or_start_mismatch_observed`, `duplicate_exact_match_blocked`, `input_rejected_no_call`,
`source_transport_failed`, `source_http_failed`, `source_parse_failed`, `quota_audit_failed`,
`publication_failed`.

## admission (fail-closed before any call)

The preflight bundle is admitted only when it strict-parses (duplicate-key and non-finite
rejection), is schema `market-contrast-screen-bundle/1.3`, mode `preflight`, terminal status
`completed_preflight_no_paid_call`, records zero odds requests and no attempted odds call, has
a valid target date (equal to `--target-date` when supplied), an all-false authority ledger, a
present pass-1 hash and positive tenant key, unique candidate gamePks, and at least one
resolved candidate whose authoritative identity is complete (home/away refs, explicit-utc
start, schedule state) and non-colliding. A resolved candidate with an incomplete or non-utc
identity is a corrupt producer artifact and rejects the whole bundle. Any rejection, and any
pre-call destination-claim failure, yields `input_rejected_no_call` with zero source calls.
The input bundle sha-256 is preserved in the artifact.

## verification

- Events-gate fixtures **33/33** (exact join; fifteen unique joins; parsed empty; team-pair
  absent; alias/Athletics variants diagnostic-only; reversed orientation; signed start
  deltas; duplicate exact match; malformed/null/non-utc/object bodies; http failure; transport
  timeout; single request + zero retries; no `/odds` reachable; no statsapi/db/model
  dependency; zero-quota accepted; bad quota fails closed over an exact match; old/paid bundle
  rejected; wrong-date rejected; authority-true rejected; candidate/event reorder determinism;
  byte-identical repeat; same-process publication race; occupied-destination no-overwrite;
  forced post-call publication failure preserves recovery; secret sentinel absent; authority
  ledger closed all-false; not a planner envelope; fixed-seed exact-match equivalence corpus).
- Full `DevCore.Api.Tests` **1449 passed / 0 failed** (was 1416).
- Full agent-service pytest **564 passed** (includes the new 1.3 replay-equivalence test).
- Builds: 0 new warnings (only the pre-existing NU1903 package advisory; 0 csproj changes on
  the branch). `git diff --check` clean. Secret/machine-path/authority-true scans clean.
- Determinism: identical injected attempt id + clock -> byte-identical artifact and sha-256;
  candidate/provider-event reordering leaves the diagnosis and its output ordering unchanged.

## external-call ledger

Model calls 0; StatsAPI 0; Odds `/events` 0; Odds `/odds` 0; database reads/writes 0; Tool
Gateway 0; generation/capture/screening/settlement/scheduling 0; push/merge 0. All tests use
an injected http handler and a non-secret sentinel key; no live call occurred anywhere.

## next governed authorization

Independent review + integration of both local events-gate branches; then, only after
integration and no earlier than 2026-07-22T12:00:00Z, one refreshed free preflight plus
exactly one zero-quota `/events` gate observation (no `/odds` request in that authorization). A
paid `/odds` attempt is proposed separately only when a current exact identity/start join
exists. No selection plan, events-gate artifact, or planner board grants execution authority.
