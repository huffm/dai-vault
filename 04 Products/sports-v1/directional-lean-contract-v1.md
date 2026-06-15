# Directional Lean Contract v1

**date:** 2026-06-15
**status:** diagnosis + one narrow read-projection fix. backend code + tests in `dai`; report/ledger in `dai-vault`. no model spend, no regeneration, no outcomes/evaluations written, no analyzer/prompt/matcher/confidence/buyer change.
**classification:** contract clarification + export-projection defect fix.

## Contract definition

Directional lean and action posture are two separate axes and the system already treats them so:

- **DirectionalLeanSide** -- which side the signal picture favors: `home`, `away`, or `null`. `null` means *no directional read was possible*, not "no action".
- **ActionPosture** -- how aggressively to act on the read: `play` / `monitor` / `wait` / `pass`. This is the buyer-action axis and may be cautious (`wait`) even when a directional side exists.
- **Confidence** (calibrated, evaluator-owned) and **EvidenceSufficiency** (richness band) are slicing/gating dimensions. They influence posture, not whether a side exists.

Doctrine: **do not force conviction; force orientation only when evaluable.** A valid low-conviction read is `LeanSide=home, Posture=wait, Confidence=0.375`. A valid null read is `LeanSide=null` *with a non-evaluable reason* (no grounded signal, missing odds/identity, source unavailable, postponed, unsupported market, or the model finding no directional edge in the available signal).

## Why the two axes must stay separate

A game can be unsuitable for a buyer-facing recommendation (thin evidence -> `wait`) while still being directionally useful for internal outcome attribution (a `home` lean that can be graded correct/incorrect after settlement). Collapsing `wait` into `null` would silently discard gradable directional signal and weaken Stage 0 calibration.

## Null-lean rules

`LeanSide = null` is permitted only for a true non-evaluable read, and the artifact must carry the reason in existing fields (the model's `lean` prose, `signalAvailability`, `groundedSignals=[]`, degraded `retrieve` note, etc.). Null must never be a side-dropping artifact of posture or a confidence gate.

## Probe behaviour (as built)

Question/Interrogate already names evidence gaps and the deterministic `interrogate.probe` summarises follow-up needs; the analyzer grounds MLB on `starting_pitching` (statsapi) and NBA on odds/market context. This slice added no new data source and triggered no new fetch. When no signal grounds (priors-only run), the model is allowed to abstain (`No clear lean`) rather than invent a side -- consistent with the "do not force fake conviction" boundary.

## Mechanism audit (where the side flows)

Traced the full path; the structured side is preserved end to end:

- **Model output** (`services/agent-service/app/services/sports_analyzer.py`): the prompt defines a structured `lean_side` field (`home`/`away`/null, lines 124-130), explicitly *separate* from prose `lean` and from posture; `_parse_response` (lines 419-420) clamps only invalid values to null and passes `home`/`away` through. No confidence/posture gate nulls it.
- **.NET compose** (`AgentRuns/SportsComposer.cs:47`): `LeanSide: analyzerOutput.LeanSide` -- straight passthrough into `AgentRunExecutionResult`.
- **Persistence** (`Controllers/AgentRunsController.cs:102`): `run.LeanSide = result.LeanSide` -- denormalised to the `AgentRun.LeanSide` column the matcher reads. No gate.
- **Artifact read DTO** (`GetArtifact`, `AgentRunArtifactDto`): **projected prose `Lean` but dropped structured `LeanSide`** -- the one defect (see below).

Answers to the slice's primary questions: (1) the model's side is **not** dropped by analyzer/compose/persistence; (2) `wait`/`pass` is **not** interpreted as null lean -- posture and lean_side are independent fields; (3) confidence/evidence gating does **not** suppress the side; (4) `AgentRun.LeanSide` is saved correctly; (5) artifact v3 represents lean separately from posture; (6) **yes -- the calibration export/read DTO dropped `leanSide` entirely** (fixed); (7) the three MLB nulls are **true non-evaluable model abstentions**, not improper abstentions.

## Findings from the 10 MLB runs

10-run diagnosis (DB `AgentRun.LeanSide` is authoritative; export `leanSide` shown after the fix):

| # | run id (8) | matchup (away @ home) | gamePk | AgentRun.LeanSide | posture | conf | evid | model lean prose | raw model side | export leanSide (post-fix) | usable | null class |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | `2303433e` | Marlins @ Phillies | 823452 | home | monitor | 0.75 | 1 | "Slight lean toward Phillies..." | home | home | yes | - |
| 2 | `2803433e` | Royals @ Nationals | 822724 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |
| 3 | `2a03433e` | Mets @ Reds | 824505 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |
| 4 | `2e03433e` | Padres @ Cardinals | 823046 | null | wait | 0.375 | 0 | "No clear lean..." | null | null | no | TrueNonEvaluable_SourceUnavailable (grounded=[]; starter data unavailable; priors-only) |
| 5 | `3403433e` | Rockies @ Cubs | 824666 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |
| 6 | `3603433e` | Twins @ Rangers | 822887 | null | wait | 0.375 | 0 | "No clear lean..." | null | null | no | TrueNonEvaluable_SourceUnavailable (grounded=[]; priors-only) |
| 7 | `3d03433e` | Tigers @ Astros | 824181 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |
| 8 | `4103433e` | Angels @ Diamondbacks | 825071 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |
| 9 | `4203433e` | Pirates @ Athletics | 824993 | null | wait | 0.5 | 1 | "No clear lean..." | null | null | no | TrueNonEvaluable_NoDirectionalSignal (starting_pitching grounded but uninformative -- unknown starter handedness) |
| 10 | `4903433e` | Rays @ Dodgers | 823938 | home | monitor | 0.75 | 1 | home lean | home | home | yes | - |

7 directionally usable (`home`), 3 null. The 7 usable map exactly to posture `monitor` / conf 0.75 / evid 1; the 3 null to posture `wait`.

## Diagnosis of the three null-lean cases

All three are **honest model abstentions**, not improper abstentions:

- The model itself emitted `lean: "No clear lean..."` and `lean_side: null` in every case (confirmed in the persisted artifacts). No downstream gate erased a stated side.
- 2e03433e and 3603433e had **zero grounded signals** (`groundedSignals=[]`, `retrieve` Degraded "priors-only run", `starting_pitching` unavailable) -- there was no directional evidence to orient on. Forcing a side here would fabricate conviction from nothing, which the slice boundary forbids.
- 4203433e had `starting_pitching` grounded but directionally empty (both starters young, handedness unknown) -- the model judged no side favoured. Also a legitimate `null` per the prompt's "no meaningful directional lean" rule.

None classify as `ImproperAbstention_WaitErasedSide`, `ImproperAbstention_ConfidenceGateErasedSide`, `AnalyzerMappingDefect`, or `PersistenceDefect`. The premise that the system conflates "no action" with "no lean" is **false at the contract/persistence layer**; it only *looked* true through the export, which never showed the side at all (defect below).

## Export projection defect and fix

**Defect:** `AgentRunArtifactDto` (the `GET /api/agent-runs/{id}/artifact` projection that the calibration export dumps) surfaced prose `Lean` but omitted the structured `LeanSide`, even though it is persisted on the artifact and denormalised to `AgentRun.LeanSide`. So every exported artifact showed no side -- making the 7 usable home leans invisible and indistinguishable from the 3 genuine nulls.

**Fix (narrow, additive, read-only):** added `LeanSide` to `AgentRunArtifactDto` and projected `artifact.LeanSide` in `GetArtifact`. No change to analyzer, prompt, matcher, confidence, posture, buyer copy, or persistence. Verified live (no spend): usable run -> `leanSide=home`, null run -> `leanSide=null`.

## Code changes

- `platform/dotnet/DevCore.Api/AgentRuns/AgentRunContracts.cs`: `AgentRunArtifactDto` += `string? LeanSide` (between `Lean` and the optional block).
- `platform/dotnet/DevCore.Api/Controllers/AgentRunsController.cs`: `GetArtifact` projects `artifact.LeanSide`.
- tests (`platform/dotnet/DevCore.Api.Tests/Integration/AgentRunsControllerTests.cs`): existing artifact test now asserts `dto.LeanSide == "home"`; new `artifact_endpoint_projects_null_lean_side_for_non_evaluable_run` asserts a `null` side passes through faithfully (`with { Lean="No clear lean", LeanSide=null }`).

No prompt/schema change was made: the model contract already requires `lean_side` separate from posture and already reserves null for non-evaluable reads. Tightening abstention would risk forcing fake conviction on zero-signal games -- explicitly out of bounds.

## Verification

- TDD: red observed (test project failed to compile -- `AgentRunArtifactDto` had no `LeanSide`), then green.
- targeted: artifact + matcher tests 54/54. full `DevCore.Api.Tests` **660/660** (was 659; +1 new test). matcher and identity-capture tests included and green.
- no-spend live smoke on real generated runs: `2303433e` -> `leanSide=home`; `2e03433e`/`3603433e` -> `leanSide=null`. Outcomes/evaluations unchanged at **12/12** -- nothing written.

## Remaining risks

- The export-projection inconsistency is closed, but other read surfaces (Angular buyer projection) consume `lean` prose / their own derivation; they were not in scope and not changed. If a future internal read surface aggregates the export, it now sees the side.
- Open doctrine question (not a bug, deferred to the operator): should a **priors-only** MLB game (zero grounded signals) still produce a low-confidence directional lean on priors (records/home-field) rather than abstain? Current behaviour abstains, which preserves honesty but yields no calibration signal for those games. Changing it is a prompt/doctrine decision with fabrication risk -- not taken here.
- The 7 usable runs remain pending settlement; reconciliation is a later slice.

## Recommended next slice

**Reconcile the 7 directionally usable MLB runs after settlement** through the proven `POST /api/agent-runs/reconcile` statsapi path, then read back evaluations -- this roughly triples the directional sample (prior 3 + 7). The 3 null-lean runs may be reconciled for completeness but will evaluate inconclusive. Separately, if the operator wants priors-only games to orient, that is a scoped **Priors-Only Orientation Doctrine** decision, not a defect fix. WNBA remains deferred.
