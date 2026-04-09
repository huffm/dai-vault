# workflows: sports-analytics

## v1 core workflow: pre-game brief

### trigger
NFL v1: runs at 10:00 AM local time on game day (or 24h before kickoff for early Sunday games). gives bettors time to act before lines tighten at kickoff. configurable per sport when expanded.

### steps
1. fetch scheduled games from odds api
2. pull current lines and line movement history
3. fetch injury reports and roster availability
4. fetch weather data (outdoor sports only)
5. score each signal category against current line
6. run summarizer agent to produce structured brief
7. deliver brief to tenant (email, webhook, or dashboard)

## brief output format
- game header: teams, time, current spread, total
- signal summary table: each category scored and flagged
- narrative paragraph: 3-5 sentences synthesizing the picture
- confidence flag: high / medium / low based on signal agreement

## agent roles used
- collector: pulls raw data from each source
- evaluator: scores each signal category against the current line
- synthesizer: writes the brief from scored signals
- compliance: confirms brief content does not constitute regulated gambling advice or violate scope constraints
- delivery: formats and dispatches the brief to the tenant's configured channel

## line movement re-alert (v1 stretch)
if a line moves 2+ points after the first brief is delivered, re-run scorer and send a short update to pro tier tenants. this is the single highest-value alert we can build.

## what this workflow does not do
- does not place bets
- does not track outcomes or maintain a model
- does not run in real-time during games (v1)
