# Decision Freshness Architecture v1

**date:** 2026-06-22
**status:** ARCHITECTURE + REQUIREMENTS PLAN. Vault-only. No code, migration, runtime, prompt, buyer-UI,
confidence/lean/posture/reconciliation, refresh-execution, or scheduler change. Planning only.
**builds on:** `decision-freshness-protocol-seed-v1.md`, `close-market-confirmation-read-v1.md`,
`market-memory-layer-v1.md`, `cross-sport-source-envelope-v1.md`, the availability/fatigue/movement docs.

## 1. Purpose

A decision artifact is generated at a single moment; market and source data keep changing. Users need to
know whether a read is **still supported, weakened, or stale** -- not just what it said when generated.
Decision Freshness is a read-only, buyer-facing interpretation over **source deltas** (baseline vs
refreshed source state). It is NOT a new prediction by itself: it re-reads already-shipped signal lanes and
the market-memory chain, and reports what changed.

The first concrete leg already exists: Close-Market Confirmation Read v1 (did the near-close market move
toward/against the lean). This plan generalizes that into a full freshness read across all source lanes.

## 2. Buyer-First Product Definition

The value is plain-language, not jargon:

- **What changed** since this artifact was generated?
- **Market support** -- did the market move toward or against this read?
- **Line quality** -- did the number get worse?
- **Source updates** -- did new lineup, availability, starter, fatigue, or venue data appear?
- **Freshness read** -- is the original read still supported, weakened, or stale?

Buyer copy uses "market support", "line quality", "source updates", "what changed", "freshness read", and
"still supported / weakened / stale / unavailable". It NEVER exposes CLV, basis points, implied-probability
deltas, provider event ids, or raw line-movement tables (those stay internal/diagnostic).

## 3. Current Infrastructure (what exists)

- **SourceSignalEnvelope** (persisted on the artifact via `AgentRunExecutionResult.SourceEnvelopes`):
  per-signal status (observed/proxy/missing/unavailable/not_attempted/not_applicable), depth, observed
  flag, freshness (UpdatedAtUtc), provenance, and buyer-safe + model-context summaries. Lanes shipped:
  `market_odds`, `market_movement`, `starting_pitching`, `lineup_injury`, `bullpen_availability`.
- **Market memory chain:** ledger (artifact_generation + near_close batches) -> query -> movement
  derivation -> movement envelope -> closing capture -> near-close trigger -> tracked resolver ->
  close-market confirmation. Batches join by ProviderEventId + gamePk + AgentRunId.
- **Close-Market Confirmation Read v1:** `GET /api/agent-runs/{id}/close-market-confirmation` -- read-only
  moved_toward/against/neutral vs the lean; TrueClvEvaluable=false (no formal market selection yet).

**What this makes possible:** a baseline (the artifact's persisted source envelopes + its
artifact_generation market snapshot + its lean) and a way to re-derive current state, so deltas are
computable.
**What is still missing:** a unified freshness read model that composes (a) market support/line quality
from close-market confirmation and (b) per-lane source deltas, plus the refresh path that produces the
refreshed envelopes to compare against.

## 4. Feature Boundary

Decision Freshness = **baseline artifact + refreshed source state + source deltas + buyer-safe summary**.

Clarifications (what freshness is NOT):
- not confidence (confidence is coherence at generation; freshness is change-over-time).
- not reconciliation (reconciliation needs a settled outcome; freshness is pre-outcome).
- not full CLV (artifacts carry a directional lean, not a formal market selection).
- not automatic rerun/regeneration (v1 plans the read only).
- not buyer UI yet (internal read first).

## 5. Domain Model (conceptual)

- **DecisionFreshnessRequest** -- AgentRunId, tenant, optional refresh policy (which lanes to refresh).
  *Future input.*
- **DecisionFreshnessBaseline** -- the persisted snapshot to compare against: the run's source envelopes
  (from OutputJson), its artifact_generation market batch, its LeanSide, generatedAtUtc, AgentRunId.
  *Derivable now from persisted data.*
- **DecisionFreshnessRefresh** -- the refreshed source state: re-resolved current envelopes + latest market
  snapshot/movement. *Future (needs a refresh path).*
- **SourceDelta** -- per lane: lane key, baseline status/depth, refreshed status/depth, delta
  classification (see 8), buyer-safe note. *Computed from baseline + refresh.*
- **MarketSupportDelta** -- from close-market confirmation: moved_toward/against/neutral, consensus flip.
  *Exists now (close-market confirmation).*
- **LineQualityDelta** -- lean-side number better/worse/similar between baseline and current. *Derivable
  from market snapshots; internal numeric, buyer copy only.*
- **AvailabilityDelta / FatigueDelta / StarterDelta / VenueDelta** -- per-lane SourceDelta specializations
  (lineup_injury, bullpen_availability, starting_pitching, weather_park later). *Computed from the lanes.*
- **DecisionFreshnessResult** -- overall status (see 11), the per-lane deltas, market support + line
  quality, a buyer-safe summary, and a diagnostic note. *Future read model.*
- **BuyerSafeFreshnessSummary** -- the single plain-language sentence(s) for the buyer. *Copy-calibrated
  later.*

## 6. Baseline Semantics

Baseline = what DAI saw at generation, never overwritten:
- the run's persisted `SourceEnvelopes` (in OutputJson) -- the per-lane status/depth at generation.
- the `artifact_generation` market snapshot batch (consensus, implied probs, lines).
- the artifact's structured `LeanSide`.
- `generatedAtUtc` (StartedUtc), the linked AgentRun.

Rules: baseline is immutable; refresh compares against it and never mutates it; multiple refreshes over
time are each compared to the SAME baseline, so a freshness trajectory is possible.

## 7. Refresh Semantics

A refresh re-fetches eligible current sources and compares to baseline. In v1 it may optionally create a
new artifact LATER, but NOT in this slice -- planning only.

- **manual / user-triggered refresh** -- the first real mode (smallest, no scheduler).
- **near-close refresh** -- reuse the existing near-close capture (already operational) as the market leg.
- **scheduled refresh** -- deferred (no scheduler this line).
- **provider-cost guardrails** -- each refreshed lane that calls a provider is a paid/rate-limited call;
  prefer reusing already-persisted near_close/market data before re-fetching; cap refresh frequency.

## 8. Source Delta Semantics

Each lane's delta classifies baseline status/depth vs refreshed status/depth:
- **unchanged** -- same status + depth.
- **strengthened** -- depth/status improved (e.g. lineup missing -> observed; market shallow -> enriched).
- **weakened** -- status/depth regressed (e.g. observed -> missing; enriched -> shallow).
- **newly_available** -- baseline missing/unavailable, now observed (e.g. lineups posted since generation).
- **no_longer_available** -- baseline observed, now missing (rare; e.g. retraction).
- **stale** -- present but older than a freshness threshold (the `stale` status reserved earlier).
- **unavailable** -- provider could not supply on refresh.
- **not_applicable** -- lane does not apply to this sport/game.
- **not_comparable** -- baseline or refresh lacks the structured data to compare.

Applies to: market support (close-market confirmation), line quality, lineup/availability
(`lineup_injury`), bullpen/fatigue (`bullpen_availability`), starter context (`starting_pitching`), and
venue/weather (`weather_park`) later.

## 9. Market Support and Line Quality

**Market support** reuses Close-Market Confirmation Read v1 directly:
- "Market moved toward this read." / "Market moved against this read." / "Market stayed mostly neutral." /
  "No close-market comparison available yet." True CLV stays deferred until artifacts carry an explicit
  market selection (type/side/line/price).

**Line quality** (lean-side number better/worse) -- internal numeric (implied probability or price), buyer
copy only:
- "Earlier number was better." / "Current number is less favorable." / "Current number is similar." /
  "Line comparison unavailable." Never expose the numeric delta to buyers.

## 10. Buyer-Safe Language Rules

Compact vocabulary (use):
- "Market moved toward this read." / "Market moved against this read." / "Market stayed mostly neutral."
- "Earlier number was better." / "New source data is available."
- "This read still has market support." / "This read has weakened since generation." / "Freshness
  unavailable."

Forbidden in buyer copy: "guaranteed edge", "sharp", "lock", "CLV", basis points, implied-probability
deltas, provider event ids, raw line tables, and any confidence overclaim.

## 11. Readiness / Status Model

Overall freshness status (diagnostic, NOT a decision mutation):
- **fresh_supported** -- market support toward (or stable) AND no weakening source deltas.
- **fresh_neutral** -- neutral market, no material source change.
- **weakened** -- market moved against, or a key lane weakened (lineup/starter/availability).
- **stale** -- baseline is old and refreshed data materially differs / new data appeared.
- **unavailable** -- cannot compute (no baseline, no refresh, or no comparable signal).
- **not_comparable** -- baseline/refresh present but not structurally comparable.
- **insufficient_history** -- market support needs >=2 snapshots (carried from the movement read).

## 12. Interaction with Confidence and Lean

- No confidence change initially; no lean mutation initially. Freshness is observed-only.
- Freshness statuses are logged and (later) calibrated against outcomes + market confirmation.
- Only AFTER outcome evidence shows freshness discriminates may it influence confidence or synthesis --
  same staged discipline as the rest of the source line.

## 13. Interaction with Artifact Surfaces (stages)

1. internal read-only service (a `GET .../freshness` like close-market-confirmation).
2. internal artifact DTO projection (derive-on-read field, like ArtifactDirectionConsistency).
3. dev/buyer-safe preview (internal copy review).
4. buyer artifact section (only after copy is validated).
5. refresh/rerun UX (last; only after backend semantics + copy stabilize).
No buyer UI now.

## 14. Implementation Slice Plan (ordered)

1. **Decision Freshness Read Model v1** -- combine baseline (persisted envelopes + artifact_generation
   batch + lean) + close-market confirmation + per-lane source-envelope state into a single read-only
   `DecisionFreshnessResult`. *Scope:* model + service over existing persisted data; no provider calls (use
   close-market confirmation + persisted baseline envelopes). *Non-goals:* refresh path, buyer UI,
   confidence. *Tests:* status mapping (fresh_supported/neutral/weakened/unavailable), market-support
   reuse, buyer-safe summary. *Verify:* read for run e5b5433e returns a freshness result.
2. **Source Delta Calculator v1** -- pure compare of baseline envelopes vs refreshed envelopes -> per-lane
   SourceDelta (classifications in 8). *Tests:* each classification. *Verify:* deterministic over seeded
   pairs.
3. **Refresh Trigger v1** -- manual/user-triggered source refresh (re-resolve current envelopes + reuse
   near_close), no artifact regeneration. *Tests:* refresh produces a refreshed envelope set; cost guard.
   *Verify:* one live MLB refresh.
4. **Decision Freshness Artifact Projection v1** -- internal artifact DTO derive-on-read field only.
   *Tests:* additive DTO, backward compatible. *Verify:* /artifact carries freshness internally.
5. **Buyer-Safe Freshness Copy v1** -- copy calibration against real artifacts (internal review). *Tests:*
   copy-safety (no forbidden terms). *Verify:* sample artifacts.
6. **Refresh/Rerun UX v1** -- buyer-facing, only after backend + copy stabilize.
7. **Freshness Calibration v1** -- compare freshness statuses against settled outcomes / market
   confirmation (read-only calibration).

## 15. Recommended Immediate Next Slice

**Decision Freshness Read Model v1.** It uses only already-persisted data (baseline envelopes + the
artifact_generation batch + close-market confirmation), needs no provider calls, no buyer UI, and no
artifact-contract change. It is the first piece that unifies market support + per-lane source-envelope
state into one read-only freshness result, and it is directly verifiable against the live-verified run
e5b5433e. Every later slice (delta calculator, refresh, projection, copy, calibration) depends on this read
model existing.

## 16. Ready-to-Paste Execution Prompt

```
Slice: Decision Freshness Read Model v1

Goal: a read-only DecisionFreshnessResult that combines an artifact's baseline (its persisted
SourceEnvelopes + artifact_generation market batch + LeanSide + generatedAt) with Close-Market Confirmation
Read v1 and the per-lane source-envelope state, producing an overall freshness status + buyer-safe summary.
No provider calls, no refresh path, no buyer UI, no confidence/lean/posture/reconciliation change.

Skills gate: dai-skill-router, dai-grill-with-vault, superpowers:test-driven-development,
dai-test-discipline, systematic-debugging, verification-before-completion, dai-agent-handoff.

Repos: dai (code), dai-vault (docs).

Preflight: git status/log both repos; confirm clean/synced; do not push. Inspect:
- dai-vault/02 Platform/architecture/sources/decision-freshness-architecture-v1.md (this plan; sections 5,6,8,11)
- dai-vault/04 Products/sports-v1/close-market-confirmation-read-v1.md
- dai/platform/dotnet/DevCore.Api/AgentRuns/CloseMarketConfirmation*.cs
- dai/platform/dotnet/DevCore.Api/AgentRuns/SourceSignalEnvelope.cs
- dai/platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs (AgentRunExecutionResult.SourceEnvelopes)
- dai/platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs (GetArtifact + close-market-confirmation)

Scope:
- DecisionFreshnessResult DTO: AgentRunId, overall Status (fresh_supported/fresh_neutral/weakened/stale/
  unavailable/not_comparable/insufficient_history), MarketSupport (reuse close-market confirmation:
  direction + summary), per-lane source-envelope freshness (lane key + baseline status/depth, observed
  flag), BuyerSafeSummary, DiagnosticNote.
- pure DecisionFreshnessCalculator: given close-market-confirmation result + baseline source envelopes ->
  overall status + buyer-safe summary. (market against OR a weakened key lane -> weakened; toward/stable +
  no weakening -> fresh_supported; neutral -> fresh_neutral; missing inputs -> unavailable/insufficient.)
- DecisionFreshnessService: load the run's persisted SourceEnvelopes (OutputJson) + call
  ICloseMarketConfirmationService, build the result. read-only.
- dev-gated GET /api/agent-runs/{agentRunId}/freshness (tenant-scoped, like close-market-confirmation).

Hard constraints: no provider calls; no refresh/rerun; no scheduler; no buyer UI; no prompt/confidence/
lean/posture/reconciliation change; no artifact-contract migration; no CLV; lowercase ascii comments.

Tests (TDD): calculator status mapping (toward->fresh_supported, against->weakened, neutral->fresh_neutral,
no market support->unavailable/insufficient_history, weakened lane->weakened); service builds from a seeded
run + envelopes + close-market confirmation; buyer-safe summary contains no forbidden terms. Run
dotnet test (DevCore.Api.Tests) to green.

Verification: GET /api/agent-runs/{e5b5433e}/freshness -> a freshness result (likely fresh_neutral, market
neutral); confirm no buyer/confidence/reconciliation change. No provider call occurs.

Docs: dai-vault/04 Products/sports-v1/decision-freshness-read-model-v1.md. Append current-slice.md handoff.
Commit dai + dai-vault separately (conventional commits). Do not push unless asked. Final handoff in ONE
fenced code block.
```
