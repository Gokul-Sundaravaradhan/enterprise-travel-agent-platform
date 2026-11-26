# 05 — Tokenomics and Capacity Forecast

| Field               | Value                                                                                                                                                                                                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status              | Architecture and ROI document. **A model, not a measurement:** every number is derived from stated assumptions and is meant to be replaced by measured values from SP06 telemetry. No billing or load-testing code is planned here                                                                                                     |
| Purpose             | Show how realistic user journeys translate into tokens, request rates and cost, how that demand meets provider limits and capacity options, and which design decisions matter most for ROI                                                                                                                                             |
| Standing constraint | **Nothing may cost money.** The paid options below are analyzed on paper as a production sizing exercise; none is purchased or required                                                                                                                                                                                                |
| Related             | [`04_PROMPT_LAYOUT_AND_PREFIX_CACHING.md`](04_PROMPT_LAYOUT_AND_PREFIX_CACHING.md) (the cache hit ratio used here), [`TASK-6.8`](subprojects/sp06_model_router_evals_and_observability/TASK-6.8_eval_reliability_pass_at_k.md) (eval token overhead), [`03_AGENT_MEMORY_ARCHITECTURE.md`](03_AGENT_MEMORY_ARCHITECTURE.md), SP04, SP06 |

---

## 1. Summary of Findings

For the reference scenario (10,000 daily active travelers, one booking journey each, 4 turns per journey, 3 tool loops per turn):

| Quantity                                     | Result (baseline design)                                                                                                                                                                                            |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| LLM calls per journey                        | **16**                                                                                                                                                                                                              |
| Input tokens per journey (gross)             | **109,280**, of which prompt **history is half** and the repeated static prefix is a third                                                                                                                          |
| Output tokens per journey                    | **1,760**                                                                                                                                                                                                           |
| Daily demand at 10,000 journeys              | 1.09 billion input tokens, 17.6 million output tokens, 160,000 requests                                                                                                                                             |
| Peak demand (15% of the day in one hour)     | about **400 requests per minute** (800 in a burst) and **0.23 to 2.7 million input tokens per minute**, depending on how well prefix caching works and how the provider counts cached tokens                        |
| Effect of prefix caching (document 04)       | Uncached input drops from 109,280 to **23,400** (within-turn reuse) or **9,300** (ideal), a 79% to 92% hit ratio                                                                                                    |
| Effect of trimming tool results from history | Input per journey drops from 109,280 to **63,920** (-42%), and combined with caching to **12,060** uncached tokens                                                                                                  |
| What loops cost                              | Input grows **faster than linearly** with tool loops per turn: 2 loops is 65,640, 3 is 109,280, 4 is 163,800                                                                                                        |
| **Free-tier ceiling**                        | A free hosted tier serves **tens to a few hundred journeys a day**, two to three orders of magnitude below the reference scenario. The $0 deployment therefore runs on a **local model**, with free tiers for demos |
| ROI shape                                    | At illustrative prices the LLM cost is **cents per journey**, well under 1% of the value of the time saved. The binding constraints are **rate limits, latency and reliability**, not the token bill                |

## 2. How to Read This Document

- **Everything is a parameter.** Section 3 lists the assumptions; changing one and recomputing is the intended use.
- **Prices are hypothetical.** They exist to show the _shape_ of the economics (ratios, sensitivities), not to quote a bill. Provider prices and limits change often and are re-verified when a decision depends on them.
- **Limits are published, not guaranteed.** Providers state that actual capacity can vary.
- **The model is calibrated later** from real telemetry (section 8).

## 3. Demand Side

### 3.1 The journey and its parameters

A **booking journey** is one traveler completing a flight-and-hotel request in conversation.

| #   | Parameter                                                            | Value        | Notes                                                                   |
| --- | -------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------- |
| P1  | Daily active travelers (DAU)                                         | 10,000       | Reference scenario; section 6 varies it                                 |
| P2  | Journeys per DAU per day                                             | 1            | Sensitivity 0.3 to 2                                                    |
| P3  | User turns per journey                                               | 4            | For example: request, clarify dates, choose a flight, confirm           |
| P4  | Tool loops per turn (m)                                              | 3            | One model call decides and calls a tool; the result feeds the next call |
| P5  | LLM calls per turn                                                   | m + 1 = 4    | m tool-calling calls plus one final answer                              |
| P6  | Static prefix T0 (system, safety rules, scoped tools, output schema) | 2,000 tokens | The stable top tier of the prompt layout in document 04                 |
| P7  | Per-user snapshot T1 (memory)                                        | 300 tokens   | Loaded once per thread                                                  |
| P8  | User message                                                         | 60 tokens    |                                                                         |
| P9  | Tool call (model output)                                             | 80 tokens    | Structured JSON                                                         |
| P10 | Tool result (flight or hotel list)                                   | 600 tokens   | About five options; retained in history in the baseline                 |
| P11 | Final answer                                                         | 200 tokens   |                                                                         |
| P12 | Peak-hour share of daily traffic                                     | 15%          | Peak-to-average ratio is 15% x 24 = 3.6                                 |
| P13 | Burst factor within the peak                                         | 2x           | For sizing request-rate limits                                          |

### 3.2 Per-call and per-journey tokens

For turn t (1 to 4) and call j within the turn (1 to 4), the prompt is the static prefix, the snapshot, the history from earlier turns, the new user message, and the results of the earlier calls in this turn:

`input(t, j) = 2,000 + 300 + 2,300 x (t - 1) + 60 + 680 x (j - 1)`

Each turn appends 2,300 tokens to history (60 user, plus 3 x (80 call + 600 result), plus 200 answer). Each tool loop adds 680 tokens within the turn.

| Turn        | Call 1 | Call 2 | Call 3 | Call 4 | Turn total               |
| ----------- | ------ | ------ | ------ | ------ | ------------------------ |
| 1           | 2,360  | 3,040  | 3,720  | 4,400  | 13,520                   |
| 2           | 4,660  | 5,340  | 6,020  | 6,700  | 22,720                   |
| 3           | 6,960  | 7,640  | 8,320  | 9,000  | 31,920                   |
| 4           | 9,260  | 9,940  | 10,620 | 11,300 | 41,120                   |
| **Journey** |        |        |        |        | **109,280 input tokens** |

Output per turn is 3 x 80 + 200 = 440 tokens, so **1,760 output tokens** and **16 calls** per journey. The largest single prompt is 11,300 tokens, so a context window of 32,000 tokens is comfortable (see 4.4).

**General form.** With m tool loops per turn (4 turns): input per journey = `(m + 1) x (11,000 + 5,440 m)`, calls = `4 x (m + 1)`, output = `4 x (80 m + 200)`.

| Loops per turn (m) | Calls per journey | Input tokens per journey | Output tokens per journey |
| ------------------ | ----------------- | ------------------------ | ------------------------- |
| 1                  | 8                 | 32,880                   | 1,120                     |
| 2                  | 12                | 65,640                   | 1,440                     |
| **3 (baseline)**   | **16**            | **109,280**              | **1,760**                 |
| 4                  | 20                | 163,800                  | 2,080                     |
| 5                  | 24                | 229,200                  | 2,400                     |

Input grows faster than linearly because every extra loop makes every later prompt longer. **Loops per turn is the strongest demand lever.**

### 3.3 Prefix caching scenarios

Document 04 explains how the application's prompt layout decides how much of the prompt a provider can reuse. Three scenarios bound the range (uncached input is what must be computed afresh):

| Scenario                                     | What is reused                                                                                                                                             | Uncached input per journey | Hit ratio |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- | --------- |
| **C0, no caching**                           | Nothing                                                                                                                                                    | 109,280                    | 0%        |
| **C1, reuse within a turn only**             | T0 is shared; history is cold at each turn start because a person's pause outlasts a cache lifetime; tool loops inside a turn reuse everything before them | 23,400                     | 78.6%     |
| **C2, ideal reuse across the whole journey** | Everything before the newest tokens is reused, including across turns                                                                                      | 9,300                      | 91.5%     |

C1 is the realistic planning case: people take minutes between turns and caches expire in minutes. (Derivation of C1: turn t uncached = 360 + 2,300 x (t - 1) for call 1, plus 3 x 680 for calls 2 to 4, giving 2,400 + 4,700 + 7,000 + 9,300.)

### 3.4 Trimming history (the second lever)

In the baseline, raw tool results stay in history. If instead each turn appends only the user message, the answer and a short structured summary of the confirmed facts (410 tokens, the "pinned facts" idea in the memory and compaction designs), the totals change:

| Variant                            | Input per journey | Uncached, scenario C1 | Reduction in uncached vs baseline C1 |
| ---------------------------------- | ----------------- | --------------------- | ------------------------------------ |
| Baseline (results kept in history) | 109,280           | 23,400                | -                                    |
| **Lean history (facts only)**      | **63,920**        | **12,060**            | **-48%**                             |

Caching and history trimming are complementary: lean history under realistic caching (12,060) lands close to the ideal-caching baseline (9,300).

### 3.5 Scaling to daily and peak demand (reference scenario, baseline design)

| Metric                        | Calculation        | Value                           |
| ----------------------------- | ------------------ | ------------------------------- |
| Journeys per day              | 10,000 x 1         | 10,000                          |
| Requests per day              | 10,000 x 16        | 160,000                         |
| Input tokens per day (gross)  | 10,000 x 109,280   | 1.093 billion                   |
| Output tokens per day         | 10,000 x 1,760     | 17.6 million                    |
| Peak-hour journeys            | 10,000 x 15%       | 1,500                           |
| **Peak requests per minute**  | 1,500 x 16 / 60    | **400** (800 with the 2x burst) |
| Peak output tokens per minute | 1,500 x 1,760 / 60 | 44,000                          |

**Peak input tokens per minute** depends on caching and on whether the provider counts cached tokens toward its limit (providers differ):

| Design and caching                                   | Input tokens per journey counted | Peak input TPM   |
| ---------------------------------------------------- | -------------------------------- | ---------------- |
| Baseline, no caching, or cached tokens fully counted | 109,280                          | **2.73 million** |
| Lean history, no caching                             | 63,920                           | 1.60 million     |
| Baseline, C1, only uncached tokens counted           | 23,400                           | 585,000          |
| Lean history, C1, only uncached counted              | 12,060                           | 301,500          |
| Baseline, C2, only uncached counted                  | 9,300                            | 232,500          |

## 4. Supply Side

### 4.1 What limits throughput

| Limit                             | Meaning                                                                           | Where it binds for this workload                                            |
| --------------------------------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **RPM** (requests per minute)     | Calls per minute, per project                                                     | 400 sustained at the reference peak; free tiers allow tens                  |
| **TPM** (input tokens per minute) | Prompt tokens per minute; some providers also limit output tokens                 | Binding for the heavy-prompt baseline; relieved by lean history and caching |
| **RPD / TPD**                     | Requests or tokens per day                                                        | **The binding limit on free tiers**                                         |
| **Context window**                | Maximum prompt plus output in one call                                            | Not binding at 4 turns (4.4); decides which models are eligible             |
| **Burst sensitivity**             | Providers throttle sharp spikes even when the per-minute average is within limits | Requires smoothing, backoff and retry in the router                         |

Providers publish the **structure** of limits (requests, input tokens and daily requests, per project, depending on usage tier); the **numbers vary by model and tier**, are visible in the provider's console, and are explicitly **not guaranteed**.

### 4.2 The free-tier ceiling (the $0 reality)

Free tiers exist for development. The maximum journeys per day a free tier can serve is `requests per day / 16`, and per hour `60 x requests per minute / 16`:

| Free-tier daily request limit (illustrative snapshots)  | Journeys per day | Equivalent DAU at 1 journey each |
| ------------------------------------------------------- | ---------------- | -------------------------------- |
| 50 per day (a snapshot of some free open-weight routes) | 3                | 3                                |
| 250 per day                                             | 15               | 15                               |
| 1,000 per day (a snapshot of some free routes)          | 62               | 62                               |
| 14,400 per day (a snapshot of a small, fast model)      | 900              | 900                              |

At 20 to 30 requests per minute the same limits allow about 75 to 112 journeys per hour. These figures are community snapshots from late 2026 and must be re-checked; they illustrate the **order of magnitude**: the reference scenario needs about 160,000 requests a day, **roughly 100 to 3,000 times** more than these free allowances. The honest conclusion for the portfolio: **free hosted tiers are for demos and small evaluations; the 10,000-DAU scenario is an analysis, not something the free deployment serves.**

### 4.3 Local model capacity (the actual $0 runtime)

A local model's capacity is set by hardware, not quotas. Per journey, compute time is approximately `uncached input / prefill speed + output / decode speed`. With illustrative speeds of 1,000 tokens per second for prefill and 30 per second for decode (to be measured on the actual machine):

| Scenario             | Prefill | Decode | Compute per journey | Journeys per hour (one slot) | Per day |
| -------------------- | ------- | ------ | ------------------- | ---------------------------- | ------- |
| C0, baseline         | 109.3 s | 58.7 s | 168.0 s             | 21                           | 514     |
| C1, baseline         | 23.4 s  | 58.7 s | 82.1 s              | 44                           | 1,052   |
| C2, baseline         | 9.3 s   | 58.7 s | 68.0 s              | 53                           | 1,270   |
| **C1, lean history** | 12.1 s  | 58.7 s | 70.8 s              | 51                           | 1,220   |

Two lessons: **decode time dominates** once prefill is cached (58.7 of 70 to 82 seconds), so shorter answers and fewer tool loops beat further prefill tuning; and **prefix caching roughly doubles local capacity** by cutting prefill. Tool latency adds waiting time but not compute. Parallel slots multiply capacity if memory allows.

### 4.4 Context window

The largest prompt in the baseline is 11,300 tokens. If tool results were three times larger (1,800 tokens), the last prompt is about 25,700 tokens, still under a 32,000-token window. An 8,000-token window would be exceeded by turn 2 or 3. **Rule: candidate models need a context window of at least 32,000 tokens** for the reference journey, and compaction (SP06) protects longer journeys.

### 4.5 Paid capacity options (analysis only, none purchased)

| Option                                  | How it works                                                                                                 | Fit                                                                        |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| **Standard pay-as-you-go**              | Shared capacity; throughput thresholds rise with usage tiers; throttling possible under contention or bursts | Default for variable load; needs smoothing, backoff and retry              |
| **Provisioned throughput**              | Reserved, isolated capacity for a fixed price; stable latency during contention                              | Steady, high, latency-sensitive load                                       |
| **Batch processing**                    | Roughly half the per-token price and a separate quota, with delayed results                                  | **Evals and offline work** (document TASK-6.8), never interactive journeys |
| **Priority tiers and caching features** | Higher priority at a premium; context caching reduces repeated work                                          | Optional levers                                                            |

**When provisioned capacity wins.** It must be sized for the **peak**, but it is paid for around the clock. Break-even utilization is `u* = cost of provisioned capacity / cost of the same tokens on pay-as-you-go at full utilization`; provisioned wins only when sustained utilization of the reserved capacity exceeds `u*`. For the reference traffic the peak-to-average ratio is 3.6, so a peak-sized reservation runs at about 28% average utilization (1 / 3.6). That favors a **hybrid**: reserve capacity for the steady baseline and let pay-as-you-go absorb peaks. The prices to fill in `u*` are taken from the provider at decision time.

## 5. Matching Demand to Supply

| Scenario                | Journeys per day | Peak RPM (burst) | Local model          | Free hosted tier         | Pay-as-you-go          | Provisioned / hybrid         |
| ----------------------- | ---------------- | ---------------- | -------------------- | ------------------------ | ---------------------- | ---------------------------- |
| **Demo** (10 users)     | 10               | under 1          | Easy                 | Fits for short bursts    | Not needed             | Not needed                   |
| **Pilot** (1,000 DAU)   | 1,000            | 40 (80)          | About one slot at C1 | **Exceeds** daily limits | Comfortable            | Not needed                   |
| **Target** (10,000 DAU) | 10,000           | 400 (800)        | Needs about 10 slots | **Far exceeds**          | Fits a high usage tier | Hybrid sized to the baseline |

The pilot figure uses 1,000 x 16 / 24 x 15% x 60... in short, one tenth of the target's request rate: about 40 requests per minute at the peak.

## 6. Sensitivity

Daily gross input tokens for the baseline design (3 loops per turn):

| Journeys per day | Input tokens per day | Output tokens per day | Requests per day |
| ---------------- | -------------------- | --------------------- | ---------------- |
| 1,000            | 109 million          | 1.8 million           | 16,000           |
| 10,000           | 1.09 billion         | 17.6 million          | 160,000          |
| 100,000          | 10.9 billion         | 176 million           | 1,600,000        |

Design levers, ranked by effect on the reference scenario:

| Lever                                                                                         | Effect                                                                                    | Owner                      |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------- |
| Fewer tool loops per turn (batch independent calls, for example flight and hotel in parallel) | 3 to 2 loops cuts input by 40% and requests by 25%                                        | SP04 planning design       |
| Trim tool results from history                                                                | Cuts input by 42%, uncached input by 48%                                                  | SP06 TASK-6.2              |
| Prefix caching layout                                                                         | Uncached input falls by 79% to 92%                                                        | SP04 TASK-4.8, document 04 |
| Shorter answers                                                                               | Decode time dominates local latency and output tokens cost more per token                 | Prompt design              |
| Sticky routing (do not switch models mid-thread)                                              | Preserves cache; a switch is a cold start                                                 | SP06 TASK-6.1              |
| Smaller models for simple steps                                                               | Lower latency and, on paid tiers, lower price; needs a reliability measurement (TASK-6.8) | SP06 TASK-6.1              |

## 7. Unit Economics and ROI (illustrative)

Hypothetical prices, to show shape only: 1.00 per million uncached input tokens, 0.10 per million cached input tokens, 5.00 per million output tokens (currency units are arbitrary).

| Design and caching   | Uncached input | Cached input | Output | Cost per journey | Cost per day at 10,000 journeys | Versus no caching |
| -------------------- | -------------- | ------------ | ------ | ---------------- | ------------------------------- | ----------------- |
| Baseline, C0         | 109,280        | 0            | 1,760  | 0.1181           | 1,181                           | 1.0x              |
| Baseline, C1         | 23,400         | 85,880       | 1,760  | 0.0408           | 408                             | 2.9x cheaper      |
| Baseline, C2         | 9,300          | 99,980       | 1,760  | 0.0281           | 281                             | 4.2x cheaper      |
| **Lean history, C1** | 12,060         | 51,860       | 1,760  | **0.0261**       | 261                             | **4.5x cheaper**  |

At the best case, **output is about a third of the cost** (0.0088 of 0.0261), so shorter answers and fewer loops matter once input is cached.

**ROI framing.** If an assisted booking saves a traveler 10 minutes against a loaded cost of 40 per hour, the value per journey is about 6.67. At 0.03 to 0.12 per journey, model cost is **0.5% to 2% of the value created**. Under these assumptions the economics are dominated by **reliability** (a failed journey wastes the value and the tokens, and pass^k in TASK-6.8 measures exactly that), by **latency**, and by **capacity limits**, not by the token bill. The sensible production KPI is **cost per successful journey**: total spend divided by journeys that passed their outcome checks.

**Eval overhead.** An evaluation case-run is roughly one journey. The standard profile of TASK-6.8 (50 cases, 4 attempts) is about 200 journeys, about 3,200 calls, which is more than a 1,000-request daily free allowance, so repeated-trial evals run **locally** (or on a batch tier if one were ever paid for). At the reference scale the same 200 journeys are 2% of one day's traffic.

## 8. Calibrating the Model from Telemetry

The assumptions in section 3.1 are replaced by measured values once the SP06 observability work exists. Each parameter maps to a signal:

| Parameter                         | Measured from                                                 | Task                         |
| --------------------------------- | ------------------------------------------------------------- | ---------------------------- |
| Calls per journey, loops per turn | Trace span counts per journey                                 | SP06 TASK-6.4                |
| Input, output and cached tokens   | Token usage on each model call                                | SP06 TASK-6.4                |
| Cache hit ratio by agent role     | `cached_tokens / input_tokens` (document 04, section 9)       | SP06 TASK-6.4, SP04 TASK-4.8 |
| Tool result sizes                 | Result size recorded per tool call (size only, never content) | SP00 TASK-0.4 logging        |
| Turn count, journey completion    | Journey events                                                | SP04                         |
| Rate-limit errors (429)           | Router events                                                 | SP06 TASK-6.1                |
| Local prefill and decode speed    | Time-to-first-token and tokens per second per call            | SP06 TASK-6.4                |
| Success per journey               | The deterministic outcome checks and pass^k                   | SP06 TASK-6.5, TASK-6.8      |

Recompute the tables at each epic boundary and whenever a model, prompt version or tool set changes. **Revisit triggers:** a change of model, a new tool with large results, a change of loops per turn, or a provider changing its limits.

## 9. Recommendations (design decisions this analysis supports)

| #   | Decision                                                                                                                                |
| --- | --------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | Design the planner to **batch independent tool calls** and cap loops per turn at 3, because input grows faster than linearly with loops |
| R2  | **Do not keep raw tool results in history**; keep structured facts (lean history), consistent with the memory and compaction designs    |
| R3  | Adopt the prefix-caching layout in document 04 from the start of SP04                                                                   |
| R4  | Keep routing **sticky within a thread**, and log every cache-losing switch                                                              |
| R5  | Require a context window of at least 32,000 tokens for candidate models                                                                 |
| R6  | Run repeated-trial evals locally or in batch; never against a free hosted quota                                                         |
| R7  | Build smoothing, backoff and retry for 429 responses into the router, because bursts are throttled even under the average limit         |
| R8  | If capacity were ever purchased, size a **hybrid** (reserved baseline plus pay-as-you-go peaks), not a peak-sized reservation           |
| R9  | Track **tokens per journey** and **cost per successful journey** as product KPIs                                                        |

## 10. Where It Lands in the Plan

This document is standalone architecture and ROI analysis; it needs no new task and no code. It informs and is calibrated by: SP04 (loops, prompt layout, TASK-4.8), SP06 TASK-6.1 (quota-aware router uses the supply model), TASK-6.2 (lean history), TASK-6.4 (the telemetry in section 8), TASK-6.8 (eval budget), and SP00 TASK-0.5 (telemetry within free tiers). An optional later task, if wanted, is a small calibration notebook that recomputes sections 3 to 7 from exported telemetry.

## 11. Assumptions and Limits

| Item                                               | Note                                                                                                                                                                                          |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Token counts per message                           | Estimates; real values depend on tokenizer, prompt wording and result formatting                                                                                                              |
| One journey per DAU, 15% peak hour, 2x burst       | Typical shapes, not observations; travel traffic is likely more clustered (weekday mornings)                                                                                                  |
| Cached tokens' treatment in rate limits and prices | Provider-specific and changing; both treatments are shown                                                                                                                                     |
| Provider limits                                    | Structure from official documentation, numbers from community snapshots in late 2026, all to be re-verified; official pages state limits are not guaranteed                                   |
| Local speeds                                       | Illustrative; measure on the actual hardware and model                                                                                                                                        |
| Prices and the value of time                       | Hypothetical, for ratios only                                                                                                                                                                 |
| Sources consulted                                  | Gemini API rate-limit documentation; Vertex AI standard pay-as-you-go and provisioned throughput documentation and its 429-reduction guidance; community free-tier snapshots (September 2026) |
