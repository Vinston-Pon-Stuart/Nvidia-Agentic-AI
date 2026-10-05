## In one line

Every new agent version — a prompt edit, a checkpoint swap, a retriever change — has to **prove it's at least as good as the version it replaces**, continuously, both before it ships (offline gate) and after (online watch), not just once at launch.

---

## Why this matters

- **What it is** — the operational discipline of comparing a new version (the "candidate") against its immediate predecessor (the "champion") as a standing habit, not a one-off analysis run when someone gets nervous.
- **Why it's needed** — agents are non-deterministic and everything around them keeps moving (base model, prompt, tool schema, retriever, user query mix). Regressions rarely crash — they show up as a slightly lower success rate, a colder tone, a tool call silently returning the wrong data shape. Five hand-picked demo prompts won't catch that.
- **Scope note** — this lesson doesn't re-teach building an eval suite (Lesson 3.1), statistical comparison methodology (Lesson 3.2), or CI/CD rollout mechanics like blue-green/canary infrastructure (Lesson 4.2). It's the connective tissue: *why* benchmarking against the prior version must be wired into every release and every hour of production traffic.

---

## The offline gate: re-running the regression suite on every change

- **What it is** — before a candidate goes near real users, re-run the existing regression eval suite and compare its score to the current production version's score on the same suite.
- **CD4ML framing (Sato, Wider, Windheuser)** — a **threshold test / ratcheting gate**: collect quality metrics (error rate, accuracy, AUC) and use them "to ensure that new models don't degrade against a known performance baseline." Worked example: a unit test that fails the build if error rate exceeds a fixed threshold, "to prevent a bad model from getting promoted."
- **What you compare with** — the eval pipeline from Lesson 3.1; must be the same suite (or versioned superset) every time, or the comparison is meaningless.
- **How you compare** — the methodology from Lesson 3.2. A raw delta ("62% vs 60%") isn't evidence of regression; Kohavi et al.'s controlled-experiment framing applies here too — need a **null hypothesis** (versions perform the same) and a test that can reject it, since "observed differences... [might be] due to random fluctuations." Paired significance testing, not eyeballing a percentage gap.
- **What this lesson adds** — the comparison runs on *every* commit/prompt edit/model swap and blocks promotion on failure. Lesson 4.2 owns the CI/CD wiring; the point here is the gate exists, runs every time, and always targets "no worse than what's currently in production" — not a fixed bar set once.

---

## Online comparison: watching the new version on real traffic

- **Why offline isn't enough** — offline benchmarks use curated/historical data; production traffic drifts. Databricks: "Offline evaluation... benchmark[s] performance... before deployment. Online evaluation monitors production traffic, scoring live traces to detect drift, regressions and new edge cases."
- Four distinct patterns (exam expects you to tell them apart):

**A/B testing**
- **What it is** — traffic split between current and candidate versions, both serving real users, compared with statistical rigor (Databricks: "randomized sampling, adequate sample sizes, confidence intervals").
- **Caveat** — expensive in time; CD4ML notes you need "enough data to make statistically significant decisions, which can take some time." Both versions are live the whole wait.

**Canary release**
- **What it is** — route a small traffic slice to the new version, watch, ramp up gradually if healthy. Fowler: different tool than A/B testing despite similar mechanics — "canary releases are a good way to detect problems and regressions, A/B testing is a way to test a hypothesis... it can take days to gather... statistical significance from an A/B test, while you would want a canary rollout to complete in minutes or hours."
- **Automated version** — Fowler's **cluster immune system** (pioneered by IMVU): monitors production metrics during canary and auto-rolls-back on a statistically significant regression — an early form of this lesson's continuous automated gate.

**Champion/challenger**
- **What it is** — one champion serves ~100% of live traffic; challengers run in parallel, scored on replayed/mirrored production data. DataRobot: "only one model is ever live at any one time... the same prediction requests are replayed against the challengers for analytical purposes" — unlike A/B testing's actual traffic split.
- **Why it fits "continuous"** — meant to run "practically all the time, across all production models," a standing cyclical process, not a time-boxed test. Promotion is usually a governed human decision, not automatic.

**Shadow (mirror) deployment**
- **What it is** — the new version receives a copy of live traffic and produces real outputs, but only the champion's response is ever shown to users. Zero user risk.
- **AWS SageMaker's framing** — "Only the responses of the production variant are returned... You can choose to discard or log the [shadow] responses for offline comparison." CD4ML calls this the "shadow model" pattern with the same rationale.
- **Cost** — answers "how would this behave on real traffic?" with zero exposure risk, at the cost of running two full inference stacks per request.

- **How the four relate** — shadow and canary both **de-risk a rollout** (shadow risks nothing, canary risks a small slice); A/B and champion/challenger both **decide which version is better** under real conditions (A/B splits live traffic between two active versions; champion/challenger keeps one live and tests challengers passively). Mature teams chain them: shadow first (cheap breakage catch) → canary (limited real impact) → A/B/challenger period (statistically defensible quality answer) — with the offline gate as the entry ticket before any online exposure. Rollout mechanics (traffic shifting, blue-green infra, automated rollback triggers) belong to Lesson 4.2.

---

## What you compare on: the production metric stack

- **Overall Evaluation Criterion (OEC)** — Kohavi et al.'s foundational idea, applicable to agent rollouts: "a quantitative measure of the experiment's objective... selecting a single metric, possibly as a weighted combination... is highly desired. A single metric forces tradeoffs to be made once... and aligns the organization." A good OEC "should not be short-term focused" — should predict long-term goals, not just the offline benchmark score.
- **Signal families feeding the criterion** (each owned elsewhere, all feed the comparison):
  - **Task success rate** — did the agent actually complete the ask end-to-end, not just produce a plausible response (Databricks: completion rate, accuracy, success rate).
  - **User feedback signals** — thumbs up/down, corrections, human escalations, session abandonment (capture mechanics: Lesson 2.5/3.3).
  - **Cost and latency** — a marginally more accurate but slower/pricier challenger may not be a net win (Databricks' efficiency metrics: cost per run, latency, iteration count, token usage).
  - **Error/safety rate** — tool-call failures, guardrail violations, hallucination flags; a regression here can outweigh a task-success gain.
- **Discipline** — decide the criterion once, up front, like fixing an OEC before an experiment — not by picking whichever dashboard metric supports a decision already made.

---

## Making it continuous and automated

- **The exam-relevant twist** — doing all of the above on a schedule, on every deploy, and reactively when something looks off — not as an occasional audit.
- **LangSmith's framing** — online evaluators "run automatically on production traces... providing real-time monitoring, anomaly detection, and alerting," closing the loop by feeding failing production traces back into the offline dataset, building targeted evaluators, validating offline, redeploying.
- **Databricks' evaluation maturity model** — level 3 = automated offline pipelines; level 4 = "continuous monitoring and feedback" (live traces scored automatically, alerts detect regressions, insights feed back); level 5 = full CI/CD integration, continuously optimized. The goal: sit at level 4/5 — scheduled re-benchmarks, a gate on every promotion, and out-of-cycle re-comparison triggered by monitored drift (Lesson 8.2). Dashboards surfacing these comparisons over time are Lesson 8.1's territory.

---

## Trade-offs and pitfalls

- **Offline pass ≠ production better** — offline data is a sample, not the live distribution; this is why online patterns exist as an independent second check.
- **A/B tests need time and traffic** — significance can take days; reading a result too early risks the same false-positive/negative problem Lesson 3.2 covers.
- **Shadow deployment doubles inference cost** — worth it before a risky change, wasteful as a permanent posture for minor tweaks.
- **Champion/challenger needs a trustworthy live scorer** — if it's an LLM-as-judge, it inherits judge biases (self-preference, verbosity, position — Lesson 2.6), so a challenger can look like it's winning on the judge's scale without actually improving user outcomes.
- **Regression sets get contaminated or overfit** — a suite that's gated every release for months stops representing current usage; teams learn to make the metric look good rather than the agent. Lesson 3.1 covers keeping suites fresh.
- **Comparing against a stale champion** — if the baseline isn't kept current with what's actually serving traffic, a candidate can look like an improvement over an already-retired champion.
- **Threshold tuning is a real design decision** — too strict blocks good changes on sampling noise and trains teams to route around the gate; too loose lets real regressions through. Noisier/smaller benchmarks need looser tolerances and more repeats.

---

## The NVIDIA angle: NeMo Evaluator's gate/compare, and the data flywheel — read critically

- **`nel compare`** — investigative tool: given two result sets, runs McNemar's exact test, paired flip analysis, confidence intervals to answer "is this regression statistically real?"
- **`nel gate`** — release-decision tool built on `nel compare`: a policy YAML classifies benchmarks into tiers — **critical**/**supporting** benchmarks block release beyond a threshold (e.g. `max_drop: 0.01`), **advisory** benchmarks are tracked but never block; also supports a *relative* drop guardrail for low-baseline benchmarks ("1 pp absolute means very different things at 80% vs 10% baseline accuracy"). Output: **GO** / **NO-GO** / **INCONCLUSIVE** (not enough paired evidence), with CI-ready exit codes. NVIDIA frames this as replacing the ad hoc alternative — "without automation, teams run `nel compare` on each benchmark separately, open a spreadsheet, copy the deltas, eyeball whether each one is within tolerance... error-prone, unreproducible, does not scale."
- **Read critically:** GO/NO-GO is only as good as human-chosen policy thresholds — NVIDIA's own tuning guidance (a "quick smoke test" at 10-point tolerance/one repeat vs. a "high-confidence" gate at 3 points/four repeats/full dataset) tacitly admits the "automated" gate is automating a manual calibration decision; miscalibrate it and you reproduce the too-strict/too-loose pitfall above. Statistical rigor also depends on an optional `scipy` dependency and per-item `results.jsonl` data — missing either degrades output to `INSUFFICIENT_EVIDENCE` rather than a real answer, easy to miss if run as a black box.
- **Data flywheel** — the same idea at production/agent level: NeMo Evaluator as one stage in an ongoing customize → evaluate → promote loop, not a pre-launch checkpoint ("evaluation benchmarks get stale" in production too). NVIDIA's worked example — an internal router agent ("NVInfo bot") continuously re-aligned, reportedly reaching "over 96% accuracy" on an 8B model while "matching... a larger 70B model" and cutting latency "by more than 70%" — is a real named case study, but a single self-reported internal deployment in a vendor blog, not independently reproduced; treat as illustration of plausibility, not generalizable evidence.
- **Lock-in note** — continuous benchmarking against a prior version is vendor-neutral; `nel gate`/`nel compare` are one well-documented implementation tightly integrated with NeMo Customizer/NIM, not the only correct way to do it.

---

## How to think about it for the exam

| If the question is about... | The answer to reach for |
|---|---|
| Blocking a release automatically when quality drops | The offline regression gate: re-run the eval suite (3.1) on every change, compare with paired significance (3.2), fail the build on a real regression |
| Testing a hypothesis about which version is better, with both live | A/B testing — traffic split, needs an OEC and enough data for statistical significance (can take days) |
| De-risking a rollout by limiting blast radius, ramping gradually | Canary release — small traffic slice first, optimized for catching a broken deploy fast (minutes/hours), not for proving subtle gains |
| Testing a challenger with zero risk to users | Shadow/mirror deployment — new version scores real traffic in parallel, output never served, doubles inference cost |
| An ongoing, standing comparison process rather than a one-time test | Champion/challenger — one model live, challenger(s) scored on replayed traffic continuously, promotion is a governed decision |
| What metric to gate on | An Overall Evaluation Criterion (Kohavi) — a single, pre-committed, long-term-oriented metric, not a post-hoc pick from a dashboard |
| Why "it passed offline eval" isn't the end of the story | Offline data is a curated sample; production traffic drifts — online comparison on live traffic is a separate, necessary check |
| Catching drift or a bad deploy fast | Continuous/automated cadence — scheduled re-benchmarks, drift-triggered re-eval (8.2), dashboards (8.1) surfacing live-vs-benchmarked divergence |
| NVIDIA-specific tooling for this | NeMo Evaluator's `nel compare` (statistical investigation) and `nel gate` (policy-driven GO/NO-GO/INCONCLUSIVE release decision), and the data flywheel's continuous evaluate-promote loop |

---

## Quick self-check

1. A candidate agent version clears the offline regression suite with no significant drop, but a week after full rollout, users report noticeably more failed requests. What production practice would most likely have caught this before full rollout, and why does passing the offline suite not guarantee it?
   **A:** A canary release or A/B test on real traffic would likely have caught it — offline data is a curated sample, and production traffic drifts, so a candidate can pass every regression test and still behave differently against the messy live distribution.
2. A teammate says "we shadow-deployed the new version and it's an A/B test." Explain what's wrong with that statement — what's the actual difference between shadow deployment and A/B testing in terms of who sees the new version's output and what each is optimized to detect?
   **A:** In shadow deployment, the new version's outputs are never shown to users — only the champion's response is served, so it's zero-risk. In A/B testing, both versions are live and visible to real users, split by traffic, specifically to compare which performs better. Shadow detects outright breakage cheaply; A/B tests a quality hypothesis with statistical rigor.
3. Why is champion/challenger described as a *cyclical, ongoing* process rather than a single test, and how does that differ from a canary release, which is meant to complete and resolve in minutes or hours?
   **A:** Champion/challenger runs "practically all the time, across all production models" — challengers are continuously scored on replayed traffic while one champion stays live, with promotion a governed decision whenever a challenger proves itself. A canary is a short, time-boxed rollout check meant to confirm a specific deploy isn't broken, then resolve.
4. `nel compare` reports that a candidate's score dropped versus baseline, but the result is INSUFFICIENT_EVIDENCE rather than a clear regression. What does that tell you, and what would you do next before deciding to ship or block?
   **A:** It means the statistical machinery (e.g., missing `scipy` or missing per-item `results.jsonl` data) couldn't produce a trustworthy paired comparison — not that there's no regression. You'd fix the missing dependency/data and re-run before trusting a GO/NO-GO verdict.
5. Explain the trade-off in setting a regression gate's threshold too tight versus too loose, using the NeMo Evaluator quick-smoke-test-vs-high-confidence-gate distinction as a concrete example.
   **A:** Too tight blocks good changes on ordinary sampling noise and trains teams to route around the gate; too loose lets real regressions ship. NVIDIA's quick smoke test uses a looser 10-point tolerance with one repeat (fast, less certain), while the high-confidence gate uses a tighter 3-point tolerance with four repeats on the full dataset (slower, more trustworthy) — the right choice depends on how much confidence the release actually needs.

---

## Sources used

- [Controlled Experiments on the Web: Survey and Practical Guide](https://ai.stanford.edu/~ronnyk/2009controlledExperimentsOnTheWebSurvey.pdf) — Kohavi, Longbotham, Sommerfield, Henne (Overall Evaluation Criterion, null hypothesis, statistical significance)
- [Continuous Delivery for Machine Learning](https://martinfowler.com/articles/cd4ml.html) — Sato, Wider, Windheuser, martinfowler.com (shadow models, competing models/A-B testing, multi-armed bandits, threshold tests/ratcheting)
- [Canary Release](https://martinfowler.com/bliki/CanaryRelease.html) — Danilo Sato, martinfowler.com (canary vs A/B testing, cluster immune system)
- [Shadow tests - Amazon SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/shadow-tests.html) — AWS documentation
- [Introducing MLOps Champion/Challenger Models](https://www.datarobot.com/blog/introducing-mlops-champion-challenger-models/) — DataRobot Blog
- [What is AI Agent Evaluation?](https://www.databricks.com/blog/what-is-agent-evaluation) — Databricks Blog (offline vs online evaluation, A/B testing, evaluation maturity model, production metrics)
- [LangSmith Evaluation](https://docs.langchain.com/langsmith/evaluation) — LangChain/LangSmith documentation (offline vs online evaluation workflow, production feedback loop)
- [CI Regression Gate — NVIDIA NeMo Evaluator](https://docs.nvidia.com/nemo/evaluator/nightly/deployment/ci-regression) — NVIDIA documentation (`nel compare`, statistical significance, threshold tuning)
- [Implementing Quality Gates — NVIDIA NeMo Evaluator](https://docs.nvidia.com/nemo/evaluator/nightly/tutorials/quality-gate) — NVIDIA documentation (`nel gate`, GO/NO-GO/INCONCLUSIVE, benchmark tiers)
- [Maximize AI Agent Performance with Data Flywheels Using NVIDIA NeMo Microservices](https://developer.nvidia.com/blog/maximize-ai-agent-performance-with-data-flywheels-using-nvidia-nemo-microservices/) — NVIDIA Technical Blog
