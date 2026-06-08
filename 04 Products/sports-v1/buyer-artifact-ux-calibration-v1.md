# Buyer Artifact UX Calibration v1

**date:** 2026-06-08
**status:** product calibration + verification. docs only. no runtime code, schema change, prompt change, model-call change, Tool Gateway change, source provider change, endpoint change, station activation, probe-refresh activation, artifact mutation, or confidence/posture/lean change.
**scope:** calibrate the buyer-facing sports artifact experience against existing generated artifacts, judge buyer-readiness, and identify the first recurring buyer-relevant signal gap before any source expansion.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read artifacts + vault + code before judging), `dai-token-tight`, `dai-agent-handoff`.
- local runtime skill consulted read-only: `dai-signal-follow-up-diagnostics` (signal vocabulary, artifact field names).
- superpowers, applied manually: `writing-plans`, `verification-before-completion`. `systematic-debugging` not triggered (no defect). `test-driven-development` not used (docs-only).

## 1. executive summary

**Buyer-readiness judgment after calibration: the buyer surface is clear, credible, and honest -- it is buyer-usable as-is.** It correctly leads with the decision (Read Stance), surfaces the dominant evidence gap plainly ("Sharp vs Public -- Not available in this run"), and keeps the stance cautious when a primary signal is absent. The Signal Summary supports the read rather than distracting from it.

**Strongest parts:** decision-first order; honest signal states; the Signal Summary turning a missing primary signal into visible, conservative context rather than hiding it; confidence shown as a band with a plain sentence.

**Weakest part:** the single material risk is **model-produced prose**, not structure or frontend copy. The `lean` text says "Edge toward <team>" in 27 of 33 artifacts (82%). The frontend label layer scrupulously avoids "edge", but the Current Lean renders model prose directly, so a betting word the product has worked to avoid reaches the buyer in the most prominent field. This is a prompt/content issue, not a frontend bug.

**Recommended next slice: Prompt Copy Safety v1** -- tighten the model prose so the lean/summary use analysis-framed language ("leans toward / favors") and stop claiming ungrounded `signals_used`. It is the highest-frequency buyer-visible copy risk, it is cheap (prompt-level, no new sources or runtime), and it protects the "decision support, not a prediction engine" positioning. Source expansion for the `sharp_public` gap stays deferred (ledger entry 19): the gap is real and recurring, but the buyer surface already handles it honestly, so it is justified-but-not-urgent and heavier.

## 2. calibration sample inventory

Reviewed 33 existing generated artifact JSONs under the sports calibration artifacts area (NBA and MLB; dated 2026-05-07 through 2026-05-29), plus the dated calibration markdown reports for context. Mix of shapes: 23 carry rich `signalAvailability`; ~10 are legacy coarse `groundedSignals` / `missingSignals` only -- both buyer-summary paths are exercised by real data. Cross-checked against the live buyer surface (`AnalyzerComponent` template + `buildBuyerSignalSummary`) and the dev `signal-table` projection. No absolute local paths recorded.

Aggregate signal evidence:
- grounded across runs: `market` and `rest_schedule` (19 each, coarse) + 9 each (rich); `starting_pitching` 14 (MLB).
- missing across runs: `sharp_public` only -- 19/33 coarse, 9/9 of the rich-path missing entries.
- `lean` contains "edge": 27/33 (82%). `lean` contains pick/lock/guarantee: 0/33.
- confidence >= 0.65 with `sharp_public` missing: 15/33 (45%).
- `artifactQualityWarnings` carrying `signals_used ... not grounded`: 25/33 (dev-only surface; e.g. MLB bullpen / lineup_form / ballpark).

## 3. buyer surface review

| section | verdict | note |
|---|---|---|
| Analyzed Matchup | Ready | clear; buyer knows what was analyzed |
| Read Stance | Ready | decision-first placement works; play/pass/monitor/wait/compare/avoid reads as analysis, not a pick |
| Current Lean | **Needs prompt/backend change** | model prose says "Edge toward X" in 82% of runs; betting word the frontend avoids |
| Summary | Ready | model prose; calm, monitor-toned in the sample set |
| Confidence | Ready | band + plain sentence; partial-evidence cases are made honest by the Signal Summary |
| Counter Case | Ready | shows the other side; trust-building |
| Watch For | Ready | actionable |
| What Could Change the Read | Ready | honest, non-absolute; sample prose ("Market movement indicating a shift in confidence") is safe |
| Signal Summary | Ready (minor copy tuning) | on legacy artifacts a row with unknown impact renders a dangling middle-dot/dash separator; cosmetic |
| Factor Breakdown | Ready | placement after the Signal Summary is correct (evidence before narrative factors) |

## 4. signal gap table

| signal / field | observed state | buyer impact | frequency / recurrence | root category | recommended follow-up | priority |
|---|---|---|---|---|---|---|
| `sharp_public` (Sharp vs Public) | Missing -> "Not available in this run" | a key money signal absent; read stays cautious | 19/33 coarse + 9/9 rich; the only missing signal | Source coverage | Line Movement Proxy v1 / Signal Source and Fallback Catalog v1 (entry 19) | P2 (recurring but handled honestly) |
| `market` (Market / Spread) | Grounded -> Supported | anchors the read | grounded in ~28 runs | Presentation | none | -- |
| `rest_schedule` (Rest & Schedule) | Grounded -> Supported | supporting context | grounded in ~28 runs | Presentation | none | -- |
| `starting_pitching` (MLB) | Grounded -> Supported | supporting (MLB) | 14 runs | Presentation | none | -- |
| Current Lean prose ("Edge toward...") | model prose | sounds like a betting pick; contradicts product positioning | 27/33 (82%) | Prompt or model prose | Prompt Copy Safety v1 | P1 |
| `signals_used` claims ungrounded signals (bullpen, lineup_form, ballpark) | model prose; dev warning | dev-only today; model overclaims coverage | 25/33 | Prompt or model prose | Prompt Copy Safety v1 (same slice) | P3 (dev-only) |
| confidence high with `sharp_public` missing | partial-evidence confidence | could overstate certainty, but the Signal Summary surfaces the gap | 15/33 (45%) | Product decision / calibration | none new; keep surfacing honestly | P3 |
| Signal Summary legacy row separator | cosmetic | dangling middle-dot/dash when impact is unknown | ~10 legacy artifacts | Presentation | Buyer Artifact Visual QA v1 (tiny) | P3 |

## 5. language risk review

Separating the two layers:

- **Frontend labels (controlled, safe):** every buyer label and state word avoids pick / lock / edge / guaranteed. "Sharp" appears only inside the plain category "Sharp vs Public". Tests assert no raw key or internal vocabulary leaks. No change needed.
- **Model-produced prose (the risk):** `lean` overwhelmingly uses "Edge toward <team>" (27/33). "Edge" is betting-market language and the most prominent line a buyer reads. `whatWouldChangeTheRead` and `summary` prose in the sample set are safe (no pick/lock/guarantee; "market movement" phrasing is acceptable). `signals_used` sometimes names signals that were not grounded (dev warning only).

Recommended safer copy (for a future Prompt Copy Safety v1, not changed here): replace "Edge toward X" with analysis framing such as "Read leans toward X" or "Model favors X on <reason>"; constrain `signals_used` to grounded signals only. Do not change prompts in this slice.

Note: a frontend regex that rewrites "Edge" in model prose was considered and rejected -- rewriting model sentences in the view is fragile and can distort meaning. The correct fix is at the prompt layer.

## 6. layout recommendation

- **Keep the current order.** Decision-first answer card (Matchup -> Read Stance -> Current Lean -> Summary -> Confidence -> Counter Case -> Watch For -> What Could Change), then full-width Signal Summary, then Factor Breakdown. Calibration found this natural and legible.
- **Signal Summary stays below the answer card, before Factor Breakdown.** Evidence before narrative factors is correct. The deeper relayout that would hoist the Signal Summary above the prose narrative remains deferred (it would break the locked two-column answer zone; ledger entry 19).
- **Factor Breakdown placement is correct** as the closing reasoning layer.
- Deferred layout item: the cosmetic dangling separator on legacy Signal Summary rows -- fold into a future Buyer Artifact Visual QA pass.

## 7. next-slice decision

**Prompt Copy Safety v1.**

Why this is next: the buyer surface is structurally and copy-safe on the parts the frontend controls; the one material, high-frequency, buyer-visible risk is model prose ("Edge" in 82% of leans, plus ungrounded `signals_used`). It directly threatens the product's "decision support, not prediction" positioning, it is the most-read field (Current Lean), and it is the cheapest fix (prompt-level, no new sources, no runtime, no schema).

Why not the alternatives first:
- **Line Movement Proxy v1 / Signal Source and Fallback Catalog v1** -- the `sharp_public` gap is confirmed recurring, but the buyer surface already handles it honestly and conservatively, so it does not currently break the read. Source expansion is heavier (providers, cost, fallback ladder) and is justified-but-not-urgent. Deferred (entry 19) until after copy safety.
- **Backend Availability Fidelity and Source Kind v1** -- metadata polish that would let the buyer table show "Unavailable" vs "Not available" and provenance; lower buyer urgency than removing betting language.
- **Buyer Artifact Visual QA v1** -- the only frontend nit found (dangling separator on legacy rows) is cosmetic and can ride along later.

## 8. deferred ledger update

- Entry 19 (source/fallback catalog) is confirmed, not resolved: calibration proves `sharp_public` is the dominant recurring gap, but the buyer surface handles it honestly, so source expansion stays deferred and is sequenced after copy safety.
- A new deferral is recorded for buyer-facing model prose copy safety (the "Edge" language and ungrounded `signals_used`), which requires a prompt change and is out of scope here. See ledger entry 20.

## risks

- The "Edge" prose is the most visible risk and cannot be fixed without a prompt change; until Prompt Copy Safety v1 ships, the Current Lean will keep reading like a betting call in most runs.
- Confidence can read high while a primary signal is missing; the Signal Summary mitigates this by making the gap explicit, but it is not eliminated.
- Sample set is NBA/MLB only and dated; NFL (the commercial v1 lead) was not represented in the artifacts and should be calibrated when NFL runs exist.

## references reviewed

- `<DAI_VAULT_ROOT>/04 Products/sports-v1/calibration/artifacts/` (33 generated NBA/MLB artifacts)
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/calibration/` (dated NBA/MLB calibration reports)
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/buyer-artifact-route-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/artifact-copy-and-section-order-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/sports-artifact-productization-review-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/analyzer.component.html`, `buyer-signal-summary.ts` (buyer surface)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/dev-artifact-review/signal-table.ts` (reused projection; not buyer-facing)

## verification performed for this note

- docs-only; no runtime, schema, prompt, Angular, or source change.
- no exact local machine paths added; placeholders used throughout.
- ASCII check for this note passed.
- git status / diff --check results recorded in the slice handoff addendum.
