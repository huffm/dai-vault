# MLB Market Odds Grounding v1

**date:** 2026-06-17
**status:** STOPPED -- docs-only. No code, no tests, no model calls, no generation, no reconciliation, no migration, no advisory/enforcement, no Probe, no buyer change. Honest-gap finding: MLB market odds are not in the artifact evidence path today, and grounding them honestly is an analyzer-behavior change (moves the lean) that this slice's boundary forbids.
**classification:** investigation + corrected-path recommendation. The source integration is real and worthwhile, but it is the wrong shape for a "grounding only, do not move the lean" slice.

**Anchor:** Ground the source; do not move the lean. (Here the two cannot both hold, so the slice stops.)

## 1. Problem statement

Source Coverage and Calibration Variance Plan v1 recommended grounding MLB `market_odds` first to lift MLB runs from the structurally-pinned `thin` band to `moderate`, on the stated assumption that odds-api baseball was "existing plumbing -- another sport key on the same client." This slice was to implement that grounding as evidence only, without changing LeanSide, confidence, or posture. Investigation shows the assumption was partly wrong, and the honest implementation is out of scope for an evidence-only slice.

## 2. Principal engineer review

1. **Where does MLB odds data currently exist?** Only as a configuration constant: `CompetitionCatalog` MLB carries `OddsApiKey="baseball_mlb"` (`CompetitionCatalog.cs:111`). Nothing reads it. No MLB odds are fetched, stored, or passed to the analyzer.
2. **Is there already odds-api baseball plumbing?** No usable path. `OddsMarketClient` is generic (keyed by sport), but only two handlers wrap it -- `MarketFootballSpreadHandler` and `MarketBasketballSpreadHandler` (`MarketSpreadHandlers.cs`), registered as `market.football.spread` and `market.basketball.spread` (`ToolRegistry.cs:112-113`). There is no `market.baseball.spread` tool, handler, or DI registration.
3. **Does MLB analysis currently receive market odds as evidence?** No. `SportsRetriever.cs:79-92` (the MLB branch) invokes only `pitching.mlb.probable_starters`. No market tool is called; `basketballMarket`/`footballMarket` stay null for MLB.
4. **Does the artifact already contain enough market info but fail to mark the signal grounded?** No. MLB `ExpectedSignalNames=["starting_pitching"]` (`CompetitionCatalog.cs:112`); `market` is not expected, not retrieved, not present. This is not a missing-trace/projection case -- the evidence is genuinely absent.
5. **Would grounding market_odds require changing the model prompt or generated output?** Yes. Honest grounding means fetching the line and passing it to the FastAPI analyzer as evidence; the model would then consume the single most directional sports signal and produce different leans/confidence. The grounded-signal accounting (`SportsRetrievalOutput.GroundedSignals`) and the sufficiency band (`SourceSufficiencyBuilder.DeriveBand`, which counts every grounded signal as decision-useful) both treat a grounded market signal as decision-useful by construction.
6. **If yes, is that out of scope for this slice?** Yes. The slice's stated gate: "No prompt/model behavior changes unless the principal engineer review concludes the market signal is already part of the evidence path and only trace/projection is missing." The market signal is NOT already in the MLB evidence path, so the gate is not met. The boundary also lists "Change LeanSide behavior" and "Change confidence thresholds" as forbidden, and the anchor is "ground the source; do not move the lean."
7. **Smallest honest implementation that grounds market_odds without changing LeanSide/confidence/posture?** None exists. In this architecture, evidence reaches the artifact only through the analyzer (the model). A grounded decision-useful signal is, by definition, fed to the decision. There is no "grounded but withheld from the model" channel, and `DeriveBand` has no grounded-but-not-counted tier. So you cannot honestly mark market grounded while guaranteeing the lean is untouched.
8. **How do we prevent market from becoming a hidden lean driver?** We cannot, if it is grounded honestly -- it is *meant* to inform the decision. The only way to keep the lean untouched is to not feed it to the model, which means it is not honestly grounded. That contradiction is why this slice stops.
9. **How do we verify market creates moderate only when actually grounded and decision-useful?** That verification is exactly what a future, correctly-scoped slice must build (deterministic grounding from a real fetch, plus tests that thin->moderate happens only on a genuine non-null fetch). It cannot be verified here because the fetch does not exist.
10. **Overengineering to avoid:** faking a grounded market signal (e.g. marking `market` grounded from the static `OddsApiKey` without a real fetch), or fetching odds but hiding them from the model to "preserve the lean" -- both corrupt the band semantics the calibration plan depends on.

## 3. Product architect review

1. **Buyer/product value of market_odds?** High in principle -- the consensus line is the strongest single signal and the clearest credibility anchor.
2. **Agree/disagree with the market?** Yes, that is its main product value -- but expressing it requires the model to actually see the line, which is the analyzer change this slice forbids.
3. **Improves credibility without tout-style claims?** Yes if handled as internal evidence first; buyer "we disagree with the market" copy stays deferred regardless.
4. **Immediate calibration variance?** Only via an analyzer change. And note a subtlety below (section 8): the variance it produces is entangled with a model-behavior change, so it is not a clean "more evidence, same model" comparison.
5. **Reusable across sports?** Yes -- the generic `OddsMarketClient` already serves football/basketball; baseball would follow the same pattern.
6. **Language that must not leak to buyer surfaces yet?** Any market-disagreement / "beat the line" / edge-vs-market copy. None of that ships here.
7. **Why before lineup/injury, weather, bullpen?** It remains the best *eventual* first source (lowest provider effort -- key already configured; highest directional value). The sequencing recommendation stands; only the slice *shape* was wrong (it is an analyzer-evidence slice, not a no-lean grounding slice).

## 4. Existing odds plumbing finding (summary)

| Element | State for MLB | Evidence |
|---|---|---|
| odds-api sport key | present but unused | `CompetitionCatalog.cs:111` `OddsApiKey="baseball_mlb"` |
| generic odds client | exists (sport-agnostic) | `OddsMarketClient` via football/basketball handlers |
| baseball market tool/handler | **absent** | only `market.football.spread`, `market.basketball.spread` in `ToolRegistry.cs:112-113` |
| MLB retriever market call | **absent** | `SportsRetriever.cs:79-92` calls only probable starters |
| MLB expected `market` signal | **absent** | `CompetitionCatalog.cs:112` `["starting_pitching"]` |
| analyzer/artifact market evidence (MLB) | **absent** | no context threaded; all 9 reconciled runs grounded `[identity_schedule, starting_pitching]` only |

Conclusion: the sport key is the only piece present. The retrieve -> analyzer -> signal-availability path for MLB market does not exist.

## 5. Implementation summary / docs-only stop reason

No code was written. Honest MLB market grounding requires, at minimum: a `market.baseball.spread` tool + `MarketBaseballSpreadHandler` (wrapping the existing `OddsMarketClient` with `baseball_mlb`), DI + `ToolRegistry` registration, a market call in the MLB branch of `SportsRetriever`, adding `market` to MLB `ExpectedSignalNames`, threading a `BaseballMarketContext` through `SportsRetrievalOutput` into the FastAPI analyzer input, and the analyzer emitting a grounded `market` signal. The analyzer consuming the line **changes the model's directional inputs**, moving LeanSide/confidence -- forbidden by this slice. Per the slice's own stop rule, we document the gap instead of forcing a false signal.

## 6-10. (Deferred to the corrected slice)

- **How market becomes grounded / maps to market_odds:** `SourceSignalTaxonomy` already maps the `market` signal -> `market_odds` (Critical) (`SourceSignalTaxonomy.cs:74`), so the taxonomy side is ready; the missing half is the real fetch + analyzer evidence. To be built in the corrected slice.
- **thin -> moderate variance:** would occur once a genuine non-null baseball market fetch grounds a second decision-useful signal alongside `starting_pitching` (`DeriveBand`: 2 grounded -> moderate). Not attainable until the fetch exists.
- **Why this does not change LeanSide/confidence/posture:** it cannot avoid changing them if grounded honestly -- which is precisely why the slice stopped rather than ship.
- **Why no advisory/enforcement:** untouched; this slice changed nothing.

## 11. Tests and verification

No code changed, so no tests were added or run (correct per TDD: there is nothing to drive red->green, and faking a signal to make a test pass would be the dishonesty the boundary forbids). Verification performed: read-only code inspection (`SportsRetriever.cs`, `CompetitionCatalog.cs`, `ToolRegistry.cs`, `MarketSpreadHandlers.cs`, `SourceSignalTaxonomy.cs`, `SportSufficiencyProfile.cs`, `ProbeFallbackCatalog.cs`); confirmed `dai` working tree clean (no code change); `ProbeFallbackCatalog` MLB `market_odds` left as `future_candidate` (NOT promoted to supported, because no honest current path exists). No model call, no generation, no reconciliation, no migration, no buyer/frontend change.

## 12. What remains deferred

advisory mode; enforcement; buyer display; market-disagreement copy; live Probe execution; lineup/injury source; weather/park source; bullpen source; WNBA. Additionally now explicitly deferred: the MLB market integration itself, until it is re-scoped as an analyzer-evidence slice with a fresh calibration baseline (below).

## 13. Recommended next slice

**MLB Market Evidence Integration v1** -- re-scoped honestly as an *analyzer-behavior* slice, not a no-lean grounding slice. Scope:
- build the real path: `market.baseball.spread` tool/handler over the existing `OddsMarketClient` (`baseball_mlb`), DI + `ToolRegistry` registration, MLB-branch fetch in `SportsRetriever`, `market` added to MLB `ExpectedSignalNames`, `BaseballMarketContext` threaded into the analyzer input, analyzer emits a grounded `market` signal; promote `ProbeFallbackCatalog` MLB `market_odds` to `supported` only once the fetch is real.
- **explicitly accept** that MLB leans/confidence change from this point (the model now sees the line). Treat it as a new analyzer baseline.
- **calibration caveat to carry (important):** moderate-band MLB runs produced after this change come from a *different decision process* than the existing thin-band cohort. Comparing pre-change thin vs post-change moderate hit rates conflates "more evidence" with "changed model behavior." The calibration comparison must therefore be within the post-change regime (thin vs moderate runs both generated by the market-aware analyzer), or use a held-out design -- not pre/post the analyzer change. This corrects an implicit assumption in Source Coverage and Calibration Variance Plan v1.
- boundaries unchanged: no advisory/enforcement, no buyer market copy, no Probe, no threshold change; TDD; full `DevCore.Api.Tests` + relevant FastAPI tests; no-spend local verification where possible.

Alternative if a no-analyzer-change variance source is preferred first: capture the **free** negative-state variance from the plan (pre-announcement `PrimaryFulfillmentRequired`, null-lean `NoDirectionalSeparation`) before any analyzer change, so some variance exists without touching the model.
