# workflows: sports-analytics

## v1 core workflow: pre-game decision artifact

### trigger
NFL v1: runs at 10:00 AM local time on game day (or 24h before kickoff for early Sunday games). gives bettors time to act before lines tighten at kickoff. configurable per sport when expanded.

### steps
1. fetch scheduled games from odds api
2. pull open line, current line, and line movement history
3. fetch injury reports and roster availability
4. fetch weather data (outdoor sports only)
5. score each signal category against current line
6. run synthesizer agent to produce the structured decision artifact
7. deliver artifact to tenant (email, webhook, or dashboard)
8. store dispatch snapshot for later review
9. after market close, compare dispatch line to closing line and append a compact audit note to history

## artifact output format
- game header: teams, time, current spread, total
- market snapshot: open line, current line, movement since open, last refresh time
- signal summary table: each category scored and flagged
- narrative paragraph: 3-5 sentences synthesizing the picture
- conflict note: one short line when major signals disagree
- confidence flag: high / medium / low based on signal agreement
- watch items: what could make the read stale or weaker before kickoff or tip-off
- audit stub: dispatch line saved for later history review

## agent roles used
- collector: pulls raw data from each source
- evaluator: scores each signal category against the current line
- synthesizer: writes the artifact from scored signals
- compliance: confirms artifact content does not constitute regulated gambling advice or violate scope constraints
- delivery: formats and dispatches the artifact to the tenant's configured channel

## line movement re-alert (v1 stretch)
if a line moves 2+ points after the first artifact is delivered, re-run scorer and send a short update to pro tier tenants. the update must explain what changed, not just that movement occurred.

## audit workflow
the audit is not a full outcome model. it is a trust mechanism.

for each delivered artifact, history should retain:

- line at dispatch
- line at close
- direction and magnitude of market move after dispatch
- whether the signal picture stayed stable or degraded

this gives the user receipts without turning v1 into a picks-tracking product.

## what this workflow does not do
- does not place bets
- does not maintain a predictive model leaderboard
- does not optimize parlays or props in v1
- does not run in real-time during games (v1)
