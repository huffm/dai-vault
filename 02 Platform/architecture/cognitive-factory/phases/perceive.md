# phase: perceive

**date:** 2026-05-11
**status:** v1 doctrine — sports pipeline is the first concrete instance.

---

## responsibility

Perceive stages grounded context for the decision. It collects and shapes the inputs that the rest of the pipeline reasons against. It does not interpret, score, or decide.

In a sentence: **perceive surfaces what is, names what is missing, and aims attention at what matters most.**

---

## current code mapping

| current step | role in perceive |
|---|---|
| `retrieve` (`SportsRetriever`, `SportsCollector`) | fetches grounded context from platform-owned tools (`OddsMarketClient`, `EspnBasketballScheduleClient`, `ActionNetworkClient`, `MlbStarterClient`). produces `SportsRetrievalOutput`. |
| `analyze` prompt `perceive` block | inside the FastAPI prompt, the perceive block asks the model to **detect**, **frame**, and **aim** based on the staged inputs. it does not retrieve. |

Existing retrieve/analyze code names stay. Perceive is the responsibility name. When a future calibration finding says "missing context entered the decision," perceive owns the fix.

---

## owns

- staging grounded context from platform-owned tools
- producing a typed retrieval contract (today: `SportsRetrievalOutput`)
- producing per-signal availability records (`SignalAvailabilityRecord`) with status, source, reason
- detecting attention triggers — anomalies in the staged context that warrant pressure later
- framing the factual matchup context for the prompt
- aiming attention at the factors that matter most for this decision

---

## must not

- decide final posture
- invent missing data
- overstate signal strength
- assert a signal is grounded when retrieval returned null
- treat model output as a perceive input — perceive runs before analyze

---

## inputs

- the user-facing run input (e.g. `CompetitionMatchupInput { competition, homeTeam, awayTeam, gameDate }`)
- platform-owned external clients (typed `HttpClient` instances)
- the per-competition expected signal list (`CompetitionCatalog.MaxGroundedSignals` style metadata)

---

## outputs

- a typed retrieval contract per niche (today: `SportsRetrievalOutput`)
- `GroundedSignals[]` and `MissingSignals[]`
- `SignalAvailability[]` with `{ Signal, Status, Source, Reason, Detail }`
- a perceive block inside the model artifact: `detect[]`, `frame`, `aim[]`

---

## guardrails the platform enforces

- enum clamping on status (`grounded / missing / not_attempted`)
- null = unavailable; perceive never silently fills nulls with plausible values
- expected signals minus grounded signals = missing signals — deterministic, not a model claim
- the perceive prompt block must reference only signals the retriever staged

---

## calibration hooks

A perceive failure looks like:

- a grounded signal was reported but no real data was attached (parsing bug, schema drift)
- a signal name does not match the platform canonical vocabulary
- an anomaly that should have triggered attention was not surfaced
- the frame omitted a fact that downstream phases needed

When calibration finds these, the fix lands in perceive — retriever code, expected signal list, or perceive prompt block — never in decide or synthesize.

---

## generalization beyond sports

For crypto, stocks, or kalshi niches, perceive is the same responsibility with different tools and a different expected signal list. The retrieval contract type changes. The doctrine does not.

Each niche owns its own retrieval contract. Perceive doctrine is shared.
