---
title: "Market Contrast Start-Instant Normalization Analysis 2026-07-22 v1"
type: "evidence-report"
date: "2026-07-22"
status: "analysis complete -- NO production predicate change; rounding hypothesis REFUTED by captured evidence"
project: "DAI"
slice: "WI-0035 offline start-instant join characterization"
repos:
  dai: "unchanged (no code, test, or fixture change; analysis was offline and read-only)"
  dai-vault: "docs only; local branch wi/0035-start-instant-normalization-analysis, NOT pushed / NOT merged"
tags:
  - evidence-operations
  - sports-v1
  - market-contrast
  - analysis
related:
  - "06 Execution/reports/market-contrast-events-gate-observation-2026-07-22-v1.md"
  - "02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md"
  - "06 Execution/reports/market-contrast-live-screen-2026-07-22-v1.md"
---

# market contrast start-instant normalization analysis 2026-07-22 v1

## purpose

The 2026-07-22 events-gate observation matched only 1 of 13 screenable candidates, with 11
same-orientation candidates off by exactly +60 seconds and one off by -240 seconds. This
report answers one question: **is the +60-second offset demonstrably a provider-rounding
equivalence, or is it schedule movement / per-event data difference?**

The question matters because the answer decides whether the production join predicate may
safely admit a bounded start tolerance. It was investigated entirely offline from
already-captured artifacts. No provider, model, gateway, database, capture, screening,
settlement, scheduling, or paid call was made.

## verdict

**The rounding-equivalence hypothesis is REFUTED. No production predicate change is
implemented.** The prompt's "if and only if the captured evidence proves a safe
deterministic rule" condition is not met.

## decisive evidence -- identical inputs, different outputs

A provider-side rounding or normalization convention is necessarily a **function of the
scheduled instant**: the same input instant must produce the same output instant. The
captured slate contains two scheduled instants shared by two candidates each, and in both
cases the provider instants disagree:

| scheduled (UTC) | candidates | provider delta | verdict |
|---|---|---|---|
| `22:40:00Z` | 823438 LAD@PHI / 824408 MIN@CLE | **+0s** / **+60s** | DIFFERENT |
| `00:10:00Z` (07-23) | 824166 MIA@HOU / 824650 DET@CHC | **+60s** / **-240s** | DIFFERENT |
| `18:10:00Z` | 823761 NYM@MIL / 824083 SF@KC | +60s / +60s | same |
| `19:40:00Z` | 823110 CIN@SEA / 825055 ATH@ARI | +60s / +60s | same |

Two counterexamples are sufficient. The same scheduled instant maps to two different
provider instants, so the offset cannot be a deterministic transform of the scheduled
instant. It reflects **per-event data difference** -- schedule movement, differing provider
sourcing per event, or both.

## full signed-delta table (provider minus scheduled)

| gamePk | matchup | scheduled | provider | delta | diagnostic |
|---|---|---|---|---|---|
| 822784 | TB@TOR | 23:07:00Z | 23:08:00Z | +60s | start_instant_mismatch |
| 822873 | CWS@TEX | 00:05:00Z | 00:06:00Z | +60s | start_instant_mismatch |
| 823110 | CIN@SEA | 19:40:00Z | 19:41:00Z | +60s | start_instant_mismatch |
| **823438** | **LAD@PHI** | **22:40:00Z** | **22:40:00Z** | **+0s** | **matched** |
| 823518 | PIT@NYY | 17:05:00Z | 17:06:00Z | +60s | skipped_preblocked (caller_start_mismatch) |
| 823761 | NYM@MIL | 18:10:00Z | 18:11:00Z | +60s | start_instant_mismatch |
| 824004 | STL@LAA | 20:07:00Z | 20:08:00Z | +60s | start_instant_mismatch |
| 824083 | SF@KC | 18:10:00Z | 18:11:00Z | +60s | start_instant_mismatch |
| 824166 | MIA@HOU | 00:10:00Z | 00:11:00Z | +60s | start_instant_mismatch |
| 824327 | WSH@COL | 19:10:00Z | 19:11:00Z | +60s | start_instant_mismatch |
| 824408 | MIN@CLE | 22:40:00Z | 22:41:00Z | +60s | start_instant_mismatch |
| **824650** | **DET@CHC** | **00:10:00Z** | **00:06:00Z** | **-240s** | start_instant_mismatch |
| **824732** | **BAL@BOS** | **23:10:00Z** | **23:10:00Z** | **+0s** | skipped_preblocked (starters_not_announced) |
| 824896 | SD@ATL | 23:15:00Z | 23:16:00Z | +60s | start_instant_mismatch |
| 825055 | ATH@ARI | 19:40:00Z | 19:41:00Z | +60s | start_instant_mismatch |

Histogram: `-240s x1`, `+0s x2`, `+60s x12`.

## the -240-second case

gamePk 824650 (DET@CHC) is scheduled `2026-07-23T00:10:00Z` while the provider carries
`2026-07-23T00:06:00Z`. Under any minute-level rounding convention a -240s offset is
unreachable, and no provider event for that pair falls within 60 seconds. **It remains
blocked** and is not equivalent to the +60s population. Nothing in the captured evidence
proves it is the same game at a different clock; it may be a genuine schedule revision.

## duplicate / ambiguity findings (doubleheaders)

Two provider team-pairs are duplicated in the same date bracket:

- `pittsburgh-pirates@new-york-yankees`: `17:06:00Z` **and** `23:05:00Z`
- `baltimore-orioles@boston-red-sox`: `17:36:00Z` **and** `23:10:00Z`

A team-pair-only join is therefore ambiguous for these matchups; today's exact-start
predicate disambiguates them correctly. Any future tolerance must preserve one-to-one
mapping. On this slate a +/-60s window happens to remain unique (the alternate game is
hours away), but that is a property of this particular slate, not a proven invariant -- and
it does not rescue the refuted hypothesis.

Reversed orientation: **0**. Unresolved identities: **0**. Normalized team identity and
orientation resolve cleanly for all 13 screenable candidates; identity is not the limiting
factor anywhere in this dataset.

## why the July-20 capture cannot corroborate

The `screen-2026-07-22/` paid screen (executed 2026-07-20) covers the same slate, but its
`market-contrast-screen-bundle/1.1` adapter recorded only join **outcomes** -- no provider
`commence_time` and no `odds_event_id` are present in any July-20 artifact (verified across
all ten JSON files). It therefore cannot supply a second observation of provider instants.

Consequently the captured evidence contains **exactly one** observation of provider start
instants (2026-07-22T12:38:37Z). Distinguishing a stable provider convention from schedule
movement requires at least two independent observations of the same events at different
times. Note also that 823438 (LAD@PHI) did **not** match on July 20 but matched exactly on
July 22 -- consistent with provider data changing over time, which is the movement
hypothesis rather than the rounding hypothesis.

## a finding that matters more than the tolerance question

gamePk **824732 (BAL@BOS) already agrees with the provider exactly** (`23:10:00Z` on both
sides, delta 0s). It produced no match only because it was preblocked upstream for
`starters_not_announced` -- a freshness condition, not an identity or start problem.

So the captured evidence says the path to more exact matches is **a fresher preflight taken
after starters are announced**, not a looser predicate. Under today's unchanged predicate
this slate already contains two candidates whose identity and start instant agree exactly.

## executable evidence

Deterministic, offline, read-only re-derivation of every table above from the captured
artifact:

- script: `<SESSION_SCRATCH>/start_instant_analysis.py`
  (sha-256 `aa9440a9666f546506d7189c616bd42e1156239b651791648a47e5c36de3c1ec`)
- input: `<DAI_WORKSPACE_ROOT>/events-gate-2026-07-22/attempt-1/events-gate-artifact.json`
  (sha-256 `26cdf20ce1a4b362929ee343134fe314ebcc0da9aca0062a69879d20ae554415`)

The script re-computes team-pair multiplicity, per-candidate signed deltas, the
identical-input/different-output test, the delta histogram, and a one-to-one uniqueness
check under a hypothetical +/-60s window. It makes no network, database, or provider call.
It is kept outside both repositories because it is analysis scaffolding, not a production
fixture -- no production fixture was added, because no production behavior changed.

## decision and its boundary

- **No change** to `MarketContrastEventsGate`, the join predicate, the source adapter, any
  operator or artifact contract version, any fixture, or any test.
- The exact-match predicate remains: identical normalized teams **and** orientation **and**
  identical start instant, within the same target-date bracket, one-to-one.
- No contract was versioned because no semantics changed.
- The -240s case stays blocked. The +60s population stays unmatched.

If a bounded normalization is ever revisited, the evidence bar is explicit: **at least two
independent observations of the same events at different times**, showing the offset is
stable per event rather than moving, plus a demonstration that identical scheduled instants
never map to differing provider instants. This dataset fails that bar on its own terms.

## what remains unauthorized

No paid `/odds` screen, capture, AgentRun, model call, scheduling, settlement,
reconciliation, migration application, or activation. The WI-0036 Slice-3 migration remains
generated and unapplied. WI-0036 Slices 1-3 keep their integrated status; Slices 4-6 remain
deferred.

## recommended next action (proposal only)

A separately governed slice that takes **one fresh preflight later in the game day**, when
starters are announced, and re-observes with the unchanged predicate. That is expected to
convert 824732 (already start-aligned) and possibly 823518 into exact matches without any
predicate change. Only if that fresh observation yields a sufficient exact-match set should
a bounded `/odds` screen be proposed -- and it would still need its own authorization.
