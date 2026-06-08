# Artifact Copy and Section Order v1

**date:** 2026-06-08
**status:** product-design slice with small frontend implementation. read-only buyer presentation. no runtime code, schema change, prompt change, model-call change, Tool Gateway change, source provider change, endpoint change, station activation, probe-refresh activation, artifact mutation, or confidence/posture/lean change.
**scope:** lock the buyer-facing reading order and copy for the sports decision artifact now that Buyer Artifact Route v1 put a real signal summary on the analyzer surface. frontend-only, using existing artifact fields and the existing buyer signal-summary mapper.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read code + vault before changing copy), `dai-token-tight`, `dai-agent-handoff`.
- local runtime skill consulted read-only: `dai-signal-follow-up-diagnostics` (signal vocabulary, so buyer labels stay honest).
- superpowers, applied manually: `writing-plans`, `test-driven-development` (buyer-summary spec updated with the copy changes), `verification-before-completion`. `systematic-debugging` not triggered.

## context

Buyer Artifact Route v1 added a read-only Signal Summary to the analyzer. This slice tunes the buyer reading experience: section order, plain labels, calm copy, and honest evidence language -- without exposing dev mechanics or implying certainty. It is deliberately small; it changes presentation, not data or logic.

## chosen buyer section order

The analyzer result has two zones, and the established ui-concept locks the pattern "answer card in the right rail + full-width supporting sections below." This slice keeps that pattern and orders within it for a buyer:

Answer card ("Matchup Read"), top to bottom:
1. Analyzed Matchup -- what game is this.
2. **Read Stance** -- moved up to sit directly under the matchup, so the decision is the first thing read.
3. Current Lean -- the directional read.
4. Summary -- the prose synthesis.
5. Confidence (with the plain-language confidence sentence).
6. Counter Case -- the other side.
7. Watch For -- what to monitor.
8. What Could Change the Read -- change conditions (renamed from "What Would Change", softer and less absolute).

Supporting full-width sections, below the answer card:
9. **Signal Summary** -- the evidence the read stands on (placed before factors, honoring "signals before deeper reasoning").
10. Factor Breakdown -- the narrative factors.

Why this order, and why not a deeper relayout: product doctrine wants the signal table to be highly legible and to precede deeper narrative. A full relayout that hoists the Signal Summary above the prose narrative would require breaking the locked two-column answer zone and would leave the tall answer card with empty space. That is a larger design change than this slice should make. Instead we made the answer card decision-first (Read Stance up) and kept the Signal Summary as the first full-width supporting section before Factor Breakdown. Promoting the Signal Summary above the prose narrative is recorded as a deferred design decision, not done here.

## final buyer-facing labels

Section labels (kept or changed):
- "Matchup Read" -- kept (clear, paid-brief appropriate).
- "Read Stance" -- kept; moved up.
- "Current Lean" -- kept. "Lean" is retained as an established, analysis-framed label (ui-concept locks it) with the supporting note "Structured lean from the current read." It is framed as a read, not a pick.
- "Summary" -- kept.
- "Confidence" -- kept; shown with the existing plain-language confidence sentence.
- "Counter Case" / "Watch For" -- kept (honest, non-hype).
- "What Could Change the Read" -- changed from "What Would Change the Read" (less absolute).
- "Signal Summary" -- kept. Chosen over "Evidence Snapshot" / "Signal Snapshot" because it matches the product doctrine ("the signal table is the product") and the value proposition ("see the signals"); "Snapshot" implies a point-in-time the data does not promise.
- "Factor Breakdown" -- kept.

Signal Summary row copy (buyer mapper, `buildBuyerSignalSummary`):
- State words: Supported / Light support / Indirect support / **Not available in this run** (was "Not available") / Unavailable / Not applicable / **Not clear enough to use** (was "Unclear").
- Evidence posture: **Backed by current data** (was "Grounded in fetched data"), Limited supporting data, Carried by a related signal, **Not available in this run** (was "No data for this game" / "Not fetched for this read"), Not applicable to this matchup, **Not clear enough to use** (was "Unclear").
- Flag phrase (how it affects the read): Supports the read / Supports the read, with caution / Pulls against the read / Keeps the stance cautious / Neutral.
- Confidence band chip: High / Medium / Low (banded by the analyzer's existing 0.70 / 0.45 thresholds).
- Evidence-strength indicator: a neutral dot reflecting support strength, with an aria-label; explicitly not a betting direction.

## labels intentionally avoided

No buyer-facing string uses any of: raw signal keys (e.g. sharp_public), probe, ToolGateway, source kind, analyzer seed, platform derived, cognitive protocol, artifact version, run id, raw evidence, diagnostics, edge, lock, guaranteed, pick. "Sharp" appears only inside the buyer category label "Sharp vs Public" (a plain description of the public/sharp money split), never as an internal token. Tests assert that buyer rows never contain a raw key (`_`) or the internal vocabulary set.

## what remains dev-only

Unchanged and still only on the dev artifact-review surface: the full Brief Signal Table (Source Type, Probe, raw Evidence text, raw signal keys), Signal Availability table, Signal Follow-Up Diagnostics table, Pipeline Steps, Cognitive Protocol, Artifact Quality warnings, Raw Artifact, analyzer confidence, artifact version, run id. The dev surface files were not touched this slice.

## source / provenance metadata still deferred

The buyer summary still collapses `Unavailable` into "Not available in this run" and shows no source provenance, because the backend does not yet carry an availability-fidelity status (not-configured vs fetched-empty) or a per-signal source-kind. Those remain deferred (ledger entry 19). This slice does not invent source labels.

## why this slice comes before source expansion

Locking buyer copy and order is cheap, reversible, and proves the brief reads as a calm paid decision artifact using data we already have. Source expansion (new providers, a fallback catalog, line-movement) is costly and only worth doing once the buyer surface shows which gaps recur and matter. Copy/order first; sources second.

## what a future source / fallback catalog should solve

- A structured registry mapping each signal to its provider(s) and fallback ladder, gated by real provider availability and cost.
- Backend availability-fidelity and source-kind metadata so the buyer summary can honestly show "Unavailable" vs "Not available" and, optionally, a buyer-safe provenance cue.
- A path to close the most common fallback need surfaced by the Signal Summary (e.g. line movement as an adjacent proxy when the sharp/public split is missing).

## accessibility and mobile

- Read Stance moving up improves scan order on mobile (decision first, less scrolling to the stance).
- Signal Summary remains a responsive card grid (one column on mobile, two at `sm`), with aria-labelled strength dots so meaning never depends on color alone.
- No dense dev-table layout on the buyer surface.

## non-goals

- No new sources, no source/fallback catalog, no ToolGateway call.
- No probe-refresh, merge writer, or station activation.
- No confidence / posture / lean / decision mutation.
- No prompt, model-call, schema, endpoint, or backend contract change.
- No change to the dev artifact-review surface or its Brief Signal Table.
- No prediction-engine, pick, lock, edge, or guaranteed language.

## risks

- The model-produced lean text (and any backend prose) is outside this slice; if it ever uses pick/edge/lock language, that is a backend/prompt copy concern, deferred. The labels and framing this slice controls avoid such language.
- Moving Read Stance up is a small reorder; verified by build and by leaving the rest of the answer card intact.

## next recommended slice

If a single signal gap dominates real runs, **Line Movement Proxy v1** or **Signal Source and Fallback Catalog v1** (ledger entry 19) to start closing the most common fallback need -- but only after confirming the gap recurs. Backend availability-fidelity / source-kind metadata remains the precondition for buyer-visible `Unavailable` and source provenance.

## references reviewed

- `<DAI_VAULT_ROOT>/04 Products/sports-v1/buyer-artifact-route-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/sports-artifact-productization-review-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/ui-concept.md`, `product-brief.md`, `value-proposition.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/analyzer.component.html` (section order)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` + `.spec.ts` (buyer copy)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/dev-artifact-review/` (confirmed unchanged)

## verification performed for this note

- frontend-only change; `npm test` (vitest) 29 passed, `npm run build` clean; `git diff --check` clean; exact-path scan clean; ASCII check passed.
- dev artifact-review surface confirmed unchanged.
- no runtime, schema, prompt, Tool Gateway, source, or endpoint change.
