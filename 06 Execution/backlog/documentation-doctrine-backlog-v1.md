# documentation doctrine backlog v1

**status:** active backlog -- governs which DAI concepts become durable doctrine next
**date:** 2026-06-23
**owner skill:** `dai-docs-architect` (write each topic as its own documentation slice using that skill's template)

## purpose

The first set of major DAI concepts to convert into durable vault doctrine, so they stop living as tribal knowledge in handoffs and code. Each entry records why it matters, whether it is already documented, the recommended next slice, and a priority. Priority weighs leverage (how much it de-risks future agent work) against current gap (how undocumented or scattered it is). Documented-but-scattered topics get a "canonical anchor" slice, not a rewrite.

## legend

- **state:** documented (canonical doc exists) | scattered (real docs exist but no single anchor) | partial (some docs, key doctrine implicit) | tribal (lives in handoffs/code/heads only)
- **priority:** P1 (do first -- high leverage, large gap) | P2 (valuable, medium gap) | P3 (mostly covered, consolidate later)

## backlog

### 1. Graphify Development Navigation Doctrine
- why it matters: keeps a code-navigation tool from becoming false architectural authority; sets the navigation-layer-not-truth pattern other tooling docs reuse.
- already documented: yes -- `02 Platform/architecture/devtooling/graphify-development-navigation-doctrine-v1.md` (model doc for this backlog).
- recommended next slice: none required; optional deep-dive once LLM/semantic decisions are revisited.
- priority: P3 (done).

### 2. Cognitive Protocol Doctrine
- why it matters: the core manufacturing model (Perceive/Interrogate/Discern/Decide/Synthesize); everything in the factory references it.
- already documented: scattered -- `cognitive-worker-doctrine.md`, `cognitive-factory/cognitive-protocol-runtime.md`, `protocol-vocabulary-map.md`, `protocol-node-specs.md`, phases/*, decision `0004-cognitive-protocol-runtime.md`. No single compact canonical entry point.
- recommended next slice: Cognitive Protocol Doctrine Anchor v1 -- one compact doc that states the doctrine and indexes the deep dives.
- priority: P2.

### 3. Source Depth and Evidence Sufficiency
- why it matters: defines when a read is grounded enough to ship; directly governs product quality and the "missing context" failure mode.
- already documented: DONE (2026-06-23) -- `04 Products/sports-v1/source-depth-and-evidence-sufficiency-doctrine-v1.md` is the canonical anchor reconciling `confidence-calibration-rules-v1`, `source-group-taxonomy-v1`, `source-depth-contract-v1`, `evidence-sufficiency-band-gate-v1`, and `buyer-copy-safety-v1` into the two-axis model (confidence vs evidence sufficiency; breadth vs depth) and the thin-evidence buyer-honesty cap.
- recommended next slice: none (complete).
- priority: P1 (done).

### 4. Decision Freshness
- why it matters: read-only freshness layer (baseline vs refreshed sources); easily confused with confidence/reconciliation/CLV.
- already documented: documented -- `sources/decision-freshness-architecture-v1.md`, `decision-freshness-protocol-seed-v1.md`, `04 Products/sports-v1/decision-freshness-read-model-v1.md`.
- recommended next slice: optional compact doctrine summary separating freshness from confidence/reconciliation; otherwise covered.
- priority: P3.

### 5. Outcome Reconciliation
- why it matters: ties decisions to settled outcomes; the calibration backbone. Many run docs, easy to lose the doctrine in the logs.
- already documented: scattered -- `04 Products/sports-v1/outcome-reconciliation-contract-v1.md`, `outcome-reconciliation-matcher-v1.md`, and many `calibration/` / `reconcile-*` run docs.
- recommended next slice: Outcome Reconciliation Doctrine Anchor v1 -- canonical doctrine over the contract + matcher, with run docs as evidence beneath it.
- priority: P2.

### 6. Tool Gateway and Agent Permissions
- why it matters: the security and capability boundary for agents; what tools a station may call. Security-adjacent, so tribal knowledge here is a real risk.
- already documented: partial -- `02 Platform/architecture/security-and-permissions.md` exists; tool-gateway behavior lives mostly in handoff addenda (Wire-In, Market Spread Wrap, Retrieve Parity). No dedicated tool-gateway doctrine doc.
- recommended next slice: Tool Gateway and Agent Permissions Doctrine v1 -- canonical doctrine tying the gateway to the permissions model and verified against the registry/policy code.
- priority: P1.

### 7. Buyer Artifact Safety and Copy Doctrine
- why it matters: governs buyer-visible language and prevents unsafe claims (CLV/lock/guarantee).
- already documented: documented -- `04 Products/sports-v1/buyer-copy-safety-v1.md`, `buyer-artifact-post-safety-calibration-v1.md`, `buyer-artifact-quality-check-pass-v1.md`, `confidence-calibration-rules-v1.md`, `artifact-copy-and-section-order-v1.md`, others.
- recommended next slice: optional Buyer Safety Doctrine Index v1 to anchor the cluster; consider extracting the recommended `dai-buyer-copy-safety` skill.
- priority: P3.

### 8. Agent Slice Workflow
- why it matters: how every slice is run (skills gate, grill, plan, verify, handoff). Pure tribal/process knowledge spread across skills and the handoff log; highest alignment leverage.
- already documented: DONE (2026-06-23) -- `06 Execution/agent-slice-workflow-doctrine-v1.md` is the canonical end-to-end slice lifecycle, gates, truth hierarchy, and handoff contract. Previously tribal across `dai-skill-router`, `dai-agent-handoff`, `dai-write-skill`, and `current-slice.md`.
- recommended next slice: none (complete). A future `dai-slice-runner` skill should encode this doctrine.
- priority: P1 (done).

### 9. Tenant as Economic Boundary
- why it matters: tenant = the business/billing boundary (stripe = truth); underpins multi-tenant isolation and monetization.
- already documented: partial -- `tenant-and-niche-model.md`, `schemas/tenant-schema.md`, `billing-and-stripe.md`. The "economic boundary" framing is implicit across them.
- recommended next slice: Tenant as Economic Boundary Doctrine v1 -- canonical statement linking tenant isolation to billing truth.
- priority: P2.

### 10. Factory Model and Product Assembly Lines
- why it matters: the top-level mental model (platform = factory, agents = workers, niches = assembly lines); frames everything else.
- already documented: scattered -- `01 Operating System/product-factory-model.md`, `cognitive-worker-doctrine.md`, `tenant-and-niche-model.md`, vault README.
- recommended next slice: Factory Model Doctrine v1 -- refresh `product-factory-model.md` to current framing as the canonical anchor.
- priority: P2 (foundational; refresh rather than greenfield).

## top 3 recommended next docs

1. Tool Gateway and Agent Permissions Doctrine v1 (#6, P1) -- security-adjacent and partly tribal; documenting it de-risks capability changes.
2. Cognitive Protocol Doctrine Anchor v1 (#2, P2) -- a compact canonical entry point over the existing scattered protocol docs.
3. Tenant as Economic Boundary Doctrine v1 (#9, P2) -- canonical statement linking tenant isolation to billing truth.

Done: Agent Slice Workflow Doctrine v1 (#8, P1) and Source Depth and Evidence Sufficiency Doctrine v1 (#3, P1) shipped 2026-06-23. The remaining items are the least-documented-yet-highest-leverage topics. The well-covered topics (graphify done; buyer copy, decision freshness all have strong docs) can wait for lower-priority "anchor/index" passes.

## related

- skill: `<DAI_REPO_ROOT>/.claude/skills/dai-docs-architect/SKILL.md`
- inventory: `06 Execution/skills/dai-skills-inventory-v1.md`
- model doctrine doc: `02 Platform/architecture/devtooling/graphify-development-navigation-doctrine-v1.md`
