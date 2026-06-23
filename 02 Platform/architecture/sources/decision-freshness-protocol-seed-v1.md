# Decision Freshness Protocol — Seed v1

**date:** 2026-06-22
**status:** PLANNING SEED ONLY. Vault-only. No runtime behavior, code, prompt, confidence, buyer-UI, or
refresh/polling change. This is a direction note, not an implementation.
**seeded by:** Close-Market Confirmation Read v1 (the first concrete piece of this future protocol).

## Definition

A future protocol that compares a baseline artifact against refreshed source state and produces a
buyer-safe freshness read -- "is this read still good?" -- without re-running the whole decision or making
unvalidated product claims.

## Core concept (pipeline)

```
baseline artifact
  -> refreshed sources (market, lineup, starter, availability, fatigue, venue, ...)
  -> source deltas (what changed since generation)
  -> market support (did the market move toward / against the read?)
  -> line quality (was the earlier number better?)
  -> freshness summary (supported / weakened / stale / unavailable)
```

Close-Market Confirmation Read v1 already implements the "market support" leg (observed-only). The other
legs (lineup/starter/availability/fatigue/venue source deltas) reuse the Cross-Sport Source Envelope lanes
already shipped.

## Future buyer-safe language (not a claim until validated)

- Market support: "moved toward this read" / "moved against this read" / "stayed mostly neutral".
- Line quality: "the earlier number was better" (informational, not a CLV claim).
- Freshness: "new source data is available".
- What changed: market, lineup, starter, availability, fatigue, venue, or other source deltas.
- Current read: "supported" / "weakened" / "stale" / "unavailable".

## How it composes existing pieces

- **Market support** <- Close-Market Confirmation Read v1 + Market Movement Derivation + near_close capture.
- **Source deltas** <- re-resolving the Cross-Sport Source Envelope lanes (Availability, Schedule/Fatigue,
  Venue later, Market Odds Depth) at refresh time vs the baseline artifact's persisted envelopes.
- **Identity** <- the snapshot ledger's ProviderEventId + gamePk + AgentRun joins.

## Explicitly deferred (do NOT build under this seed)

- prompt changes;
- confidence changes;
- buyer UI / buyer-facing freshness claims before validation;
- automatic rerun / refresh of artifacts;
- source-delta synthesis (turning deltas into a new decision);
- paid / live polling;
- making freshness a product claim before it is outcome-validated.

## Recommended first real slice (when pursued)

**Decision Freshness Architecture v1** -- a planning/architecture slice that specifies the freshness read
contract (inputs: baseline artifact envelopes + refreshed envelopes + close-market confirmation; output: a
read-only freshness summary), staged like the market-memory line: derive read-only first, surface
internally, validate against outcomes, and only then consider any buyer claim.
