# Cross-Sport Source Component Architecture v1

**date:** 2026-06-22
**status:** DIRECTIONAL ARCHITECTURE PLAN. Vault-only. No application code, prompt, runtime, migration,
buyer-UI, or reconciliation change. No source framework is built by this slice.
**relation to prior work:** builds on `04 Products/sports-v1/source-depth-contract-v1.md`,
`04 Products/sports-v1/market-odds-depth-v1.md`, and the `SourceSignalEnvelope` conceptual contract
already defined in `02 Platform/architecture/inference/grpc-inference-orchestration-boundary-v1.md` §6.
Reconciles category naming with the live `SourceSignalTaxonomy.SourceGroups` rather than inventing a
parallel vocabulary.

## 1. Purpose

Make future source/data work reusable across sports. The platform should add richer signals through
**cross-sport signal contracts** (stable category shapes) served by **sport-specific adapters**, instead
of accumulating one-off MLB-only branches. This is a contract- and roadmap-shaping plan so the next few
source slices each improve the factory, not just one product line. It is not an implementation slice.

Important framing: the reusable categories are not new. `SourceSignalTaxonomy.SourceGroups` already
enumerates `market_odds`, `starting_pitching`, `bullpen_availability`, `lineup_injury`, `team_form`,
`rest_travel`, `weather_park`, `market_movement`, `identity_schedule`. This plan **promotes that latent
taxonomy into an explicit source-component model** with normalized envelopes, adapters, and freshness --
the vocabulary is already in the code; what is missing is the component shape and the adapters that fill
most groups.

## 2. Current Context

- **Market Odds Depth v1 (shipped)** deepened the market source: multi-book moneyline/spread/total ->
  `market_odds` now classifies `shallow` vs `enriched`. First proof that a source group can carry observed
  depth + provenance.
- **MLB starter depth exists** (`MlbStarterClient` -> `starting_pitching` enriched/identity_only/none).
- **Most non-market, non-starter groups are defined but unfilled:** `bullpen_availability`,
  `lineup_injury`, `rest_travel` (basketball-only today), `team_form`, `weather_park` have taxonomy slots
  and tiers but **no adapters** -- so many baseball-specific risks (lineup, bullpen, park/weather) remain
  under-sourced. This is the root of the standing "named-risk Ungrounded" weakness.
- **gRPC inference orchestration is deferred and transport-agnostic.** Source acquisition stays REST/JSON;
  the `SourceSignalEnvelope` is the promotion target for a later protobuf contract, not built now.
- **Direction:** future source work should be reusable across sports wherever the signal is not inherently
  sport-specific.

## 3. Design Principle

**Reusable signal contract, sport-specific adapter.**

- **Reusable contract (cross-sport):** the *category shape* and its normalized envelope -- what an
  availability/fatigue/venue/market signal means in decision terms, its status, depth, freshness, and
  buyer-safe summary. One shape, all sports.
- **Sport adapter (sport-specific):** how the category is *filled* for one sport from one provider. MLB
  bullpen workload and NBA minutes-load are different domains but both reduce to a `FatigueContext`
  envelope. The adapter owns the domain math; the contract owns the meaning.
- **Provider-specific (stays in the adapter):** raw API shapes, endpoints, auth, rate limits, name
  normalization. Never leak provider field names above the adapter.
- **Deferred:** any category with no near-term high-value adapter, and any cross-sport abstraction whose
  two sports do not yet share a real decision use. Do not force false universality.

Do not over-abstract: a category earns its contract only when at least two sports (or one
high-value sport-specific risk) need it. MLB bullpen is not NBA minutes load -- but both map to
`FatigueContext` because both answer "how depleted is this side going in?"

## 4. Proposed Source Component Model

Conceptual components (fields, not code). Several promote existing types
(`SportsRetrievalOutput`, `SourceDepthRecord`, `SignalAvailabilityRecord`, `MarketDepthSummary`).

- **SourceComponent** -- a reusable category producer. Fields: categoryKey (maps to a `SourceGroups`
  value), tier (Critical/Supporting/Contextual, from the taxonomy), appliesTo(sport) predicate, the
  adapter it delegates to, and the envelope it emits.
- **SourceAdapter** -- sport+provider-specific fetch/normalize unit. Fields: sport, provider, the category
  it fills, fetch(matchup, ct) -> envelope, and its own fail-soft policy. Owns all provider detail.
- **SourceSignalEnvelope** -- the normalized output (reuse the shape from the gRPC inference doc §6).
  Fields: categoryKey, signalName, sourceProvider, status (`SourceSignalStatus`), depthLevel
  (`SourceDepthLevels`), observed flag, freshness (`SourceFreshness`), provenance (`SourceProvenance`),
  quality (`SourceQuality`), modelContextSummary, artifactSafeSummary, typed payload.
- **SourceSignalStatus** -- observed | proxy | missing | stale | not_applicable. (Today's code has
  grounded/missing/unavailable/not_attempted/proxy/not_applicable; `stale` is NEW here, derived from
  freshness, and would be added when freshness lands.)
- **SourceFreshness** -- fetchedAt, sourceUpdatedAt, asOf, ageSeconds, stalenessThreshold, isStale.
  (Today only market `UpdatedAt` exists, unstructured; this formalizes it.)
- **SourceProvenance** -- provider, endpoint/method class, sourceRef, retrievedBy (run/correlation id),
  confidenceOfMatch (for fuzzy event/team matching).
- **SourceQuality** -- official vs unofficial, completeness, matchConfidence; optional, defaults safe.
- **ArtifactSafeSummary** -- buyer-facing, overclaim-safe phrasing of the signal (no raw provider terms,
  no internal gap language). Feeds the buyer source-depth surface.
- **ModelContextSummary** -- the factual block injected into the analyzer prompt (data only, not prompt
  doctrine) -- the generalization of today's per-sport market/starter injection.

## 5. Reusable Signal Categories

First recommended cross-sport categories (each maps to an existing `SourceGroups` key):

**MarketContext** (`market_odds`, Critical)
- Purpose: where the market sets the line and how firmly. Applies: MLB, NBA, NFL, NCAAF, NCAAMB.
- Sport examples: moneyline/spread/total/book count/consensus/disagreement (all sports).
- Buyer value: high (concrete, explainable). Named-risk value: medium (grounds market risks).
- Complexity: low -- largely shipped (Market Odds Depth v1); needs envelope promotion + NBA/NFL extension.

**AvailabilityContext** (`lineup_injury`, Supporting)
- Purpose: who is actually playing / available. Applies: MLB, NBA, NFL (NCAA later).
- Sport examples: MLB confirmed/probable lineup + player availability; NBA injury report
  (active/inactive, probable/questionable/out); NFL injury report + inactive list + depth-chart.
- Buyer value: high. Named-risk value: **highest** -- most ungrounded named risks are availability risks.
- Complexity: medium (timing-sensitive; lineups post late).

**ScheduleRestContext** (`rest_travel`, Supporting)
- Purpose: rest/schedule spot. Applies: NBA, NCAAMB (today), NFL, MLB (lighter).
- Sport examples: NBA days rest + back-to-back (shipped via ESPN); NFL short week; MLB series leg.
- Buyer value: medium. Named-risk value: medium. Complexity: low-medium (NBA already grounded).

**FatigueContext** (cross-cut: `bullpen_availability` for MLB, `rest_travel` for NBA/NFL)
- Purpose: depletion going in. Applies: MLB, NBA, NFL.
- Sport examples: MLB bullpen workload / relief usage / starter short-outing proxy; NBA minutes load +
  back-to-back + travel; NFL short week + injury burden + snap load.
- Buyer value: medium-high. Named-risk value: high (bullpen is a frequent ungrounded MLB risk).
- Complexity: medium-high (mostly derived from recent game logs -> proxy/observed).

**VenueEnvironmentContext** (`weather_park`, Contextual)
- Purpose: environment effect on play. Applies: MLB, NFL, NCAAF; mostly not_applicable for NBA/NCAAMB.
- Sport examples: MLB park factor + wind/temp/precip; NFL/NCAAF weather + surface + stadium type;
  NBA/NCAAMB indoor -> not_applicable, home-court context only.
- Buyer value: medium (mostly affects totals). Named-risk value: medium. Complexity: low-medium
  (open-meteo + static park factors).

**ParticipantFormContext** (`team_form` + form aspect of `starting_pitching`, Contextual)
- Purpose: recent form of the key participants. Applies: all sports.
- Sport examples: MLB starter season form (shipped) + lineup recent form; NBA player/team form; NFL form.
- Buyer value: medium. Named-risk value: medium. Complexity: medium (volume of data; overclaim risk).

**MatchupContext** (`team_form` / matchup_style, Contextual)
- Purpose: style/matchup interactions (handedness splits, pace, scheme). Applies: all, value varies.
- Sport examples: MLB team-vs-handedness splits; NBA pace/style; NFL scheme matchup.
- Buyer value: medium. Named-risk value: medium. Complexity: high (analytical; defer).

**SourceFreshnessContext** (cross-cutting metadata, not a `SourceGroups` member)
- Purpose: how current each signal is; drives the `stale` status. Applies: all.
- Buyer value: medium (trust). Named-risk value: low-direct, high-indirect (prevents stale grounding).
- Complexity: low (formalize fetchedAt/updatedAt already partly present).

## 6. Sport-Specific Adapter Examples

Each adapter fills one reusable category for one sport; the contract is shared.

- **MLBAvailabilityAdapter** -> AvailabilityContext. statsapi lineups/roster/injuries -> confirmed vs
  probable lineup, player availability.
- **NBAAvailabilityAdapter** -> AvailabilityContext. injury report -> active/inactive,
  probable/questionable/out.
- **NFLAvailabilityAdapter** -> AvailabilityContext. injury report + inactive list + depth-chart.
- **MLBBullpenFatigueAdapter** -> FatigueContext. recent relief usage / rest -> bullpen depletion (proxy
  when derived from game logs).
- **NBAScheduleFatigueAdapter** -> FatigueContext (+ ScheduleRestContext). back-to-back + minutes load +
  travel spot.
- **NFLShortWeekFatigueAdapter** -> FatigueContext. short week + injury burden + snap load if available.
- **MLBVenueEnvironmentAdapter** -> VenueEnvironmentContext. park factor + wind/temp/precip.
- **NFLWeatherEnvironmentAdapter** -> VenueEnvironmentContext. weather + surface + stadium type.

Shared rule: the adapter outputs a `SourceSignalEnvelope` with the category's normalized fields; the
analyzer and synthesis never see provider shapes or adapter internals.

## 7. Dynamic Source Availability

Behavior per state (envelope `status`), with examples:

- **observed** -- real data retrieved at usable depth. Ex: multi-book market -> observed/enriched;
  confirmed MLB lineup; ESPN rest data.
- **proxy** -- a related/adjacent signal stands in. Ex: bullpen usage derived from recent game logs
  (proxy or observed depending on completeness); market direction standing in for a missing sharp/public
  split.
- **missing / pending** -- expected but not present yet. Ex: MLB lineup not posted yet -> missing (or a
  `pending` nuance if pre-deadline).
- **stale** (NEW) -- present but older than its freshness threshold. Ex: an injury report not refreshed
  near game time. Drives a caution, not a hard block.
- **not_applicable** -- category does not apply to this sport/game. Ex: NBA indoor weather -> not_applicable
  (never "missing"; absence here is correct, not a gap).

Depth interacts with status: `single-book odds only -> observed but shallow/proxy`; `multi-book market ->
observed/enriched`. Missing/not_applicable carry no depth.

## 8. Interaction with SourceDepth and EvidenceSufficiency

- **SourceDepth:** each observed envelope yields a `SourceDepthRecord` (none/identity_only/shallow/
  enriched) for its group -- generalizing today's MLB-only `SourceDepthEvaluator` so every grounded
  category reports observed depth, not just starting_pitching and market_odds.
- **EvidenceSufficiency:** breadth (how many critical/supporting groups grounded) stays owned by
  `SourceSufficiency` (bands insufficient/thin/moderate/rich); adding filled categories raises breadth
  honestly. not_applicable groups must not count against sufficiency.
- **Named-risk grounding:** reusable categories give named risks real groups to map to. A bullpen risk
  maps to `bullpen_availability`; a lineup risk to `lineup_injury`; once those adapters exist, today's
  `NamedRiskGrounding=Ungrounded` results become Grounded/NotMappable correctly. This is the single
  biggest artifact-quality payoff of the whole plan.
- **Confidence calibration:** OUT OF SCOPE here. More/clearer sources create variance the calibration loop
  can later read; this slice recommends no confidence change.
- **Buyer-safe summaries:** each envelope's `ArtifactSafeSummary` feeds the existing buyer source-depth
  surface without new internal-term leakage.

## 9. Interaction with Synthesis

When a synthesis phase exists (see the gRPC inference doc), it should:
- **integrate source packets** (envelopes) as the evidence base, alongside primary output + deterministic
  checks + verifier findings.
- **never invent evidence** -- absent/missing/not_applicable envelopes stay absent; synthesis may
  down-weight or add humility, not fabricate.
- **demote unsupported named risks** -- a named risk whose category envelope is missing/stale is flagged,
  not silently grounded.
- **preserve missingness** -- the artifact must continue to distinguish "no signal" from "signal says no
  edge."
- **separate observed evidence from model inference** -- envelopes are observed; model commentary is
  clearly secondary.
- **support a future verifier pass** -- envelopes are exactly what a verifier checks the primary artifact
  against.

## 10. Future Slice Planning Updates (ranked source roadmap)

1. **Cross-Sport Source Envelope v1** -- introduce the normalized `SourceSignalEnvelope` + status/freshness/
   provenance shape and retrofit the *existing* market + starter-depth signals onto it. Why: establishes
   the reusable contract with zero new providers; de-risks everything after. Reuse: total. Adapters: none
   new (wrap existing). Must not change: lean/confidence/posture logic, buyer UI, prompts (data shape only).
2. **Availability Context v1 (MLB first)** -- MLBAvailabilityAdapter -> lineup/injury envelope. Why:
   biggest named-risk-grounding win. Reuse: high (NBA/NFL adapters follow). Adapters: MLB now; NBA/NFL
   later. Must not change: confidence logic; observed-only.
3. **Schedule/Fatigue Context v1** -- generalize NBA rest + add MLB bullpen-fatigue proxy. Why: grounds
   the second-most-common ungrounded risk. Reuse: high. Adapters: MLB bullpen, NBA schedule. Must not
   change: posture logic.
4. **Venue/Environment Context v1** -- MLB park/weather + NFL weather; NBA not_applicable. Why: grounds
   environment risks; cheap providers. Reuse: medium. Adapters: MLB, NFL. Must not change: totals/lean
   logic.
5. **Matchup/Participant Form Context v1** -- handedness splits / form. Why: analytical lift. Reuse: medium.
   Adapters: sport-specific. Higher overclaim risk -> later. Must not change: prompt doctrine.
6. **Market Agreement Projection v1** -- read-time lean-vs-market agreement from the already-shipped
   multi-book consensus (carried over from Market Odds Depth v1's deferred piece). Why: turns existing
   depth into a calibration dimension. Reuse: high. Adapters: none. Must not change: confidence/lean.
7. **Inference Worker Contract v1 (FUTURE ONLY)** -- typed orchestrator<->inference contracts consuming
   envelopes. Gated on the gRPC-doc §5 trigger. Not now.

## 11. Non-Goals

Explicitly deferred: implementing any source framework now; gRPC implementation; confidence tuning;
prompt-doctrine rewrite; buyer-UI changes; reconciliation-logic changes; dashboards; auth/billing/tenant
work; a full provider marketplace; adding every possible sports stat. Componentization serves artifact
quality -- not a source framework for its own sake.

## 12. Recommended Immediate Next Slice

**Cross-Sport Source Envelope v1.** Introduce the reusable `SourceSignalEnvelope` (status, depth,
freshness, provenance, artifact-safe + model-context summaries) and retrofit the **already-grounded**
market and starter-depth signals onto it -- before adding any new provider. This establishes the reusable
contract with no new API risk, generalizes the MLB-only `SourceDepthEvaluator`, and makes every subsequent
source slice (availability, fatigue, venue) a drop-in adapter rather than a one-off branch. It is the
smallest step that converts the latent `SourceGroups` taxonomy into a real component model, and it directly
sets up named-risk grounding and a future verifier/synthesis pass. Chosen over starting with Availability
because the envelope must exist first or availability becomes another bespoke branch.

## 13. Ready-to-Paste Execution Prompt

```
Slice: Cross-Sport Source Envelope v1

Goal: introduce a reusable, normalized SourceSignalEnvelope and retrofit the existing market_odds and
starting_pitching signals onto it -- no new providers, no new sports. Establish the source-component
contract that later availability/fatigue/venue adapters drop into. Observed-only; changes no decision logic.

Skills gate: dai-skill-router, dai-grill-with-vault, superpowers:test-driven-development,
dai-test-discipline, verification-before-completion, dai-agent-handoff.

Repos: dai (code), dai-vault (docs).

Inspect first:
- dai/platform/dotnet/DevCore.Api/AgentRuns/SportsRetrievalOutput.cs
- dai/platform/dotnet/DevCore.Api/AgentRuns/SourceDepth.cs
- dai/platform/dotnet/DevCore.Api/AgentRuns/SourceSignalTaxonomy.cs  (SourceGroups, tiers)
- dai/platform/dotnet/DevCore.Api/AgentRuns/MarketDepth.cs
- dai/platform/dotnet/DevCore.Api/Sports/MlbStarterClient.cs
- dai-vault/02 Platform/architecture/sources/cross-sport-source-component-architecture-v1.md (this plan)
- dai-vault/02 Platform/architecture/inference/grpc-inference-orchestration-boundary-v1.md (envelope shape)

Scope:
- C# SourceSignalEnvelope record: categoryKey (SourceGroups), signalName, sourceProvider, status
  (observed|proxy|missing|stale|not_applicable), depthLevel (SourceDepthLevels), observed, freshness
  (fetchedAt/sourceUpdatedAt/ageSeconds/isStale), provenance (provider/sourceRef/matchConfidence),
  artifactSafeSummary, modelContextSummary, typed payload reference.
- Retrofit market_odds (from MarketDepth/BaseballMarketContext) and starting_pitching (from MlbStarterClient)
  into envelopes; keep existing SourceDepthRecord derivation, now sourced from the envelope.
- Pure mapping + fail-soft; missing/not_applicable carry no depth; stale derived from freshness threshold.

Non-goals: no new providers/sports; no gRPC/.proto; no confidence/posture/lean logic change; no
prompt-doctrine rewrite; no buyer-UI change; no migration; no Probe Refresh / Protocol Node Runner.

Tests (TDD): envelope mapping from market + starter signals; status/depth/not_applicable matrix;
freshness->stale; fail-soft on missing. Verify dotnet test (DevCore.Api.Tests) green.

Verification: unit tests green; one live MLB run shows market + starter envelopes populated with correct
status/depth/freshness; SourceDepth output unchanged in meaning; no buyer/confidence change.

Commit dai + dai-vault separately (conventional commits). Do not push unless asked. Handoff: result /
repo state / files changed / tests / verification / commit hashes / next slice (Availability Context v1).
```
