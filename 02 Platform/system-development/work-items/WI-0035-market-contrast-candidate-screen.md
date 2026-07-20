---
title: "WI-0035 Market-Contrast Candidate Screen v1"
type: "work-item"
date: "2026-07-19"
status: "in-progress"
project: "DAI"
slice: "WI-0035 Slice 1: offline deterministic classifier core + planner-envelope projection"
repos:
  dai: "code+tests (C# classifier + market-math reuse + python consumer compatibility)"
  dai-vault: "docs (this WI, architecture record, closeout, MOC, handoff)"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md"
  - "06 Execution/patterns/cohort-selection-and-run-discipline-v1.md"
---

# wi-0035 market-contrast candidate screen v1

## problem

The corrected 2.x planner refuses market-objective cohorts without grounded
`input.market_contrast_screen` evidence, and no producer of that evidence exists (verified
during the WI-0034 Slice-2 review: platform market machinery is run-scoped, not a
pre-generation candidate classifier). This WI builds the niche/domain producer. It is a
separate parent WI (not a WI-0034 slice) because it owns a different authority: screen
policy and market-fact classification, versus the planner's evidence-orchestration
decisions -- a producer/consumer pair, independently reviewable and versioned.

## objective

Classify pre-generation MLB candidates by expected discrimination-evidence value
(identity-safe, source-ready, market-grounded, balanced, timely, nonduplicative). The
screen never predicts DAI-vs-market disagreement (`unknown_until_generation`) and its
result authorizes nothing.

## operating loop (two-pass)

canonical calibration verdict -> planner pass 1 -> missing
`input.market_contrast_screen` -> operator separately authorizes bounded source screening
-> source adapter retrieves normalized market/readiness facts -> offline market-contrast
classifier emits typed evidence envelopes -> planner pass 2 consumes them -> proposed
primary/reserve cohort for operator review -> separate capture authorization -> generation
-> settlement -> calibration feedback. Neither planner pass authorizes screening or
capture; the classifier core performs no source access.

## acceptance criteria (slice 1)

- deterministic pure C# classifier with versioned policy profile
  `market-contrast-screen/1.0` (thresholds only in the policy class; source =
  cohort-selection-and-run-discipline-v1)
- blocker vs exclusion vs includable primary/secondary distinguished with stable reasons
- market math reused, never re-implemented (single de-vig authority; canonical h2h count)
- canonical planner-envelope projection (`input-evidence-envelope/1.1`) with no authority
- planner consumer preserves tier + priority through eligibility and ordering
- cross-language vectors identical in both suites; full suites green

## test plan / verification commands

- `dotnet test platform/dotnet/DevCore.Api.Tests --no-restore` (full; targeted filters
  MarketContrastScreenTests / MarketDepthTests / PromptTraceServiceTests)
- `./.venv/Scripts/python.exe -m pytest` in services/agent-service (full; targeted planner
  + cli suites)
- cross-process canonical determinism snippet (sha-256 equality, two fresh processes)

## decisions

- **language/authority:** C# beside the existing market authorities (MarketDepth,
  SourceReadinessClassifier, prompt-trace de-vig). the python planner is the consumer,
  never the owner of market normalization. verified live before code.
- **de-vig single authority:** the private prompt-trace pairwise de-vig was extracted to
  `MarketDepth.DevigPair`; prompt-trace now delegates (behavior-equivalent; suite green).
  the classifier calls the same helper; de-vig values are not classifier inputs, so
  contradictory caller-supplied de-vig is unrepresentable.
- **h2h book count:** `MarketDepthSummary.BookCount` (any-usable-field) is NOT redefined;
  additive `H2hBookCount` counts distinct bookmakers quoting a two-sided h2h pair
  (spread-only and one-sided books never count; duplicate names deduped). additive
  optional field -> existing constructors unaffected; contract consequence recorded in the
  architecture record.
- **policy boundaries:** gap compared after rounding to 12 decimals so binary
  representation dust cannot flip an exact boundary (0.55/0.45 is exactly the 0.10
  primary boundary).
- **freshness:** slice 1 consumes an explicit normalized market observation status
  ("evaluated" = evaluated AND fresh per the producing feed); no invented duration
  threshold -- the live adapter owns timestamp-to-freshness policy later.
- **primary/secondary consumer compatibility:** envelope normalized result extended
  (classification + screen_tier + priority_components) as `input-evidence-envelope/1.1`;
  request/board/planner/cli bumped to 2.1; no silent extension of closed 1.0/2.0
  contracts. planner ranks (tier, known-disagreement-first, greater-first, start,
  provider, id) and exposes candidate screen tiers; it consumes tier facts and never
  re-derives screen thresholds.
- **wi-0031 seam:** niche policy stays out of generic capability-selection types; no
  runtime dependency in either direction.

## decomposition

- **Slice 1 (this slice, delivered): offline deterministic classifier core** -- policy
  profile, normalized facts, classification, reasons, priority, envelope projection,
  consumer compatibility, cross-language vectors. local commits only.
- **Slice 2 (delivered 2026-07-20, local commits only): bounded default-off source
  adapter** -- process-gated batch adapter on branch `wi/0035-market-contrast-source-adapter`
  (dai `a05b63b`): ONE odds request per slate (h2h,spreads x us = 2 credits, zero
  retries, usage headers audited, key never recorded), ONE statsapi schedule/hydrate
  request (new batch path preserving generation semantics; pitcher reads deduped),
  fail-closed cross-provider identity join (teams-in-orientation AND start instant),
  freshness policy market-contrast-source/1.0 (5-minute quote age; fresh paired
  population only), one tenant-scoped batch active-run read, canonical screen bundle
  market-contrast-screen-bundle/1.0 with atomic exclusive-create publication. 17 fixture
  tests; no live call occurred. NOT pushed / NOT merged.
- **Slice 3 (complete 2026-07-20): one governed live screening + planner replay** --
  executed for MLB 2026-07-22 under a dedicated operator authorization (see
  [[market-contrast-live-screen-2026-07-22-v1]]): ONE Odds `/odds` request (tenant 1, us,
  h2h,spreads, `x-requests-last="0"`, zero retries); truthful terminal bundle
  `market-contrast-screen-bundle/1.1` with NO eligible cohort (all candidates blocked
  `source_unavailable` / `market_not_evaluated`); deterministic pass-2 replay (run twice,
  byte-identical) -> `EVIDENCE_NEEDED_INPUT_TYPES_NOT_ADDRESSABLE`, empty pools. Every
  authority boolean false; zero db/application writes; zero generation/capture; no code
  changed (live execution of the integrated Slice-2 surface). r5 correction: proven only
  that no returned Odds event exact-joined any screenable candidate on both team refs AND
  the exact start instant; the returned event count and precise no-match cause were not
  persisted, so "returned no priced events" is withdrawn and no production defect is
  declared. The next paid attempt must be preceded by a free cross-provider
  identity/readiness gate (see [[market-contrast-live-screen-2026-07-22-v1]]).
- **Slice 4 (deferred, justify first): operating integration** -- only after one reviewed
  live screen; no autonomous scheduling or capture.

## risks

Vector drift between suites (mitigated by the update-together rule + byte-equality tests
on both sides); policy-threshold scatter (bounded by the single policy class); future
adapter re-implementing market math (bounded by the recorded reuse decisions).

## links  <!-- LITE -->

- work item: WI-0035 (ADO: AB#- when wired; no ADO item created)
- branch: `wi/0035-market-contrast-candidate-screen` (dai + dai-vault, matching, from
  9147549 / 83a055f; local only, not pushed / not merged)
- pr: - (not pushed / not merged this slice)
- commits: dai `a6e213b` (WI: WI-0035); vault commit recorded in the closeout
- tests: `platform/dotnet/DevCore.Api.Tests/AgentRuns/MarketContrastScreenTests.cs`;
  planner/cli suites updated (tier + vectors)
- verification notes: Slice-1 closeout + current-slice handoff
- docs updated: this WI; market-contrast architecture record; orchestrator record
  (two-pass loop); WI-0034 links; MOC; closeout; current-slice
- lessons: producers own thresholds and emit envelopes; consumers rank on producer facts;
  extract shared math before writing a second copy

## final handoff requirements

Slice handoff appended to `06 Execution/handoffs/current-slice.md`; status stays
`in-progress` (Slices 2-4 deferred); disposition: Slice 1 implementation complete / merge
ready / not integrated; next governed action = independent review + integration of the
market-contrast offline core before any source adapter or paid screen.
