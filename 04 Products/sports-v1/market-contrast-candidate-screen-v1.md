# market-contrast candidate screen v1

**status:** active doctrine (slice 1 delivered; source adapter + live screening deferred)
**date:** 2026-07-19

## purpose

The niche/domain producer for the planner capability `input.market_contrast_screen`: a
pre-generation classification of whether an MLB candidate is expected to be valuable for
collecting discrimination evidence.

## what it is

A pure, offline, deterministic C# classifier
(`platform/dotnet/DevCore.Api/AgentRuns/MarketContrastScreen.cs`) with a versioned policy
profile `market-contrast-screen/1.0` whose numeric rules live in one policy class and come
from `06 Execution/patterns/cohort-selection-and-run-discipline-v1.md`:

- hard prerequisites: provider-scoped identity matched; schedule scheduled/pregame; first
  pitch >= 60 min after the injected as-of; source readiness predicts
  `starter_enriched_market_backed_depth` AND generation-eligible; market observation
  evaluated and fresh (explicit normalized status; the future live adapter owns
  timestamp-to-freshness policy); valid two-sided h2h probability pair; >= 5 distinct h2h
  books; active-run check evaluated with zero active runs.
- tiers from the canonical pairwise de-vig side gap (policy-rounded to 12 decimals):
  <= 0.10 primary; 0.10 < gap <= 0.15 secondary; > 0.15 excluded.
- blocker vs exclusion: a blocker (identity, readiness, market availability/staleness,
  depth, duplication, contradictory input) is a trust failure, never a validly screened
  exclusion. exclusions (out of filter, not pregame, insufficient margin) are valid
  screen decisions.
- soft priority: within a tier, greater KNOWN cross-book disagreement carries more
  evidence value; represented as deterministic priority facts (known-before-unknown,
  greater-first), never a hard threshold.

## what it is not

- not a prediction of DAI-vs-market disagreement: every result carries
  `actual_dai_market_disagreement: unknown_until_generation`.
- not an authority: every result and envelope keeps screening/capture/execution/
  scheduling/tool-selection false; a valid screen authorizes nothing.
- not a second market-math implementation: pairwise de-vig is `MarketDepth.DevigPair`
  (the single authority, extracted from the prompt-trace path which now delegates);
  book aggregation is `MarketDepth.Summarize`; readiness facts come from
  `SourceReadinessClassifier` and are consumed, never reconstructed.

## market-depth contract consequence (h2h book count)

`MarketDepthSummary.BookCount` keeps its original any-usable-field semantics. The
canonical cohort-doctrine fact ("bookmakers pricing h2h") is the additive
`H2hBookCount`: distinct bookmakers (deduped by name) quoting a TWO-SIDED h2h moneyline
pair. Spread-only books, one-sided moneylines, and duplicate names never satisfy the h2h
threshold. Additive optional field; existing constructors and consumers unchanged.

## planner-envelope projection and versions

`MarketContrastScreen.ProjectToPlannerEnvelopeJson` emits the planner-owned typed
evidence contract as canonical json: `input-evidence-envelope/1.1` with classifier
version, evaluation status (blockers map to non-grounding states: stale->stale,
source-unavailable->unavailable, market/active-run-not-evaluated->not_evaluated, all
other blockers->invalid), observation timestamp, source provenance, target date,
provider-scoped identity, stable producer reasons, and a normalized result
(classification + screen_tier + priority_components). Envelope 1.1 / request 2.1 / board
2.1 / planner 2.1 / cli 2.1 -- the closed 1.0/2.0 shapes were not silently extended.
The planner consumer preserves the screen decision end to end: excluded/blocked never
eligible; primary ranks ahead of secondary; known disagreement before unknown, greater
first; remaining ties by start/provider/event id; boards expose candidate screen tiers
and producer reasons.

## cross-language contract

Exact vectors (primary, secondary, excluded, blocker/not-evaluated, poisoned 1.0-schema)
are embedded verbatim in BOTH suites -- `MarketContrastScreenTests.CrossLanguageVectors`
(C#, the producing authority for canonical json) and the planner cli suite (python
consumer) -- under an update-together ownership rule, each side asserting byte equality /
end-to-end flow.

## future paid-source boundary

Slice 2 (source adapter) and Slice 3 (one governed live screen) are separate
authorizations with explicit paid-call bounds; the classifier core never gains source
access. Nothing in this record authorizes screening, capture, or generation.

## related docs

- `02 Platform/system-development/work-items/WI-0035-market-contrast-candidate-screen.md`
- `04 Products/sports-v1/daily-evidence-acquisition-orchestrator-v1.md`
- `06 Execution/patterns/cohort-selection-and-run-discipline-v1.md`

## recommended next slice

Independent review + integration of the WI-0035 Slice-1 branches; afterwards a separate
authorization may consider the bounded source adapter. A recommendation is not an
authorization.
