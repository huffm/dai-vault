# backlog: sports-v1

**product:** NFL and NBA pre-game brief
**date:** 2026-04-10

items are grouped by layer: platform work (reusable across niches) vs sports-specific work (owned by sports-analytics). build platform primitives only when the sports-specific work requires them — not speculatively.

items marked `stretch` are in scope but lower priority.

---

## platform

work that lives in the shared platform layer and is reusable across niches.

| task | notes | priority |
|---|---|---|
| implement tenant record and config storage | fields: tenant_id, niche, tier, stripe IDs, delivery config, niche config | high |
| implement niche config validation | reject bad sport values, missing timing fields; called at tenant creation | high |
| implement brief envelope schema | brief_id, tenant_id, sections[], trigger, delivery_status, generated_at | high |
| implement aggregate score computation | sum of non-null scores from evaluator output; confidence from agreement % | high |
| implement stripe subscription status check | call stripe at dispatch time; do not cache subscription status | high |
| implement tier enforcement | free = 1 game/week, starter = all games, pro = all + alert | high |
| implement scheduled trigger system | cron-style timing, configurable per niche via config | high |
| implement threshold-alert trigger | fires when line moves 2+ points; runs evaluator + delivery for that game | high |
| implement observability logging | per workflow run and per agent step | medium |
| implement delivery retry logic | retry once on failure; mark failed and log on second failure | medium |
| implement delivery status tracking on brief envelope | pending → delivered or failed | medium |

---

## sports-specific: data sources

work that integrates external data for NFL and NBA. each item is specific to a sport unless noted as shared.

| task | sport | notes | priority |
|---|---|---|---|
| integrate the odds api — NFL games and lines | NFL | endpoint: sports/americanfootball_nfl/odds | high |
| integrate the odds api — NFL line movement history | NFL | endpoint: sports/americanfootball_nfl/historical/odds | high |
| integrate the odds api — NBA games and lines | NBA | endpoint: sports/basketball_nba/odds | high |
| integrate actionnetwork — sharp/public split (NFL) | NFL | confirm full-season availability and data freshness | high |
| integrate actionnetwork — sharp/public split (NBA) | NBA | confirm NBA coverage; same endpoint or separate scrape path | high |
| integrate rotowire — NFL injury reports | NFL | confirm freshness vs official NFL injury report timing | high |
| integrate rotowire — NBA injury reports | NBA | confirm basketball feed available on current plan tier | high |
| integrate open-meteo — game-day forecast by lat/lon | NFL only | outdoor venues only; skip entirely for NBA | high |
| build stadium lat/lon lookup for outdoor NFL venues | NFL only | static mapping is fine for v1 | medium |
| build back-to-back schedule flag for NBA | NBA | check if team's previous game was the night before; include travel flag | medium |
| build stale data detection per source | both | flag data older than configured threshold; do not pass nulls silently | medium |
| validate sources under high-volume NFL Sunday window | NFL | confirm rate limits and latency under 16-game load | medium |
| validate sources under high-volume NBA nightly window | NBA | NBA can have up to 13 games per night during season | medium |

---

## sports-specific: niche logic

signal scoring rules and sport-aware logic. these live in the evaluator and niche config, not the platform.

| task | sport | notes | priority |
|---|---|---|---|
| implement signal scoring for line movement | both | opening vs current; directional flag | high |
| implement signal scoring for sharp/public split | both | sharp % vs public %, reverse line movement flag | high |
| implement signal scoring for injury and availability | both | key player status impact per team | high |
| implement signal scoring for situational context | both | rest days, travel, home/away; divisional flag (NFL), back-to-back flag (NBA) | high |
| implement signal scoring for weather | NFL only | wind speed/direction, precipitation; null for NBA or indoor venues | high |
| implement null handling for missing signal inputs | both | explicit null score, data_freshness: missing — do not coerce to zero | high |
| implement confidence flag from signal agreement | both | high ≥80% agreement, medium mixed, low <50% scored | high |
| implement line movement poller | both | runs every 15 min during game-day window (8 AM to kickoff/tip-off) | high |
| implement "best game of week" selector for free tier | NFL | highest aggregate score from full week; NFL is the free-tier sport | medium |
| implement compliance check for sports brief content | both | confirm no regulated gambling advice language; block dispatch on violation | medium |

---

## sports-specific: prompts

| task | sport | notes | priority |
|---|---|---|---|
| finalize collector prompt | both | draft in prompts/collector-prompt.md — test against real NFL and NBA game data | high |
| finalize evaluator prompt | both | draft in prompts/evaluator-prompt.md — test scoring on 10 known games (mix of NFL/NBA) | high |
| finalize synthesizer prompt | both | draft in prompts/synthesizer-prompt.md — review 5 NFL and 5 NBA briefs manually | high |
| finalize compliance prompt | both | constraint list: no picks, no outcome predictions, no gambling advice | medium |
| finalize delivery prompt | both | draft in prompts/delivery-prompt.md — separate subject line formats for NFL and NBA | medium |
| test evaluator on edge cases | both | all nulls, all neutral, conflicting sharp/public, back-to-back with no other signals | medium |
| test synthesizer on low-confidence brief | both | should acknowledge mixed picture; must not force a narrative | medium |
| prompt token audit | both | confirm all five prompts stay under 800 tokens per shared-system-prompt-pattern | stretch |

---

## platform: delivery

delivery formatting and channel integration.

| task | notes | priority |
|---|---|---|
| implement email formatter for sports brief | signal table as html/markdown, narrative as paragraph block | high |
| integrate transactional email provider (resend, sendgrid, or postmark) | confirm sender domain and deliverability | high |
| implement webhook formatter | full brief envelope as JSON | high |
| implement slack formatter for sports brief | header block, compact signal list, narrative | medium |
| implement subject line logic | NFL: kickoff time; NBA: tip-off time; scheduled vs alert subject formats | medium |
| test email rendering in gmail, outlook, apple mail | check table formatting across clients | medium |
| implement email word cap enforcement | keep brief under 600 words for standard delivery | stretch |
| test webhook retry behavior under endpoint downtime | confirm retry fires once, logs on failure | stretch |
