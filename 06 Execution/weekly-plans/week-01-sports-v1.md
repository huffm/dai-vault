# week 01: sports-v1

**dates:** 2026-04-13 — 2026-04-18
**goal:** produce one complete NFL brief end-to-end without manual intervention
**scope:** NFL only this week — NBA added in week 02 after the NFL pipeline is stable

do not start on platform infrastructure (billing, scheduling, tier enforcement) until the brief itself is worth delivering.

---

## monday — data source access and NFL baseline

- sign up for the odds api; confirm `americanfootball_nfl` endpoint returns games, spread, and opening line for a live week
- confirm actionnetwork returns sharp % and public % for at least one current NFL game
- sign up for rotowire; confirm NFL injury report endpoint returns at least one team's data and check the timestamp
- confirm open-meteo returns an hourly forecast for at least one outdoor NFL stadium lat/lon (e.g., Lambeau Field: 44.5013, -88.0622)
- document rate limits, endpoint shapes, and any concerns per source

**end of day check:** do all four sources return usable data for the same NFL game? if not, identify which source is the blocker before moving on.

---

## tuesday — collector: NFL single game

- build the collector to call all four sources for a single NFL game
- return a structured data object with a field per signal category
- flag stale or missing fields explicitly — a missing injury report is `data_freshness: missing`, not a null passed downstream
- run the collector against one live NFL game and print the output

**end of day check:** does the collector return a complete object for one real NFL game, with all fields present or explicitly flagged? if weather is missing for an outdoor game, that is a blocker.

---

## wednesday — evaluator and aggregate score: NFL

- build the evaluator to score all five signal categories against the collector output
- scoring: +1 (favorable), 0 (neutral), -1 (unfavorable), null (missing data — not scored)
- platform computes aggregate score (sum of non-null) and confidence flag (% of signals scored and in agreement)
- run the evaluator against the same game used on Tuesday
- spot-check the scores manually against what you know about the game

**end of day check:** does the evaluator return a scored table with an aggregate and confidence flag? are the scores defensible against the actual game context?

---

## thursday — synthesizer and brief output

- build the synthesizer to produce a brief from the evaluator output: header, signal table, narrative (3–5 sentences), confidence flag
- validate the output against the brief envelope schema
- run the synthesizer for 3 NFL games and read each brief manually
- check for: accuracy, no hallucinated details, readable in under 90 seconds, no picks or outcome predictions

**end of day check:** are all three briefs readable and accurate? if any brief hallucinates a player name or makes a pick, fix the prompt before Friday.

---

## friday — delivery and end-to-end integration test

- wire collector → evaluator → synthesizer → delivery into a single pipeline run
- email the brief to a test inbox; confirm formatting and readability
- send the brief as JSON to a test webhook endpoint; confirm the payload is valid
- record delivery status on the brief envelope
- run the full pipeline for one NFL game with zero manual steps

**end of day check:** does one complete NFL brief produce and deliver without manual intervention? if yes, week 01 is done. if not, identify the specific failure and log it.

---

## week 01 exit criteria

- [ ] four data sources return data for the same NFL game
- [ ] collector produces a complete structured object for a live NFL game
- [ ] evaluator scores all five signal categories; aggregate and confidence flag compute correctly
- [ ] synthesizer produces a readable brief for 3 NFL games — no hallucinations, no picks
- [ ] brief delivers via email and webhook without manual intervention
- [ ] delivery status recorded on the brief envelope

---

## what is explicitly out of scope this week

- NBA (week 02)
- tenant records, billing, stripe integration (phase 3)
- scheduled triggers and cron logic (phase 3)
- line movement poller and re-alert (phase 3)
- slack delivery (phase 3)
- compliance agent (phase 3)
- free-tier best-game selector (phase 3)
