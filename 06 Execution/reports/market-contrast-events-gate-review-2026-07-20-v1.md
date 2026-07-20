---
title: "Market-Contrast Events-Gate Independent Review + Integration v1 (2026-07-20)"
type: "evidence-report"
date: "2026-07-20"
status: "complete"
project: "DAI"
slice: "WI-0035 Slice 3 r7B: independent events-gate semantic review and integration"
repos:
  dai: "code review + adversarial tests (DevCore.Api events gate; branch wi/0035-market-contrast-events-gate)"
  dai-vault: "review report + current-state"
tags:
  - system-development
  - platform
  - market-contrast
  - review
  - cross-provider-identity
related:
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "06 Execution/reports/market-contrast-events-gate-slice-3-2026-07-20-v1.md"
  - "04 Products/sports-v1/market-contrast-candidate-screen-v1.md"
---

# market-contrast events-gate independent review + integration v1

## purpose

Independently review the WI-0035 events-gate r7A correction (dai `aea9465`, vault `6e746c5`)
without trusting its terminal report, using adversarial executable fixtures; correct any
confirmed defect in new commits; then fast-forward integrate both mains. Offline only; no
StatsAPI, Odds `/events`, Odds `/odds`, database, model, Tool Gateway, application, generation,
capture, execution, settlement, or scheduling call.

## method

Re-derived git truth (original r7 `e4b0196`/`6f76cde` preserved as ancestors; mains unmoved ==
origin and ancestors of the reviewed tips). Read the gate source directly and added ten
adversarial probes targeting edges the 60 existing fixtures might not cover -- inspecting each
probe's inputs and assertions, not its name. All ten passed; no production defect was confirmed,
so no production code changed. The probes were retained as a permanent tests-only commit (dai
`c7d4a79`).

## review questions (each proven by executable evidence)

1. **Preblocked -> ready?** No. Only screenable candidates enter matching; preblocked go through
   the `skipped_preblocked` diagnostic with zero counts (rb07, rb10; r7a01-04).
2. **Stale/future/cross-date reaching the call?** No. Freshness rejects stale/future/non-UTC
   completion before claim (r7a06-08); a cross-date candidate is non-screenable (r7a09).
3. **Screenability at gate time incl. margin?** Yes. Computed at the frozen observation start:
   preblock null + scheduled/pregame + start in the dst-aware Eastern day `[start,end)` + margin
   `>= MarketContrastPolicy.MinStartMarginMinutes` (r7a09-11).
4. **Zero screenable -> zero claim + zero call?** Yes, including an all-unresolved bundle (rb01;
   r7a05) -- no destination file is created.
5. **Authority ledger exact closed all-false?** Yes. Empty/missing/extra/true fail (r7a12-15);
   a duplicate ledger key and a non-boolean value also fail (rb02, rb03).
6. **Missing key -> zero claim + zero call?** Yes; null/whitespace rejected before claim
   (r7a16-17).
7. **pass-1/count/gamePk/external cross-checked?** Yes: 64-lowercase-hex pass-1, candidate-count
   == array length, unique positive gamePks, unique external ids each == gamePk, and a numeric
   (non-string) external id is rejected (r7a18-20, rb04).
8. **Parsed empty array vs failure?** Distinguished: `events_received=0` only for a parsed `[]`;
   `null` for every transport/http/parse failure (fixture 3; r7a22-25).
9. **Provider ids/teams/uniqueness/UTC strict?** Yes: non-blank id/home/away, explicit-UTC
   commence, unique event ids, and duplicate keys inside an event object all enforced
   (r7a22-25, rb09).
10. **Timezone-less ISO ever environment-dependent?** No. Explicit-UTC is proven from the
    original text (trailing `Z`/`+00:00`); a lowercase `z` and a bare timestamp are rejected
    (rb05; r7a21), while a valid millisecond-precision UTC still matches (rb06).
11. **Ready requires a screenable exact match AND a passing zero-quota audit?** Yes; a nonzero
    last-cost fails closed even over a screenable exact match (rb08; r7 fixture 20).
12. **Preblocked represented as skipped, not evaluated?** Yes; excluded from evaluated totals
    (rb10; r7a03).
13. **Original exact-match predicate unchanged?** Yes; the fixed-seed corpus proves the gate's
    exact-match set equals a direct `MarketJoinDiagnostics.Analyze` (fixture 33).
14. **Command/DI/named-client seam works under an injected handler?** Yes; one `/events`, `/odds`
    unreachable, structured status, missing-config reject before the handler (r7a26).
15. **Post-claim exception leaving an unexplained claim or deleting another writer's file?** No;
    the outer catch releases only this writer's own claim, staging/recovery are pid+attempt
    unique (r7a28; r7 fixtures 27, 29).
16. **All authority booleans false / plainly non-authorizing?** Yes; the output ledger is
    booleans only, every one false (incl. `odds_request_authorized`, `paid_attempt_authorized`)
    (r7 fixture 31).
17. **1.1/1.2 replay unchanged, 1.3 replay-inert?** Yes; the planner replay-equivalence tests
    pass for 1.1/1.2/1.3.
18. **Docs match code semantics?** Yes; the r7 closeout superseding addendum, WI-0035 entry, and
    product record state the screenable/freshness/strict-boundary semantics as implemented.

## final definitions (as reviewed)

- **Screenable** (at the frozen gate observation start): resolved + unique authoritative
  identity; `preblock_reason` null; canonical scheduled/pregame state; scheduled start inside the
  target dst-aware Eastern day `[start, end)`; and the canonical minimum decision margin still
  satisfied. A resolved identity alone is insufficient because the producer's free gates (active
  runs, starter announcement, schedule state, margin) and the passage of time between preflight
  and gate can make a resolved candidate unscreenable; only a screenable candidate's exact match
  can support a later paid-attempt proposal.
- **Fresh**: `operation.completed_as_of_utc` present, explicitly UTC, not after the observation
  start, `<= 5` minutes old at that start, and not before `operation.started_at_utc`. Boundary:
  0 and exactly 5:00 accepted; 5:01 and future rejected with zero calls.
- **Ready**: admission + freshness pass; the response parses and passes strict shape validation;
  the zero-quota audit passes (`x-requests-last` exactly 0); at least one screenable candidate
  has exactly one exact match; and no screenable candidate has a duplicate exact match. Even
  then the result grants no authority.
- **Response/quota**: a json array only; a parsed empty array is `events_received=0`; every
  failure is `events_received=null`; the zero-quota audit requires all three usage headers
  present, finite, nonnegative, and last cost exactly zero.

## verification

Events-gate fixtures 70/70 (60 r7A + 10 r7B probes). Full DevCore.Api.Tests **1486 passed / 0
failed** (was 1476). Adjacent market-contrast suites 72/72. Full agent-service pytest **564
passed**. Planner replay 1.1/1.2/1.3 inert. 0 new build warnings (only the pre-existing NU1903
advisory; 0 project/package changes). `git diff --check`, secret, machine-path, and
authority-grant scans clean. Strict planning snapshot 24 WIs / 0 warnings. Protected/classified
drift byte-identical open to close.

## integration

Both reviewed feature branches pushed; local feature tip == remote feature tip; each remote main
proven unchanged from the reviewed base and an ancestor of its reviewed tip; fast-forward-only
merges; mains pushed; re-fetch proved both local mains == origin/main == the reviewed tips.
Original r7 and r7A commits remain ancestors. No non-fast-forward, force push, history rewrite,
or partial integration.

## next step

The next action is NOT a paid attempt. It is a separately authorized, time-gated 2026-07-22
refreshed free preflight followed by at most one zero-quota `/events` observation. No `/odds`
call is authorized by this review.

## superseding correction addendum -- r7C (2026-07-20)

This addendum qualifies the "no production defect was confirmed" conclusion above. That
conclusion was correct FOR THE TEN PROBES r7B ran, but the r7B probe set was incomplete: it did
not exercise a MISSING `preblock_reason` property (as opposed to a present explicit null) or a
MISSING/malformed `operation.started_at_utc`. r7C (see
[[market-contrast-events-gate-boundary-completion-2026-07-20-v1]]) reproduced both as executable
RED failures against the integrated r7B gate and corrected them:

- a missing `preblock_reason` property is malformed input and is rejected before any claim or
  call (never silently treated as null, which could otherwise turn malformed input into a
  screenable candidate); a blank/whitespace preblock string is likewise rejected; a present
  explicit null remains the evaluated "no preblock" case; a present nonblank string remains a
  skip reason.
- `operation.started_at_utc` is required present, an explicit-utc string, and no later than
  completion; missing/null/malformed/timezone-less/nonzero-offset all fail closed before any
  call.

Operator version bumped `market-contrast-events-gate-operator/1.1`; the artifact schema stays
`market-contrast-events-gate/1.0` (output shape unchanged); no new capability or authority. The
r7B integration proof and the ten probe findings remain valid as written; this addendum records
that the "no defect" claim was scoped to the probes actually run.
