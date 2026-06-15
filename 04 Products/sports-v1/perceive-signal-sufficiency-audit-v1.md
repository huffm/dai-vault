# Perceive Signal Sufficiency Audit v1

**date:** 2026-06-15
**status:** audit only. read-only inspection of existing artifacts/DB output. no code, no model calls, no generation, no reconciliation, no outcomes/evaluations changed (12/12). no sufficiency gate implemented.
**classification:** observability/sufficiency diagnosis.

## Core question

Do current sports artifacts capture enough raw-signal and protocol-stage trace to attribute a directional lean / null / confidence to source coverage and protocol-stage behavior?

Short answer: **raw-signal coverage and protocol-stage prose are well instrumented; what is missing is structure** -- source grouping, tiering, a persisted sufficiency band, and a typed reason code distinguishing "null from source insufficiency" vs "null from low directional separation". The first is structurally observable today; the second is only in model prose.

## What signal trace exists today (per artifact)

- `groundedSignals` / `missingSignals`: named-signal lists (not just a summary). Named categories come from a fixed catalog (`_KNOWN_SIGNAL_CATEGORIES` in `sports_analyzer.py`): market, injury_report, situational, weather, rest_fatigue, matchup_style, home_court, starting_pitching, bullpen, lineup_form, ballpark, sharp_public.
- `evidenceRichness`: integer count = len(groundedSignals). The only structured evidence metric.
- `signalAvailability[]`: per-signal records `{signal, status (grounded/missing), source (e.g. mlb_statsapi), reason, detail, quality (usable/unavailable), decisionUse (supporting_context/not_usable), followUpSignals, confidenceEffect}`. This is the richest structured surface.
- `signalFollowUps[]`: the deterministic fallback-ladder diagnostics surface (empty on all 13 runs here -- no follow-up was raised at runtime).
- denormalized: `LeanSide` (home/away/null), `confidence`, `posture`, plus the prose `lean`.

## What protocol micro-stage trace exists today

`cognitiveProtocol` records model-emitted prose for every micro-stage: `perceive{detect,frame,aim}`, `interrogate{question,probe,verify}`, `discern{weigh,contrast,stress}`, `decide{resolve,position,justify}`, `synthesize{...}`. Every stage is present as narrative. Notable: `interrogate.probe` is **null** on all inspected runs (the deterministic probe field surfaces a structured gap only when raised; it is not a record of a live fallback fetch -- the probe-refresh chain is dormant, ledger entries 1-3). `discern.contrast` is null on the low-separation run.

## Audit table (representative runs)

| run | gamePk | leanSide | conf | ev | grounded | signalAvailability.status | discern.contrast | null/low driver (from trace) |
|---|---|---|---|---|---|---|---|---|
| 2303433e (usable orig) | 823452 | home | 0.75 | 1 | [starting_pitching] | grounded | present | directional separation from starter |
| 2e03433e (old null) | 823046 | null | 0.375 | 0 | [] | **missing/unavailable** | n/a | **source insufficiency** (starter unannounced) |
| 3603433e (old null) | 822887 | null | 0.375 | 0 | [] | **missing/unavailable** | n/a | **source insufficiency** |
| 4203433e (old null) | 824993 | null | 0.5 | 1 | [starting_pitching] | grounded | null | **low directional separation** (prose only) |
| 5003433e (rerun) | 823046 | home | 0.75 | 1 | [starting_pitching] | grounded | present | starter now available -> separation |
| 5403433e (rerun) | 822887 | home | 0.75 | 1 | [starting_pitching] | grounded | present | starter now available -> separation |
| 5703433e (rerun) | 824993 | null | 0.5 | 1 | [starting_pitching] | grounded | null | low separation persists (both RH starters) |

## Old-vs-new null rerun signal comparison

- **823046 (Padres@Cardinals) & 822887 (Twins@Rangers):** old `signalAvailability.status = missing/unavailable` for starting_pitching, `groundedSignals=[]` -> null. Rerun `status = grounded` -> `home`. The **single signal that changed is starting_pitching** (became available between 12:37Z and 20:06Z). This is **structurally observable** from `signalAvailability.status` and `groundedSignals` alone -- no prose needed.
- **824993 (Pirates@Athletics):** starting_pitching `grounded` in BOTH runs, leanSide null BOTH times. The trace shows why only in **prose**: `perceive.detect` "two right-handed starters with no notable advantages", `discern.weigh` "starting pitching is noted, but ... bullpen and lineup form are missing", `decide.justify` "lack of clear signals beyond starting pitching", `discern.contrast = null`. So the cause is **low directional separation + thin supporting coverage**, inferable but not typed.

## Answers to the audit questions

1. Raw signals tracked as named signals? **Yes** (catalog-backed), not just summarized.
2. Grouped by source group? **No.** Individual signals only; no group concept in code (grep clean except venv noise).
3. Source groups tiered? **No.** No groups, no critical/supporting tiers.
4. Missing critical signals recorded? **Partial.** `missingSignals` + `signalAvailability.decisionUse (not_usable)` exist, but no "critical" designation -- nothing marks identity_schedule/market_odds/starting_pitching as critical.
5. Perceive output recorded? **Yes** (`cognitiveProtocol.perceive`).
6. Question/Probe/Verify recorded? **Question/Verify yes** (prose). **Probe present-as-field but null** in practice.
7. Can we tell whether Probe attempted fallbacks? **No.** `interrogate.probe` null + `signalFollowUps` empty; the refresh/fallback chain is dormant, so there are no live fallback attempts to record.
8. Which signal changed rerun null->home? **starting_pitching** (became grounded) -- structurally observable.
9. Why did 824993 stay null? **Low directional separation** (both RH starters, no edge) + missing supporting signals -- observable only in prose.
10. Can we distinguish the four causes?
    - **source insufficiency: yes, structurally** (`signalAvailability.status = missing/unavailable`, `groundedSignals=[]`).
    - **low directional separation: no structured field** -- prose only (grounded-but-null is the structural fingerprint, but the reason is not typed).
    - **model abstention vs low separation: not separable** structurally (both look like grounded + leanSide null).
    - **missing trace/instrumentation: yes** (`cognitiveProtocol == null` marks analyze-failure).
11. Phase-level value attribution? **Partial.** Per-stage prose exists; no structured per-stage contribution/score, no directional-separation metric. Cannot numerically attribute "Discern added X".
12. What is missing before a Perceive Sufficiency Gate?
    - a **source-group taxonomy** (named signal -> group) -- absent.
    - **group tiering** (critical vs supporting) -- absent.
    - a **persisted sufficiency band** (insufficient/thin/moderate/rich). Today only `evidenceRichness` (count) is persisted; the band is derived in the Angular buyer projection, not in the artifact.
    - a **typed null/low-confidence reason code** (source_insufficient | low_separation | abstention) -- prose only today.

## Source-group / tier readiness

A **first-cut source-group coverage score is derivable from existing data today** -- the named signals map deterministically onto the proposed groups (e.g. `starting_pitching` -> starting_pitching group; `market` -> market_odds; the `SourceProvider/ExternalGameId` identity columns -> identity_schedule), and grounded-group counts come straight from `groundedSignals`. So a deterministic Source Group Taxonomy + coverage/band can be built with **no model or prompt change**.

What it could NOT yet do: distinguish a grounded-but-null read's cause as low-separation vs abstention (no typed reason), and there is no criticality tier to gate on. Critically, a deterministic null-reason **could** be derived today: `groundedSignals=[]` + critical signal unavailable => `source_insufficient`; all critical signals grounded + leanSide null => `low_separation_or_abstention`. That inference needs the group/tier definitions first.

## Whether current artifacts support Perceive sufficiency scoring

- **Source-group COVERAGE scoring: yes** -- derivable deterministically from existing named-signal coverage; supports a band (insufficient/thin/moderate/rich) without new model output.
- **Criticality-driven GATING: not yet** -- needs a tiered group taxonomy (which signals are critical).
- **Cause attribution (insufficiency vs low-separation): partial** -- source insufficiency is structural; low-separation is prose-only until a typed reason code (deterministically derivable once groups/tiers exist, or model-emitted) is added.
- **Phase-level value attribution: not yet** -- needs structured per-stage contribution, not just prose.

## Recommended next slice

**Source Group Taxonomy v1** (contract + deterministic projection, no model/prompt change) is the right next observability step: map the existing named signals into the proposed groups, assign critical/supporting tiers, and derive a coverage/sufficiency band + a deterministic null-reason from existing artifact fields. It is the prerequisite that a later **Perceive Sufficiency Gate Contract v1** needs, and it is buildable from data already captured.

Sequencing note: this observability work is **independent of reconciliation**. Reconciling the 9 active usable MLB runs after tonight's settlement remains the calibration-evidence priority and is not blocked by this audit; the taxonomy slice can run in parallel or after. Protocol Trace Enrichment v1 (typed reason codes / per-stage contribution) and Probe Fallback Catalog v1 are later steps, gated on the taxonomy and on a decision to activate any live fallback.

## Risks / deferred decisions

- The grounded-but-null cause (824993) is currently legible only as model prose; treating prose as a gate input would be fragile. A typed reason code (deterministic where possible) should precede any gate.
- The sufficiency band lives only in the frontend buyer projection today; a gate needs it server-side/persisted -- a promotion decision (ledger entry 23 adjacency) that this slice does not take.
- No source grouping/tiering exists; defining "critical" groups (identity_schedule, market_odds, starting_pitching) is a doctrine decision for the taxonomy slice, not assumed here.
- No code, prompt, confidence, buyer, or matcher change made; no sufficiency gate implemented (per slice boundary).
