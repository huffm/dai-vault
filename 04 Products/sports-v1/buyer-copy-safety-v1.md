# Buyer Copy Safety v1

**date:** 2026-06-09
**status:** implementation slice. prompt, parser, and buyer projection hardening. no new data sources, no source/fallback catalog, no probe refresh activation, no confidence/posture/lean mutation engine, no schema change, no tenant/auth/billing work.
**scope:** keep buyer artifacts directional and useful while removing betting-edge posture and internal diagnostic leak-through from buyer-facing copy.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault`, `dai-token-tight`, `dai-agent-handoff`.
- local runtime skill (read-only): `dai-signal-follow-up-diagnostics` for signal vocabulary and internal diagnostic terms.
- superpowers, applied manually: `writing-plans`, `test-driven-development`, `verification-before-completion`.

## product doctrine implemented

Buyer copy should read like a composed analyst, not a debug report.

Buyer-facing copy may state direction:

- slight lean toward a team
- cautious lean toward a team
- available evidence supports the direction
- mixed read or no clear lean
- measured confidence
- not strong enough for a firmer stance

Buyer-facing copy must not use betting-edge posture:

- edge toward
- best bet
- prediction
- lock
- sharp side
- value play
- expected to cover
- should win
- take or hammer language

Buyer-facing copy must not expose internal implementation gaps:

- `sharp_public`
- missing signal
- unavailable signal
- fallback failed
- probe required
- source unavailable
- confidence capped because a named signal is missing

Those details remain valid and useful in internal diagnostics, artifact quality warnings, signal availability records, signal follow-up records, and dev artifact review surfaces.

## implementation decision

The slice uses a layered boundary rather than a broad copy rewrite.

1. The FastAPI analyzer prompt now instructs top-level buyer fields (`lean`, `summary`, `factors`, `counter_case`, `watch_for`, `what_would_change_the_read`) to avoid betting posture and internal gap terms.
2. The FastAPI parser applies a small deterministic buyer-copy sanitizer to those top-level fields. This protects the buyer surface if the model drifts or older prompt habits reappear.
3. The internal `protocol` object is not sanitized. It can still name `missing sharp_public confirmation`, probe gaps, and other diagnostic facts for calibration and dev review.
4. The Angular buyer signal mapper no longer renders non-grounded signal inventory rows such as `Sharp vs Public - Not available`. It replaces missing/unavailable/proxy/unknown rows with one aggregate `Confirmation strength` row.
5. `signals_used` remains internal/dev-only. The prompt now tells the model to list only categories with real context blocks, and the existing quality checker still flags ungrounded `signals_used` claims for internal calibration.

## buyer-facing behavior

Allowed example:

> Slight lean toward Celtics. The available evidence supports the direction, but not strongly enough to justify a firmer stance.

Signal Summary now shows grounded support rows plus aggregate confirmation language when internal gaps exist:

- Market / Spread - Supported
- Rest & Schedule - Supported
- Confirmation strength - Measured

It does not show:

- Sharp vs Public - Not available
- missing `sharp_public`
- fallback failed
- probe required

## dev/internal behavior preserved

The following remain explicit where they belong:

- `signalAvailability`
- `signalFollowUps`
- `missingSignals`
- `artifactQualityWarnings`
- `signals_used contains ... was not grounded`
- `interrogate.probe`
- raw signal keys such as `sharp_public`
- fallback status and equivalence fields

This slice does not hide truth from the system. It only prevents internal implementation gaps from becoming buyer copy.

## files changed

`dai`:

- `services/agent-service/app/services/sports_analyzer.py` - prompt constraints and deterministic top-level buyer-copy sanitizer.
- `services/agent-service/tests/test_sports_analyzer.py` - red/green parser tests for unsafe buyer phrases.
- `apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` - aggregate confirmation-strength row for internal gap states.
- `apps/sports-app/src/app/analyzer/buyer-signal-summary.spec.ts` - red/green buyer mapper tests for unavailable/internal signal rows.
- `apps/sports-app/src/app/analyzer/analyzer.component.html` - Signal Summary header copy no longer says "not a prediction" or "missing signals".
- `apps/sports-app/src/app/analyzer/analyzer.component.ts` - no-lean fallback and aria label made product-safe.
- `apps/sports-app/src/app/sports-api.service.ts` - stub lean now says "Slight lean toward".

`dai-vault`:

- `04 Products/sports-v1/buyer-copy-safety-v1.md` - this note.
- `02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md` - entry 20 resolved; Buyer Copy Polish Review v1 recorded as entry 21.
- `06 Execution/handoffs/current-slice.md` - slice handoff addendum.

## non-goals

- No new Sharp vs Public provider.
- No line movement proxy.
- No source/fallback catalog.
- No availability-fidelity metadata.
- No probe refresh activation.
- No confidence/posture/lean mutation engine.
- No frontend redesign.
- No tone-polish pass beyond safety corrections.

## deferred follow-up

**Buyer Copy Polish Review v1** remains deferred. It should review tone, cadence, repetition, buyer trust language, and prompt efficiency after the safety boundary has held across fresh generated artifacts.

Entry 19 remains deferred. Source expansion and backend availability/source metadata are still separate future slices.

## verification

- Targeted FastAPI parser suite: `105 passed`.
- Targeted Angular buyer signal summary spec: `15 passed`.
- Broader verification and final git status are recorded in `06 Execution/handoffs/current-slice.md`.
