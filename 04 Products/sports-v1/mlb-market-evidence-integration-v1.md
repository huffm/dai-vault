# MLB Market Evidence Integration v1

**date:** 2026-06-17
**status:** IMPLEMENTED. Real MLB market evidence path built end to end (.NET retrieve + FastAPI analyzer). MLB runs with a grounded run line now ground the `market` signal (-> `market_odds`) and can reach `SourceSufficiency = moderate`. No advisory/enforcement, no buyer copy, no confidence/posture/threshold change, no migration, no live model spend. `DevCore.Api.Tests` 710/710; FastAPI 113/113.
**classification:** analyzer-behavior slice. MLB analyzer runs are now market-aware; future MLB leans/confidence may differ from the pre-change thin cohort. This is a deliberate new calibration regime.

**Anchor:** If market odds are evidence, the analyzer must see them.

## 1. Problem statement

The previous slice (MLB Market Odds Grounding v1) stopped because MLB market could not be grounded as decision-useful evidence while hidden from the model. This slice accepts the consequence: MLB market evidence is wired honestly into the analyzer so a starter+market MLB run grounds two decision-useful signals and leaves the `thin`-only bucket, producing the band variance the calibration plan requires.

## 2. Principal engineer review

1. **Exact MLB market path missing today?** Everything between the configured key and the model. The sport key `baseball_mlb` existed in `CompetitionCatalog` but was unread; there was no `market.baseball.spread` tool/handler, no MLB market fetch in `SportsRetriever`, no `market` in MLB expected signals, and no MLB market field in the analyzer request.
2. **Is `OddsApiKey="baseball_mlb"` enough to reuse `OddsMarketClient`?** Yes. The client is sport-generic; the new `GetBaseballSpreadAsync` mirrors `GetBasketballSpreadAsync` on the same `/v4/sports/{key}/odds?markets=spreads` endpoint. For MLB, `spreads` is the run line; the signed point (`-1.5` / `+1.5`) names the favorite, so it is honest directional market evidence.
3. **Smallest honest path odds -> analyzer evidence?** Retrieve in .NET (`market.baseball.spread` tool over `OddsMarketClient`), thread `BaseballMarketContext` through `SportsRetrievalOutput` -> `SportsAnalysisRequest` -> FastAPI `analyze_mlb`, which injects a `[market data]` block. Grounding is deterministic in .NET (`SportsRetrievalOutput`), exactly as for football/basketball.
4. **Where introduced?** All of: .NET retriever + grounding, the analyzer request payload, FastAPI `analyze_mlb` context + prompt, signal availability (`market` -> `market_odds` via existing `SourceSignalTaxonomy`), and the artifact's grounded-signal set. No new artifact version -- it rides existing fields.
5. **Does it require prompt/model input changes?** Yes -- by design. `analyze_mlb` now receives the run line and is instructed to use it in the market factor and name the favorite. This is the honest core of the slice.
6. **How is market prevented from being a hidden lean engine?** It is not hidden and not forced: it is one evidence block among starters; the model still emits its own `lean_side`; no code sets the lean from the line, no confidence/threshold/posture rule changed. It informs the decision (as evidence should) without dictating it.
7. **How are future runs distinguished as market-aware?** MLB `ExpectedSignalNames` now includes `market`, so a market-aware MLB run grounds `market` (visible in `GroundedSignals`/`SignalAvailability` and the moderate band). Pre-change runs grounded only `starting_pitching` (thin). The band/grounded-set is the regime marker.
8. **Artifact version bump?** No. The grounded-signal set, signal availability, and `BaseballMarketContext` use existing fields/shapes; no persisted contract field was added. Artifact v3 carries it.
9. **Dangerous to change here?** Letting the odds-event identity overwrite the MLB reconciliation key. Guarded: `SportsRetriever` keeps the statsapi `gamePk` as the canonical `(SourceProvider, ExternalGameId)` and discards the odds-event identity for MLB (test: `retrieve_grounds_mlb_market_and_keeps_statsapi_identity`).
10. **Overengineering avoided?** No moneyline/price extraction, no line-movement, no new artifact version, no per-tenant source policy, no buyer surface. Reused the basketball path verbatim.

## 3. Product architect review

1. **Product value?** The market line is the strongest single sports signal and the clearest credibility anchor.
2. **Credibility?** Real, sourced (bookmaker + timestamp) market evidence rather than priors-only reasoning.
3. **Supports "DAI agrees/disagrees with the market" later?** Yes -- the model now sees the line, so a future buyer slice can express agreement/disagreement. No such copy ships here.
4. **Immediate calibration variance?** Yes -- the first MLB path to `moderate`. (See the regime caveat in section 10.)
5. **Reusable across sports?** Yes -- same generic client/handler pattern as football/basketball.
6. **Tout-style claims prevented?** No buyer copy changed; the prompt instructs use of the line as a factor, not as a guarantee; no "beat the line"/edge language.
7. **Why before lineup/injury, weather, bullpen?** Lowest effort (provider+key already wired) and highest directional value; it is the natural first variance lever.

## 4. Existing gap (from MLB Market Odds Grounding v1)

Confirmed and now closed: `CompetitionCatalog.cs` had `OddsApiKey="baseball_mlb"` unread; `SportsRetriever` MLB branch fetched only starters; MLB expected only `starting_pitching`; no `market.baseball.spread`. The market signal was absent end to end -- this slice supplies the whole path.

## 5. Implementation summary

**.NET (`dai`):**
- `DevCore.AiClient/BaseballMarketContext.cs` (new) -- run-line market context; `SportAnalysisContracts.cs` -- request gains `BaseballMarketContext`.
- `Sports/GameIdentity.cs` -- `BaseballMarketGrounding`; `Sports/OddsMarketClient.cs` -- `GetBaseballSpreadAsync` (baseball family, `spreads`/run line).
- `Tools/Handlers/MarketSpreadHandlers.cs` -- `MarketBaseballSpreadHandler`; `Tools/ToolRegistry.cs` -- `market.baseball.spread` (Retriever, PlatformRetrieve, PaidExternal, 15m); `Tools/ToolGatewayServiceCollectionExtensions.cs` -- DI.
- `AgentRuns/SportsRetriever.cs` -- MLB branch fetches the run line via the gateway and keeps the statsapi `gamePk` as canonical identity; `AgentRuns/SportsRetrievalOutput.cs` -- accepts `baseballMarketContext`, grounds `market` when present; `AgentRuns/SportsAnalyzer.cs` -- passes it to the request.
- `Sports/CompetitionCatalog.cs` -- MLB `ExpectedSignalNames` = `["starting_pitching", "market"]`; `AgentRuns/ProbeFallbackCatalog.cs` -- MLB `market_odds` `future_candidate` -> `supported` (primary `market.baseball.spread`, High).

**FastAPI (`dai/services/agent-service`):**
- `app/models/sports.py` -- `BaseballMarketContext` + `SportsAnalysisRequest.baseballMarketContext`.
- `app/services/sports_analyzer.py` -- `analyze_mlb` gains `market_context` and injects a `[market data]` run-line block (mirror of basketball), or an explicit no-data instruction when absent.
- `app/routes/sports.py` -- passes `req.baseballMarketContext` to `analyze_mlb`.

## 6. Market evidence path

provider: odds-api (`baseball_mlb`, `markets=spreads` = run line) -> retrieval: `market.baseball.spread` tool over `OddsMarketClient.GetBaseballSpreadAsync`, called in the MLB branch of `SportsRetriever` -> analyzer input: `BaseballMarketContext` on `SportsAnalysisRequest` -> FastAPI `analyze_mlb` `[market data]` prompt block -> signal availability: `SportsRetrievalOutput` grounds `market` (source `odds_api`) deterministically when the context is non-null -> taxonomy: `market` -> `market_odds` (Critical) -> sufficiency: `starting_pitching` + `market` = 2 grounded decision-useful signals -> `moderate`.

## 7. Artifact version

Unchanged (v3). No persisted contract field added; the grounded-signal set, signal availability, and request context use existing shapes.

## 8. How this creates thin -> moderate variance

`SourceSufficiencyBuilder.DeriveBand` counts grounded decision-useful signals: 1 -> thin, >=2 -> moderate. MLB previously grounded only `starting_pitching` (thin). With a grounded run line it grounds `starting_pitching` + `market` -> `moderate`. Verified: `mlb_starting_pitching_plus_market_reaches_moderate_band`.

## 9. Why this is NOT lean-neutral

The model now receives the run line in its prompt and is told to use it in the market factor. The line is the strongest directional signal, so MLB `lean_side`/`confidence` distributions will shift versus the pre-change thin cohort. This is intended and honest -- the prior slice stopped precisely because pretending otherwise was impossible. No code sets the lean from the market; the shift is the model reasoning over more evidence.

## 10. Why post-change runs are a new market-aware calibration cohort

Pre-change MLB runs were produced by a starter-only analyzer; post-change runs are produced by a market-aware analyzer. They are different decision processes. **Do not compare pre-change thin vs post-change moderate hit rates as if only the band changed** -- the model input changed too. Calibration must compare thin vs moderate *within the market-aware regime* (both generated after this slice), or use a held-out design. This supersedes the implicit lean-neutral assumption in Source Coverage and Calibration Variance Plan v1.

## 11. Why buyer/advisory/enforcement remain unchanged

No buyer surface, no advisory mode, no enforcement, no Probe, no Tool Gateway authority change, no confidence threshold, no posture rule. `PerceiveFulfillment` stays observed/read-only; a moderate MLB read simply projects `Fulfilled`/observed instead of `FulfilledWithThinCoverage` when two signals ground. Standing decision (observed-only) is intact.

## 12. Tests and verification

New/updated tests:
- `SportsRetrievalOutputTests`: baseball market grounds `market`; starter+market grounds both; no market -> market missing.
- `SourceSignalTaxonomyTests`: starter+market -> moderate; `market` -> `market_odds`.
- `SportsRetrieverTests`: gateway-routed MLB market grounds `market` AND keeps statsapi `gamePk` identity (reconciliation-key guard); existing MLB tests updated to register the new handler.
- `ProbeFallbackCatalogTests` / `SportSufficiencyProfileTests`: MLB `market_odds` now `supported` (was `future_candidate`), still recommended-not-required.
- FastAPI `test_sports_analyzer.py`: `analyze_mlb` includes the `[market data]` block when context present; the no-data instruction when absent.

Verification (evidence): `DevCore.Api.Tests` **710/710**; FastAPI `pytest` **113/113**. No live model call (tests mock `_call_model`); no live generation; no reconciliation; no migration; no buyer/frontend change; diff is code+tests only.

## 13. What remains deferred

advisory mode; soft/hard enforcement; buyer market-disagreement copy; confidence threshold changes; posture changes; live Probe execution; lineup/injury source; weather/park source; bullpen source; richer MLB moneyline/price + line-movement enrichment; WNBA. Also: actually generating a market-aware MLB cohort (a budgeted model-spend slice) and the within-regime calibration comparison.

## 14. Recommended next slice

**Market-Aware MLB Stage 0 Capture v1** (budgeted, model spend): generate a batch of market-aware MLB runs (some grounding the run line -> moderate, some pre-market/odds-missing -> thin) before settlement, then reconcile post-settlement. This yields the first *within-regime* thin-vs-moderate comparison for PerceiveFulfillment calibration -- the data the variance plan needs -- without the pre/post-regime confound. Companion (free): also capture `PrimaryFulfillmentRequired` (pre-announcement) and `NoDirectionalSeparation` (null-lean) cases. No advisory/enforcement until that data exists.
