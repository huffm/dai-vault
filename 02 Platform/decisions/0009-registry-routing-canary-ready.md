# decision 0009: registry-authoritative routing is canary-ready (backed_depth)

**date:** 2026-07-03
**status:** accepted (validation slice; `dai` test-only; docs-only in vault). registry routing remains
DEFAULT-OFF.

## context

The mature DAI design routes source ingredients -> observedDataRegime -> selectedDataRegime -> registry
recipe -> assembled prompt -> provenance, rather than depending permanently on the live prompt path.
Registry-Authoritative Routing Canary v2 validated that chain in dry-run/shadow (no model calls), building
on the Attribution Contract (ADR 0007) and the Source Readiness gate (ADR 0008).

## decision

Classify registry-authoritative routing as **paid-canary-ready for the `starter_enriched_market_backed_depth`
regime**, on this evidence:

- **10 shadow recipes** (base + starter + market overlays), hash-verified on load; **4 allowlisted** incl.
  the v8 target.
- The backed_depth recipe **assembles and is byte-identical to the live prompt** (proven:
  `test_select_allowlisted_uses_registry_byte_identical`, `test_registry_success_decision_full_provenance`).
- Registry selection populates **complete provenance** -- promptSource=registry, selectedDataRegime,
  recipeId/version/assembledHash (never fabricated), plus attributionStatus=complete / selectedPromptPath=
  registry (Attribution Contract v1).
- **Fail-closed on every error class** with specific reasons: disabled / regime_not_allowlisted / mismatch /
  assembly_error (bounded detail). The cardinal invariant holds: the model NEVER receives bytes different
  from the live prompt.
- `/rows` already distinguishes registry runs from live runs (promptSource, selectedDataRegime,
  recipeId/version, selectedPromptPath); no read-model change needed.

Because the model input is byte-identical, a paid registry canary carries **no prompt-behavior risk** -- it
changes only WHICH builder produced the identical bytes and populates registry provenance.

## consequences

- A paid registry canary may run on ONE source-readiness-eligible backed_depth game with explicit operator
  approval (packet in the slice report); it doubles as a real backed_depth ROUTE measurement row for v8.
- Registry routing stays DEFAULT-OFF in local/dev/prod config; the canary sets `DAI_MLB_REGISTRY_PROMPT_
  CANARY=1` for its run only.
- Non-allowlisted regimes (enriched_market_backed, all named/asymmetric combos) remain blocked pending real
  soak evidence -- promotion is a separate, deliberate decision, never implicit.

## references

- Slice: `06 Execution/reconciliations/registry-authoritative-routing-canary-v2.md`.
- Lineage: ADR 0007 (attribution), ADR 0008 (source readiness); manifest
  `services/agent-service/app/prompting/templates/manifest.json`; canary
  `app/services/registry_prompt_canary.py`.
