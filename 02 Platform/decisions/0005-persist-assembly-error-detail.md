# decision 0005: persist assembly error detail on prompt-route provenance

**date:** 2026-07-01
**status:** accepted (implemented in `dai`; docs-only in vault)

## context

The registry prompt canary fails closed to the live prompt with `fallbackReason = "assembly_error"` when a
game classifies as an allowlisted regime but its evidence is un-representable by the strict recipe. Registry
Assembly Error Diagnostic v1 established this is safe, deterministic containment -- not a defect -- but flagged
that the concrete `PromptAssemblyError` message (which slot failed) was only logged, never persisted, so the
exact failing slot for the one observed case (run 260018) is unrecoverable and the next occurrence would be
equally opaque.

## decision

Persist a bounded diagnostic string on prompt-route provenance:

- Field `fallbackDetail: string | null`, value = `str(PromptAssemblyError)` truncated to 500 chars.
- Populated ONLY when `fallbackReason == "assembly_error"`; null for every other row and every legacy row.
- Diagnostic-only: it is NOT a route-key or metrics-grouping input, and a `registry` decision is contractually
  forbidden from carrying it (the coherence validator enforces this).
- Additive and back-compatible on both stacks (python pydantic + C# record); no DB migration
  (`AgentRuns.PromptRouteProvenanceJson` is `nvarchar`); surfaced read-only in the `/rows` calibration export.

## consequences

- Future registry `assembly_error` fallbacks are diagnosable without log archaeology; the value accrues on the
  next occurrence (run 260018 stays null -- not back-fillable).
- Metrics output is unchanged (byte-identical `/metrics` verified); only the `/rows` shape gains a trailing
  nullable field, following the additive pattern set by Calibration Rows Export Endpoint v1.
- Structured slot data was deliberately NOT modeled (the 6+ `PromptAssemblyError` message forms have no shared
  parseable structure); a string is uniform and non-fragile, and structured data can layer on later if needed.

## references

- Full technical plan: `dai/docs/superpowers/specs/2026-07-01-persist-assembly-error-detail-design.md` and
  `dai/docs/superpowers/plans/2026-07-01-persist-assembly-error-detail.md`.
- Evidence report: `06 Execution/persist-assembly-error-detail-v1.md`.
- Origin: `06 Execution/diagnostics/registry-assembly-error-diagnostic-v1.md` (deferred item) +
  `06 Execution/calibration-route-attribution-fix-v1.md`.
