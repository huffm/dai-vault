---
title: "Model-Assisted Capability Recommender Slice 3 Closeout v1 (2026-07-19)"
type: "evidence-report"
date: "2026-07-19"
status: "complete"
project: "DAI"
slice: "WI-0031 Model-Assisted Capability Recommendation and Tool Selection v1 (Slice 3)"
repos:
  dai: "code+tests (python recommender + .net receiving projection; additive; branch wi/0031-model-assisted-capability-recommender)"
  dai-vault: "docs-only"
tags:
  - platform-architecture
  - capability-selection
  - model-boundary
  - implementation
related:
  - "02 Platform/system-development/work-items/WI-0031-model-assisted-capability-recommendation-and-tool-selection-v1.md"
  - "02 Platform/architecture/capability-recommendation-and-tool-selection-standard-v1.md"
  - "06 Execution/reports/capability-tool-registry-resolution-slice-2-closeout-2026-07-18-v1.md"
---

# model-assisted capability recommender slice 3 closeout v1

## purpose

Record WI-0031 Slice 3: the semantic recommendation stage. A model extracts normalized
ingredients and recommends implementation-independent capabilities from a bounded, versioned
ontology; all model output is treated as untrusted data behind a strict validation boundary; only
validated recommendations project into the Slice 2 `CapabilityRecommendationInput` contract. The
model selects no tool, sees and produces no access facts, and authorizes nothing.

## context

Coordinated branches from the integrated Slice-2 heads: dai
`wi/0031-model-assisted-capability-recommender` from `b1c068a`, vault same-named branch from
`f608329`. No endpoint, persistence, live model call, or execution authority.

## model-call authority decision (the gating step)

**Python `services/agent-service` owns external model calls** -- the canonical seam is the
`AsyncOpenAI` singleton (`_get_client()` in `sports_analyzer.py`); `DevCore.AiClient` (C#) is an
HTTP client to FastAPI, not a provider stack. Per the binding rules the recommender therefore
lives at the python model boundary and reuses that authority: a provider-neutral `ModelPort`
protocol whose production adapter is the existing AsyncOpenAI seam (no second provider stack, no
new prompt-provenance dialect), with metering through the existing
`model_metering.estimate_cost`/pricing-status conventions. Execution authority and the resolution
contracts remain in .NET; the two sides communicate through a strict language-neutral JSON
contract (`capability-recommendation/1.0`) with NO endpoint or cross-service runtime connection in
this slice -- the receiving projection is tested with fixtures.

## scope

Included: python recommender module (input contracts, ontology contract, versioned prompt, strict
output validation/normalization, statuses, metering, canonical projection JSON) + tests; .NET
receiving projection + tests incl. Slice 1-3 offline composition; WI-0031 Slice-3 disposition;
this closeout; the current-slice append. Excluded: endpoint/controller/UI/scheduler; live tool
execution; registry or recommendation persistence; telemetry database; schema migrations; second
provider stack; production prompt-routing changes; recipe compilation (Slice 4); Daily Evidence
Planner integration; any live or paid model call.

## key decisions and invariants proven

1. **Untrusted-output trust boundary.** model response -> strict parse -> schema validation
   (unknown fields rejected) -> semantic validation (ontology membership, references, ranges,
   duplicates) -> deterministic normalization. The model can never construct
   `CandidateAccessFacts`, `ResolvedCandidate`, `ToolCandidateInput`, `SelectionTrace`, gateway
   commands, recipes, or permission decisions; a forbidden-key scan
   (tool/access/permission/execution/recipe families) fails the whole response closed.
2. **Capabilities, not tools.** The ontology gives the model capability ids + descriptions only
   (no tool implementation ids, no permissions, no cost authority); unknown needs go to the
   unmapped section, which is retained for gap telemetry and never projected into an executable
   capability id.
3. **Prompt contract** (`capability-recommender.v1`): signals are declared untrusted data inside
   delimited envelopes; embedded instructions must be ignored; ontology-only recommendations; no
   tool naming; no permission inference; strict-schema output; no hidden chain-of-thought
   requested.
4. **Deterministic handling.** Duplicates (capability or ingredient) reject the full response --
   one documented rule, no input-order tie-break; normalization sorts sections by stable identity;
   the canonical projection JSON is byte-identical for semantically identical outputs.
5. **Failures fabricate nothing.** timeout/refusal/unavailable/empty/malformed produce structured
   statuses with zero recommendations; metering/provenance are still retained; input-bound
   violations (too many signals etc.) fail before any model request is sent.
6. **Receiving projection re-validates independently** (.NET): schema + ontology version
   mismatches fail closed; forbidden keys fail closed; only `completed*` statuses may project;
   unmapped never projects; ordering is ordinal; projected candidates carry no access facts and no
   score authority.
7. **Slice 1-3 composition (offline):** fake model output -> validated recommendation ->
   `CapabilityRecommendationInput` -> Slice 2 resolver -> Slice 1 `BuildTrace`: a policy-denied
   candidate stays shadow regardless of model relevance; only accessible-under-evaluated-checks
   candidates can be selected; capture and screening remain false; no tool executes.

## evidence

- files (dai, additive; branch `wi/0031-model-assisted-capability-recommender`, commit `cb5396b`):
  `services/agent-service/app/services/capability_recommender.py` (new),
  `services/agent-service/tests/test_capability_recommender.py` (new),
  `platform/dotnet/DevCore.Domain/CapabilitySelection/RecommendationProjection.cs` (new),
  `platform/dotnet/DevCore.Api.Tests/CapabilitySelection/RecommendationProjectionTests.cs` (new).
  No existing source modified; no `.csproj`/`.slnx`/package change; csproj phantom untouched.
- tests: python 11 new (fake ports; injection boundary; duplicates; refs; bounds; failure
  statuses; determinism; input bounds with request-never-sent proof) -- full agent-service suite
  **470 passed**; C# 6 new (projection validity, mismatch fail-closed, failure statuses,
  forbidden keys, invalid ids/ranges, Slice 1-3 composition) -- full DevCore.Api.Tests
  **1312 passed / 0 failed / 0 skipped**. `DevCore.Domain` builds 0 warnings. Zero live/paid/
  network model calls anywhere in verification.
- pre-write gate: dai `b1c068a` 0/0, vault `f608329` 0/0; drift classified + disjoint; strict
  snapshot exit 0 / 0 warnings / 21 WIs; branches created before first write from verified heads.

## security findings

Credentials/tokens/connection strings/transport config cannot enter the model request (the input
contract has no such fields; signals are bounded text with declared sensitivity); model output is
never trusted as policy; tool ids are rejected wherever they appear; access facts remain
adapter-produced (Slice 2); tenant contexts are not combined (single opaque scope per request);
raw unrestricted signals are not logged or persisted by this module; diagnostics carry codes and
ids, not signal bodies; unmapped suggestions are non-executable; no model-created identifier can
become a registered capability without a separate governed action.

## safety / non-actions

0 live/paid model calls; 0 network calls; 0 tool calls; 0 services; 0 database reads/writes; 0
application-data writes; 0 endpoint/controller/DI wiring; 0 persistence; 0 schema/migration; 0
project/package changes; 0 production prompt-routing changes (the existing sports prompt path is
untouched); 0 pushes/merges. The recommender and projection are unwired library code.

## review correction (2026-07-19, dai commit `ac634b5`, WI: WI-0031)

The independent Slice 3 review found and corrected, on the branch before integration:
(1) **aggregate request budgets** -- per-item bounds alone did not bound the complete request; added
deterministic character budgets (total signal content 48k, ontology entries 128 / text 16k, final
serialized prompt 60k) rejected BEFORE any ModelPort call (`not_sent`, zero port calls; documented
as conservative character budgets, no tokenizer or network dependency); (2) **strict json** --
python's default `json.loads` accepts `NaN`/`Infinity` constants and silently keeps the last of
duplicate object keys; both now reject as `malformed_json` (`parse_constant` + `object_pairs_hook`);
(3) **prompt fingerprint** -- the static instruction template is sha256-fingerprinted
(`PROMPT_FINGERPRINT`) and carried in provenance so the version label cannot silently refer to
different rule text; (4) **cross-language contract vectors** -- python `to_projection_json` is the
producing authority for `capability-recommendation/1.0`; the exact canonical valid vector (and a
poisoned `tool_id` variant) is now embedded by value in BOTH suites with a documented
update-together ownership rule; (5) **forbidden-key scan proven prose-safe** -- field names only;
tool/permission/execution words inside bounded description values are accepted. Updated totals:
python 16 tests / full suite **475 passed**; C# 8 projection tests / full suite **1314 passed / 0
failed**. Sensitivity classification remains descriptive metadata, not enforced redaction.

## next step

Independent Slice 3 integration (this review), then **WI-0031 Slice 4 -- Deterministic
Eligible-Candidate Ranking and Bounded Recipe Planning** under separate authorization: versioned
contextual weight profiles over ELIGIBLE candidates only, deterministic tie-breaking, and a bounded
recipe PLAN. Slice 4 must not invent or duplicate missing canonical policy seams, must never treat
`NotEvaluated` as `Allowed`, and any recipe produced before all required execution policies are
evaluated is marked authorization-pending / non-executable (fail-closed per the canonical
contract). A recommendation is not an authorization.
