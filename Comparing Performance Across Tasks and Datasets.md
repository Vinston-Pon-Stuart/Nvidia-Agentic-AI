## In one line

Comparing agent performance across tasks and datasets is a methodology problem, not a "run eval, read the number" problem — hold everything except the thing under test constant, aggregate fairly across tasks that have nothing in common, and confirm an observed delta is real before shipping on the strength of it.

---

## Scope

- Builds on Lesson 3.1 (building the eval pipeline) and feeds Lesson 3.5 (what to do with results). Here the only question: given results you can already measure, how do you compare them fairly, and how do you know the difference is trustworthy?
- Example of the stakes: version 2 scores 84.1%, version 1 scored 83.8%. That 0.3-point delta could be real — or the same noise you'd get re-running version 1 against itself twice.

---

## The fair-comparison principle: vary one thing

- **What it is** — hold the dataset, harness, judge (model + prompt), and sampling policy (temperature, seed) constant; vary only the agent under test. Swap the model *and* the prompt *and* the judge between two runs, and a score change can't be attributed to any one of them.
- LangSmith's model operationalizes this: a **dataset** is a fixed input/reference-output collection; each run against it produces an **experiment** ("the results of running a specific version of your application on the dataset"). Its comparison view shows experiments side-by-side, letting you "run the same dataset against different prompt versions, model providers, or agent systems."
- NAT's `nat eval` enforces the same rule operationally: keep the **project name** constant across runs to allow comparison, and vary only `workflow_alias` (e.g., `nat-simple-llama-31` vs. `nat-simple-llama-33`) — the tooling forces you to hold the harness identity constant and label only what changed.
- **Key caveat**: changing the harness *itself* between runs — a new judge model, a reworded judge prompt, an updated tool version, a different sampling temperature — silently breaks the comparison even if nobody meant to change the agent. "Same eval, different result" should always raise the question of whether the eval moved, not the agent.

---

## Aggregating heterogeneous tasks: macro vs. micro

- **What it is** — how you combine per-task scores into one number. Two defensible methods can rank the same two systems in opposite orders.
- **Macro averaging** — unweighted mean across tasks; every task counts equally regardless of size (scikit-learn's definition, generalizes past classification).
- **Micro averaging** — pools all examples first, computes the metric once; implicitly weights each *example* equally, not each task.
- **Weighted average** — averages per-task scores but weights by each task's example count.
- **Consequence**: a system great on one huge, easy dataset and mediocre on several small, hard ones looks strong under micro-averaging (dominated by the big set) and weak under macro-averaging (each small hard task pulls the average down as much as the big easy one) — and vice versa for a consistently-decent-everywhere system.
- Structurally the same as **Simpson's paradox**: aggregated numbers can reverse a trend that's consistent in every subgroup, purely because subgroup weights differ (same mechanism as the classic Berkeley-admissions case, where the overall rate favored one group even though every department favored the other).
- **Key caveat**: never trust a single headline average — report per-task breakdowns alongside whichever aggregate you choose, and state explicitly which averaging method produced the headline number.

---

## Statistical reliability: is the delta even real?

- **What it is** — LLM/agent outputs are noisy even at low temperature; a single-run score is an estimate, not a fact.
- arXiv:2407.10457 ("The Good, The Bad, and The Greedy: Evaluation of LLMs Should Not Ignore Non-Determinism") finds even greedy decoding yields different results across runs; decoding config shifts scores meaningfully on some benchmark families (math reasoning, code generation) while leaving fixed-answer-space benchmarks like MMLU comparatively stable. A single-run number overstates precision it doesn't have.
- Anthropic's "Adding Error Bars to Evals" (arXiv:2411.00640): eval questions are a sample from an underlying "super-population" of possible questions, so a score is an *estimate* with a standard error, not a fact — reporting it without that error bar is close to meaningless for deciding whether one model is really better. Two refinements:
  - **Clustered standard errors** — many benchmarks (DROP, SQuAD) group related questions under one passage; treating every question as independent understates uncertainty. In real data, clustered SEs can run **over 3x larger** than the naive calculation.
  - **Power analysis / minimum-detectable-effect** — a formula for calculating ahead of time whether a given eval has enough questions to reliably detect the size of difference you care about; sometimes the honest answer is "this eval can't tell these two models apart."
- **Practical playbook**: run each config multiple times (multiple seeds, not one pass); report mean and confidence interval, not a single number; use a **paired** significance test (paired bootstrap over score differences, or McNemar's test for pass/fail outcomes) when comparing two configs on the *same* items, rather than comparing two independent-looking averages.

---

## Pairwise/relative comparison and arena-style ranking

- **What it is** — instead of an absolute score, ask a judge "which of these two is better?"
- Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (arXiv:2306.05685), formalizes **pairwise comparison** alongside single-answer and reference-guided grading, and shows GPT-4-as-judge agrees with human preference **over 80%** of the time on pairwise judging — "the same level of agreement between humans."
- **Why needed** — absolute scoring ("7/10") is hard to calibrate consistently across independent judge calls; a relative "A is better than B" is a simpler, more stable unit of comparison.
- **Key caveat** — pairwise judging has its own failure modes: **position bias** (favoring answer order), **verbosity bias** (rewarding length over quality), **self-enhancement bias** (judge preferring its own style). Mitigations: swap answer order and average, add a reference answer, have the judge reason before scoring.
- **Chatbot Arena** scales this: real users see two anonymized model responses and vote for the better one; votes are aggregated via a **Bradley-Terry** model (Arena's current successor to raw Elo) — estimates a latent "strength" per system so beating a stronger opponent moves your rating more than beating a weak one, giving a more stable ranking than naive Elo (not sensitive to match order or over-weighting recent games).
- **Generalization**: to rank more than two systems from noisy pairwise judgments (human or LLM), aggregate through a Bradley-Terry/Elo-style model rather than forcing every system onto one shared absolute scale.

---

## Online vs. offline comparison

- **Offline** — run configs against a fixed dataset outside production. Cheap, fast, repeatable — but "looks good on the test set" doesn't guarantee production behavior once distributions shift and real user behavior diverges from benchmark assumptions.
- **Online (A/B testing, canary)** — compares configs on live traffic. Kohavi et al.'s standard treatment of controlled experiments (drawn from Microsoft, Google, LinkedIn practice) frames the core discipline as choosing an **Overall Evaluation Criterion (OEC)**: a short-term-measurable metric that's genuinely predictive of the long-term goal. Their example of a *bad* OEC is raw "profit" — short-term tactics like price hikes can inflate it while damaging the business. Applied to agents: pick an OEC measurable within the test window (task success rate, user-rated helpfulness, escalation rate) that actually correlates with the real goal, not just whatever's easiest to log.
- **Canary**: route a small percentage of real traffic to the new version first, watch for regressions, ramp up only if it holds — bridges "looked good offline" and "works under real conditions."
- **Trade-off**: online is the most trustworthy signal (measures real usage) but slower, riskier (real users exposed to a worse agent), and needs traffic volume for statistical significance. Standard practice: offline for fast iteration, online A/B/canary reserved for candidates that already cleared the offline bar.

---

## Version-to-version regression comparison

- A special case of everything above: comparing a new version against its immediate predecessor, run automatically on every agent change, to catch a regression before shipping.
- Same discipline (same dataset, same judge, multiple runs, per-task breakdown, paired significance test) applied continuously as a gate rather than a one-time analysis.
- "Benchmark against prior versions" as ongoing production practice is Domain 8's; this lesson supplies the comparison methodology that gate runs on.

---

## Dataset issues that quietly invalidate a comparison

- **Contamination** — the model being compared has already seen the benchmark data during pretraining/fine-tuning, so its score reflects memorization, not capability.
- **Tuning on the comparison set** — iterating prompts or parameters against the same dataset used to declare a winner turns a "held-out" comparison into an overfit one.
- **Distribution shift** — even an uncontaminated benchmark may not reflect production traffic's actual task mix, so a benchmark ranking can reverse under real usage — structurally the same aggregation-weight problem as macro/micro, just applied to how representative the dataset is rather than how it's averaged.

---

## Trade-offs and pitfalls

- **Cherry-picking a favorable benchmark or task subset** instead of reporting the full per-task breakdown.
- **Simpson's-paradox-style aggregation traps** — macro vs. micro (or two differently-weighted benchmarks) ranking the same two systems oppositely.
- **Ignoring variance** — trusting a single run's delta as real improvement when it's within the noise band a few reruns would reveal.
- **Judge inconsistency across runs** — an LLM judge is itself nondeterministic; confounds agent variance with judge variance.
- **Overfitting to the comparison set** — contamination or repeated tuning against the same held-out data.
- **Unfair comparisons from a changed harness** — a different judge, prompt, tool version, or sampling policy sneaking in between compared runs.

---

## NVIDIA angle (read critically)

- `nat eval` keeps the **project name** constant across runs and uses **`workflow_alias`** to label each config, so the same dataset/evaluators are demonstrably reused rather than silently swapped. Its evaluation UI's **Compare** view shows metrics across selected runs, with per-metric scores inspectable per dataset entry — a built-in per-task, per-run breakdown rather than one headline score.
- The profiler output (`inference_optimization.json`) reports 90/95/99% confidence intervals for latency/throughput per run — NVIDIA's own tooling treats *performance* as a distribution worth bounding; the same discipline (per Anthropic's error-bars argument above) should apply to *quality* metrics too.
- **Key caveat**: the comparison dashboard makes consistent naming convenient, but doesn't solve the statistical-reliability problem (you still run multiple seeds and check significance yourself) or the aggregation problem (still your job to choose macro vs. micro and inspect per-task results).
- Don't conflate with **DGX Cloud Benchmarking**, which measures infrastructure/software-stack throughput (training/inference performance across compute and platform combinations) — a different kind of "benchmarking" entirely from agent task quality.

---

## How to think about it for the exam

| Comparison technique | What it protects against / when to use |
|---|---|
| **Fair-comparison principle (vary one thing)** | Attributing a score delta to the wrong cause when harness, judge, or prompt also changed between runs |
| **Macro vs. micro averaging + per-task breakdown** | A single headline number hiding which tasks got worse; rankings flipping depending on averaging method |
| **Multiple runs/seeds + confidence intervals** | A single-run delta being sampling noise — LLM/agent outputs are nondeterministic even at low temperature |
| **Paired significance testing (bootstrap, McNemar)** | Telling a "probably real" difference apart from a "probably noise" one on the same eval set |
| **Pairwise / arena-style comparison (Bradley-Terry, Elo)** | Open-ended quality judgments where an absolute 1–10 score is hard to calibrate consistently across calls |
| **Offline vs. online (A/B, canary) comparison** | Offline "looks good on the test set" not guaranteeing production behavior under real traffic |
| **Version-to-version regression comparison** | Catching a quality regression before shipping a new version (ongoing gate, Domain 8 territory) |

---

## Quick self-check

1. Two agent configurations are compared, but one used a newer version of the judge model. Why does this invalidate the comparison, even if the agents themselves are identical to the previous run?
   **A:** It breaks the "vary one thing" principle — the harness itself changed (a new judge), so any score change can't be cleanly attributed to the agent; it may just be the judge scoring differently.
2. A system scores higher on a macro-averaged benchmark but lower on the same benchmark's micro-average. Explain mechanically how this can happen, and which per-task information would resolve the disagreement.
   **A:** Macro-averaging weights every task equally; micro-averaging pools all examples first, so it's implicitly dominated by whichever task has the most examples. A system that's excellent on one huge, easy dataset and mediocre on several small, hard ones scores high under micro (dominated by the big set) but low under macro (each small task drags the average down equally). A per-task breakdown shows exactly where the two rankings diverge.
3. Why does Anthropic's "Adding Error Bars to Evals" argue that a single eval score without a confidence interval is close to meaningless, and what does a clustered standard error correct for that a naive standard error misses?
   **A:** Eval questions are a sample from an underlying "super-population" of possible questions, so a score is an estimate with a standard error, not a fact. Clustered SEs correct for benchmarks like DROP/SQuAD that group multiple related questions under one passage — treating each as independent understates uncertainty; clustered SEs can run over 3x larger than the naive calculation.
4. What practical problem does pairwise/relative comparison (as in MT-Bench and Chatbot Arena) solve that absolute scoring struggles with, and what statistical model does Chatbot Arena use to turn pairwise votes into a ranking?
   **A:** Absolute scoring is hard to calibrate consistently across independent judge calls (a "7/10" isn't guaranteed to mean the same thing twice); a relative "A is better than B" judgment is simpler and more stable. Chatbot Arena aggregates pairwise votes using a Bradley-Terry model.
5. Why would a team run an offline comparison before an online A/B test rather than going straight to production traffic, and what does an Overall Evaluation Criterion protect against in the online case?
   **A:** Offline comparison is cheap, fast, and repeatable, letting you filter candidates before exposing real users to a worse agent — online testing is slower and riskier. An OEC protects against optimizing a short-term-measurable metric (like raw profit) that doesn't actually predict the long-term goal you care about.

---

## Sources used

- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena — arXiv:2306.05685](https://arxiv.org/abs/2306.05685)
- [Adding Error Bars to Evals: A Statistical Approach to Language Model Evaluations — arXiv:2411.00640](https://arxiv.org/abs/2411.00640)
- [The Good, The Bad, and The Greedy: Evaluation of LLMs Should Not Ignore Non-Determinism — arXiv:2407.10457](https://arxiv.org/abs/2407.10457)
- [Evaluation concepts — Docs by LangChain (LangSmith)](https://docs.langchain.com/langsmith/evaluation-concepts)
- [3. Model evaluation: quantifying the quality of predictions — scikit-learn documentation](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [Online Controlled Experiments and A/B Testing — Kohavi & Longbotham, Encyclopedia of Machine Learning and Data Science (exp-platform.com)](https://exp-platform.com/Documents/2023-03-11EncyclopeiaMLDSABTestingFinal.pdf)
- [Agent Evaluation in NVIDIA NeMo Agent Toolkit — NVIDIA Docs](https://docs.nvidia.com/nemo/agent-toolkit/latest/improve-workflows/evaluate.html)
- [NVIDIA DGX Cloud Benchmarking — Optimize AI Workload Performance](https://www.nvidia.com/en-gb/data-center/dgx-cloud-benchmarking/)
