# roadmap: sports-v1

**product:** NFL pre-game brief
**goal:** one paying subscriber before expanding to a second sport
**date:** 2026-04-09

---

## phase 1 — research and foundation
*lock the design, confirm data access, set up infrastructure primitives*

### 1.1 data source validation
- [ ] sign up for the odds api — confirm line and movement history endpoints work for NFL
- [ ] confirm actionnetwork scrape or api returns sharp/public split for current week
- [ ] sign up for rotowire — confirm injury report format and freshness relative to official NFL injury report timing
- [ ] confirm open-meteo returns hourly forecast by lat/lon for outdoor stadium locations
- [ ] document any rate limits, latency concerns, or ToS risks per source

### 1.2 platform primitives
- [ ] implement tenant record (tenant_id, niche, tier, stripe IDs, config, delivery config)
- [ ] implement niche config for sports-analytics (sport list, timing, alert threshold)
- [ ] implement brief envelope schema (brief_id, sections[], trigger, delivery_status)
- [ ] set up stripe products: free, starter ($29), pro ($79)
- [ ] confirm transactional email provider is configured and sending

**exit criteria:** data source keys in hand and tested, tenant record creates and persists, stripe products exist

---

## phase 2 — prototype
*get one brief produced end-to-end for a single game, no billing gate*

### 2.1 collector
- [ ] integrate the odds api — scheduled games, current spread, opening line, line movement
- [ ] integrate actionnetwork — sharp %, public %, reverse line movement flag
- [ ] integrate rotowire — injury report per team (player, position, status, timestamp)
- [ ] integrate open-meteo — weather at game venue for outdoor games
- [ ] collector returns structured game data object with stale flags on any missing field

### 2.2 evaluator
- [ ] implement signal scoring for all five categories against collector output
- [ ] platform computes aggregate score and confidence flag from scored table
- [ ] evaluator prompt tested against 3 known games with verified expected scores

### 2.3 synthesizer
- [ ] synthesizer prompt produces header, signal table, narrative, confidence flag
- [ ] output validated against output-brief-schema
- [ ] five briefs read manually — quality bar: accurate, no hallucinations, readable in 90 seconds

### 2.4 delivery (prototype)
- [ ] email delivery sends brief to a test inbox with correct formatting
- [ ] webhook delivery sends brief envelope as valid JSON to a test endpoint
- [ ] delivery status recorded on brief envelope

**exit criteria:** one full brief produced and delivered via email for a live NFL game without manual intervention

---

## phase 3 — v1
*add billing gate, tier logic, scheduled trigger, and line movement re-alert*

### 3.1 scheduled trigger
- [ ] trigger fires at 10 AM local on game day for standard Sunday NFL games
- [ ] trigger fires 24 hours before kickoff for Thursday and Monday night games
- [ ] missed trigger handling: if trigger fires after window, log and skip (do not deliver late)

### 3.2 billing and tier enforcement
- [ ] stripe subscription status checked before each brief dispatch
- [ ] free tier: one game per week (highest aggregate signal score), email only
- [ ] starter tier: all games, email
- [ ] pro tier: all games, email + slack/webhook, line movement alert enabled
- [ ] tier enforcement tested for all three tiers with test stripe subscriptions

### 3.3 compliance
- [ ] compliance agent runs on each brief before delivery
- [ ] confirms output is signal context, not regulated gambling advice
- [ ] any constraint violation blocks dispatch and logs the reason

### 3.4 line movement re-alert (pro)
- [ ] poller checks spread every 15 minutes during game-day window (8 AM – kickoff)
- [ ] alert fires if spread moves 2+ points from brief-time spread
- [ ] alert brief includes alert-reason section, labeled as an update
- [ ] pro tenants receive alert; free and starter do not

### 3.5 observability
- [ ] every workflow run logs tenant_id, niche, workflow, trigger, start/end time
- [ ] every agent step logs role, duration, and output summary
- [ ] failed runs produce diagnostic log entry

**exit criteria:** billing-gated briefs deliver on schedule for a full NFL week across all three tiers

---

## phase 4 — first paying subscriber
*real users, real money, real feedback*

- [ ] soft launch to 10–20 people from r/sportsbook, betting discord, or direct outreach
- [ ] offer free trial covering all NFL games for one week
- [ ] collect feedback: signal quality, brief format, delivery timing, missing context
- [ ] convert at least one person to a paid starter or pro subscription

**exit criteria:** one active paying stripe subscription, at least 5 pieces of structured feedback collected

---

## post-v1 (do not build yet)
- NBA expansion
- outcome tracking and signal accuracy scoring
- player prop coverage
- NCAAF / NCAAB
- live / in-game signals
