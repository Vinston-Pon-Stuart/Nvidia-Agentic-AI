## In one line

An aggregate eval score only tells you *if* you have a problem, not *where* or *why* — analyzing results for optimization means reading failing traces into a ranked taxonomy, slicing metrics to find the subgroup the average hides, tracing each failure to the pipeline stage that actually broke, profiling cost/latency separately, and shipping one targeted, re-evaluated fix.

---

## Scope

- Lesson 3.1 builds the evaluation pipeline; 3.2 covers comparing runs and aggregation traps — both get you *a number*. This lesson is what happens after.
- Two agents can both score 78% and need completely different fixes: one is uniformly mediocre everywhere, the other is excellent on 90% of traffic and catastrophic on a 10% subgroup the average quietly buries.
- "The score is low, let's try a better prompt" is a guess dressed up as an action — without reading actual failing traces, the fix you ship has a real chance of missing the dominant failure entirely.

---

## Error analysis: turning failing traces into a taxonomy

- **What it is**: a 3-step methodology (Hamel Husain and Shreya Shankar's AI Evals curriculum) borrowed from qualitative research coding.
- **Open coding** — a domain expert reads real failing traces and writes open-ended notes on what went wrong, focused specifically on the *first* place a trace went off the rails ("the first upstream failure... before the cascading consequences") — one early mistake typically produces a chain of downstream symptoms that all trace back to the same root; annotating every downstream symptom separately just inflates noise.
- **Axial coding** — group the notes into a small named taxonomy (e.g., retrieval miss, wrong tool chosen, hallucinated argument, reasoning error, formatting/parse failure, unwarranted refusal). Hamel calls this "the most important" step — it turns a pile of anecdotes into something actionable.
- **Counting** — tally traces per category, converting the taxonomy into a ranked list: not "the agent sometimes messes up arguments" but "38% of failures are hallucinated arguments, 22% are retrieval misses."
- **When to stop**: annotate until *theoretical saturation* — new traces stop revealing new categories. Rule of thumb: "if ~20 traces don't turn up a new category, you can stop (but review at least 100 to start)."

---

## Slicing: the average that hides the failure

- **What it is**: break the same aggregate metric down by task type, input length, language, user segment, or difficulty tier — a model fine on average can be quietly failing an entire subgroup nobody was watching.
- Practical guidance treats subgroup evaluation as first-class: break every headline metric down "by dimensions that carry risk: customer segment, geography, language, product line, user expertise, workflow type, query complexity, document category, and business function," because "the worst-performing segment may tell you more about system risk than the overall score."
- Direct continuation of Lesson 3.2's aggregation traps: an average is a lossy compression of a distribution, and slicing is how you decompress it. The formal version — models with high overall accuracy can still make "systematic errors on important subsets (or slices) of data" — is established in ML literature on slice discovery.
- **Where to start**: cut your eval set by dimensions you already know carry risk (short vs. long queries, common vs. rare languages, easy vs. hard categories) before reaching for automated slice discovery. Treat any slice meaningfully worse than the aggregate as its own diagnostic target.

---

## Root-cause attribution: which stage actually broke

- **What it is**: once a failure category and its cluster are known, find *which stage* — retrieval, reasoning/tool-selection, tool-call execution, or generation — produced it, because the fix is completely different depending on the answer.
- **RAG triad** (context relevance, groundedness, answer relevance): "the failing leg localizes the bug." Low context relevance → retriever problem (tune chunking, top-k, reranking). Healthy context relevance + low groundedness → generator hallucinating against good context (tighten the prompt to answer only from retrieved content, or lower temperature). High groundedness + low answer relevance → the model faithfully summarized the wrong thing. **Never assume a bad final answer means generation is broken** — trace backward to the first stage that actually failed.
- **MAST** (Cemri et al., Multi-Agent System Failure Taxonomy, arXiv:2503.13657): the most empirically grounded multi-agent failure taxonomy, built from **1,600+ human-annotated traces** across 7 multi-agent frameworks (Cohen's κ = **0.88**). 14 failure modes group into 3 stage-mapped categories: **specification/system-design** (~**42%** of failures — task misinterpretation, ambiguous roles, poor decomposition, missing termination conditions; pre/start of execution), **inter-agent misalignment** (~**37%** — communication breakdowns, lost context on handoff, step repetition, format mismatches; mid-execution), **task verification** (~**21%** — premature termination, incomplete/incorrect verification; end of execution, nothing checked whether the output was right). The verification category is the sharpest illustration of stage attribution: a system can reason and act correctly the whole way and still fail purely because no verification step existed.
- **Tooling**: LangSmith's tracing nests every LLM call/tool execution as child runs of the parent agent run, associating any exception or bad output with the specific step that produced it — answering "did the model hallucinate in step 3?" or tracing a bad answer to "the retrieval step returned an outdated document version." Without step-level trace structure, you're diagnosing a five-step agent from its final output alone — the guessing game this lesson replaces.

---

## Profiling: separating "slow and expensive" from "wrong"

- **What it is**: a separate axis entirely from quality — which step dominates wall-clock time or token spend, independent of whether the output was correct.
- NVIDIA's NeMo Agent Toolkit ships a dedicated **profiler**: callbacks record per-invocation usage stats (tokens used, time between calls, number of LLM calls). Its `bottleneck_analysis` config's `nested_stack` mode identifies "nested bottlenecks like tool calls inside other tool calls" and renders a Gantt-chart timeline — a nine-step trace where one call eats 80% of total latency becomes immediately visible instead of buried in an average. `concurrency_spike_analysis` flags points where simultaneously-running functions cross a threshold, useful for throughput problems that only appear under load. Output: a `workflow_profiling_report.txt` combining latency, token efficiency, and bottleneck highlights.
- **Why it's separate**: quality and cost are orthogonal failure axes. An agent can be cheap and fast while wrong, or slow and expensive while right — and a fix targeted at one axis can regress the other (a bigger, slower model to fix a reasoning error; a tighter retrieval budget that speeds things up but starves the generator of context).

---

## From diagnosis to a targeted fix — then re-evaluate

- The payoff converts "the score is low" into a falsifiable claim: *this stage, on this slice, fails this way, this often.* That points to a specific lever, roughly cheapest to most expensive: clearer prompt/tool description (repeated wrong-tool-selection or malformed arguments) → guardrail/schema check (structural failure) → retrieval tuning (RAG triad points at context relevance) → parameter change (Lesson 3.4's territory, if groundedness is low but retrieval is fine) → more/better training data (Lesson 3.3's territory, if it's a systematic capability gap).
- Discipline: **change one thing**, driven by the highest-ranked category from counting, then re-run the Lesson 3.1 pipeline and compare via Lesson 3.2's methodology — on the overall metric *and* the targeted slice, since a fix aimed at one failure category can silently regress another.
- Same logic as Andrew Ng's classic **ceiling analysis**: manually count which error category dominates ("if 50% of mistakes are dogs, the project can have a big impact") before investing engineering effort — a fix for a category responsible for 3% of errors is a rounding error even if it's technically an improvement.

---

## Trade-offs and pitfalls

- **Chasing the loudest metric instead of the highest-impact failure** — a 2-point dashboard drop feels urgent; a failure category affecting 30% of a low-traffic-but-high-value slice doesn't move the aggregate at all. Frequency × severity, not visibility, should drive priority.
- **Fixing symptoms, not root cause** — a bad final answer downstream of a retrieval miss looks like a generation problem if you only read the last message; trace backward to the first broken stage (RAG triad, MAST-style attribution) before touching a prompt.
- **Changing many things at once** — Anthropic's agent-building guidance: "find the simplest solution possible, and only increasing complexity when needed." Bundle a prompt rewrite with a retrieval change and a parameter tweak, and a re-eval improvement can't be attributed to any one of them.
- **Overfitting the fix to the eval set** — iterating against the same fixed set of failing traces risks tuning to those specific examples rather than the underlying failure mode (Goodhart's law). Hold out fresh traces to confirm a fix generalizes.
- **Ignoring cost/latency while chasing accuracy** — a fix that improves quality but doubles token spend or adds 10 seconds of latency isn't obviously a win; profile before and after, not just score before and after.
- **Small-sample error analysis over-generalizing** — building a taxonomy from 10–15 traces and treating the counts as ground truth is exactly what the "at least 100 traces, stop when ~20 more add nothing new" heuristic exists to prevent.

---

## NVIDIA angle (read critically)

- NeMo Agent Toolkit's profiler is a genuinely useful, concrete instrument for the cost/latency half of this lesson: per-invocation token/timing callbacks, nested bottleneck analysis with Gantt-chart visualization, and concurrency-spike detection answer "which step is slow and expensive" with actual data, not intuition.
- Read critically: the profiler measures *where time and tokens go*, not *why an answer was wrong*. No failure taxonomy, no subgroup slicing, no automated root-cause localization for quality failures — those are handled by the toolkit's separate `trajectory` and `ragas` evaluators (Lesson 2.6 territory), and even those only score a trajectory; a human (or an LLM assistant reading traces) still has to do the open-coding, taxonomy-building, and stage-localization work described above. Treat the profiler as a strong instrument for one axis, not evidence that NVIDIA's tooling automates the harder, judgment-driven half of this lesson — that's still a fundamentally human-in-the-loop discipline, regardless of vendor.

---

## How to think about it for the exam

| Step | Question it answers | Key tool/concept |
|---|---|---|
| **Error analysis** | What *kinds* of failures happen, and how often? | Open coding → axial coding (taxonomy) → counting; read traces, focus on the first upstream failure |
| **Slicing** | *Where* do failures cluster that the average hides? | Break metrics down by task type, length, language, difficulty, segment (extends 3.2's aggregation traps) |
| **Root-cause attribution** | *Which stage* — retrieval, reasoning/tool-selection, tool-call, generation — actually broke? | RAG triad (context relevance / groundedness / answer relevance); MAST's stage-mapped failure categories; step-level trace inspection |
| **Profiling** | Is the problem quality, or cost/latency — and where does time/token spend concentrate? | NeMo Agent Toolkit profiler: bottleneck analysis, Gantt charts, concurrency spikes — a separate axis from quality |
| **Targeted optimization** | Given the diagnosis, what's the cheapest effective lever? | Prompt/tool-description fix → guardrail → retrieval tuning → parameter tuning (3.4) → training data (3.3); change one thing |
| **Re-evaluate** | Did the change actually help, and not regress elsewhere? | Re-run the 3.1 pipeline, compare via 3.2's methodology on overall metric *and* the targeted slice |

---

## Quick self-check

1. Why is an aggregate eval score insufficient to tell you *where* or *why* an agent is failing, and what two analyses does this lesson add on top of it?
   **A:** An aggregate score is a lossy compression of a distribution — it tells you only whether a problem exists, not which kind of failure or which subgroup is affected. This lesson adds error analysis (a ranked failure taxonomy from reading traces) and slicing (breaking the metric down by subgroup to find where failures cluster).
2. Walk through Hamel Husain's three-step error analysis process (open coding, axial coding, counting) and explain why "focus on the first upstream failure" matters when reading a trace.
   **A:** Open coding reads failing traces and writes open-ended notes; axial coding groups those notes into a small named taxonomy — the "most important" step; counting tallies traces per category into a ranked list. Focusing on the first upstream failure matters because one early mistake typically produces a chain of downstream symptoms tracing back to the same root — annotating each symptom separately just inflates noise.
3. An agent's final answer is wrong. Using the RAG triad, explain how you would determine whether the failure is a retrieval problem, a groundedness (hallucination) problem, or an answer-relevance problem — and why you shouldn't just rewrite the final-answer prompt first.
   **A:** Check context relevance first — low means the retriever is the problem (chunking/top-k/reranking). If context relevance is healthy but groundedness is low, the generator is hallucinating against good context. If groundedness is high but answer relevance is low, the model summarized the wrong thing faithfully. Rewriting the final-answer prompt first risks patching a symptom of an earlier-stage failure instead of the actual root cause.
4. What does the NeMo Agent Toolkit profiler measure that error analysis and slicing do not, and why do you need both kinds of analysis rather than just one?
   **A:** The profiler measures cost/latency — where time and tokens go — not whether or why an answer was wrong. Both are needed because quality and cost are orthogonal axes: an agent can be cheap and fast while wrong, or slow and expensive while right, and a fix on one axis can regress the other.
5. Why does "change one thing, then re-evaluate" matter more than it sounds — what specifically goes wrong if you change three things at once and the eval score improves?
   **A:** If a prompt rewrite, a retrieval change, and a parameter tweak all ship together and the score improves, you can't tell which change actually helped, or whether one change's regression is being silently masked by another's improvement — you lose the ability to attribute the result and confidently keep the good change while reverting the bad one.

---

## Sources used

- [Q: Why is "error analysis" so important in LLM evals, and how is it performed? — Hamel's Blog (Hamel Husain)](https://hamel.dev/blog/posts/evals-faq/why-is-error-analysis-so-important-in-llm-evals-and-how-is-it-performed.html)
- [Why Do Multi-Agent LLM Systems Fail? — arXiv:2503.13657 (Cemri et al., MAST taxonomy)](https://arxiv.org/abs/2503.13657)
- [Profiling and Performance Monitoring of NVIDIA NeMo Agent Toolkit Workflows — NVIDIA NeMo Agent Toolkit (1.2) Docs](https://docs.nvidia.com/nemo/agent-toolkit/1.2/workflows/profiler.html)
- [Building Effective AI Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [Machine Learning Yearning — Andrew Ng (PDF)](https://home-wordpress.deeplearning.ai/wp-content/uploads/2022/03/andrew-ng-machine-learning-yearning.pdf)
- [Debugging Deep Agents with LangSmith — LangChain Blog](https://blog.langchain.com/debugging-deep-agents-with-langsmith/)
- [Agentic RAG Failure Modes: Retrieval Thrash, Tool Storms, and Context Bloat — Towards Data Science](https://towardsdatascience.com/agentic-rag-failure-modes-retrieval-thrash-tool-storms-and-context-bloat-and-how-to-spot-them-early/)
- [RAG Triad — TruLens](https://www.trulens.org/getting_started/core_concepts/rag_triad/)
- [Your AI Dashboard Is Hiding Its Worst Failures — Context and Chaos (Substack)](https://contextandchaos.substack.com/p/your-ai-dashboard-is-hiding-its-worst)
