# TASK-6.8 — Eval Reliability: pass@k and pass^k

| Field         | Value                                                                                                                                                                                                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Epic          | SP06 — Dynamic Model Router, Egress Validation, Observability, Evals & AI-BOM                                                                                                                                                                                                                                                               |
| PR            | Single PR (feat)                                                                                                                                                                                                                                                                                                                            |
| Branch        | `feat/sp06-t08-eval-reliability-pass-at-k`                                                                                                                                                                                                                                                                                                  |
| Target branch | `productionize`                                                                                                                                                                                                                                                                                                                             |
| Status        | **Fully specified; implementation deferred.** Writing this document changes no code, tests or configuration                                                                                                                                                                                                                                 |
| Depends on    | SP06 TASK-6.5 (golden dataset and deterministic graders), TASK-6.3 (groundedness verifier), TASK-6.4 (trace storage), TASK-6.7 (adversarial set); SP04 TASK-4.4 (tool-call validity metrics)                                                                                                                                                |
| Related       | [`04_PROMPT_LAYOUT_AND_PREFIX_CACHING.md`](../../04_PROMPT_LAYOUT_AND_PREFIX_CACHING.md) (cuts the cost of repeated runs), [`05_TOKENOMICS_AND_CAPACITY_FORECAST.md`](../../05_TOKENOMICS_AND_CAPACITY_FORECAST.md), [`02_AI_AND_MCP_SECURITY_THREAT_MODEL.md`](../../02_AI_AND_MCP_SECURITY_THREAT_MODEL.md), [`EPIC_PRD.md`](EPIC_PRD.md) |

---

## 1. Objective

A single eval run answers "did it pass this time?" That is not enough for an LLM agent, because the same case can pass on one run and fail on the next. This task adds **repeated-trial evaluation** and reports two complementary numbers per case and per suite:

- **pass@k** is the **capability ceiling**: the chance that _at least one_ of k attempts succeeds. It says whether the agent _can_ solve the task.
- **pass^k** (read "pass hat k") is the **consistency**: the chance that _all_ k attempts succeed. It says whether a user can _rely_ on it.

The **gap between them is the flakiness**. It separates two very different problems that a single run cannot tell apart: tasks the agent never solves (a capability, tooling or grader bug) and tasks it solves only sometimes (a stochastic reliability problem that prompts, structured output and repair loops can fix). The task also defines how passing trajectories from stochastic runs can later improve prompts, and optionally fine-tuning data, under the project's rule that **nothing may cost money**.

## 2. Scope

### In scope

| Area                                    | Change                                                                                                                                                        |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Repeated-trial runner (design)          | Runs each eval case n times with a configurable k, records full trajectories and outcomes                                                                     |
| Estimators and reporting                | Unbiased pass@k and pass^k estimators, per-case pass rate with confidence intervals, a flakiness gap, and an always-pass / always-fail / flaky classification |
| Sampling profiles and adaptive sampling | `smoke`, `standard`, `deep`, and an adaptive mode that spends repeats only where outcomes are mixed (3.4)                                                     |
| Deterministic determinism check         | A cheap k=2 run of the **deterministic** track to catch hidden non-determinism in non-LLM code                                                                |
| Model comparison                        | The same cases scored per candidate free model, feeding the SP06 router (TASK-6.1)                                                                            |
| Trajectory harvesting policy            | Rules for turning passing and failing trajectories into prompt hardening, new regression cases, and optional fine-tuning data (3.8)                           |
| Results storage                         | Versioned result artifacts with full provenance (3.9)                                                                                                         |
| Docs                                    | A short "how to read an eval report" guide in the README section for evals                                                                                    |

### Out of scope

- **Gating CI on stochastic results.** Model-in-the-loop runs are advisory (3.3). Gating stays deterministic.
- LLM-as-judge graders. A hosted judge would cost money and add its own variance; graders here are deterministic code.
- Actually fine-tuning a model (3.8 defines policy only).
- Running local models inside GitHub Actions runners (CPU-only, too slow to be useful).
- Load or latency benchmarking (see the tokenomics document).

## 3. Architectural Notes

### 3.1 Definitions and estimators

For one case, run n independent attempts and count c successes. For a chosen k (with k at most n):

| Metric     | Meaning                           | Unbiased estimator        | If the success rate is p (independent attempts) |
| ---------- | --------------------------------- | ------------------------- | ----------------------------------------------- |
| **pass@k** | At least one of k attempts passes | `1 - C(n-c, k) / C(n, k)` | `1 - (1 - p)^k`                                 |
| **pass^k** | All k attempts pass               | `C(c, k) / C(n, k)`       | `p^k`                                           |

Suite values are the mean over cases. When n equals k the estimators reduce to "any pass" and "all pass". Using n larger than k reduces variance at the same k and is preferred when the budget allows.

**Why the gap is the point.** The same per-attempt success rate looks very different under the two metrics:

| Per-attempt success p | pass@8     | pass^8 | Reading                                            |
| --------------------- | ---------- | ------ | -------------------------------------------------- |
| 0.00                  | 0%         | 0%     | Never solved: a capability, tool or grader problem |
| 0.50                  | 99.6%      | 0.4%   | Solvable but a coin flip: a reliability problem    |
| 0.90                  | about 100% | 43%    | Looks good on one run, fails often across eight    |
| 0.95                  | about 100% | 66%    | Still unreliable at scale                          |
| 1.00                  | 100%       | 100%   | Stable                                             |

A worked estimator example with n=8 attempts and c=6 successes: pass@4 = 1 - C(2,4)/C(8,4) = 1 - 0/70 = **100%**, and pass^4 = C(6,4)/C(8,4) = 15/70 = **21%**. The agent "can" do it, and you can rely on it about a fifth of the time. A single run reports "pass" 75% of the time and hides this.

### 3.2 What counts as a pass (deterministic graders only)

A trajectory-level outcome is graded by **code, not by another model**, in three checks that must all hold:

1. **Tool-call correctness.** The calls made were the expected ones, or an accepted equivalent: right tools, valid arguments (the same contract validation as SP04 TASK-4.4), dependencies respected.
2. **Grounded answer.** Every price, flight identifier and date in the final answer matches the tool results (the SP06 TASK-6.3 verifier is the grader).
3. **Invariants.** No state-changing action without approval, budgets respected, no untrusted-content instruction obeyed, no secret disclosed.

This keeps the grader free, reproducible and immune to judge noise, and it means every failure has a specific, machine-readable reason.

### 3.3 Two tracks, and why only one repeats

| Track                             | Provider                                          | Variance       | Gates CI?        | Repeats?                                                                                                                                                                                                            |
| --------------------------------- | ------------------------------------------------- | -------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Deterministic** (TASK-6.5, 6.7) | Mock provider with scripted behavior              | None by design | **Yes**          | k=1 is complete: with no variance pass@k and pass^k equal pass@1. A **k=2 determinism check** confirms identical results and catches hidden non-determinism (iteration order, time, randomness) in the non-LLM code |
| **Stochastic** (this task)        | Local models (Ollama) and free-tier hosted models | Real           | **No**, advisory | Yes: this is where pass@k and pass^k carry information                                                                                                                                                              |

Because the stochastic track costs time and quota and is non-deterministic, it **never blocks a merge**. Its output is reviewed, tracked over time, and used to decide what to harden.

### 3.4 Sampling plan and the cost of k

Every attempt costs model calls, so k multiplies the cost. Cost per full run is approximately:

`calls = cases x n x L` (L = average LLM calls per case: plan, tool loops, summary, repairs)
`input tokens = calls x T_in`, `output tokens = calls x T_out`, reduced by the cache hit ratio `h` on the repeated prefix (see the prefix-caching document)

| Profile      | Cases                   | n = k           | Cost vs one pass        | Use                                                                              |
| ------------ | ----------------------- | --------------- | ----------------------- | -------------------------------------------------------------------------------- |
| **smoke**    | 10 representative cases | 1               | 0.2x of a full pass     | Quick local sanity check after a prompt or model change                          |
| **standard** | The 50-case golden set  | 4               | 4x                      | Regular reliability read on a local model                                        |
| **deep**     | The cases flagged flaky | 8               | up to 8x of that subset | Investigating a specific instability; the headline pass^8 number                 |
| **adaptive** | The 50-case golden set  | 2, then up to 8 | about 2 to 4x           | Default. Run 2 attempts per case; **expand only cases with mixed outcomes** to 8 |

**Adaptive sampling** is the practical answer for a free-tier project. After two attempts, a case that passed twice is probably stable, and one that failed twice is probably a hard failure or a bug; both are cheap to leave alone. Only mixed cases (one pass, one fail) can be flaky, and only they earn the extra six attempts. If 30% of cases are mixed, the total is about 2 + 0.3 x 6 = 3.8 attempts per case on average instead of 8, roughly half the cost of fixed k=8, with the same headline conclusion.

**Illustrative budget (assumptions to be measured, not forecasts):** 50 cases, L = 8 calls per case, 3,000 input tokens and 250 output tokens per call. One pass is 400 calls, about 1.2 million input tokens (less with prefix caching) and 100,000 output tokens. A fixed k=8 run is 3.2 million input and 0.8 million output tokens; the adaptive plan is roughly half that. On a local model the **decode (output) time dominates**, so reducing attempts matters more than reducing prompt size. On a free hosted tier the binding constraint is the daily request and token quota, so the planner checks the estimate against the provider's current free limits (see the tokenomics document) before starting and refuses a run that would exceed them.

### 3.5 Statistics and reporting

- **Per-case:** attempts, successes, pass rate with a **Wilson confidence interval** (honest about small n), and the class: _always pass_ (c = n), _always fail_ (c = 0), _flaky_ (0 < c < n).
- **Per-suite and per-category** (happy path, ambiguous, multi-step with dependencies, policy compliance, adversarial, malformed-output pressure): mean pass@1, pass@k, pass^k, the **flakiness gap** (pass@k minus pass^k), and counts of always-pass, always-fail and flaky cases.
- **Suite-level confidence** by bootstrapping over cases. With only 50 cases the intervals are wide, and the report says so rather than implying precision.
- **Comparisons between runs** (before and after a prompt change) are reported with their intervals; a difference inside the noise is labeled as such.

### 3.6 How to read the result

| Pattern                                          | Diagnosis                                  | Typical response                                                                                                        |
| ------------------------------------------------ | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Low pass@k, low pass^k                           | Capability ceiling, or a deterministic bug | Check the grader and the tools first; then context, tool descriptions, a stronger free model                            |
| High pass@k, low pass^k (large gap)              | **Stochastic flakiness**                   | Harden: provider-native structured output, tighter prompts, the repair loop, fixed examples in the stable prompt prefix |
| High and close                                   | Stable                                     | Watch for regressions                                                                                                   |
| Always-fail case that passes on a stronger model | Model capacity                             | Let the router escalate that task class (TASK-6.1)                                                                      |
| Case that flips with prompt wording              | Fragile prompt                             | Version the prompt change and re-measure                                                                                |

**Safety invariants are measured at two levels.** The _system-level_ invariant (a forbidden action is **executed**) is enforced by deterministic guards and must hold on **every** attempt, so it is expected to be k out of k. The _model-level_ behavior (the model **attempts** the forbidden action, which the guard then blocks) is reported separately as an attempt rate. A rising attempt rate with a perfect system-level result means the guards are doing their job and the prompts need hardening; any executed violation is a defect.

### 3.7 Model comparison and the router

The same cases are scored per candidate free model, giving each a pass@1, pass^k and an approximate cost in time or quota per case. This is the evidence the router (TASK-6.1) needs: a small, fast model can be assigned to a task class only if its pass^k on that class clears a stated threshold; otherwise the class escalates. It also reports the effect of the reliability layers by **ablation**: with and without provider-native structured output, with and without the repair loop, to show what each layer is worth.

### 3.8 Trajectory harvesting (what to do with the runs)

Stochastic runs produce trajectories, and the contrast between passing and failing attempts of the same case is useful data. Three uses, in increasing order of effort:

1. **Prompt hardening (the main use).** Compare passing and failing trajectories of flaky cases, identify the divergence (a missed argument, a wrong tool order), and fix it in the versioned prompt, the output schema or the validation feedback. Any fixed examples added go in the **static tier of the prompt** (see the prefix-caching document) and are versioned.
2. **New regression cases.** A newly discovered failure becomes a permanent case in the golden set (the deterministic track can often reproduce it with a scripted mock).
3. **Fine-tuning candidates (optional, later).** Trajectories that pass **all** deterministic graders and violate no invariant can be kept as supervised examples or as accepted samples for rejection-sampling fine-tuning.

**Conditions for any harvested data:**

| Rule                                                                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Synthetic or scripted inputs only.** Real user conversations are never harvested                                                                                                           |
| **Quality filter is deterministic:** all graders pass and no invariant is violated; duplicates are removed                                                                                   |
| **License and terms:** use only outputs of models whose terms allow using outputs to train; permissive open models are the safe default, and a hosted provider's terms must be checked first |
| **Provenance recorded:** model digest, prompt version, tool-manifest hash, sampling settings, date; the dataset is a data component in the AI-BOM                                            |
| **No secrets or personal data**, enforced by the same redaction rules as logs                                                                                                                |
| **Fine-tuning is out of scope here** and, if ever done, runs only on local hardware to keep the $0 rule                                                                                      |

### 3.9 Reproducibility and storage

Each run writes a versioned results artifact containing: the case results and trajectories, the estimator outputs, and the full **provenance** (model identifier and digest, prompt version or T0 hash, tool-manifest hash, sampling parameters, seed where the runtime supports one, n, k, date, code version). Runs are comparable only when provenance matches, and the report refuses to compare across mismatched tool manifests. Results live as artifacts and, within the free tier, as traces in the SP06 observability stack (TASK-6.4). Raw trajectories contain only synthetic inputs by construction.

### 3.10 Free-tier constraints

- Default profile is **adaptive**, on a **local** model; hosted free tiers are used only for small, planned runs that fit the quota check in 3.4.
- No paid API, no hosted judge, no cloud GPU. Local compute time is the only cost.
- Nothing in CI calls a model; the deterministic track (mock provider) is the CI gate.

## 4. Acceptance Criteria

| #    | Criterion                                                                                                                                                                                                                                        |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| AC1  | The runner executes each case n times with a configurable k and records full trajectories and graded outcomes.                                                                                                                                   |
| AC2  | pass@k and pass^k use the unbiased estimators in 3.1, validated against hand-computed examples (including n=8, c=6, k=4, which gives 100% and 21%).                                                                                              |
| AC3  | Each case is classified always-pass, always-fail or flaky, with a Wilson interval on its pass rate; suite and category summaries include the flakiness gap and bootstrap intervals.                                                              |
| AC4  | Pass is decided only by deterministic graders (tool-call correctness, groundedness, invariants); no model-based judge is used.                                                                                                                   |
| AC5  | The deterministic track runs a k=2 determinism check in CI and fails if any result differs between the two runs.                                                                                                                                 |
| AC6  | The stochastic track never gates CI and calls no paid service.                                                                                                                                                                                   |
| AC7  | `smoke`, `standard`, `deep` and `adaptive` profiles exist; adaptive sampling expands only mixed-outcome cases; a pre-run estimate of calls and tokens is checked against the provider's free limits and a run that would exceed them is refused. |
| AC8  | The report separates model-level attempts from system-level executed violations for the safety invariants, and any executed violation is flagged as a defect.                                                                                    |
| AC9  | Per-model comparison and the ablation of the reliability layers (structured output, repair loop) are available and feed the router's decisions.                                                                                                  |
| AC10 | Results carry full provenance, and comparisons across mismatched provenance are refused.                                                                                                                                                         |
| AC11 | The trajectory-harvesting rules in 3.8 are documented and enforced by a filter (synthetic only, graders pass, license check recorded, no secrets).                                                                                               |
| AC12 | The README explains how to run a profile and how to read pass@k, pass^k and the gap.                                                                                                                                                             |

## 5. Verification Commands

Run after implementation, from the repository root (PowerShell). A local model server must be running for steps 3 to 5; steps 1 and 2 need no model. The script and parameter names are placeholders for whatever the implementation chooses.

```powershell
# 0. Install dependencies and confirm the deterministic suite is green
.\scripts\sync-dev.ps1
pytest -m "not db and not integration"

# 1. Estimator unit tests against hand-computed values (AC2)
pytest tests/test_eval_estimators.py -v                                       # expect: n=8,c=6,k=4 -> pass@4 = 1.0, pass^4 = 0.2143

# 2. Determinism check on the deterministic track (AC5)
python scripts/eval_reliability.py --track deterministic --k 2                # expect: identical results across the two runs; non-zero exit otherwise

# 3. Plan a stochastic run and see the estimate before spending anything (AC7)
python scripts/eval_reliability.py --track stochastic --profile adaptive --model <local-model> --dry-run   # expect: calls and tokens estimate, checked against the free limits

# 4. Smoke, then adaptive run on a local model (AC1, AC3, AC7)
python scripts/eval_reliability.py --track stochastic --profile smoke --model <local-model>
python scripts/eval_reliability.py --track stochastic --profile adaptive --model <local-model>
Get-Content eval-results\latest\summary.md                                    # expect: per-category pass@1, pass@k, pass^k, gap, always-pass/fail/flaky counts, intervals

# 5. Ablation and model comparison (AC9)
python scripts/eval_reliability.py --track stochastic --profile standard --model <local-model> --ablate structured-output,repair-loop
python scripts/eval_reliability.py --compare eval-results\<runA> eval-results\<runB>      # expect: refused if provenance (tool manifest, prompt version) differs

# 6. Confirm nothing paid or gating (AC4, AC6)
Select-String -Path .github\workflows\*.yml -Pattern "eval_reliability.py --track stochastic"   # expect: no output (stochastic runs are not in CI)

# 7. Harvest filter dry run (AC11)
python scripts/eval_reliability.py --harvest eval-results\latest --dry-run     # expect: only synthetic, grader-passing, invariant-clean trajectories listed
```

## 6. Rollout & Rollback

- **Rollout:** merge the PR. The deterministic determinism check joins CI; the stochastic runner is a local tool and does not change CI duration. Record a first baseline for each candidate free model.
- **Using it:** run `adaptive` after a meaningful prompt, tool or model change, read the gap, and act on the diagnosis table. Re-run only the cases that changed.
- **Rollback:** remove the determinism check from CI or revert the PR. The stochastic runner is optional tooling; nothing at runtime depends on it.

## 7. Risks

| Risk                                                                                  | Mitigation                                                                                                                                      |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Fifty cases give wide confidence intervals, so small differences are noise            | Intervals are always reported; a difference inside the noise is labeled as such; cases are added as failures are found                          |
| k=8 on a local model is slow                                                          | Adaptive sampling, smaller profiles, prefix caching, and the pre-run estimate; decode time is the limit, so fewer attempts beat shorter prompts |
| Free hosted quotas are exhausted by a run                                             | The pre-run estimate checks the provider's current free limits and refuses to start                                                             |
| Deterministic graders miss a subtle wrong answer an expert would catch                | Graders check facts (groundedness) and invariants; the golden set is reviewed by hand when cases are added; no judge model adds its own noise   |
| Attempts are not truly independent (shared server state, cache effects, a fixed seed) | Fresh thread per attempt, seeds varied where supported, and the report notes when independence cannot be guaranteed                             |
| Over-reading a stochastic metric and treating it as a gate                            | Explicitly advisory; only deterministic checks block a merge                                                                                    |
| Harvested trajectories leak data or break a model's terms                             | Synthetic-only rule, license check recorded per source model, redaction filter, AI-BOM provenance                                               |
| Local hardware varies, so runs are not comparable between machines                    | Provenance records the model digest and settings; comparisons are made on the same machine, and cross-machine runs compare rates, not times     |
