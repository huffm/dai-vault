---
title: "Market-Contrast Events-Gate Fail-Closed Boundary Completion v1 (r7C, 2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 3 r7C: events-gate fail-closed boundary completion + fast-forward integration"
repos:
  dai: "code+tests (DevCore.Api events gate admission; branch wi/0035-events-gate-boundary-completion)"
  dai-vault: "review report addendum + current-state reconciliation"
tags:
  - system-development
  - platform
  - market-contrast
  - cross-provider-identity
  - fail-closed-boundary
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "06 Execution/reports/market-contrast-events-gate-slice-3-2026-07-20-v1.md"
  - "06 Execution/reports/market-contrast-events-gate-review-2026-07-20-v1.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
---

# market-contrast events-gate fail-closed boundary completion (r7C)

## purpose

Correct two remaining fail-closed input-validation defects in the integrated events-gate
operator, reconcile the stale WI/product/orchestrator current-state, bump the operator version,
and fast-forward integrate both mains. Offline only; no StatsAPI, Odds `/events`, Odds `/odds`,
database, model, Tool Gateway, application, generation, capture, execution, settlement, or
scheduling call.

## the two defects (reproduced RED against integrated r7B `c7d4a79`)

- **A -- missing preblock_reason treated as null.** The r7B admission treated a MISSING
  `preblock_reason` property as explicit null (no preblock), so malformed input could yield a
  screenable candidate and reach the source-call boundary; a blank/whitespace preblock string
  was treated as a skip reason rather than rejected. RED fixtures: `rcA3` (missing -> was
  ready/called), `rcA4` (blank/whitespace -> was skipped, not rejected), `rcA7` (missing cannot
  manufacture ready).
- **B -- missing/malformed operation.started_at_utc ignored.** The r7B admission only checked
  `started_at_utc` when it was present and parseable (`completed < started`), so a missing or
  malformed value was ignored. RED fixtures: `rcB_missing`, `rcB_invalid` (malformed /
  timezone-less / nonzero-offset). Nine fixtures failed RED against r7B before the fix.

## corrections (new commits; r7/r7A/r7B preserved as ancestors)

`platform/dotnet/DevCore.Api/AgentRuns/MarketContrastEventsGate.cs`
(`PreflightBundleAdmission.Validate`):

- **preblock_reason:** the producer always emits the field. A MISSING property now rejects
  before any claim or call (`candidate <pk> is missing preblock_reason`); a present explicit
  null is the evaluated "no preblock" (may remain screenable); a present nonblank string is a
  preblock reason (retained, `skipped_preblocked`, zero counts, never ready); a blank/whitespace
  string rejects (`... is a blank string`); a non-string value rejects (unchanged). Missing is
  never treated as null.
- **operation.started_at_utc:** now required present, a string, explicit-utc under the same
  strict textual rule as completion, and `<=` completion; missing / null / malformed /
  timezone-less / nonzero-offset / later-than-completion all reject before any claim or call. The
  observation clock is still captured once; the five-minute freshness boundary is unchanged.
- **version:** operator `market-contrast-events-gate-operator/1.0` -> `1.1`; the artifact schema
  stays `market-contrast-events-gate/1.0` (output shape unchanged). 1.1 tightens malformed-input
  rejection and grants no new capability or authority.

## preserved semantics (unchanged)

Exact-match predicate; normalized identity; orientation; exact-UTC start equality; screenability
requirements; minimum start margin; target Eastern-day bracket; one `/events` request max; zero
retries; quota audit; strict provider-response parsing; destination claim/publication/recovery;
secret handling; replay (1.1/1.2/1.3 inert); bundle schema 1.3; adapter 1.3; planner CLI 2.5;
all-false authority ledgers; the `/odds` prohibition; WI-0031. No controller/endpoint/scheduler/
hosted service/persistence/new switch.

## verification

RED->GREEN: 9 r7C fixtures failed against r7B, all 18 pass after the fix. Events-gate class
**88/88** (70 r7B + 18 r7C). Full DevCore.Api.Tests **1504 passed / 0 failed** (was 1486).
Adjacent market-contrast suites 72/72. Full agent-service pytest **564 passed**; planner replay
1.1/1.2/1.3 inert (4/4). 0 new build warnings (only the pre-existing NU1903 advisory; 0 project/
package changes). `git diff --check`, secret, machine-path, and authority-grant scans clean.
Strict planning snapshot 24 WIs / 0 warnings. Protected/classified drift byte-identical open to
close. `DAI CODE REVIEW`: approve, no blocking findings.

## integration

Both feature branches pushed; local feature tip == remote feature tip; each remote main proven
unchanged from the reviewed base and an ancestor of its reviewed tip; fast-forward-only merges;
mains pushed; re-fetch proved both local mains == origin/main == the reviewed tips. Original
r7/r7A/r7B commits remain ancestors. No non-fast-forward, force push, history rewrite, or
partial integration.

## external-call ledger

model 0; StatsAPI 0; Odds `/events` 0; Odds `/odds` 0; database 0; Tool Gateway 0; generation/
capture/screening/settlement/scheduling 0. Tests use an injected HTTP handler and a non-secret
sentinel key; the live command was not invoked.

## next step

Not a paid attempt. A separately authorized, time-gated action no earlier than
2026-07-22T12:00:00Z: one refreshed free preflight followed by at most one zero-quota `/events`
observation. No `/odds` call is authorized. A later paid proposal requires at least one exact
current identity/start join from a screenable candidate and a separate authorization.
