# roadmap: sports-v1

**product:** NFL and NBA pre-game brief
**goal:** one paying subscriber before expanding to soccer or other sports
**date:** 2026-04-10

strategy: build the thinnest vertical slice that produces a real brief for a real game. validate each layer before adding the next. do not build scheduling, billing, or alerts until the brief itself is worth delivering.

---

## phase 1 — one brief, one game, no billing gate
*prove the pipeline end-to-end on a single NFL game before touching NBA or platform infrastructure*

### 1.1 data sources — NFL only
- [ ] sign up for the odds api — confirm `americanfootball_nfl` endpoint returns games, current spread, and opening line
- [ ] confirm actionnetwork returns sharp % and public % for an NFL game currently in market
- [ ] sign up for rotowire — confirm NFL injury report data is fresh (within 2 hours of official report)
- [ ] confirm open-meteo returns hourly forecast for a known outdoor NFL stadium lat/lon
- [ ] document rate limits and any ToS risks for each source

### 1.2 collector — NFL only
- [ ] collector calls all four sources for a single NFL game and returns a structured data object
- [ ] stale or missing fields are flagged explicitly — no silent nulls passed downstream
- [ ] collector output matches the expected input shape for the evaluator

### 1.3 evaluator — NFL only
- [ ] evaluator scores all five signal categories for the game using collector output
- [ ] aggregate score and confidence flag compute correctly from the scored table
- [ ] evaluator tested against 3 known NFL games with manually verified expected scores

### 1.4 synthesizer — NFL only
- [ ] synthesizer produces a brief with header, signal table, narrative, and confidence flag
- [ ] brief read manually for one real game — accurate, no hallucinations, readable in 90 seconds
- [ ] output validated against output-brief-schema

### 1.5 delivery — prototype
- [ ] email delivery sends the brief to a test inbox with correct formatting
- [ ] webhook delivery sends the full brief envelope as valid JSON
- [ ] delivery status recorded on the brief envelope

**exit criteria:** one complete NFL brief produced and delivered without manual intervention for a live game

---

## phase 2 — expand to NBA
*add basketball using the same pipeline; confirm sport-specific differences are handled correctly*

### 2.1 data sources — NBA
- [ ] confirm the odds api `basketball_nba` endpoint returns games and current lines
- [ ] confirm actionnetwork returns sharp/public split for NBA games (confirm endpoint or scrape path)
- [ ] confirm rotowire NBA injury feed is available on the current plan tier
- [ ] weather: confirm collector correctly skips open-meteo for NBA (null, not missing)

### 2.2 collector and evaluator — NBA
- [ ] collector handles NBA games: no weather call, back-to-back flag logic included
- [ ] evaluator scores NBA games: weather score is null (not zero), back-to-back flagged correctly
- [ ] brief produced for one NBA game and reviewed manually

### 2.3 quality bar — both sports
- [ ] 5 NFL briefs and 5 NBA briefs reviewed manually across different weeks and game contexts
- [ ] signal scoring spot-checked against publicly known game context for at least 3 games per sport
- [ ] synthesizer tested on edge cases: all nulls, mixed signals, conflicting sharp/public

**exit criteria:** one complete NBA brief produced and delivered without manual intervention; pipeline handles both sports without hard-coded sport-specific logic

---

## phase 3 — platform and billing
*add tenant records, tier enforcement, scheduled triggers, and stripe gate*

### 3.1 tenant and config
- [ ] tenant record creates and persists with sports-analytics niche config
- [ ] niche config validates sport list, timing fields, and alert threshold at creation time
- [ ] stripe products created: free, starter ($29), pro ($79)
- [ ] stripe webhook configured and receiving subscription events

### 3.2 scheduled triggers
- [ ] trigger fires at 10 AM local on game day for Sunday NFL games
- [ ] trigger fires 24 hours before kickoff for Thursday and Monday night NFL games
- [ ] trigger fires morning of game day for NBA (nightly schedule)
- [ ] missed trigger: if trigger fires after window, log and skip — do not deliver late

### 3.3 tier enforcement
- [ ] stripe subscription status checked before each brief dispatch — not cached
- [ ] free tier: one NFL game per week (highest aggregate score), email only
- [ ] starter tier: all NFL and NBA games, email
- [ ] pro tier: all NFL and NBA games, email + slack/webhook, line movement alert enabled
- [ ] tier enforcement tested for all three tiers with test stripe subscriptions

### 3.4 line movement re-alert (pro)
- [ ] poller checks spread every 15 minutes during game-day window (8 AM to kickoff/tip-off)
- [ ] alert fires if spread moves 2+ points from brief-time spread
- [ ] alert brief includes alert-reason section and is labeled as an update
- [ ] only pro tier tenants receive the alert

### 3.5 observability
- [ ] every workflow run logs: tenant_id, niche, trigger, start/end time
- [ ] every agent step logs: role, duration, output summary
- [ ] failed runs produce a usable diagnostic log entry

**exit criteria:** billing-gated briefs deliver on schedule across a full NFL week and a full NBA game night for all three tiers

---

## phase 4 — first paying subscriber
*real users, real money, real feedback*

- [ ] soft launch to 10–20 people (r/sportsbook, betting discord, or direct outreach)
- [ ] offer free trial: all NFL games for one week
- [ ] collect structured feedback: signal quality, brief format, delivery timing, missing context
- [ ] convert at least one person to a paid starter or pro subscription

**exit criteria:** one active paying stripe subscription; at least 5 pieces of structured feedback collected

---

## post-v1 (do not build yet)
- soccer (MLS, EPL, Champions League)
- MLB, NHL
- NCAAF / NCAAB
- outcome tracking and signal accuracy scoring
- player prop coverage
- live / in-game signals
