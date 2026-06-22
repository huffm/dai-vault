# Directional-Contrast Cohort Capture v1

**date:** 2026-06-22
**status:** CAPTURE complete. 10 fresh MLB runs generated through the normal analyzer path. NOT reconciled
(all games first-pitch tonight, not Final). No tuning. No code/prompt/model/confidence/posture/buyer/
source-depth/reconciliation change.
**classification:** generation + capture slice. Optimized for diagnostic contrast, not hit rate.

**Anchor:** The prior enriched cohort (190013-190018) was an all-home-lean / all-home-win sweep, so it
could not show whether enriched starter depth or confidence add signal beyond home base rate. This cohort
exists to break that confound.

## Repo State (at capture)

- dai: main, HEAD 2d7ade2, clean (no code change this slice).
- dai-vault: main, HEAD 45f396f at capture start; this doc + handoff + harness report committed on top.
- Push: 80679be and 45f396f pushed to origin/main before this slice (vault synced before capture).

## Cohort Selection Criteria

- Competition MLB, upcoming window 5 days, take 10 (harness hard cap 10).
- Generated via the blessed harness `scripts/dev/sports/run-artifact-calibration.ps1 -Competition mlb
  -Days 5 -Take 10 -Force` (one gpt-4o-mini call per run, sequential).
- Games are the 2026-06-22 MLB slate; selection was the harness's first-10 of 13 upcoming, not
  cherry-picked. The model chose each lean; balance was not forced (per slice: document bias rather than
  force it).

## Generated Runs

New cohort: AgentRunKey 200013-200022, all `mlb_statsapi` + gamePk identity-bearing, all `completed`,
artifact v3, TenantKey 1, `ExclusionReason=null`, 0 outcome / 0 eval (pre-reconcile).

- **200013** A6B5433E | gamePk 824261 | Yankees at Tigers | lean **away** (Yankees) | conf 0.75 | evRich 2 | monitor | enriched | suff moderate | dir Consistent | namedRisk NotMappable | first pitch 18:10Z
- **200014** ABB5433E | gamePk 822964 | Royals at Rays | lean **home** (Rays) | 0.75 | 2 | monitor | enriched | moderate | Consistent | Ungrounded | 18:40Z
- **200015** B2B5433E | gamePk 823851 | Rangers at Marlins | lean **home** (Marlins) | 0.75 | 2 | monitor | enriched | moderate | Consistent | Ungrounded | 18:40Z
- **200016** B8B5433E | gamePk 822720 | Phillies at Nationals | lean **null** | conf 0.45 | evRich 1 | monitor | none | thin | NotEvaluable | Ungrounded | 18:45Z
- **200017** B9B5433E | gamePk 822800 | Astros at Blue Jays | lean **home** (Blue Jays) | 0.75 | 2 | monitor | enriched | moderate | Consistent | Ungrounded | 19:07Z
- **200018** BEB5433E | gamePk 823613 | Cubs at Mets | lean **away** (Cubs) | 0.75 | 2 | monitor | enriched | moderate | Consistent | **Grounded** | 19:10Z
- **200019** C0B5433E | gamePk 824502 | Brewers at Reds | lean **null** | conf 0.54 | evRich 1 | monitor | none | thin | NotEvaluable | Ungrounded | 19:10Z
- **200020** C7B5433E | gamePk 824585 | Guardians at White Sox | lean **away** (Guardians) | 0.75 | 2 | monitor | enriched | moderate | Consistent | **Grounded** | 19:40Z
- **200021** CAB5433E | gamePk 823692 | Dodgers at Twins | lean **home** (Twins) | 0.75 | 2 | monitor | enriched | moderate | Consistent | Ungrounded | 19:40Z
- **200022** D1B5433E | gamePk 823040 | Diamondbacks at Cardinals | lean **home** (Cardinals) | 0.75 | 2 | monitor | enriched | moderate | Consistent | Ungrounded | 19:45Z

Raw artifact JSON: `04 Products/sports-v1/calibration/artifacts/20260622-1116-mlb-*.json`. Harness report:
`04 Products/sports-v1/calibration/20260622-1116-mlb-calibration.md`.

## Distributions

- **Lean:** home 5 / away 3 / null 2.
- **Source depth (starting_pitching):** enriched 8 / none 2.
- **Evidence-sufficiency band:** moderate 8 / thin 2.
- **Confidence:** 0.75 on all 8 directional runs; 0.54 and 0.45 on the 2 null-lean runs. Range 0.45-0.75.
- **Posture:** monitor 10 / 10.
- **Market context:** market_odds grounded but `shallow` on all 10 (run line present, e.g. Yankees -1.5;
  no sharp/public, no line-movement depth).
- **Direction consistency:** Consistent 8 / NotEvaluable 2 (the null leans) / PotentialMismatch 0.
- **Named-risk grounding (top-level):** Grounded 2 / Ungrounded 7 / NotMappable 1.

## Market-Context Notes

Every run grounded the market at shallow depth only (run line, no sharp/public split, no line movement).
The two null-lean runs (200016, 200019) had market as their *only* grounded signal -- starters were not
announced, so starting_pitching depth was `none`. Both produced a null lean at reduced confidence
(0.45 / 0.54). Market-only, no starter depth -> no directional lean.

## Direction-Consistency Notes

Eight directional runs are all `Consistent` (prose direction matches structured LeanSide). The two null
leans are `NotEvaluable` (no structured lean to check). Zero `PotentialMismatch` this cohort -- a cleaner
result than 190013-190018, which had one (190017).

## Did the Cohort Achieve Useful Contrast?

**Yes.** This cohort has 3 away leans and 5 home leans (plus 2 nulls), unlike the prior all-home enriched
sweep. When these settle it will be possible to compare home-lean vs away-lean accuracy at enriched depth
and begin separating the home-lean base-rate effect (Reconciled-Outcome Calibration Read v1, Finding 1)
from any real directional signal. The away leans include likely road favorites (Yankees, Cubs) and a road
read at White Sox, giving some favorite/underdog spread. Two clean diagnostic patterns already emerged
pre-settlement, independent of outcomes:

1. **Depth gates the lean.** Enriched starter depth present -> directional lean at 0.75; starters absent
   (depth none) -> null lean at lower confidence. The 8/2 split lines up exactly with the lean/null split.
2. **Confidence still does not vary within directional runs.** All 8 directional runs sit at 0.75
   regardless of home/away or grounded-named-risk status. Confidence drops only when there is no lean.
   This re-confirms: confidence encodes "lean + enriched evidence present," not strength or correctness.
3. **Named-risk grounding remains weak (7/10 Ungrounded),** corroborating the standing weak spot.

## When Reconciliation Should Happen

After all 10 games are Final (first pitches 18:10-19:45Z on 2026-06-22; expect Final ~2026-06-23T04:00Z+).
Reconcile via `POST /api/agent-runs/reconcile` on each `(mlb_statsapi, gamePk)`; grade on structured
`AgentRun.LeanSide`. The 8 directional runs will evaluate correct/incorrect; the 2 null-lean runs
(200016, 200019) will evaluate `inconclusive` by contract. Do not reconcile before Final; do not post
in-progress scores. Keep this cohort separate from 190013-190018 and from the identity-only cohort.

## Recommended Next Slice

**Reconcile Directional-Contrast Cohort v1 -- settlement-completion pass**, after the 10 games are Final.
Then a within-cohort calibration read that compares home-lean vs away-lean accuracy at enriched depth and
tests whether the enriched cohort beats the home-lean base rate established in Reconciled-Outcome
Calibration Read v1. Only after that read shows a discrimination signal should any confidence-calibration
slice be considered. The hard rule (no confidence/prompt/posture/buyer/source-depth change) stays in force
until then.

## Verification

- Services up for capture: docker `devcore-sql`, DevCore.Api `:5007`, agent-service `:8000`
  (`/api/ping` ok). OpenAI key present and funded (all 10 model calls returned `completed`; no 429).
- All 10 runs `status=completed`; artifacts fetched and written.
- Per-run dimensions read from DB (`sqlcmd`: LeanSide, Confidence, EvidenceRichness, Posture,
  SourceDepth) and from `GET /api/agent-runs/{id}/artifact` (sourceSufficiency band, direction
  consistency, named-risk grounding, scheduledStartUtc).
- Source-depth enrichment confirmed still reaching the analyzer: 8/10 runs show
  `starting_pitching=enriched` with season-stat detail, identical pipeline to 190013-190018.
- Buyer/source-depth projection unchanged: no code edited this slice (dai clean).
- No reconciliation performed; AgentRunOutcomes/Evaluations remain 35/35. No outcome rows written.
