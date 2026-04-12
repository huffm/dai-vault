# launch checklist: sports-v1

**product:** NFL and NBA pre-game brief
**date:** 2026-04-10

check each item only when it is confirmed working — not when it is assumed working. do not advance to a later section while earlier sections have open items.

---

## section 1 — data sources

### NFL
- [ ] the odds api key active, returns NFL games and lines for current week
- [ ] actionnetwork returns sharp % and public % for at least 5 NFL games in a test week
- [ ] rotowire NFL injury data confirmed fresh (within 2 hours of official report)
- [ ] open-meteo returns correct forecast for 3 outdoor NFL venue lat/lon coordinates
- [ ] collector returns complete output for a full NFL test week (up to 16 games)

### NBA
- [ ] the odds api returns NBA games and lines for a current game night
- [ ] actionnetwork returns sharp % and public % for at least 5 NBA games
- [ ] rotowire NBA injury data available on the current plan tier and confirms freshness
- [ ] collector skips weather entirely for NBA games — null score, not missing field

### both sports
- [ ] stale data handling confirmed: missing fields flagged explicitly, no silent nulls downstream
- [ ] rate limits documented and confirmed acceptable under Sunday NFL volume (up to 16 games) and NBA nightly volume (up to 13 games)

---

## section 2 — agent pipeline

- [ ] collector output passes schema validation for a full NFL week
- [ ] collector output passes schema validation for a full NBA game night
- [ ] evaluator scores all five signal categories for NFL — no null scores on complete input
- [ ] evaluator scores NBA correctly: weather score null, back-to-back flag scored when applicable
- [ ] aggregate score and confidence flag compute correctly from evaluator output for both sports
- [ ] synthesizer produces a readable 3–5 sentence narrative for 10 test games (mix of NFL and NBA)
- [ ] synthesizer tested on edge cases: all nulls, conflicting sharp/public, back-to-back with no other signals
- [ ] no hallucinated player names, team details, or scores in any reviewed brief
- [ ] compliance agent runs on each brief and blocks any output with picks or outcome predictions

---

## section 3 — platform and billing

- [ ] tenant record creates and persists with sports-analytics niche config
- [ ] niche config validation rejects bad sport values and missing timing fields
- [ ] stripe products created: free, starter ($29/month), pro ($79/month)
- [ ] stripe webhook configured and receiving subscription events in production
- [ ] subscription status check gates brief dispatch correctly at runtime (not cached)
- [ ] free tier: one NFL game/week (highest aggregate score), email only — confirmed
- [ ] starter tier: all NFL and NBA games, email — confirmed
- [ ] pro tier: all NFL and NBA games, email + slack/webhook, alert enabled — confirmed
- [ ] tier enforcement tested for all three tiers using test stripe subscriptions

---

## section 4 — delivery

- [ ] email brief arrives in test inbox with correct formatting and readable signal table
- [ ] email rendering checked in at least two clients (e.g., Gmail and Outlook)
- [ ] webhook brief delivers valid JSON to a test endpoint
- [ ] slack brief delivers correctly to a test workspace (pro tier)
- [ ] subject lines correct: NFL uses "kickoff time," NBA uses "tip-off time"
- [ ] delivery status recorded on brief envelope after every dispatch attempt
- [ ] failed dispatch retries once and marks `failed` with log on second failure

---

## section 5 — triggers

- [ ] NFL Sunday trigger fires at 10 AM local on game day
- [ ] NFL Thursday / Monday night trigger fires 24 hours before kickoff
- [ ] NBA trigger fires morning of game day (nightly schedule)
- [ ] missed trigger behavior confirmed: logs and skips, does not deliver late

---

## section 6 — line movement re-alert (pro tier)

- [ ] poller runs every 15 minutes during game-day window (8 AM to kickoff/tip-off)
- [ ] alert fires when spread moves 2+ points from brief-time value
- [ ] alert brief labeled as update, alert-reason section present and accurate
- [ ] only pro tier tenants receive the alert — free and starter confirmed blocked

---

## section 7 — observability

- [ ] every workflow run produces a log entry: tenant_id, niche, trigger, start/end time
- [ ] every agent step produces a log entry: role, duration, output summary
- [ ] at least one full workflow run reviewed in logs manually before go-live
- [ ] at least one failure scenario tested and confirmed to produce a usable error log

---

## section 8 — content quality

- [ ] 10 briefs reviewed manually: at least 5 NFL, at least 5 NBA
- [ ] signal scoring spot-checked against publicly known context for at least 3 games per sport
- [ ] no brief recommends a pick or makes an outcome prediction
- [ ] delivery timing confirmed: brief arrives when expected relative to kickoff or tip-off

---

## go / no-go

all sections must be fully checked before launch. one open item in any section = not ready.

| section | all items confirmed |
|---|---|
| data sources | |
| agent pipeline | |
| platform and billing | |
| delivery | |
| triggers | |
| line movement alert (pro) | |
| observability | |
| content quality | |
