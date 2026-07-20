# daily evidence acquisition orchestrator v1

**status:** active architecture — Stage 0 (Daily Evidence Planner) offline core implemented under
WI-0034 Slice 1; all later stages and interfaces deferred and separately gated
**date:** 2026-07-19

## purpose

Name the umbrella control loop for sports evidence acquisition and bind the architecture and
authority boundaries of its first stage, the Daily Evidence Planner, whose canonical result is
the Daily Evidence Board. This is niche/domain architecture for the sports-v1 evidence operation;
it is deliberately NOT platform core.

## problem it solves

The 2026-07-17/18 operations sessions showed the recurring failure modes of ad-hoc evidence
planning: schedules mistaken for cohort worthiness; "no eligible candidates" claimed without an
evaluated slate; deficits pursued with cohorts that cannot close them (market-agreeing volume vs a
disagreement deficit); and administrative latency consuming usable windows. The planner encodes
those decisions deterministically so the operator reviews a board instead of re-deriving doctrine.

## mental model

```text
calibration + settled evidence state (canonical authority verdict)
-> evidence need -> cohort worthiness -> required input capabilities
-> input addressability -> slate evaluation (only when inputs support it)
-> identity + schedule-state validation -> eligibility
-> lexicographic ranking of eligible candidates only
-> deterministic primary/reserve allocation
-> Daily Evidence Board (exactly one closed outcome)
```

Stages of the umbrella (context only; nothing beyond Stage 0 is designed or authorized here):
Stage 0 planner -> screening -> capture authorization -> capture -> settlement -> calibration
feedback. Each later stage requires its own authorization.

## what it is

- A pure, offline, deterministic decision core: `services/agent-service/app/services/
  daily_evidence_planner.py` (WI-0034 Slice 1), tested by fixtures only.
- A consumer of the canonical evidence-sufficiency authority: it takes the
  `pooled_calibration` verdict (`discrimination_hybrid_v1`: conclusionsAllowed + failingReasons)
  as normalized input and never recomputes sufficiency or copies thresholds.
- The owner of the Daily Evidence Board contract: six mutually exclusive outcomes
  (cohort proposed / no additional evidence warranted / evidence needed but input types not
  addressable / validated slate with zero eligible candidates / diagnostic required / narrow
  operator decision required), stable reason records, canonical byte-deterministic JSON, and a
  Markdown projection that introduces no decision absent from JSON.

## what it is not

- Not screening, capture, settlement, tuning, or scheduling — every board carries
  `screening_authorized/capture_authorized/execution_authorized: false`.
- Not a platform facility: WI-0031 capability selection remains the generic platform layer. The
  planner emits implementation-independent missing-capability records (required input type +
  blocked objective + reasons; never a tool id); a later, separately authorized WI-0031
  pilot/consumer boundary may map them to capability recommendations.
- Not a market oracle: with no market input, `market_availability` is
  `unknown_until_paid_screening`; schedule presence never implies market state.
- Not a CLI, adapter, wrapper, skill, service, or persistence layer (Slices 2-4, deferred).

## authority model

- Evidence sufficiency: `pooled_calibration` (single canonical authority; the planner consumes
  its verdict; unknown criterion = fail-closed POLICY_VERSION_MISMATCH).
- Domain semantics (failing-reason -> objective -> required input capabilities; worthiness of
  capture per objective; eligibility vocabulary): this planner, as niche doctrine.
- Execution/screening/capture authority: none here; later stages under their own authorizations.
- Identity: two-level — candidate-level failures exclude the candidate; terminal
  UNRESOLVED_IDENTITY only when no trustworthy candidate remains.

## key invariants (implemented and test-proven)

Pure functions with explicit injected time; eligibility precedes ranking; rank never rescues an
exclusion; unknown is never favorable; `slate_evaluated == false` implies slate status
NOT_EVALUATED and a null (never zero) eligible count; zero-eligible is reportable only after an
evaluated slate; pools are disjoint, eligible-only, limit-bounded, deterministically allocated;
identical inputs (any candidate order) yield byte-identical canonical JSON; no board outcome
grants any authority.

## truth hierarchy

1. Planner source + tests (`daily_evidence_planner.py`, `test_daily_evidence_planner.py`).
2. The canonical policy authority (`pooled_calibration.py`) for sufficiency.
3. This record + WI-0034 for boundaries and intent.
4. Handoffs and navigation records.

## related docs

- `02 Platform/system-development/work-items/WI-0034-daily-evidence-planner-stage-0.md`
- `02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md`
- `06 Execution/reports/gate4-discrimination-sufficiency-criterion-2026-07-05-v1.md`
- `04 Products/sports-v1/calibration/` (settled evidence corpus doctrine)

## typed input-authority contract (2.0, slice-2 review correction, 2026-07-19)

**Corrected defect (blocker):** the 1.0 contract derived input addressability from
caller-declared capability identifier strings. A caller asserting
`input.market_divergence_screen` while supplying only schedule candidates obtained
`COHORT_PROPOSED_FOR_OPERATOR_REVIEW` (reproduced before correction, both market paths).
A capability identifier asserted by a caller is not evidence that the capability was
evaluated.

**Contract:** capability availability is now derived entirely from typed input-evidence
envelopes (`input-evidence-envelope/1.0`): capability id; envelope schema version;
policy/classifier version; closed evaluation status (`evaluated | not_evaluated |
unavailable | stale | invalid` -- NotEvaluated never equals Available); observation
timestamp; source provenance; target date; provider-scoped candidate identity
(`source_provider` + `external_event_id`) for candidate-scoped capabilities; stable
producer reason codes; and a normalized result (`classification: includable | excluded`
for screens). The planner rejects or excludes: wrong-candidate evidence (orphans),
wrong-target-date, stale, duplicate/conflicting records, incompatible versions, records
missing provenance or observation timestamps, and evaluated screens without an explicit
result. Global availability never implies per-candidate coverage: partial coverage is
explicit, ungrounded candidates are excluded, and a proposed pool contains only candidates
with every required typed input satisfied. Producers own screen thresholds; the planner
consumes verdicts and reasons only.

**Semantic migration:** actual dai-versus-market disagreement exists only after a dai
directional decision and a decision-time market baseline; no pre-generation input can
observe it (every board carries `dai_market_disagreement: unknown_until_generation`). The
pre-generation input is a market-contrast candidate screen. The overloaded id
`input.market_divergence_screen` is therefore RETIRED and replaced by
`input.market_contrast_screen`; the retired id in an envelope is rejected with
`retired_capability_id`, never silently remapped. `input.market_coverage` keeps
`input.market_snapshot` (decision-time market-observation coverage), now candidate-scoped
and envelope-grounded.

**Versions:** request `daily-evidence-planner-request/2.0`, board `daily-evidence-board/2.0`,
planner `daily-evidence-planner/2.0`, cli `daily-evidence-planner-cli/2.0`. 1.0 requests
and boards remain parseable but are rejected as version mismatches (exit 5); no silent
migration -- producers re-emit under 2.0. Version ownership: envelope schema and board
contract are owned here (planner core); classifier versions are owned by the screen
producer; the policy criterion remains owned by pooled_calibration.

## review-corrected contracts (2026-07-19)

The independent Slice-1 review tightened four contracts recorded here: candidate identity is
provider-scoped (`source_provider` + `external_event_id`); an unrecognized failing-reason code
from the canonical criterion routes to `DIAGNOSTIC_REQUIRED_BEFORE_TRUSTWORTHY_DECISION`
(`unrecognized_deficit_code`) instead of becoming a generic objective; missing start time or
team identity excludes a candidate (unknown values never rank favorably); and
`allowed_operator_actions` is outcome-specific and never authority-bearing. Details in the
Slice-1 closeout's review-corrections section.

## recommended next slice

Independent review + integration of the WI-0034 Slice-1 branches; afterwards, separate
authorizations may consider planner Slice 2 (CLI) and WI-0031 Slice 4 informed by this real
consumer. Nothing further is authorized by this record.
