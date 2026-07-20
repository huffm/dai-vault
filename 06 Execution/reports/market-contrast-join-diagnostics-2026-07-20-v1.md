---
title: "Market-Contrast Join Diagnostics Hardening v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 join-diagnostics hardening (offline; local commits only)"
repos:
  dai: "code+tests (join diagnostics; branch wi/0035-market-contrast-join-diagnostics)"
  dai-vault: "docs-only"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "06 Execution/reports/market-contrast-live-screen-2026-07-22-v1.md"
---

# market-contrast join diagnostics hardening v1

## purpose

The first live screen reported eleven `no_market_match` candidates truthfully but without
enough detail to tell apart: no events returned; events for different teams; correct teams
at a different start; reversed home/away; multiple matches; or exactly one match. This
slice makes those cases distinguishable in future bundles WITHOUT changing which events
count as matches and without making matching more permissive. Offline implementation and
testing only; no source calls, no spend; local commits only (dai `0aae858`).

## limitation reproduced first

Before the production change, tests establish at the helper boundary that five
meaningfully different situations (empty response; unrelated teams; correct teams wrong
start; reversed teams; duplicate exact matches) were previously indistinguishable at the
production-visible verdict, and are now five distinct closed statuses.

## what changed (dai `0aae858`)

- `platform/dotnet/DevCore.Api/AgentRuns/MarketJoinDiagnostics.cs` (new): a pure
  deterministic helper computing, per candidate, the closed diagnostic status, the
  same-/reversed-orientation counts, the exact-match count, the signed nearest start delta
  among same-orientation team pairs, and the odds event id (only when exactly one exact
  match). The exact-match set is the identical predicate the adapter already used.
- `MarketContrastSourceAdapter.cs`: the join decision now derives from the helper's
  exact-match set (behaviorally identical); the bundle
  (`market-contrast-screen-bundle/1.2`, adapter `market-contrast-source/1.2`) gains, under
  `source_calls.odds_api`, `events_received`, `evaluated_candidate_exact_match_count`,
  `response_parsed`, and a per-candidate `market_join_diagnostic` block.
- `daily_evidence_planner_cli.py`: CLI 2.4 accepts the additive 1.2 bundle in the replay
  seam and keeps 1.1 supported.
- tests: `MarketJoinDiagnosticsTests.cs` (helper + all 21 required cases + the limitation
  reproduction + determinism/authority invariants + a fixed-seed 600-fixture
  predicate-equivalence corpus), adapter integration tests (bundle carries diagnostics;
  response-ledger zero-vs-unparsed; evaluated-count reconciliation; commence-time boundary
  shapes; start-mismatch vs team-mismatch distinguishable; decision unchanged), and a
  real-shape python replay test (1.1 and 1.2 -- including a mutated diagnostic -- produce
  byte-identical pass-2 request and board).

## r6a review corrections (2026-07-20)

An independent review corrected four semantic/testing defects in a new commit (dai
`5e81b83`; original `0aae858` preserved):

1. **zero really means zero (RQ1):** `events_received` was 0 on every attempted response
   including failures; now it is a count ONLY on a parsed event array (a parsed empty array
   is 0) and `null` otherwise. `response_parsed` is true only on a parsed array. A JSON
   `null` body is `source_failed` (the `/odds` array contract never returns `null`), never
   `no_events_returned`.
2. **truthful total name (RQ2):** `exact_matches_total` -> `evaluated_candidate_exact_match_count`,
   documented as evaluated-candidates-only (preblocked/source-failed/not-attempted never
   contribute) and proven to reconcile with the per-candidate diagnostics; preblocked
   candidates are not inspected to inflate it.
3. **real replay shape + boundary (RQ3):** the equivalence test now uses the production
   shape (`source_calls.odds_api` + per-candidate `market_join`/`market_join_diagnostic`).
   The replay-validation boundary is **option B**: replay verifies version, target date,
   candidate identities, and envelopes only, not the diagnostic section; a mutated
   diagnostic cannot change the pass-2 request or board; the CLI never claims full bundle
   validation (documented in the seam).
4. **matching-unchanged proof + honest start coverage (RQ4/RQ5):** a fixed-seed
   600-fixture corpus proves the helper's exact-match set equals the original inline
   predicate; `commence_time` shapes (null/malformed -> `source_failed`; valid explicit-utc
   -> parsed) are tested at the HTTP/JSON boundary, distinct from a valid large start delta.

## closed status vocabulary and precedence

`duplicate_exact_match` > `matched` > `start_instant_mismatch` > `orientation_mismatch` >
`team_pair_not_found` > `no_events_returned`; context statuses `not_attempted` /
`source_failed` / `skipped_preblocked` are set before market analysis. Meanings are as
documented in the architecture record. A reversed event and any nonzero start delta are
recorded but never count as a match; alias and short-name divergence remain non-matches
(diagnostic-only).

## semantics explicitly unchanged

The diagnostics explain the match attempt only. Market availability, blocking/inclusion,
classification, tier, priority, source readiness, planner eligibility, the source-call
count, and the paid-call lease are all unchanged; every authority boolean stays false. The
first live attempt remains valid; its unrecorded Odds response count cannot be
reconstructed, and no team-matching defect is declared without provider evidence.

## versioning

adapter `market-contrast-source/1.2`; bundle `market-contrast-screen-bundle/1.2`; planner
CLI `daily-evidence-planner-cli/2.4`. Unchanged (no behavior/shape change): classifier
`market-contrast-screen/1.2`, request `2.1`, board `2.2`, planner-core `2.2`, envelope
`input-evidence-envelope/1.1`. The historical 1.1 bundle is NOT silently converted to 1.2
and does not gain diagnostic evidence; it remains explicitly replayable.

## verification

Targeted diagnostics + adapter 49/49; full DevCore.Api.Tests **1394 / 0** (was 1363, +31
after the r6a-review tests); planner + cli 88/88; full agent-service **563 / 0**. Only build
warnings are the pre-existing NU1903 package advisory (0 csproj changes on the branch).
`git diff --check` clean; secret / machine-path / authority scans clean. Historical
attempt-1 1.1 bundle replay proven hash-preserving (pass-2 request sha-256
`32bb0851a5bc69a0406ad3b9dcb0923a84104ced0d3e032811c385d1f27ffefc`, unchanged); the
1.1-vs-1.2 replay equivalence test proves the additive fields never change the pass-2
request or board. Diagnostics helper proven input-order-invariant, authority-free, and
free of non-finite numbers.

## next step

Independent review + integration of the local `wi/0035-market-contrast-join-diagnostics`
branches (both repos). The July 22 free `/events` cross-provider identity/readiness gate
remains a separate, time-gated action; any later paid attempt keeps the one-request,
zero-retry, <=2-credit, all-authority-false contract. A recommendation is not an
authorization.
