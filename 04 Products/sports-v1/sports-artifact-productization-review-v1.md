# Sports Artifact Productization Review v1

**date:** 2026-06-08
**status:** product + design review. docs only. no runtime code, schema change, prompt change, model-call change, Tool Gateway change, source provider change, endpoint, station activation, probe-refresh activation, artifact mutation, or confidence/posture/lean change.
**scope:** a field-by-field productization audit of the sports decision artifact and the new Brief Signal Table, judged for whether they are clear, credible, reviewable, and buyer-ready without adding new runtime machinery.

## skills / guidance used

- local DAI pack (read-only): `dai-grill-with-vault` (read code + vault before judging), `dai-token-tight` (dense reporting), `dai-agent-handoff` (current-slice addendum). `dai-write-skill` not used.
- local runtime skill consulted read-only: `dai-signal-follow-up-diagnostics` -- for the canonical signal vocabulary and artifact field names, so the review never invents signals or sources. Kept separate from the development skills.
- superpowers, applied manually: `writing-plans`, `verification-before-completion`. `systematic-debugging` and `test-driven-development` not triggered -- this slice is docs-only and found no obvious low-risk presentation bug requiring a code change.

## 1. executive summary

**Current buyer-readiness judgment: the artifact is credible and reviewer-ready, but it is not yet buyer-ready -- because the signal table is not on the route a buyer actually sees.**

The product's own doctrine is explicit: "the signal table is the product," and the buyer is a sharp recreational bettor who wants signal context, not a conclusion. The Brief Signal Table built in the prior slice is honest, legible, and credible -- but it lives only on the dev artifact-review surface (`dev-artifact-review`), which also exposes Pipeline Steps, Cognitive Protocol, and Raw Artifact. The buyer-facing analyzer surface (`AnalyzerComponent`) renders only `AgentRunResultDto` (summary, confidence, read stance, counter-case, watch-for, what-would-change, factors). It does not fetch the artifact and shows no signal table at all. So the central promise is structurally unmet on the buyer path.

**What is strong**
- Honesty: the artifact distinguishes grounded / weak / missing / proxy / not-applicable states without pretending unavailable data was fetched. The Brief Signal Table inherits that discipline and expresses fallback as a need, never as a fetched fact.
- Credibility: signal evidence, follow-up diagnostics, posture as "Read Stance" (not "Pick"), counter-case, watch-for, and what-would-change are all present and decision-shaped, not hype.
- Technical alignment: the table is a frontend projection over existing DTO fields; no domain logic is duplicated in the wrong layer; legacy artifacts are handled safely.

**What is not ready**
- The buyer never sees the signal table; the analyzer surface has no signal-context module.
- The dev table speaks builder language: raw signal keys (`sharp_public`), internal source labels (`ToolFetched`), a `Probe` column, and an `Evidence` column carrying internal reason/detail text. Appropriate for a dev surface; not buyer-facing copy.
- Two state/source labels (`Unavailable`, and the source kinds `AnalyzerSeed` / `PlatformDerived`) are unreachable from current backend fields and should not be promoted until backend metadata supports them.

**Recommended next slice: Buyer Artifact Route v1** -- a thin, read-only buyer signal-table module that renders a curated, plain-language subset of the existing projection on the buyer path, using the existing artifact read endpoint. Backup: **Artifact Copy and Section Order v1** (docs-only) if the team prefers to lock buyer labels and the category mapping before building.

## 2. artifact section inventory

Sections currently on the dev artifact-review surface, top to bottom:

| section | audience fit | verdict |
|---|---|---|
| Run Overview (run id, status, competition, game date, Read Stance, evidence richness, confidence, analyzer confidence, artifact version) | mixed | **needs regrouping** -- a buyer subset (matchup, read stance, confidence) is buyer-ready; run id / analyzer confidence / artifact version are dev-only |
| Brief Signal Table | builder now, buyer with curation | **needs rename + curation for buyer**; buyer-ready as a concept, dev-ready as built |
| Pipeline Steps | internal | **keep dev-only** |
| Signal Coverage (grounded / missing chips) | buyer-adjacent | **regroup** -- subsumed by the buyer signal table; keep raw chips dev-only |
| Signal Follow-Up Diagnostics (table) | internal | **keep dev-only** |
| Signal Availability (table) | internal | **keep dev-only** -- it is the raw source the buyer table is derived from |
| Artifact Quality (warnings) | internal | **keep dev-only** -- review notes, not buyer copy |
| Decision Fields (Lean, Summary, Counter Case, Watch For, What Would Change) | buyer | **buyer-ready** with light copy review; this is the narrative spine of the buyer brief |
| Cognitive Protocol (Perceive/Interrogate/Discern/Decide/Synthesize) | internal | **keep dev-only** -- exposes factory internals; must not leak to a buyer |
| Raw Artifact (JSON) | internal | **keep dev-only** |

## 3. field-by-field review

Priority: P1 = needed before/with a buyer route; P2 = soon after; P3 = later or optional.

| section | field / element | current label | buyer clarity | decision usefulness | trust risk | technical risk | recommendation | priority |
|---|---|---|---|---|---|---|---|---|
| Run Overview | matchup / competition / game date | Competition, Game Date | clear | orients the read | low | low | keep, surface on buyer header | P1 |
| Run Overview | posture | Read Stance | good (already de-risked from "Pick") | states stance without a lock | low | low | keep label; it is the right call | P1 |
| Run Overview | confidence | Confidence (percent) | a raw percent can over-read as precision | supporting context | medium -- percent implies false precision | low | show as band (High/Medium/Low) + one sentence on the buyer route; keep percent dev-only | P1 |
| Run Overview | analyzerConfidence, artifactVersion, agentRunId | Analyzer Confidence, Artifact Version, Agent Run ID | internal | none for buyer | low | low | keep dev-only | P3 |
| Brief Signal Table | Signal | raw key (`sharp_public`) | poor for buyer | high once relabeled | low | low | map to plain category labels on buyer route | P1 |
| Brief Signal Table | State | Grounded/Weak/Missing/Unavailable/Proxy/NotApplicable/Unknown | mostly clear; `NotApplicable`/`Unknown` read as system words | high -- this is the trust core | low | medium -- `Unavailable` unreachable today | keep for dev; on buyer route reduce to Grounded/Weak/Missing/Proxy + a muted dash for not-applicable | P1 |
| Brief Signal Table | Source Type | Artifact/ToolFetched/Proxy/Missing/Unknown | internal vocabulary | medium | low | medium -- `AnalyzerSeed`/`PlatformDerived` never emitted | hide on buyer route until backend source-kind exists; keep dev-only | P2 |
| Brief Signal Table | Impact on Read | Supports / Supports with caution / Dampens confidence / Caps aggressive posture / Neutral / Unknown | mostly clear | high | low -- descriptive only, not a mutation | low | keep; soften "Caps aggressive posture" to buyer phrasing | P1 |
| Brief Signal Table | Fallback Need | "needs line_movement (not implemented)" | leaks internal key + status | medium | low | low | on buyer route, convert to plain phrase or a simple "more data would help" cue; keep raw need dev-only | P2 |
| Brief Signal Table | Probe | Eligible / dash | opaque to a buyer | none for buyer | medium -- "Probe" hints at machinery | low | drop entirely from buyer route; keep dev-only | P2 |
| Brief Signal Table | Evidence | reason / detail text | internal phrasing | medium | low | low | replace with one short flag phrase per signal on buyer route | P1 |
| Brief Signal Table | What Would Change | templated phrase | clear and honest | high | low | low | keep; it directly answers a buyer question | P1 |
| Decision Fields | Summary | Summary | clear | the narrative synthesis | low | low | buyer-ready; light copy pass | P1 |
| Decision Fields | Counter Case | Counter Case | clear | shows the other side -- builds trust | low | low | buyer-ready | P1 |
| Decision Fields | Watch For | Watch For | clear | actionable | low | low | buyer-ready | P1 |
| Decision Fields | What Would Change the Read | What Would Change the Read | clear | actionable | low | low | buyer-ready | P1 |
| Decision Fields | Lean | Lean | mostly clear; ensure it never reads as a guaranteed pick | directional cue | medium -- could read as a pick | low | keep as directional lean; never lock language | P1 |

## 4. Brief Signal Table review (column by column)

- **Signal** -- correct data, wrong register for a buyer. Raw canonical keys are perfect for a builder and opaque for a customer. Map to plain category labels on the buyer route (see section 6). Dev: keep raw.
- **State** -- the trust core, and it is honest. `Grounded`, `Weak`, `Missing`, `Proxy` are buyer-meaningful. `NotApplicable` should render as a muted dash (matching the product concept's "null signals shown as a dash, not hidden"); `Unknown` should be rare and also render as a dash. `Unavailable` is currently unreachable (backend encodes unavailability in `quality`, not `status`), so today every absent signal reads as `Missing` -- acceptable and honest, but the `Unavailable` rung needs backend metadata before it can be populated.
- **Source Type** -- truthful but internal. `ToolFetched` / `Artifact` / `Missing` are builder words; `AnalyzerSeed` and `PlatformDerived` are reserved and never emitted. Hide on the buyer route until a backend source-kind field exists; otherwise it risks both jargon and an empty/over-precise feel.
- **Impact on Read** -- strong and honest; it is a description of the already-computed confidence effect, not a new judgment. Soften "Caps aggressive posture" to plain buyer language.
- **Fallback Need** -- the honesty is excellent (a need, never a fetched fact) but the string leaks an internal signal key and status. Convert to a plain cue on the buyer route; keep the precise need dev-only.
- **Probe** -- drop from the buyer route. "Probe" implies internal machinery and gives a buyer nothing. Dev-only.
- **Evidence** -- internal reason/detail text. Replace with a single short flag phrase per signal for buyers (the product concept asks for exactly that: "line moved 2.5 pts against 68% public").
- **What Would Change** -- keep. It directly answers "what would change the read if confirmed" and is templated, non-predictive, and honest.

Overall: the table is a strong builder/reviewer instrument and a strong *foundation* for the buyer table. It is not the buyer table yet -- the buyer version is a curated, relabeled, narrower projection of the same rows.

## 5. buyer-facing artifact recommendation

What the first buyer-facing version (a brief view or a signal-table module on the analyzer result) should include.

**Must show**
- Matchup header: teams, competition, game date.
- Read Stance (posture) -- never a lock/pick.
- Signal table, plain-language: one row per buyer category, a directional indicator, a short flag phrase, and an honest state (grounded / weak / missing / proxy, with a muted dash for not-applicable).
- Confidence as a band (High / Medium / Low) with one supporting sentence.
- Narrative: Summary.
- What Would Change the Read; Watch For; Counter Case.

**Should show later**
- A compact "what would strengthen this read" cue derived from fallback needs, in plain language.
- Line-movement / update affordance (pro tier), once that source exists.

**Keep dev-only**
- Pipeline Steps, Cognitive Protocol, Raw Artifact, Signal Availability table, Signal Follow-Up Diagnostics table, Artifact Quality warnings, Source Type, Probe column, raw signal keys, analyzer confidence, artifact version, run id.

**Needs backend metadata first**
- A real availability fidelity status that distinguishes "not configured / never fetched" from "fetched, empty" (to populate `Unavailable` honestly).
- A source-kind field (analyzer-seed vs platform-derived vs tool-fetched) before showing source provenance to a buyer.

**Remove or rename (for the buyer route only)**
- Rename raw signal keys to category labels; rename "Impact on Read" phrasings to plain language; remove "Probe"; replace "Evidence" with a flag phrase.

## 6. language and copy recommendations

Principle: decision-support language, not prediction-engine language. The buyer wants to see the signals, not be sold a call.

Suggested buyer-facing category labels (mapping canonical keys -> plain labels; final wording to be locked in the next slice):

- `market` -> "Market / Spread"
- `sharp_public` -> "Sharp vs Public"
- `line_movement` -> "Line Movement"
- `rest_schedule` -> "Rest & Schedule"
- `starting_pitching` -> "Starting Pitching" (MLB)
- `injury_report` -> "Injuries & Availability"
- `weather` -> "Weather"
- (others map to their plain-English category; never show the raw key)

State and effect copy:

- `Grounded` -> "Grounded" or a solid directional indicator; `Weak` -> "Light"; `Missing` -> a muted dash with "not available for this game"; `Proxy` -> "Proxy" with a one-line "carried by a related signal" note.
- "Caps aggressive posture" -> "Keeps the stance cautious."
- Fallback need -> "More data would sharpen this" rather than "needs line_movement (not implemented)."

Words worth using where they actually fit: decision artifact, reviewable signal context, evidence posture, signal availability, decision support, structured reasoning. Words to avoid: pick, lock, guaranteed, edge, +EV framing, anything that implies a settled outcome.

Trust rule to preserve: confidence is supporting context. Low confidence means mixed or incomplete signals, shown matter-of-factly -- no warning colors, no exclamation, no implication the brief is broken.

## 7. engineering recommendations (recommendations only -- no code in this slice)

- **Availability fidelity:** add a backend status that separates "not configured" from "fetched, empty" so the buyer table can show `Unavailable` vs `Missing` honestly. Today both collapse to `Missing` (honest, but coarse).
- **Source kind:** add a per-signal source-kind (analyzer-seed / platform-derived / tool-fetched) so `Source Type` can be truthful if ever shown; until then the projection correctly refuses to guess.
- **Fallback ladder exposure:** the backend `SignalFollowUpEvaluator` already computes `fallbackType` / `equivalence` / `confidencePermission` / `posturePermission`, but the frontend `SignalFollowUpDto` does not carry them. If buyer copy ever needs proxy nuance, serialize a minimal, buyer-safe subset rather than re-deriving it in the frontend.
- **No new sources, no activation:** none of the above adds a provider, activates probe-refresh, or mutates confidence/posture/lean. They are read-only metadata enrichments, each its own future slice.

## 8. deferred decisions

This review does not resolve any deferred runtime or product decision. It clarifies the productization direction and the metadata preconditions.

- Deferred-ledger **entry 19 (Sports Signal Source and Fallback Catalog)** is clarified: this review confirms the source/fallback gaps the Brief Signal Table surfaces are real and recurring-worthy, and adds that a buyer route additionally needs availability-fidelity and source-kind metadata before the `Unavailable` / source-provenance labels can be shown to a buyer. Source expansion remains **Deferred** and **not resolved**.
- Probe-refresh activation, confidence/posture/lean mutation, and the tenant/Stripe economic boundary remain **Deferred** and explicitly **not resolved** by this review.

## 9. next slice recommendation

**Primary: Buyer Artifact Route v1.**

Build a thin, read-only buyer signal-table module on the buyer path (a brief view, or a signal-context module on the analyzer result), reusing the existing `buildSignalTable` projection and the existing artifact read endpoint. Scope: curated, plain-language category labels (section 6), a directional indicator, a short flag phrase, an honest state with a muted dash for not-applicable, confidence as a band, and the narrative spine (summary, counter-case, watch-for, what-would-change). It must not add a source, activate probe-refresh, mutate confidence/posture/lean, expose dev-only sections, or leak factory internals. This is the single highest-value step toward the product promise ("the signal table is the product") and toward money as validation.

**Backup: Artifact Copy and Section Order v1.**

A docs-only slice that locks the buyer label set, the canonical-key -> category mapping, the directional-indicator semantics, the missing/proxy/not-applicable copy, and the confidence-band phrasing before any buyer surface is built. Choose this if the team wants the overclaim-risk surface (buyer copy) reviewed and fixed in docs before writing the route.

## anti-goals

- Do not build the buyer route in this review slice.
- Do not add a data source, provider, or ToolGateway call.
- Do not activate probe-refresh, a station runner, or a merge writer.
- Do not mutate confidence, posture, lean, or any decision field.
- Do not change schema, prompts, model-call count, or the artifact contract.
- Do not promote dev-only sections (Cognitive Protocol, Pipeline Steps, Raw Artifact, Signal Availability/Follow-Up tables, Source Type, Probe) to a buyer surface.
- Do not introduce prediction-engine or lock/pick language.

## references reviewed

- `<DAI_VAULT_ROOT>/04 Products/sports-v1/product-brief.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/value-proposition.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/ui-concept.md`
- `<DAI_VAULT_ROOT>/04 Products/sports-v1/product-vs-factory-deliberation-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/design-readiness-and-focus-selection-v1.md`
- `<DAI_VAULT_ROOT>/02 Platform/architecture/cognitive-factory/deferred-runtime-decisions-ledger-v1.md`
- `<DAI_VAULT_ROOT>/06 Execution/handoffs/current-slice.md`
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/dev-artifact-review/` (signal-table projection + artifact-review surface)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/analyzer/analyzer.component.ts` (buyer surface; consumes AgentRunResultDto, no signal table)
- code: `<DAI_REPO_ROOT>/apps/sports-app/src/app/core/models/agent-run.model.ts` (DTO shapes)
- code: `<DAI_REPO_ROOT>/platform/dotnet/DevCore.Api/AgentRuns/SignalQualityEvaluator.cs`, `SignalFollowUpEvaluator.cs` (status/quality/confidence-effect/fallback vocabulary)

## verification performed for this note

- docs-only change; no runtime, schema, prompt, or Angular code changed.
- no exact local machine paths added; placeholders used throughout.
- ASCII check for this note passed.
- git status / diff --check results recorded in the slice handoff addendum.
