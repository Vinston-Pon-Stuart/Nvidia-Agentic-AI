## In one line

An **evaluation pipeline** is a repeatable, automatable harness — dataset, runner, scorers, aggregation — that runs an agent against curated tasks and produces quantitative, comparable scores instead of one-off "it worked when I tried it" impressions.

---

## Scope vs. sibling lessons

- Lesson 2.6 owns the developer-side loop of evaluating agent *decisions* mid-build (heuristic checks, LLM-as-judge on trajectories, self-consistency, reflection, evaluator-optimizer) — not repeated here.
- Lesson 3.2 owns comparing performance across runs once you have pipeline output; Lesson 3.5 owns analyzing and acting on results.
- This lesson: how to build the harness, and what to run through it.

---

## Why agent evaluation is hard

- **Nondeterminism** — same prompt/model, even at temperature 0, can produce different tool calls and answers across runs. A single run is a sample, not a measurement.
- **Multi-step trajectories, not single outputs** — a correct final answer can hide an inefficient/fragile path (excess tool calls, wrong tool chosen then corrected); a wrong answer can still reflect mostly-sound reasoning. Scoring only the last message throws away most of the signal.
- **No single ground truth** for open-ended tasks ("book a flight that fits constraints," "summarize this diff") — unlike a math benchmark, there's no one string to match against.
- **Real-world side effects** — an agent can change external state (a DB row, a filed ticket, a sent email), so correctness sometimes has to be checked against the *state the world ended up in*, not the text produced.
- **τ-bench** (Yao, Shinn, Razavi & Narasimhan, 2024, arXiv:2406.12045) makes the state-based case concrete: it grades by comparing end-of-conversation database state against an annotated goal state. Headline finding: GPT-4o-class agents **succeed on <50% of tasks**, and **pass^8 (all 8 repeated trials succeeding) is under 25%** on the retail domain — the same agent, same task, often can't repeat its own success.

---

## Anatomy of an evaluation pipeline (memorize the shape)

**Dataset → runner → scorers → aggregation/report.** NVIDIA's NeMo Agent Toolkit (`nat eval`) is structured exactly this way:

1. **Dataset (gold set)** — curated inputs, sometimes with expected outputs/steps. NAT supports `json`/`jsonl`/`csv`/`xls`/`parquet`/`custom` loaders, and splits **structured** entries (fixed `id`/`question`/`answer` the harness understands) from **unstructured** ones (raw record handed to the workflow, a custom evaluator interprets it) — this is how a public benchmark like `SWE-bench_Verified` (pulled from a HF parquet file, `structure.disable: true`) drops into the same harness as a hand-written internal eval set.
2. **Runner** — executes the workflow on every entry (parallel; NAT's default `max_concurrency` is 8) and captures the full trajectory, not just the final answer. NAT's `workflow_output.json` logs `question`, expected `answer`, `generated_answer`, `intermediate_steps` per sample.
3. **Scorers/evaluators** — exact-match, LLM judge, RAG-specific metric, trajectory judge, etc. — each producing a per-item score plus, ideally, a machine-readable reason.
4. **Aggregation/reporting** — per-item scores rolled into an average (ideally with variance, not just a mean). NAT writes one `<evaluator>_output.json` per configured evaluator, a console summary table, and reproducibility snapshots (`config_original.yml`, `config_effective.yml`, `config_metadata.json`) so a score traces back to the exact config that produced it.

---

## Metric families

- **Exact-match / programmatic** — string equality, regex, unit test pass, clean diff apply. Cheap, deterministic, zero judge-bias risk. NAT's `langsmith` evaluator wraps LangChain's `openevals` (`exact_match`, `levenshtein_distance`). SWE-bench's core scoring ("does the patch make the held-out tests pass") is this family at benchmark scale.
- **LLM-as-judge** — for open-ended outputs with no single correct string. NAT's `langsmith_judge` (built on `openevals`'s `create_llm_as_judge`) supports prebuilt rubrics (`correctness`, `hallucination`), custom prompts, `few_shot_examples` to calibrate, and `use_reasoning` (CoT before scoring, on by default). Bias mechanics (position/verbosity/self-enhancement) are Lesson 2.6's territory; here the point is narrower — a judge is one swappable **component**, not the whole pipeline.
- **RAG-specific** — RAGAS (Es, James, Espinosa-Anke & Schockaert, 2023, arXiv:2309.15217): reference-free RAG evaluation, built because RAG quality has several independent failure points. **Faithfulness** (claims traceable to context, computed as supported-claims / total-claims, 0–1) and **Answer Relevancy** score the *generation* side; **Context Precision** (are retrieved chunks useful or noise) and **Context Recall** (did retrieval surface everything needed) score the *retrieval* side. Low faithfulness + high context precision → generator ignoring good context (a prompting problem); low context recall → retriever itself is the bottleneck (an indexing/chunking problem) — one end-to-end score can't localize which.
- **End-to-end vs. component metrics** — one pass/fail for the whole run (τ-bench's DB-state match, SWE-bench's tests-pass, GAIA's exact-match) vs. decomposed scores per pipeline stage. NAT's example config runs both at once: `accuracy` (RAGAS `AnswerAccuracy`, end-to-end), `groundedness` (RAGAS `ResponseGroundedness`, generation), `relevance` (RAGAS `ContextRelevance`, retrieval), `trajectory` (judge-LLM over the whole intermediate-steps path) — four independent scorers on one run, because a single number can't tell you *where* in the pipeline a problem lives.

---

## Task benchmarks: what each one stresses

| Benchmark | What it stresses | Concrete numbers |
|---|---|---|
| **GAIA** (Mialon et al., 2023, arXiv:2311.12983) | Multi-step reasoning + multimodality + web browsing + tool-use; "simple for humans, hard for AIs" | 466 questions (300 answers withheld); **humans 92% vs. GPT-4+plugins 15%** |
| **SWE-bench** (Jimenez, Yang, Wettig et al., 2023, arXiv:2310.06770) | Real software engineering: edit a codebase to resolve a real GitHub issue across multiple files | 2,294 problems from 12 Python repos; Claude 2 resolved 4.8%, GPT-4 resolved 1.7% (with oracle retriever) |
| **τ-bench** (Yao et al., 2024, arXiv:2406.12045) | Tool-agent-**user** interaction + domain-policy adherence; introduces **pass^k**, a consistency-across-trials metric | gpt-4o-class agents succeed on <50% of tasks; pass^8 <25% in retail domain |
| **AgentBench** (Liu, Yu, Zhang et al., 2023, arXiv:2308.03688) | Breadth: 8 interactive environments testing general reasoning/decision-making, not one task type in depth | 25 LLMs tested; large commercial/open-source gap; weak long-term reasoning, decision-making, instruction-following flagged as main failure causes |

- **Not interchangeable**: GAIA = tool+reasoning breadth on assistant tasks; SWE-bench = depth on one hard, verifiable skill (code); τ-bench = policy adherence + reliability under repeated trials; AgentBench = coverage across many environments. Match the benchmark's stress profile to what you need to know, don't default to the best-known one.

**Public benchmark vs. your own domain eval set**
- A public benchmark buys standardization/comparability, not domain fit — your tools, policies, and failure modes aren't in someone else's fixed dataset.
- NAT's architecture treats a public benchmark as just another dataset: the same abstraction that loads a hand-written `json` set also loads `SWE-bench_Verified` from a HF `parquet` file, with `filter.allowlist` on `instance_id` to run a couple of rows while debugging. Same plumbing — only the task-distribution match differs.
- Most mature setups run both: a public benchmark for external comparability/contamination checks, and a hand-curated gold set as the primary regression signal.

---

## Offline vs. online evaluation

- **Offline** — fixed dataset, usually reference outputs, runs before/independent of production traffic. Classic regression-testing mode: hold inputs constant, vary one thing, re-run, compare.
- **Online** — scores live traffic in place, no pre-written reference, uses proxies (LLM-judge rubrics, heuristics, user feedback), watches trends over time rather than a pass/fail number.
- Offline tells you a change is *safe to ship*; online tells you what shipped is *still working*. Feedback discovered online should feed back into growing the offline gold set.
- NAT's `--skip_workflow` flag re-runs only the evaluators over a dataset with pre-populated `generated_answer`, skipping re-execution of the agent — pure offline scoring of fixed outputs, and structurally the same shape you'd use to retroactively score exported production transcripts.

---

## Running evals repeatably

- **Versioned datasets** — the gold set is a versioned artifact; if it silently changes between runs, score changes are uninterpretable. NAT's `config_original.yml`/`config_effective.yml`/`config_metadata.json` snapshots tie a score to the exact dataset/overrides/endpoint used.
- **Repetition for variance, not just a mean** — nondeterminism means a single run's average is a noisy point estimate. NAT's `--reps N` re-runs the full eval N times for average *and* variance — directly relevant given τ-bench's finding that the same agent can succeed then fail on the same task.
- **CI-gated regression testing** — run the eval suite on every PR touching a prompt/tool/model; fail or flag the build if the score drops below a threshold or vs. the previous run.
- **Resumability at scale** — large datasets (SWE-bench-Verified alone: 500 entries) make full reruns expensive; NAT's `--skip_completed_entries` resumes an interrupted run instead of restarting from zero.

---

## Trade-offs and pitfalls

- **Benchmark contamination / leaderboard overfitting** — an audit of SWE-bench (Aleithan et al., 2024, arXiv:2410.06992) manually reviewed a leaderboard-topping system's "solved" patches and found **32.67%** involved solution leakage (the fix was present in the issue text/comments — copied, not solved) and **31.08%** passed due to weak test cases that didn't actually verify correctness. Filtered, the reported resolution rate dropped from 12.47% to 3.97%. Over 94% of SWE-bench issues predate common LLM knowledge cutoffs — some "solved" instances may just be memorized. A high public-benchmark score can reflect leakage/artifacts, not genuine capability.
- **Judge cost and bias** — every LLM-judge call is extra inference cost/latency and inherits position/verbosity/self-enhancement bias (Lesson 2.6); a pipeline leaning on one unblinded judge measures the judge's preferences as much as the agent's quality.
- **Gold-set staleness** — a hand-curated eval set reflects the product/tools/policies at the time it was written and silently stops representing reality unless actively maintained.
- **Metric gaming (Goodhart's law)** — optimizing hard against a single metric (a judge score, exact-match rate, a RAGAS sub-score) teaches the system, or the developer tuning prompts against the eval, to satisfy the metric's letter rather than its intent.
- **Eval sets too small for significance** — 20 hand-written examples can't reliably distinguish a genuine 3-point regression from run-to-run noise, especially given agent nondeterminism (sizing this properly is Lesson 3.2's territory).

---

## NVIDIA angle (read critically)

- NeMo Agent Toolkit's `nat eval` is a working implementation of this lesson's anatomy: dataset loaders (`json`/`jsonl`/`csv`/`xls`/`parquet`, or a public HF-hosted benchmark), and configurable evaluators — `ragas` (wraps the external RAGAS library, including NV-flavored `AnswerAccuracy`/`ContextRelevance`/`ResponseGroundedness`/`FactualCorrectness`), `trajectory` (judge-LLM score over intermediate steps), `tunable_rag_evaluator` (customizable weighted coverage/correctness/relevance judge), and `langsmith`/`langsmith_judge` (wraps LangChain's `openevals`).
- Read this as **orchestration/tooling**, not novel evaluation science — RAGAS is an independent open-source project, `openevals`/LangSmith are LangChain's. The genuine engineering contributions are the plumbing: the shared dataset abstraction (internal set + public benchmark, one config schema), `--reps`/`--skip_completed_entries` for CI-friendly repeatable/resumable runs, `--skip_workflow` for offline re-scoring, and `config_effective.yml` reproducibility.
- Worth a skeptical read: NAT's docs surface a "leaderboard for the judge LLM" (sourced from RAGAS's own NV-metrics leaderboard) with **`nvidia/Llama-3_3-Nemotron-Super-49B-v1` ranked first**. It's evidence-based (RAGAS's own published comparison), but it's still vendor documentation prominently recommending the vendor's own NIM-hosted model as the best judge — verify independently before trusting a single judge's ranking.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Why agent eval is hard | Nondeterminism, multi-step trajectories (not single outputs), no single ground truth for open-ended tasks, real-world side effects/state changes |
| Pipeline anatomy | Dataset/gold set → runner (executes agent, captures full trajectory) → scorers/evaluators → aggregation & versioned reporting |
| Metric families | Exact-match/programmatic (verifiable tasks); LLM-as-judge (open-ended, biases in 2.6); RAG-specific (RAGAS: faithfulness/answer relevancy = generation, context precision/recall = retrieval); end-to-end vs. component metrics |
| Benchmarks and what they stress | GAIA = multi-modal/tool-use/web reasoning (92% human vs 15% GPT-4+plugins); SWE-bench = real coding depth (GPT-4 1.7% w/ oracle retriever); τ-bench = tool+user+policy adherence, pass^k reliability (<50% success, <25% pass^8); AgentBench = breadth across 8 environments |
| Public benchmark vs. own eval set | Public = standardized/comparable, not domain-representative; own gold set = domain-accurate, no external comparability; same pipeline plumbing runs both |
| Offline vs. online | Offline = fixed dataset, pre-ship regression/benchmarking; online = live traffic, no reference, trend-watching; `--skip_workflow` re-scoring is the offline pattern in concrete form |
| Repeatability | Versioned datasets + config snapshots, `--reps` for variance (not just mean), CI-gated thresholds, resumability for large datasets |
| Biggest pitfalls | Benchmark contamination/solution leakage (SWE-bench: 32.67% leaked), judge cost/bias, gold-set staleness, Goodhart's-law metric gaming, eval sets too small for significance (→ 3.2) |

---

## Quick self-check

1. Why does scoring only an agent's final answer (and not its trajectory) risk missing real quality problems? Give a concrete example.
   **A:** A correct final answer can hide an inefficient or fragile path (excess tool calls, a wrong tool chosen then corrected), and a wrong final answer can still reflect mostly-sound intermediate reasoning — scoring only the last message throws away that signal.
2. Name the four RAGAS metric families and say which two score the retrieval side of a RAG pipeline versus the generation side, and what a low-faithfulness/high-context-precision combination would tell you about where the problem lives.
   **A:** Faithfulness and Answer Relevancy score generation; Context Precision and Context Recall score retrieval. Low faithfulness with high context precision means the generator is ignoring good context — a prompting problem, not a retrieval problem.
3. Pick two of GAIA, SWE-bench, τ-bench, and AgentBench, and explain concretely what each stresses that the other doesn't.
   **A:** GAIA stresses multi-modal, tool-use, and web-browsing reasoning breadth on assistant-style tasks (humans 92% vs. GPT-4+plugins 15%); SWE-bench stresses depth on one hard, verifiable real-world skill — code (GPT-4 resolved only 1.7% even with an oracle retriever). Breadth vs. narrow-but-verifiable depth.
4. What's the difference between offline and online evaluation, and how does NAT's `--skip_workflow` flag concretely embody the offline pattern?
   **A:** Offline evaluation runs against a fixed dataset with reference outputs, pre/independent of production traffic; online evaluation scores live traffic with no reference, using proxies, and watches trends. `--skip_workflow` re-runs only the evaluators over a dataset whose `generated_answer` is already populated, skipping re-execution of the agent — pure offline scoring of fixed, already-collected outputs.
5. What did the SWE-bench audit (arXiv:2410.06992) find about "solution leakage," and why is that finding a warning against building an evaluation pipeline exclusively around a single public benchmark?
   **A:** It found 32.67% of a leaderboard-topping system's "solved" patches had the fix already present in the issue text/comments, and 31.08% passed due to weak test cases; filtering these dropped the resolution rate from 12.47% to 3.97%. A high public-benchmark score can therefore reflect data leakage or dataset artifacts rather than genuine capability.

---

## Sources used

- [Ragas: Automated Evaluation of Retrieval Augmented Generation — arXiv:2309.15217](https://arxiv.org/abs/2309.15217)
- [τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains — arXiv:2406.12045](https://arxiv.org/abs/2406.12045)
- [GAIA: a benchmark for General AI Assistants — arXiv:2311.12983](https://arxiv.org/abs/2311.12983)
- [AgentBench: Evaluating LLMs as Agents — arXiv:2308.03688](https://arxiv.org/abs/2308.03688)
- [SWE-bench: Can Language Models Resolve Real-World GitHub Issues? — arXiv:2310.06770](https://arxiv.org/abs/2310.06770)
- [SWE-Bench+: Enhanced Coding Benchmark for LLMs — arXiv:2410.06992](https://arxiv.org/abs/2410.06992)
- [Agent Evaluation in NVIDIA NeMo Agent Toolkit (evaluate.md, GitHub source)](https://raw.githubusercontent.com/NVIDIA/NeMo-Agent-Toolkit/develop/docs/source/improve-workflows/evaluate.md)
- [List of available metrics — Ragas docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)
- [Faithfulness — Ragas docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/)
- [Evaluation concepts (offline vs. online evaluation) — Docs by LangChain (LangSmith)](https://docs.langchain.com/langsmith/evaluation-concepts)
- [Mastering Agentic Techniques: AI Agent Evaluation — NVIDIA Technical Blog](https://developer.nvidia.com/blog/mastering-agentic-techniques-ai-agent-evaluation/)
- [Evaluating RAG and Semantic Search Systems — NVIDIA Deep Learning Institute course](https://learn.nvidia.com/courses/course-detail?course_id=course-v1%3ADLI+S-FX-32+V1)
