# Buyer Artifact Route v1

**date:** 2026-06-08
**status:** implementation slice (frontend-only). read-only buyer presentation. no runtime code, schema change, prompt change, model-call change, Tool Gateway change, source provider change, endpoint addition, station activation, probe-refresh activation, artifact mutation, or confidence/posture/lean change.
**scope:** add a thin, read-only buyer-safe signal summary to the buyer-facing analyzer surface, closing the gap named by Sports Artifact Productization Review v1 (the signal table existed only on the dev artifact-review surface).

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read code + vault before building), `dai-token-tight`, `dai-agent-handoff`.
- local runtime skill consulted read-only: `dai-signal-follow-up-diagnostics` -- canonical signal vocabulary so buyer category labels map honestly.
- superpowers, applied manually: `writing-plans`, `test-driven-development` (vitest spec written with the projection), `verification-before-completion`. `systematic-debugging` not triggered.

## 1. context

Sports Artifact Productization Review v1 found the artifact credible and reviewer-ready but not buyer-ready: the Brief Signal Table lived only on the dev artifact-review surface, and the buyer `AnalyzerComponent` consumed only `AgentRunResultDto` (summary, confidence, read stance, counter-case, watch-for, what-would-change, factors) with no signal table. Product doctrine is "the signal table is the product," so the central promise was unmet on the route a buyer sees.

This slice adds the smallest buyer-facing signal experience: a read-only "Signal Summary" card on the analyzer result, built from the existing artifact via the existing read endpoint.

## 2. product decision

Add a buyer-safe signal summary to the existing analyzer surface rather than building a new route or navigation. After a run completes, the analyzer fetches the artifact read-only (`getAgentRunArtifact`, the same endpoint the dev surface uses) and renders a calm card grid. The summary reuses the dev `buildSignalTable` projection through a thin buyer mapper (`buildBuyerSignalSummary`) so signal meaning has one source of truth and no domain logic is duplicated.

Integration point chosen: the analyzer result, not a new page. A full-width "Signal Summary" section sits between the Matchup Read card and the Factor Breakdown, giving the signal picture room to breathe without disturbing the existing read card.

## 3. what the buyer-facing route shows

Per signal (one calm card, responsive: one column on mobile, two on larger screens):

- **Plain category label** (e.g. "Sharp vs Public", "Market / Spread") -- never a raw signal key.
- **Evidence-strength indicator** -- a dot reflecting how grounded the evidence is (supported / light / none). It is explicitly an evidence-strength cue, not a betting direction; the artifact carries no per-signal side.
- **State word** -- Supported / Light support / Indirect support / Not available / Unavailable / Not applicable / Unclear.
- **Evidence posture** -- e.g. "Grounded in fetched data", "No data for this game", "Carried by a related signal".
- **Flag phrase** -- how it affects the read in calm language: "Supports the read", "Pulls against the read", "Keeps the stance cautious".
- **What would change** -- a non-predictive note, shown only when present.

Plus a single **Confidence band** chip (High / Medium / Low) on the section header, banded by the analyzer's existing 0.70 / 0.45 thresholds.

The section renders only when the artifact yields signal rows; otherwise it hides cleanly (fail-soft).

## 4. what remains dev-only

Unchanged and still only on the dev artifact-review surface: the full Brief Signal Table (Source Type, Probe, raw Evidence text, raw signal keys), Signal Availability table, Signal Follow-Up Diagnostics table, Pipeline Steps, Cognitive Protocol, Artifact Quality warnings, Raw Artifact, analyzer confidence, artifact version, and run id. None of these were promoted to the buyer surface.

## 5. reused projections and endpoints

- **Endpoint:** existing `GET /api/agent-runs/{id}/artifact` via `SportsApiService.getAgentRunArtifact` -- no new endpoint, no schema change.
- **Projection:** existing `buildSignalTable` (dev-artifact-review/signal-table.ts) -- the buyer mapper depends on it; signal state/source/impact derivation is not re-implemented.
- **DTO:** existing `AgentRunArtifactDto` -- no new contract.

## 6. copy and label choices

- Canonical key -> plain label map (market -> "Market / Spread", sharp_public -> "Sharp vs Public", line_movement -> "Line Movement", rest_schedule -> "Rest & Schedule", starting_pitching -> "Starting Pitching", injury_report -> "Injuries & Availability", etc.); unknown keys are title-cased so a new backend signal never leaks as snake_case.
- State translation: Grounded -> "Supported"; Weak -> "Light support"; Proxy -> "Indirect support"; Missing -> "Not available"; Unavailable -> "Unavailable"; NotApplicable -> "Not applicable"; Unknown -> "Unclear".
- Impact translation: "Caps aggressive posture" -> "Keeps the stance cautious"; "Dampens confidence" -> "Pulls against the read"; "Supports with caution" preserved as "Supports the read, with caution".
- Framing line on the card: "Decision support, not a prediction -- missing or light signals lower confidence rather than being hidden." No pick / lock / edge / guaranteed language anywhere.

## 7. accessibility and mobile considerations

- Section is an `aria-labelledby` region; the strength indicator dot carries `role="img"` and an `aria-label` ("Supported by evidence" / "Limited evidence" / "No evidence available") so it is not a color-only signal.
- Card grid is single-column on mobile, two-column at the `sm` breakpoint; no horizontal scroll, no dense dev-table feel.
- Color tones (emerald / amber / muted) are paired with text state words and the aria label, so meaning never depends on color alone.

## 8. explicit non-goals

- No new source provider, no source/fallback catalog, no ToolGateway call.
- No probe-refresh activation, no merge writer, no station activation.
- No confidence / posture / lean / decision mutation; the summary only describes existing values.
- No prompt, model-call, schema, or backend contract change.
- No new navigation, route, or product flow; the dev artifact-review surface and its Brief Signal Table are untouched.
- No betting-prediction language; the summary is evidence posture, not a call.

## 9. risks

- In stub/dev mode there is no artifact endpoint, so `getAgentRunArtifact` fails and the summary hides (fail-soft). This is expected; the summary appears against the real API.
- The evidence-strength indicator could be misread as a betting direction; mitigated by labeling it as evidence strength and using neutral dots, not arrows toward a team.
- The buyer state collapses `Unavailable` into "Not available" today because the backend encodes unavailability in `quality`, not a distinct status; an honest `Unavailable` and any source-provenance label still need backend metadata (deferred-ledger entry 19).

## 10. next recommended slice

**Artifact Copy and Section Order v1** -- lock the buyer label set, the read-section ordering (so the signal summary sits where product doctrine wants it relative to the narrative), and the confidence-band copy, now that a real buyer signal surface exists to tune. Alternatively, if a single signal gap dominates real runs, **Signal Source and Fallback Catalog v1** / **Line Movement Proxy v1** (deferred-ledger entry 19) to start closing the most common fallback need. Backend metadata (availability fidelity, source kind) remains a precondition for showing `Unavailable` / source provenance to a buyer.

## references reviewed

- `<DAI_VAULT_ROOT>/04 Products/sports-v1/sports-artifact-productization-review-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/product-brief.md`, `value-proposition.md`, `ui-concept.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/analyzer.component.ts` + `.html` (buyer surface)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/buyer-signal-summary.ts` + `.spec.ts` (new buyer mapper + tests)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/dev-artifact-review/signal-table.ts` (reused projection)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/sports-api.service.ts` (`getAgentRunArtifact`)

## verification performed for this note

- frontend-only change; `npm test` (vitest) and `npm run build` both green; `git diff --check` clean; exact-path scan clean; ASCII check passed.
- dev Brief Signal Table and artifact-review surface confirmed unchanged.
- no runtime, schema, prompt, Tool Gateway, or source provider change.
