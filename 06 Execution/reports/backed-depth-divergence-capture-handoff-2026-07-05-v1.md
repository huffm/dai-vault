# HANDOFF: Backed-Depth Divergence Capture (PAID) v1

## 1. Objective
Paid, approval-gated (`PAID_CAPTURE_APPROVED=true`) capture of <=12 registry-routed backed_depth MLB runs favoring plausible DAI-market divergence (close favorites), to build a settlement-safe cohort where independent signal can later be measured. GAME_DATE=2026-07-05, gpt-4o-mini, $0.05 cap, capture-only (no settlement).

## 2. Outcome
**PARTIAL -- blocked before any paid generation. 0 runs, 0 paid model calls, $0.00.** The 2026-07-05 slate was nearly exhausted at run time (17:44 ET): 13 of 15 games already Final/In-Progress, only 2 pre-game (823931 SD@LAD, 824010 BOS@LAA). Both are eligible backed_depth (9 books, enriched, identity-matched) but **neither passes the divergence prefilter** -- 823931 is an overwhelming favorite (LAD implied 0.690, gap 0.344) and 824010 is a moderate favorite (BOS 0.613, gap 0.188), both above the `<=0.10` close-favorite target. Slate frozen, block recorded, no spend.

## 3. Repo State
### Before
- dai: main, `c6d4f43`, dirty (pre-existing `DevCore.Data.csproj` phantom only), 0 ahead / 0 behind.
- dai-vault: main, `245e4dd`, untracked `06 Execution/system-state-synopsis-v1.md`, 0 ahead / 0 behind.
### After
- dai: main, `c6d4f43` (UNCHANGED), same csproj phantom, 0 ahead / 0 behind.
- dai-vault: main, `<tip = this slice's docs commit>`, **1 ahead / 0 behind**, synopsis still untracked.

## 4. Services Used
- devcore-sql: used (already up) -- DB count reads.
- DevCore.Api :5007: used (already up, health 200) -- `/source-readiness` screen + AgentRuns count.
- agent-service :8000: NOT started (generation never reached) -> proof of 0 model calls.
- sports-app :4201: not started.

## 5. Work Performed
- Verified repo state (both clean vs handoff; both pushed).
- Verified guardrails from source: gpt-4o-mini (sports_analyzer.py:647), single model call (line 650), cost metering present, caps, registry canary default-off.
- Discovered 2026-07-05 slate via StatsAPI (15 games) -> only 2 pre-game.
- Screened both pre-game games via `/source-readiness` (eligible backed_depth) + one the-odds-api read (implied-prob gaps).
- Applied divergence prefilter -> 0 close-favorite candidates -> blocked before spend.
- Froze slate, wrote capture report + this handoff, committed docs (no push).

## 6. Files Changed
- `dai-vault/06 Execution/reports/backed-depth-divergence-candidate-slate-2026-07-05-v1.md` — frozen slate + screening + block decision.
- `dai-vault/06 Execution/reports/backed-depth-divergence-capture-2026-07-05-v1.md` — capture attempt (blocked) report.
- `dai-vault/06 Execution/reports/backed-depth-divergence-capture-handoff-2026-07-05-v1.md` — this handoff.
- dai: none.

## 7. DB Writes / External Side Effects
- AgentRuns created: 0
- Outcomes written: 0
- Evaluations written: 0
- Other: ~3 the-odds-api read units for screening (2x /source-readiness + 1 direct h2h read). No OpenAI spend. No DB writes.

## 8. Paid Calls / Cost
- paid model calls: 0
- estimated cost: $0.00
- proof: agent-service :8000 never started (DOWN throughout); AgentRuns 273 before==after; no generation invoked.

## 9. Validation Proof
- AgentRuns count: 273 before and after (0 new runs).
- agent-service :8000: DOWN throughout (0 paid model calls).
- `$env:DAI_MLB_REGISTRY_PROMPT_CANARY`: empty (registry default-off preserved; .env untouched).
- dai `git status`: only pre-existing csproj phantom; no runtime/registry/config/prompt/routing/schema file changed.
- No reconciliation writes; no outcome/evaluation rows created.

## 10. What Did Not Change
- prompts: unchanged
- routing: unchanged (registry default-off preserved)
- confidence logic: unchanged
- buyer copy: unchanged
- migrations/schema: unchanged
- runtime behavior: unchanged (read-only screening only; agent-service never started)
- reconciliation outcomes: unchanged (no cohort created)

## 11. Open Issues
- No divergence cohort captured today -> the independent-signal measurement gap remains open.
- Root cause: capture ran too late in the day (17:44 ET); pre-game slate exhausted. Fix: run 10:00-13:00 ET.
- dai-vault will have 1 unpushed docs commit after this slice (push only on instruction).

## 12. Recommended Next Slice
Re-run Backed-Depth Divergence Capture (PAID) on a game day, executed 10:00-13:00 ET, screening the full pre-game slate and selecting close favorites (implied-prob gap `<= ~0.10`) first. Same caps/guardrails. Keep capture separate from settlement.

## 13. Suggested Next Prompt
```text
SLICE: Backed-Depth Divergence Capture (PAID) v2
Date: <a game day>  |  RUN WINDOW: 10:00-13:00 ET (full pre-game slate required)
PAID_CAPTURE_APPROVED=true
Caps: max 12 paid runs, gpt-4o-mini only, 1 model call/run, total cost cap $0.05.
Execute dai-vault/06 Execution/reports/backed-depth-divergence-capture-plan-2026-07-05-v1.md:
1. StatsAPI schedule for the date -> keep only Pre-Game games.
2. One the-odds-api h2h slate read -> per-game favorite + implied-prob gap + book count.
3. Divergence prefilter: prefer close favorites (gap <= ~0.10), mixed books, sufficient depth; EXCLUDE overwhelming favorites.
4. Confirm the shortlist via GET /api/agent-runs/source-readiness (eligible starter_enriched_market_backed_depth, identity matched).
5. FREEZE the slate doc BEFORE generation.
6. Start agent-service with DAI_MLB_REGISTRY_PROMPT_CANARY=1 process-scoped; generate <=12 runs via run-artifact-calibration.ps1 -Competition mlb; record EVERY run; then restart agent-service default-off.
7. Do NOT settle this cohort (separate slice after finals).
Hard constraints: no prompt/routing/confidence/buyer/schema/migration change; stop on any fallback/promptSource!=registry/model!=gpt-4o-mini/cost-cap; do not push; no co-authored-by.
End with a continuation-grade handoff brief (per dai-agent-handoff canonical template).
```
