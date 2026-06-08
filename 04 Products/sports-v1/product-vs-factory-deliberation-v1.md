# Product vs Factory Deliberation v1

**date:** 2026-06-07
**status:** deliberation note. docs only. no runtime code, schema change, prompt change, Angular change, Tool Gateway change, endpoint, station activation, or artifact mutation.
**scope:** decide whether the next DAI slice should prioritize factory maturity or sports product polish, using the principle that the platform is the factory, not the product.

## Executive conclusion

Recommend one next slice: **Sports Brief Signal Table v1**.

This is a product-polish slice focused on artifact usefulness. It should turn the signal picture already present in the artifact into a buyer-facing brief surface: signal category, available/missing/proxy/weak status, short flag phrase, confidence context, counter-case, watch-for, and what-would-change. It should not activate stations, mutate artifacts, change confidence/posture/lean, expand probe tool power, or add new retrieval sources.

Reason: more factory work is now mostly architecture-forward. Station cards, status semantics, diagnostics, Perceive intake contracts, and dormant Discern/Synthesize ideas improve future safety, but they do not directly make the current paid-user brief more useful. The sports product's own doctrine says the signal table is the product. The current product still has a gap between the internal artifact quality surface and the compact buyer-facing read.

Synthesize Runtime Inspection v1 remains a backup only. It has no named product or operator caller today.

## Decision frame

The platform is the factory. It owns repeatable orchestration, permissions, diagnostics, artifact contracts, usage, tenant boundaries, and station governance. The product is the packaged decision artifact a buyer values.

Factory work should be prioritized when it removes a current blocker to artifact quality, safety, observability, cost control, or tenant delivery. Product work should be prioritized when the factory is already producing enough internal evidence and the next bottleneck is whether a buyer can understand and trust the brief.

Current state favors product polish:

- station metadata and status semantics are now strong enough for future governance.
- no station runtime caller is named.
- the sports product still lacks the buyer-facing signal table promised by the product concept.
- existing artifact fields already carry signal availability, quality, follow-up diagnostics, posture, confidence, counter-case, watch-for, and what-would-change.
- making those existing fields useful to a buyer is more valuable than inspecting Synthesize internals.

## Does more factory work directly improve buyer-visible artifact quality?

Not enough to justify being next.

The recent factory slices improved safety and future optionality:

- Perceive signals can be normalized, projected, and collected.
- Discern has dormant runner groundwork.
- station cards carry ownership, maturity, mutation policy, input/output contracts, fallback behavior, forbidden behavior, and status semantics.
- diagnostics can inspect station cards read-only.

Those are platform health gains. They do not, by themselves, make a paid user's morning brief clearer, more trusted, or more actionable.

The direct buyer-visible quality lever now is not another runtime station caller. It is presenting the current artifact's evidence plainly: which signals are grounded, which are missing, which are weak, what is proxy-backed, how confidence should be interpreted, and what would change the read.

## What the current sports artifact likely lacks for a paid user

From the sports product docs and calibration notes, the likely paid-user gaps are:

- **buyer-facing signal table:** the product concept says the signal table is the product, but current user-facing delivery is still compact. Internal `SignalAvailability` and follow-up diagnostics exist, but they are primarily inspection/developer surfaces.
- **explicit missing/proxy labels:** the hardening doc names proxy label surfacing as a gap. Missing and proxy context should not remain buried in internal diagnostics.
- **injury/availability context:** product scope names injury/availability as expected for NFL/NBA, but current source strategy says no injury/availability source is wired yet. This is a product quality gap, not a station runtime gap.
- **confidence explanation fit:** calibration notes show recurring `confidence_high_for_partial_evidence` flags in small batches. Threshold changes remain deferred, but buyer-facing confidence context should make partial evidence clear.
- **unsupported or overbroad signal claims:** MLB calibration showed warnings where `signals_used` referenced ungrounded concepts such as bullpen, lineup form, or ballpark. A buyer-facing brief must make grounded vs ungrounded evidence obvious.
- **delivery packaging:** the product brief targets email and Slack/webhook; current dev surfaces are still analyzer and internal artifact review oriented.
- **history/account/billing polish:** current scope docs describe History and Account as mostly shell/mock surfaces. Those matter later, but the brief itself should be worth archiving before investing there.

The highest-value first product slice is therefore not injury sourcing or delivery. It is the brief's signal table/read model using fields the factory already produces.

## Does a concrete read-only runtime caller exist yet?

No.

Existing internal callers and surfaces are enough for inspection:

- `/dev/artifacts` already provides internal artifact review.
- calibration reports already consume artifacts for offline quality review.
- station diagnostics and rollups already inspect station-card metadata.

None of these requires Synthesize Runtime Inspection v1, Discern runner adoption, Perceive runtime routing, or probe-refresh activation. A future read-only station caller should name who consumes it, what decision it helps them make, and why existing artifact/diagnostic surfaces are insufficient. That caller is not present in the current docs.

## Should Synthesize Runtime Inspection v1 become next?

No. It remains a backup.

Synthesize is the safest technical station family for inspection because it is platform-owned, deterministic, no-tool, no-model, and already part of artifact assembly. But "safest" is not the same as "needed."

Synthesize Runtime Inspection v1 should become next only if a named caller needs to inspect Synthesize output boundaries and cannot use existing artifacts or diagnostics. No such caller exists. Building it now would be factory-forward: useful future scaffolding, but not directly buyer-visible.

## Architecture-forward work to defer

Defer these until a concrete caller, product need, or safety gate requires them:

- Synthesize Runtime Inspection v1.
- Perceive Signal Intake Read Model v1.
- Discern Station Runner Adoption v1.
- any station runtime adoption or activation.
- expanding `ProtocolNodeRunner` beyond `interrogate.probe`.
- probe-refresh chain activation.
- direct Interrogate -> Perceive self-invocation.
- direct tool power on `interrogate.probe`.
- merge writer or artifact mutation path.
- confidence/posture/lean mutation.
- per-station model-call splitting.
- Tool Gateway behavior changes.
- endpoint work for station diagnostics.
- memory/pgvector-backed probe work.

These are not rejected. They are deferred because the current factory can already produce enough internal artifact evidence to support product polish.

## Recommended next slice

**Sports Brief Signal Table v1**

Purpose: make the existing sports artifact more useful to a paid user by presenting the evidence picture clearly.

Candidate scope for the next implementation slice:

- define a buyer-facing signal table/read model from existing artifact fields.
- show grounded, missing, weak, unavailable, and proxy-backed signal states.
- include one short flag phrase per signal, not raw internal diagnostics.
- label proxy or missing primary evidence explicitly.
- keep confidence as supporting context, with partial-evidence caveats visible.
- keep posture language as "Read Stance", not "Pick".
- preserve counter-case, watch-for, and what-would-change as concise buyer-facing fields.
- do not add new signals or retrieval sources in this slice.
- do not change confidence, posture, or lean.
- do not mutate artifacts or add a merge writer.
- do not activate stations.
- do not expand probe tool power.

Why this slice:

- It directly addresses paid-user artifact usefulness.
- It uses current factory output instead of inventing new station runtime.
- It follows the product doctrine that the signal table is the product.
- It helps validate whether the artifact is worth paying for before deeper factory investment.
- It keeps platform and product boundaries clean: the factory produces evidence; the product packages it.

## Explicit non-recommendation

Do not make the next slice a factory-maturity slice unless it names a concrete read-only caller.

A valid factory-maturity next slice would need a sentence like: "Caller X needs read-only station runtime output Y to make decision Z, and existing artifact/diagnostic surfaces cannot provide it." The current review did not find that sentence.

## Guardrails carried forward

- Do not activate any station.
- Do not treat station-card metadata as execution permission.
- Do not expand Probe tool power.
- Do not allow artifact mutation.
- Do not change confidence, posture, or lean without calibration proof.
- Keep runtime adoption deferred unless a named caller exists.
- Keep the platform as the factory, not the product.

## Docs reviewed

- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/protocol-station-runtime-adoption-readiness-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/product-brief.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/value-proposition.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/target-customer.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/v1-scope.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/ui-concept.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/decision-factory-hardening-v1.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/current-sports-analysis-flow.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/calibration/20260528-0931-nba-canonical-protocol-spot-check.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/calibration/20260529-1421-mlb-calibration.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/calibration/protocol-runs/2026-05-18-cognitive-protocol-outcome-reconciliation.md`

## Implementation status (2026-06-08)

Recommended slice implemented as **Sports Brief Signal Table v1**. A read-only buyer-facing table was added to the artifact-review surface (`apps/sports-app/.../dev-artifact-review`) via a tested pure projection (`signal-table.ts`) over existing artifact fields. It surfaces grounded/weak/missing/unavailable/proxy state, a source-type label, impact-on-read (from the existing confidence effect, not a mutation), a fallback NEED, probe-eligibility as a label only, and a non-predictive what-would-change note. No new sources, no ToolGateway calls, no station activation, no probe-refresh activation, no schema/prompt change, and no confidence/posture/lean mutation. The structured source/fallback catalog the table makes visible remains deferred (ledger entry 19).

## Verification performed for this note

- docs-only change.
- no runtime files changed.
- no schema/migration files changed.
- `git diff --check` passed with the new doc included through an intent-to-add diff.
- exact local path scan found no exact local machine paths in added lines.
- ASCII check for this note passed.
