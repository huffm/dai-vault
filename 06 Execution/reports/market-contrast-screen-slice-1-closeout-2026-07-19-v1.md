---
title: "Market-Contrast Screen Slice 1 Closeout v1 (2026-07-19)"
type: "evidence-report"
date: "2026-07-19"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 1: offline deterministic market-contrast classifier core"
repos:
  dai: "code+tests (C# classifier + market-math reuse + python consumer; branch wi/0035-market-contrast-candidate-screen)"
  dai-vault: "docs-only"
tags:
  - system-development
  - evidence-operations
  - sports-v1
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
  - "06 Execution/reports/daily-evidence-planner-cli-slice-2-closeout-2026-07-19-v1.md"
---

# market-contrast screen slice 1 closeout v1

## purpose

Record WI-0035 Slice 1: the offline deterministic market-contrast classifier core, its
versioned policy profile, its typed planner-envelope projection, and cross-language
compatibility -- committed locally on matching governed branches. NOT pushed / NOT merged.

## context and allocation

Executed after the WI-0034 Slice-2 contract-integrity integration (dai main `9147549`,
vault main `83a055f`, both == origin, re-verified live). WI-0035 allocated read-only (files
through WI-0034; reserved band respected; zero MOC/filename/branch collisions). Matching
branches `wi/0035-market-contrast-candidate-screen` created from those exact mains before
the first write. Snapshot 24 WIs / 0 warnings at close; Obsidian closed; drift = the known
classified set, byte-identical.

## what shipped (dai `a6e213b`, WI: WI-0035)

- `platform/dotnet/DevCore.Api/AgentRuns/MarketContrastScreen.cs` (new): policy profile
  `market-contrast-screen/1.0` (single policy class; source = cohort-selection doctrine);
  closed immutable normalized facts (identity pre-resolved; de-vig intentionally NOT an
  input -- derived via the single authority so contradictions are unrepresentable);
  blocker-vs-exclusion-vs-includable primary/secondary; 17+ stable reason codes;
  deterministic priority (known disagreement before unknown, greater first); canonical
  envelope projection with non-grounding blocker status mapping; all-false authority
  posture; `unknown_until_generation` on every result.
- `platform/dotnet/DevCore.Api.Tests/AgentRuns/MarketContrastScreenTests.cs` (new): 53
  targeted assertions incl. exact tier boundaries (0.55/0.45 == the 0.10 primary boundary
  after 12-decimal policy rounding), h2h depth semantics, blockers, exclusions,
  determinism, authority, and the byte-exact cross-language vectors.
- `MarketDepth.cs` (bounded): `DevigPair` extracted as the single pairwise de-vig
  authority; additive `H2hBookCount` (distinct two-sided h2h books; `BookCount` semantics
  preserved, not redefined). `PromptTrace.cs` (bounded): delegates to `DevigPair`
  (behavior-equivalent; prompt-trace suite green).
- python consumer (bounded): envelope `input-evidence-envelope/1.1` (normalized result =
  classification + screen_tier + priority_components; closed, validated); planner/request/
  board/cli bumped to 2.1; tier-aware deterministic rank tuple; boards expose candidate
  `screen_tier`; markdown projection updated; cli strict parser extended.

## cross-language compatibility result

Five exact vectors (primary / secondary / excluded / blocker-not-evaluated / poisoned
1.0-schema) embedded verbatim in BOTH suites with the update-together rule; C# is the
producing authority. Proven end to end: C# classifier -> projected envelope json (byte-
equal to the vectors in C# tests) -> strict python parser -> planner board -> candidate
tier + eligibility -> deterministic cohort ordering (primary `824732` pooled ahead of the
EARLIER-starting secondary `823518`; screen-excluded and blocked candidates never
eligible; the poisoned old-schema vector never grounds). No network or subprocess service
involved.

## verification (actual totals and hashes)

- DevCore.Api.Tests: **1338 passed / 0 failed** (full suite; targeted market-contrast +
  market-depth + prompt-trace filters: 53/53). Build: zero NEW warnings (the only two are
  the pre-existing NU1903 Microsoft.OpenApi advisory, present before this slice).
- agent-service pytest: **555 passed / 0 failed** (full suite).
- cross-process canonical determinism (two fresh python processes, 2.1 board with tiered
  screen envelopes), sha-256 equal:
  `756288cb8f7a39e2641b5b24babd030a1fefb7cf8b3a2b54e7aa2c99096e524b`
- `git diff --check` clean in both repos; strict snapshot 24 WIs / 0 warnings; MOC
  cardinality: WI-0035 registered exactly once; machine-path/secret/network/authority
  scans clean; protected drift byte-identical (dai csproj `63ef2488`; vault `b3d68588` /
  `9127e464` / `68948ebd` / `25835e6c`; Welcome.md deleted).

## safety / non-actions

0 provider clients, 0 live screening, 0 Odds API / StatsAPI / Action Network / model /
Tool Gateway / endpoint / database calls, 0 AgentRun generation, 0 capture/execution/
settlement/scheduling/tuning, 0 DI/controller/persistence/migration/prompt/config
changes, 0 pushes/merges. WI-0034 Slices 3-4 and WI-0031 Slice 4/pilot untouched. The
classifier's result and envelope authorize nothing.

## independent review corrections (2026-07-20, superseding addendum)

A separately authorized contract-integrity review reproduced and corrected, in a new
commit on the same branch (dai `ecd0ddb`; originals preserved):

1. **missing-tier blocker (reproduced):** an evaluated includable envelope with a null or
   missing screen_tier stayed eligible, took tier rank 0 (primary), and entered the
   primary pool; a negative disagreement_range (-5) produced the best possible rank.
   Corrected via centralized capability-specific semantic validation (includable requires
   a real tier; excluded requires none; priority facts must be a real boolean plus a
   finite 0..1 number or null; market-snapshot semantics kept separate).
2. **superseded claim -- "canonical json":** the projector concatenated unescaped strings;
   quotes/backslashes/non-ascii in provider data could corrupt or inject. Replaced with a
   standards-correct deterministic ascii escaping writer; an escaping/non-ascii vector was
   added to both suites.
3. **superseded claim -- "unknown/null facts are never favorable":** the 1.0 classifier
   substituted readiness/as-of timestamps and a generic provenance for missing market
   observation facts, accepted future-dated evidence, and treated negative active-run
   counts as countable. Corrected under market-contrast-screen/1.1: evaluated market
   evidence requires its real timestamp + provenance; readiness and active-run checks
   require checked-at + provenance; evidence never postdates as-of; negative counts are
   invalid; blocked projections carry the screen's own evaluation instant and the
   classifier as producer (never an invented market observation). The original vectors'
   temporal ordering (as-of before its own evidence) was itself wrong and is fixed.
4. **superseded claim -- "same paired population":** H2hBookCount counted paired books
   while medians/disagreement still used the mixed one-sided non-deduped population.
   Corrected: additive H2hPairedHomeMedian / H2hPairedAwayMedian /
   H2hPairedDisagreementRange derive from exactly the deduplicated valid-pair set that
   grounds the count (BookmakerKey dedupe, title fallback; zero/non-finite/one-sided/
   duplicate records never contribute); legacy BookCount and legacy medians unchanged.
5. **board truthfulness:** an evaluated screen no longer sits under the generic
   `market_availability: unknown_until_paid_screening`; boards now carry
   `decision_time_market_baseline: unknown_until_generation_capture` (board/planner/cli
   2.2; request unchanged at 2.1) -- a screening-time observation is never a decision-time
   baseline. The cli board validator additionally reports `interior_validated: false`.

Test counts after review: DevCore.Api.Tests **1345 / 0**; agent-service **560 / 0**;
cross-process 2.2 board sha-256
`2403c51079eb84cc423acc364c94b1af35ad96a6d14244abf1da4e0ca36fc315`; all six vectors
byte-identical across suites.

## next step

Independent review + integration of the local `wi/0035-market-contrast-candidate-screen`
branches (both repos) before any source adapter or paid screen. Slice 2 (bounded source
adapter) and Slice 3 (one governed live screen) remain separately gated. A recommendation
is not an authorization.
