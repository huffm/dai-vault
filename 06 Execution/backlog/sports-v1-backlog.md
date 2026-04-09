# backlog: sports-v1

**product:** NFL pre-game brief
**date:** 2026-04-09

v1 build tasks organized by layer. use this alongside the roadmap. items marked `stretch` are in scope but lower priority.

---

## platform

| task | notes | priority |
|---|---|---|
| implement tenant record and config storage | fields: tenant_id, niche, tier, stripe IDs, delivery config, niche config | high |
| implement niche config validation for sports-analytics | reject bad sport values, missing timing fields | high |
| implement brief envelope schema | brief_id, sections[], trigger, delivery_status, generated_at | high |
| implement aggregate score computation from evaluator output | sum of non-null scores, confidence from agreement % | high |
| implement stripe subscription status check | check at brief dispatch time, not cached | high |
| implement tier enforcement | free = 1 game/week, starter = all, pro = all + alert | high |
| implement scheduled trigger system | cron-style, configurable per niche | high |
| implement threshold-alert trigger | fires when line moves 2+ points, runs evaluator + delivery | high |
| implement observability logging | per workflow run and per agent step | medium |
| implement delivery retry logic | retry once on failure, mark failed and log on second | medium |
| implement delivery status tracking on brief envelope | `pending` → `delivered` or `failed` | medium |

---

## niche logic

| task | notes | priority |
|---|---|---|
| implement signal scoring for line movement | opening vs current, directional flag | high |
| implement signal scoring for sharp/public split | sharp % vs public %, reverse line movement flag | high |
| implement signal scoring for injury and availability | key player status impact per team | high |
| implement signal scoring for situational context | rest days, travel, divisional flag, schedule spot | high |
| implement signal scoring for weather | wind speed/direction, precipitation — outdoor only | high |
| implement null handling for missing signal inputs | explicit null score, data_freshness: missing | high |
| implement confidence flag from signal agreement | high ≥80% agreement, medium mixed, low <50% scored | high |
| implement line movement poller | runs every 15 min during game-day window | high |
| implement "best game of week" selector for free tier | highest aggregate score from full week | medium |
| implement compliance check for sports brief content | confirm no regulated gambling advice language | medium |

---

## data sources

| task | notes | priority |
|---|---|---|
| integrate the odds api — games and lines | endpoint: sports/americanfootball_nfl/odds | high |
| integrate the odds api — line movement history | endpoint: sports/americanfootball_nfl/historical/odds | high |
| integrate actionnetwork — sharp/public split | scrape or api, confirm availability for full season | high |
| integrate rotowire — injury reports | confirm freshness vs official NFL injury report timing | high |
| integrate open-meteo — game-day forecast by lat/lon | fetch once per game, 6 hours before kickoff | high |
| build stadium lat/lon lookup for outdoor venues | static mapping is fine for v1 | medium |
| build stale data detection per source | flag if data is older than configured threshold | medium |
| validate sources handle high-volume Sunday window | confirm rate limits and latency under 16-game load | medium |

---

## prompts

| task | notes | priority |
|---|---|---|
| finalize collector prompt | draft in `prompts/collector-prompt.md` — test against real game data | high |
| finalize evaluator prompt | draft in `prompts/evaluator-prompt.md` — test scoring accuracy on 10 known games | high |
| finalize synthesizer prompt | draft in `prompts/synthesizer-prompt.md` — review 5 generated briefs manually | high |
| finalize compliance prompt | define constraint list: no picks, no advice, no outcome predictions | medium |
| finalize delivery prompt | draft in `prompts/delivery-prompt.md` — test email and webhook rendering | medium |
| test evaluator prompt on edge cases | all nulls, all neutral, conflicting sharp/public signals | medium |
| test synthesizer on low-confidence brief | should acknowledge mixed picture, not force a narrative | medium |
| prompt token audit | confirm all five prompts stay under 800 tokens per shared-system-prompt-pattern | stretch |

---

## delivery

| task | notes | priority |
|---|---|---|
| implement email formatter for sports brief | signal table as html/markdown, narrative as paragraph block | high |
| integrate transactional email provider (resend, sendgrid, or postmark) | confirm sender domain and deliverability | high |
| implement webhook formatter | full brief envelope as JSON | high |
| implement slack formatter for sports brief | header block, compact signal list, narrative | medium |
| implement subject line logic | scheduled vs threshold-alert subject formats | medium |
| test email rendering in gmail, outlook, apple mail | check table formatting across clients | medium |
| implement email word cap enforcement | keep brief under 600 words for standard delivery | stretch |
| test webhook retry behavior under endpoint downtime | confirm retry fires once, logs on failure | stretch |
