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

> **Current integration state (2026-07-20, superseding the inline "NOT pushed / NOT
> merged" / "local only" flags below).** Confirmed against git: dai `main` contains every
> WI-0035 commit -- Slice-1 core `a6e213b`+`ecd0ddb`, Slice-2 source adapter `a05b63b`,
> r3 corrections `8e044a4`, join diagnostics `0aae858`, r6a-review corrections `5e81b83`
> -- and both WI-0031 Slice-4 commits `f926484`+`209d485`. The Slice-1/Slice-2/diagnostics
> branches are integrated, not local-only. Reconciled sequencing: (1) WI-0031 Slice 4 is
> integrated; production weights, telemetry (Slice 5), and pilot (Slice 6) remain
> separately deferred. (2) The WI-0035 source adapter, the first live `/odds` attempt, and
> join diagnostics are integrated. (3) The next time-sensitive gate is the zero-quota
> 2026-07-22 `/events` cross-provider identity observation (events-gate operator below,
> delivered locally this slice). (4) A later `/odds` attempt needs a separate authorization
> AND at least one exact current identity/start join. (5) WI-0031 Slice 5 telemetry is not a
> prerequisite for the 2026-07-22 evidence loop. (6) No selection plan, events-gate
> artifact, or planner board grants execution authority. The historical bullets are
> preserved as written; this note is the authoritative current state.

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
- **Join-diagnostics hardening (complete 2026-07-20, local commits): explain no-match
  causes** -- offline, no source calls (see
  [[market-contrast-join-diagnostics-2026-07-20-v1]]): a pure `MarketJoinDiagnostics`
  helper classifies each candidate's Odds match attempt into a closed status vocabulary
  (`no_events_returned` / `team_pair_not_found` / `orientation_mismatch` /
  `start_instant_mismatch` / `matched` / `duplicate_exact_match` + context statuses)
  using EXACTLY the existing exact-match predicate; bundle
  `market-contrast-screen-bundle/1.2` and adapter `market-contrast-source/1.2` add
  response-level (`events_received`, `evaluated_candidate_exact_match_count`,
  `response_parsed`; r6a-review corrected: parsed-only counts, null on failure, evaluated-
  only total, replay option B) and
  per-candidate diagnostics; planner CLI 2.4 accepts 1.2 while keeping 1.1 explicitly
  replayable (hash-preserving). Matching did NOT become more permissive; no
  classification/tier/priority/readiness/eligibility/authority change; the attempt-1 1.1
  bundle is unchanged and no team-matching defect is declared without provider evidence.
  Branch `wi/0035-market-contrast-join-diagnostics`; NOT pushed / NOT merged.
- **Events-gate operator (delivered 2026-07-20, local commits): zero-quota cross-provider
  `/events` identity observation** -- offline implementation and testing only, no source
  call (see [[market-contrast-events-gate-slice-3-2026-07-20-v1]]). A local-only one-shot
  command `market-contrast-events-gate --preflight-bundle <f> --out <f> [--target-date <d>]`
  that observes the provider's ZERO-QUOTA `/events` endpoint once (never `/odds`), and joins
  the returned events against the authoritative statsapi identities carried by a freshly
  generated preflight bundle -- with NO statsapi, database, model, or application path of its
  own. Producer parity: the market-contrast preflight bundle bumps to
  `market-contrast-screen-bundle/1.3` / adapter `market-contrast-source/1.3`, additively
  emitting a per-candidate `authoritative_identity` (normalized home/away refs, exact
  scheduled utc, schedule state); planner CLI 2.5 accepts 1.3 (1.1/1.2/1.3 replay-identical,
  authoritative_identity is replay-inert). The exact-match predicate is the integrated
  `MarketJoinDiagnostics` predicate verbatim -- aliases, shortened names, reversed
  orientation, and nonzero start deltas are observation-only and never match. Artifact
  `market-contrast-events-gate/1.0` (canonical json, authority ledger booleans-only all
  false, zero-quota audit requiring `x-requests-last` exactly 0, no api key, no tenant data,
  no planner envelope). 33 offline fixtures; reused authorities only (EasternDayBracket, the
  `OddsApiEvent` DTO, GameIdentityDerivation.NormalizeTeamRef, MarketJoinDiagnostics,
  ScreenBundlePublisher) -- no second client/normalizer/bracket/publisher. Branch
  `wi/0035-market-contrast-events-gate`; local commits only, NOT pushed / NOT merged.
  **r7A pre-integration correction (2026-07-20, new commit; original r7 preserved as an
  ancestor; see the superseding addendum in
  [[market-contrast-events-gate-slice-3-2026-07-20-v1]]):** the r7 operator over-counted
  resolved-but-preblocked candidates -- a resolved identity is NOT the same as a screenable
  candidate. A match is `exact_match_ready_for_separate_operator_decision` only when it belongs
  to a candidate still screenable at the gate observation start (preblock_reason null, canonical
  scheduled/pregame state, start inside the target dst-aware Eastern day `[start, end)`, and the
  canonical minimum decision margin still satisfied) AND the preflight bundle is fresh (<= 5
  minutes old) and strictly valid. Non-screenable candidates are retained as `skipped_preblocked`
  (zero counts, never matched, never ready); zero screenable candidates reject before any call
  with the new closed status `no_screenable_candidates`. Added: bounded freshness contract;
  strict closed input validation (schema/mode/terminal/target-date/candidate-count/unique
  gamePks+external ids each == gamePk/64-lowercase-hex pass1/closed all-false authority ledger);
  missing-or-blank API key rejected before claim and call; strict provider-response validation
  (json array; non-blank id/home/away; explicit-UTC commence proven from text; no duplicate
  ids/keys); strengthened publication/recovery so no post-claim path leaves an unexplained
  claim; a real command/DI/named-client seam test. Version set unchanged (bundle/adapter 1.3,
  operator/artifact 1.0, CLI 2.5). 60 offline fixtures (33 retained + 27); the six defects were
  reproduced as executable RED failures against the committed r7 gate before correcting.
  **r7B independent review + INTEGRATION (2026-07-20; see
  [[market-contrast-events-gate-review-2026-07-20-v1]]):** the r7A correction was independently
  reviewed with ten adversarial probes across all 18 review questions; the reviewed feature
  branches were fast-forward integrated into both mains (dai `c7d4a79`, vault `32a1d99`) and
  pushed. The events-gate operator is INTEGRATED as of r7B (superseding the r7/r7A "NOT pushed /
  NOT merged" flags above).
  **r7C fail-closed boundary completion (2026-07-20; see
  [[market-contrast-events-gate-boundary-completion-2026-07-20-v1]]):** r7B correctly reviewed
  its listed probes but omitted two malformed-input cases. r7C corrects them: a MISSING
  `preblock_reason` property (distinct from a present explicit null) and a blank/whitespace
  preblock string are now rejected before any claim or call, and `operation.started_at_utc` is
  now required present, an explicit-utc string, and no later than completion (missing/malformed/
  timezone-less/nonzero-offset all fail closed). Operator version bumped
  `market-contrast-events-gate-operator/1.0` -> `1.1` (artifact schema stays 1.0; the output
  shape is unchanged; no new capability or authority). Integrated fast-forward into both mains.
  WI-0035 remains in progress; operating-integration Slice 4 remains deferred.
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
`in-progress` (operating-integration Slice 4 deferred). Current disposition (2026-07-20,
superseding the earlier "events-gate NOT integrated" line): Slice 1 core, Slice 2 source
adapter, the first live `/odds` screen, join diagnostics, AND the events-gate operator
(r7A correction -> r7B review+integration -> r7C fail-closed boundary completion, operator
1.1) are all INTEGRATED on both mains (dai `main`, vault `main`). Next governed action =
a separately authorized, time-gated action no earlier than 2026-07-22T12:00:00Z: one
refreshed free preflight plus exactly one zero-quota `/events` gate observation (no `/odds`
request in that authorization). A paid `/odds` attempt is proposed separately only when a
current exact identity/start join from a screenable candidate exists.
