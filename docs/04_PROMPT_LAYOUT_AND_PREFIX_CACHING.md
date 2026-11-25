# 04 — Prompt Layout and Prefix Caching

| Field               | Value                                                                                                                                                                                                                                                    |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status              | Architecture decision record, planned ahead of the work. Implementation: **SP04 TASK-4.8 (new)**, with inputs from SP03, SP06 and the memory design                                                                                                      |
| Decision            | Treat prompt layout as an **architectural contract**: assemble every prompt deterministically, static content first and volatile content last, and **measure** cache hits instead of hoping for them                                                     |
| Standing constraint | **Nothing may cost money.** Caching is a free optimization here: it stretches free-tier quotas and cuts latency on local models. It is never a correctness condition                                                                                     |
| Related             | [`03_AGENT_MEMORY_ARCHITECTURE.md`](03_AGENT_MEMORY_ARCHITECTURE.md), [`02_AI_AND_MCP_SECURITY_THREAT_MODEL.md`](02_AI_AND_MCP_SECURITY_THREAT_MODEL.md), [`05_TOKENOMICS_AND_CAPACITY_FORECAST.md`](05_TOKENOMICS_AND_CAPACITY_FORECAST.md), SP04, SP06 |

---

## 1. Why This Matters

An agent re-sends a large, mostly unchanging prompt on every step: the system instructions, the tool schemas, the memory snapshot and the whole conversation so far. Inference engines can **reuse the computed attention state (the KV cache) for an identical prefix** instead of recomputing it. When the reuse works, the expensive "prefill" work for the repeated part is skipped: first-token latency drops, free-tier token quotas stretch further, and (on paid providers) the repeated tokens are billed at a discount.

The application does not control the inference engine, but it controls **one thing that decides whether the cache hits: the exact bytes and order of the prompt.** That is what this document specifies.

## 2. Background: How Prefix Caching Behaves

| Fact                                                                                                                                                                                        | Consequence for design                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Caching is **exact prefix reuse**: the cached state is valid only up to the first byte where the new prompt differs                                                                         | One request-specific value early in the prompt (a timestamp, a request ID, a reordered tool) reduces the hit rate to zero for everything after it |
| Engines such as vLLM match **fixed-size blocks** by hash, so an almost identical prompt does not partially hit                                                                              | "Nearly the same" is worthless; the prefix must be **identical**                                                                                  |
| Most providers have a **minimum cacheable prefix** (about 1,024 tokens for OpenAI; roughly 1,024 to 4,096 for Gemini and Anthropic depending on the model)                                  | A short system prompt can be "configured for caching" and cache nothing, with no error. Prefix size must be **measured**                          |
| Providers differ on control: OpenAI and Gemini cache implicitly; Anthropic uses explicit `cache_control` breakpoints; newer OpenAI models add a `prompt_cache_key` and explicit breakpoints | The application states **vendor-neutral hints**; an adapter per provider translates them (section 8)                                              |
| Cache entries expire (minutes to hours depending on provider and settings)                                                                                                                  | Long pauses, such as a human-approval interrupt, can cost a cold start                                                                            |
| Some providers bill a **cache write** at a premium, so a large prompt sent only once can cost more with caching on                                                                          | Caching must be a **measured** decision, not a blanket setting (moot under the $0 rule, relevant if the project ever uses a paid tier)            |
| Self-hosted engines (Ollama, llama.cpp, vLLM) reuse state for a matching prefix in memory                                                                                                   | Applies to the default local-model path; behavior depends on keeping the same model loaded and the same context settings                          |

Reported savings on agent workloads are large (published agent evaluations show tens of percent lower input cost), because agents are the worst case for waste and the best case for reuse. Exact numbers vary by provider and are re-verified when this is implemented.

**Three things that are not this.** Prompt caching reuses attention state and still generates a fresh answer. **Semantic caching** (return a stored answer for a similar question) and **response caching** change behavior and carry correctness and privacy risk; they are out of scope.

## 3. Where This Stands in the Current Plan

| Area                                    | Today                                                                                                          | Gap                                                                                                                                                   |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| The agent today (PoC)                   | Two ad hoc prompts built by string concatenation, with the tool catalog and the user text in the same f-string | No layout discipline; fine for a PoC, but the rewrite must not inherit it                                                                             |
| SP04 (LangGraph rewrite)                | Supervisor plus three scoped sub-agents, typed state, structured outputs                                       | **Not accounted for.** Nothing says how prompts are assembled or that their layout is stable                                                          |
| SP03 tool manifest pinning (C-03, C-01) | Tool descriptions and schemas are agent-owned and hash-pinned                                                  | A **favorable accident**: pinned, versioned tool definitions are exactly the stable prefix caching wants. Ordering and serialization still need rules |
| Memory (M1, per-user snapshot)          | Preferences are loaded deterministically at the start of a turn                                                | Unplanned interaction: reloading or reformatting per turn would break the prefix                                                                      |
| SP06 TASK-6.2 (compaction)              | Summaries keep pinned facts                                                                                    | Compaction **rewrites history, which invalidates every cache entry after the edit**. It must be designed as cache-aware                               |
| SP06 TASK-6.1 (router)                  | Picks a model per task, falls back on failure                                                                  | Switching model or provider **starts a cold cache**; routing needs a stickiness rule                                                                  |
| SP06 TASK-6.4 (observability)           | Token and model-choice metrics                                                                                 | Cache hit ratio and cached tokens are not mentioned                                                                                                   |

**Verdict:** a design rule for SP04 (a deterministic prompt assembler), measured in SP06, specified here. It does not warrant its own epic or subproject; a dedicated task inside SP04 is enough.

## 4. Principles

| #   | Principle                                                                                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| P1  | **One assembler.** Prompts are built in a single, tested component. No other code concatenates prompt strings                                                                                          |
| P2  | **Static first, volatile last.** Order content by how often it changes, least volatile first                                                                                                           |
| P3  | **Byte-stable.** Anything that precedes the cache boundary is serialized canonically (fixed key order, fixed whitespace, sorted tools) and contains no time, no random value and no request identifier |
| P4  | **Append-only history.** Within a thread, new content is appended after what was already sent; earlier content is not edited or reordered                                                              |
| P5  | **Caching is an optimization, never correctness.** The agent behaves identically with caching disabled, and hints a provider cannot honor are silently ignored                                         |
| P6  | **Measure.** Prefix size, hit ratio and cached tokens are recorded and visible; layout stability is a CI test, not a hope                                                                              |
| P7  | **Security first.** Cache-friendliness never moves untrusted content earlier, mixes users in a shared prefix, or weakens the demarcation of untrusted data (threat-model rule R3)                      |
| P8  | **Version prompts deliberately.** A prompt change is a versioned event that is expected to cool the cache once, not an accident                                                                        |

## 5. The Prompt Layout Contract

Every assembled prompt has four tiers, ordered from most to least stable. The **cache boundaries** fall between tiers.

```text
+---------------------------------------------------------------+
| T0  GLOBAL STATIC                (same for every user and thread of this agent role)
|     - system instructions and role description        (versioned)
|     - safety and demarcation rules, output contract   (versioned)
|     - tool definitions: agent-owned, sorted by name   (pinned manifest hash)
|     - structured-output schema                        (versioned)
+-------------------------- cache boundary A ------------------+
| T1  PER-USER SNAPSHOT            (changes rarely; same across a thread)
|     - memory snapshot loaded ONCE at thread start, canonical JSON
|     - role tier and locale, if used
+-------------------------- cache boundary B ------------------+
| T2  PER-THREAD HISTORY           (append-only)
|     - prior turns, in order, never edited between turns
+-------------------------- cache boundary C ------------------+
| T3  PER-TURN VOLATILE            (everything that changes each call)
|     - the current user message
|     - fresh tool results, demarcated as untrusted data
|     - the current date, if the task needs it
+---------------------------------------------------------------+
```

| Tier | Volatility                                 | Shared across          | Typical content size         | Rules                                                                                                                                                        |
| ---- | ------------------------------------------ | ---------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| T0   | Changes only on a versioned release        | All users, all threads | The bulk of a short prompt   | No user data, no dates, no IDs. Tool list fixed per agent role and sorted. Serialized canonically. Its token count is measured against provider minimums     |
| T1   | Changes when the user changes a preference | One user               | Small                        | A **snapshot** taken at thread start and reused; a preference changed mid-thread is announced in T3 and folded into T1 at the next thread, not by editing T1 |
| T2   | Grows every turn                           | One thread             | Grows; bounded by compaction | Append-only. Compaction (SP06 TASK-6.2) is the only thing allowed to rewrite it, under the cache-aware policy in 7.2                                         |
| T3   | Changes every call                         | Nothing                | Small to medium              | Always last. Untrusted content is demarcated with **constant** marker strings, so the markers themselves never perturb a prefix                              |

**Each agent role has its own T0.** The supervisor and the three sub-agents (Booking, Compliance, FinOps) have different system prompts and scoped tools, so they form four separate cache families. This is expected, and it is a further reason to keep each sub-agent's tool set fixed.

## 6. What Breaks the Cache, and What to Do Instead

| Anti-pattern                                                                   | Why it breaks the prefix                      | Instead                                                                             |
| ------------------------------------------------------------------------------ | --------------------------------------------- | ----------------------------------------------------------------------------------- |
| "Current date and time" in the system prompt                                   | Changes every call                            | Put the date in T3, at day granularity, and only when the task needs it             |
| Request IDs, trace IDs, session UUIDs or user names in T0                      | Unique per request or user                    | Keep identifiers in logs and telemetry (SP00), never in the prompt prefix           |
| Tool definitions in non-deterministic order (dictionary or set iteration)      | Same tools, different bytes                   | Sort by tool name; serialize with sorted keys and fixed separators                  |
| Adding or removing a tool between turns                                        | Changes the whole prefix after the tool block | A fixed tool set per agent role; scope by role, not by turn                         |
| Re-reading and re-formatting memory on every turn                              | The snapshot text differs from the last turn  | Snapshot once per thread (T1) and reuse the same bytes                              |
| Editing earlier turns (redaction, "tidying", truncation)                       | Invalidates everything after the edit         | Redact **before** a message first enters history; compact only at policy boundaries |
| Changing model options mid-thread (context size, tool schema, response format) | May reload the model or change the prefix     | Fix options per agent role; treat a change as a versioned event                     |
| Random few-shot example selection                                              | Different examples, different bytes           | Fixed examples in T0, versioned; if retrieved dynamically, place them in T3         |
| A "helpful" footer or header that varies                                       | Any varying byte before the boundary          | Constant text only before boundary C                                                |
| Whitespace and formatting differences between code paths                       | Bytes differ even if the meaning is identical | Single assembler (P1) and canonical serialization (P3)                              |

## 7. Interactions with Other Parts of the Design

### 7.1 Memory (stage M1)

The preference snapshot belongs in **T1**, loaded once per thread, serialized canonically (sorted keys, fixed formatting). It is **untrusted data** when read back ([`03_AGENT_MEMORY_ARCHITECTURE.md`](03_AGENT_MEMORY_ARCHITECTURE.md) 5.7), so it sits after the static instructions and carries the same constant demarcation markers as any untrusted content. Per-user data is never placed in T0, so a shared prefix can never contain one user's data.

### 7.2 Cache-aware compaction (SP06 TASK-6.2)

Compaction rewrites T2, so it invalidates the cache from the first edited byte onward. The policy:

- **Compact rarely and in one step**, at a threshold (for example when history exceeds a fraction of the context window), not on every turn.
- **Keep the head stable:** the summary replaces the oldest turns, and everything the cache already holds before the summary (T0 and T1) stays untouched.
- **Pay once, then reuse:** after a compaction the new, shorter prefix becomes the stable base for the following turns.
- **Compare costs:** compaction is worthwhile when the tokens saved over the remaining turns exceed the cost of the one-time cold prefix. A simple estimate is part of the policy.

### 7.3 Routing (SP06 TASK-6.1)

Within a thread the router is **sticky**: it keeps the same provider and model for an agent role unless it must fall back (failure, timeout, exhausted quota). A switch is a known cache loss and is logged as such. When a weaker free model is chosen for a simple task, it gets its own prefix family.

### 7.4 Human-approval pauses (SP04 TASK-4.5)

A manager may take hours to approve, far longer than any cache lifetime, so the run resumes cold. This is accepted and measured, not engineered around. Short tool-call gaps inside a run are the case worth protecting.

### 7.5 Parallel tool loops

When several tool calls run in parallel, their results are appended to T3 in a **deterministic order** (by step ID), so two runs of the same plan produce the same bytes.

### 7.6 Structured output schema

For providers that count the response schema as part of the prompt, the schema is part of T0 and versioned. It is not regenerated per call.

## 8. Provider Adapters (vendor-neutral hints)

The assembler produces a **neutral description** of the prompt: ordered segments, each tagged with its tier and whether a cache boundary follows it. A small adapter per provider translates that into the vendor's mechanism and silently ignores what it cannot honor.

| Provider path               | Translation                                                                                                                                                                   |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ollama (default, local)** | Nothing to mark. Keep the same model loaded and the same context settings for an agent role so the matching prefix is reused; no extra cost                                   |
| OpenAI                      | Implicit prefix caching above the minimum size; a stable cache key per agent role and prompt version on models that use one; no per-user keys, which would fragment the cache |
| Anthropic                   | A breakpoint after T0 (and one after T1 if it is large enough), only when the prefix is above the model's minimum                                                             |
| Gemini                      | Implicit caching; keep the stable content first                                                                                                                               |
| Mock provider               | A **deterministic cache simulator** (section 10): reports the longest common prefix with the previous request, so layout is testable with no model                            |

Only the free paths are exercised by default. The paid-provider rows document that the design is portable; they cost nothing to describe.

## 9. Observability

Recorded per model call (and fed into the SP00 telemetry and the SP06 Langfuse traces):

| Signal                   | Meaning                                                                                                                                                     |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prefix_tokens` per tier | Size of T0, T1, T2 and T3; shows whether T0 clears the provider minimum                                                                                     |
| `cached_tokens`          | Tokens the provider reports as served from cache (where it reports them)                                                                                    |
| `cache_hit_ratio`        | `cached_tokens / input_tokens`, by agent role and model                                                                                                     |
| `time_to_first_token`    | The latency benefit, comparable between cold and warm calls                                                                                                 |
| `cache_break_reason`     | When the assembler detects its own prefix changed versus the previous call: which tier and why (prompt version bump, tool change, compaction, model switch) |

These are logged as structured events and, within the free budgets of SP00 TASK-0.5, a few bounded-label metrics. The hit ratio is the parameter `h` used in the [tokenomics forecast](05_TOKENOMICS_AND_CAPACITY_FORECAST.md).

## 10. Test Strategy (deterministic, free)

| Test                           | Proves                                                                                                                                                                                                                    |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Prefix-stability contract**  | Two prompts for different users, threads and turns are byte-identical up to boundary A, and up to B for the same user                                                                                                     |
| **Forbidden-content scan**     | T0 contains no timestamps, UUIDs, request IDs, user names or random values (pattern scan over the assembled T0)                                                                                                           |
| **Canonical serialization**    | Permuting tool order or dictionary insertion order produces identical bytes                                                                                                                                               |
| **Append-only history**        | Adding a turn changes only the end of the prompt; earlier bytes are unchanged                                                                                                                                             |
| **Cache simulator regression** | A scripted multi-turn conversation run through the mock provider's simulator meets an expected hit ratio; a layout change that lowers it fails the build. This is the CI gate and needs no model, no network and no money |
| **Compaction policy**          | Compaction touches only T2, leaves T0 and T1 unchanged, and triggers only at the configured threshold                                                                                                                     |
| **Size report**                | The measured T0 size per agent role is reported against the documented provider minimums (informational, not a gate)                                                                                                      |
| **Security invariants**        | No per-user data in T0; untrusted-content markers are constant strings; disabling caching changes no behavior                                                                                                             |

A live measurement against a local model (cold versus warm time-to-first-token) is a **non-gating** check, recorded in the results table of the SP04 task.

## 11. Where It Lands in the Plan

| Item                                                                                                 | Task                                                                             |
| ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Deterministic prompt assembler, layout contract, canonical serialization, adapters, simulator, tests | **SP04 TASK-4.8 (new)**: Deterministic prompt assembly and cache-friendly layout |
| Prompts and tool sets defined per agent role (the T0 inputs)                                         | SP04 TASK-4.3                                                                    |
| Stable structured-output schema                                                                      | SP04 TASK-4.4                                                                    |
| Memory snapshot in T1, loaded once per thread                                                        | SP04 TASK-4.7                                                                    |
| Cache-aware compaction policy                                                                        | SP06 TASK-6.2                                                                    |
| Sticky routing and cache-loss logging                                                                | SP06 TASK-6.1                                                                    |
| Cache hit ratio, cached tokens and time to first token                                               | SP06 TASK-6.4                                                                    |
| Tool definitions sorted and pinned                                                                   | SP03 TASK-3.7                                                                    |

**Definition of done for TASK-4.8:** every prompt in the system is produced by the assembler; the stability, forbidden-content, serialization, append-only and simulator tests pass in CI; the hit-ratio and prefix-size signals are emitted; the layout contract above is the documented reference; and the agent behaves identically with caching disabled.

## 12. Risks, Limits and Non-Goals

| Item                                                                                      | Handling                                                                                                                            |
| ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| A short T0 never reaches the provider minimum, so nothing is cached on hosted APIs        | Measured and reported; local-model reuse does not have this floor; consolidating instructions into T0 is acceptable, padding is not |
| Cache behavior, minimums, lifetimes and pricing change between providers and over time    | The design depends only on exact-prefix reuse; provider details live in adapters and are re-verified at implementation              |
| Hit ratios on free-tier hosted APIs are not guaranteed (automatic caching is best effort) | Reported honestly; the CI gate uses the deterministic simulator, not a provider                                                     |
| Layout discipline conflicts with a feature that wants fresh context early                 | Such content goes in T3; if it must go earlier, that is a conscious, documented cache loss                                          |
| Over-optimizing for caching at the expense of clarity of prompts                          | Principle P5 and the single assembler keep prompts readable; correctness never depends on caching                                   |
| **Non-goals**                                                                             | Semantic caching, response caching, shared per-user prefixes, provider lock-in, and any paid cache feature                          |
