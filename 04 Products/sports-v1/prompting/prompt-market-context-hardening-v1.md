# Prompt Market Context Hardening v1

**date:** 2026-07-08
**status:** implemented (dai `ce8f21f`, agent-service only); live registry-routed paid canary PASS
**classification:** first authorized prompt-behavior change since the market-attribution failure chain; prevention half of the Option E staged hybrid (guard shipped 2026-07-07 as the detection half).

## why

the attribution debug (2026-07-07) located the mutation point: model generation enabled by
(1) a bare side-label consensus line ("moneyline consensus favors the {side} side." -- team never
named, the model resolves "away side" to its own lean's team) and (2) a raw home-only median
(not de-vigged, rendered "50%" beside an away consensus). corpus correlation: incoherent blocks
misattributed 5/7 vs coherent 3/30. guard baseline on 285 rows: Pass 72 / FAIL 10 / Unclear 203.

## what changed (backed_depth market depth block only)

1. **team-named consensus:** `moneyline consensus favors the away side (Chicago Cubs).`
2. **de-vigged both-side probabilities** replace the raw home-only median:
   `de-vigged win probabilities: {home} 48% | {away} 52%.` (new pure `devig_pair` in
   sports_analyzer: normalizes the two-way implied pair to sum to 100%; degenerate total
   falls back to raw). if the away median is ever absent, the fallback line is explicitly
   labeled `median implied win probability (home side only, raw): ...` -- and that partial
   shape still fails closed in registry assembly (no template for it, by standing policy).
3. **required market-vs-lean acknowledgment** appended to the depth guidance: if the lean
   opposes the consensus, the model must say so in lean and summary, naming the
   market-favored team; never claim market support the consensus does not give.

single-book (run-line-only) and no-market branches are untouched. no confidence/threshold/
posture/buyer/schema/.NET change; the wire already carried medianAwayImpliedProb.

## registry reversioning (this slice owns the invariants)

- new `mlb.overlay.market.backed_depth.v2.txt` (sha256 727ab756...62f), byte-identical to the
  hardened live full shape; v1 overlay lifecycle -> deprecated.
- 4 recipes reversioned: `mlb.pregame.analysis.starter_{named,enriched,missing,asymmetric}
  _market_backed_depth.v2` (v1 recipes deprecated -- selection stays unambiguous/fail-closed).
- slot oracle (`migration_readiness._market_slots`): +consensus_team, +devig_home_prob/
  devig_away_prob (shared `devig_pair`), -median_home_prob.
- manifest integrity OK: 10 templates / 14 recipes; hash-verified on load.

## verification

- TDD: new `tests/test_market_context_hardening.py` (12 tests: devig math incl. degenerate,
  team naming both sides, both-side display, labeled fallback, acknowledgment presence and
  absence, v2 selection for all 4 backed_depth regimes, v2 byte-equality). stale v1
  expectations updated in 8 test files. **full agent-service suite 448 passed / 0 failed**
  (436 baseline + 12). `check_prompt_manifest.py` OK exit 0. `git diff --check` clean.
- the 9-regime live-vs-shadow byte-equality lattice re-proves registry==live after the change.
- **live paid canary (1 gpt-4o-mini call, operator-approved):** COL@LAD 823928 screened
  ELIGIBLE via /source-readiness (enriched Sasaki/Hughes, 9 books, consensus home);
  agent-service started with DAI_MLB_REGISTRY_PROMPT_CANARY=1 PROCESS-SCOPED (.env untouched),
  one run, service stopped after. run `aa46433e`: promptSource=registry, recipe
  `...backed_depth.v2@v2`, attributionStatus=complete, observed==selected regime, conf 0.80,
  lean home, marketAgreement=true; **guard: Pass / prose_matches_staged_consensus /
  MarketAligned**; summary prose names the market side explicitly. run excluded `diagnostic`
  immediately per capture-closeout-run-eligibility-rule-v1 (first live application);
  duplicate-active sweep remains 0.
- dai-code-reviewer: approve with notes (cosmetic rounding-sum note; no required fixes).

## measurement discipline

v2-route rows are a DISTINCT regime era: never pool v2 attribution rates with v1 rows. the
frozen baseline for comparison is Pass 72 / FAIL 10 / Unclear 203 (285 rows, 2026-07-07).
improvement is claimable only from settled hardened cohorts, not from the n=1 canary.

## next

Hardened-Regime Baseline Measurement v1 after the next capture cohort settles on v2 routes;
capture-cadence resumption remains a separate operator decision.
