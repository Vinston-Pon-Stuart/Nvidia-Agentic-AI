## In one line

Once a dashboard metric (Lesson 8.1) says *something* broke, **logs, traces, and drift detection** are the instruments that answer **why** — logs give discrete event detail, traces give the causal path through an agent's steps, drift detection flags that the underlying traffic/behavior pattern shifted.

---

## Why this matters

- **What it is** — the live, production-incident counterpart to Lesson 3.5's *offline* error analysis (curated eval traces, annotated at leisure). This lesson diagnoses one real failing request or a just-fired anomaly, using whatever logging/tracing was (or wasn't) captured at the time.
- **Why agents are hard to diagnose** — a single agent "turn" is a **tree**, not one operation: a planning step, one or more tool calls, a retrieval, a generation step, maybe a nested sub-agent call. A bad final answer could originate at any node; aggregate metrics (8.1) only say *some* node on *some* traffic is unhealthy — logs/traces/drift detection localize it to the specific node, specific request.
- **Cross-reference** — Lesson 2.4 covers what to do once the cause is known (retry, circuit-break, fallback); this lesson is entirely about getting to that cause.

---

## The three pillars, and where this lesson sits

- IBM's framing: **metrics** = quantitative, answer "what"; **logs** = immutable exhaustive event records, answer "why"; **traces** = a request's end-to-end journey across components, answer "where/how." Combined: "metrics alert teams to problems, traces show their path of execution, logs provide the context needed to resolve them."
- Division of labor: 8.1 owns "what" (dashboard/metric/alert); this lesson owns "why/where" (logs = granular detail, traces = causal path).
- **Why traces aren't just "fancy logs"** — a pile of timestamped log lines from five steps doesn't tell you which step *caused* which; you reconstruct by hand. A trace encodes causality directly as a tree of **spans**, each with start/end time, a parent, and nested children.
- **Caveat (IBM)** — "tracing alone cannot reveal data trends or patterns," and distributed tracing requires instrumentation wired up *before* the failure — not a free upgrade over logs.

---

## Structured logging: correlation IDs make a log line findable later

- **Structured logging** — emit machine-parseable records (JSON, consistent fields) instead of free text; IBM: makes logs "simpler to parse and analyze" vs. regex archaeology on unstructured text.
- **Correlation ID** — a single identifier tracking one request end-to-end across all services (Last9), typically an `X-Correlation-ID` header or queue-message property, echoed into every log line. Works across any tech stack with minimal instrumentation (generate a UUID at entry, propagate, log everywhere).
- **Trace ID** — a stricter, richer version: "the root identifier in distributed tracing... groups all spans across services into one trace tree, with span IDs tracking individual operations" (Last9). In mature stacks, "the trace ID functions as your correlation ID" — logs, metrics, and traces all link through the same IDs.
- **Exam-likely:** without a propagated correlation/trace ID, a distributed agent's logs are just an unordered pile of events with no way to reassemble a single failed request — the reason Lesson 4.1 requires fleet-wide tracing, not per-process logging alone, for a distributed agent fleet.

---

## Distributed tracing for agents: why a trace localizes a multi-step failure

- **The question this pillar answers**, per OpenTelemetry's blog: "Your AI agent just took 45 seconds... Was it the model? A slow tool call? A retry loop?... without observability, you are guessing." A log line saying "generation took 30 seconds" doesn't say *why*; only a full span tree does.
- **OpenTelemetry Semantic Conventions for Generative AI** — the emerging vendor-neutral span standard. Worked example: a top-level `invoke_agent` span with child `chat` spans (per LLM call) and `execute_tool` spans (per tool call) — mirroring the actual plan → tool call → retrieval → generation control flow.
  - Standardized attributes per span: `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens`, `gen_ai.response.finish_reasons`, and (if content capture is on) `gen_ai.input.messages`/`gen_ai.output.messages`.
  - This is the mechanism that can answer "did the model hallucinate in step 3, or did retrieval return the wrong document" — impossible from flat logs, which don't encode which step's output fed which step's input.
- **LangSmith** operationalizes the same idea at the app level: wrapping an LLM client auto-logs every call as a nested span; `@traceable` on application functions nests their inputs/outputs/child spans into one trace. Worked example: an `assistant` function calling `get_context` — outer span captures the assistant's I/O, nested spans capture the tool call and the exact prompt/response.
  - Lesson 3.5 already used this same nesting for offline analysis ("associating any exception or bad output directly with the specific step that produced it"). Difference here: this lesson pulls up the same span tree the moment a live alert fires, for the one failing request, with less time and less certainty.
- **Two adjacent tracing layers, don't conflate:** Lesson 4.3's **Nsight Systems** traces are hardware/kernel-level (CUDA calls, GPU kernels, NCCL collectives) — a different stack tier than this lesson's application/request-level tracing. A perfect Nsight trace can't reveal a stale retrieved document; a perfect LangSmith/OTel trace can't reveal a stalled GPU kernel. Lesson 7.3 covers NVIDIA's NeMo Agent Toolkit contribution to the application layer in depth.

---

## Error tracking and classification

- **Exception** (API threw, network timeout) — Lesson 2.4's retry/circuit-breaker/fallback territory; diagnosis here is just confirming which span threw and whether it's transient (429, 5xx) or not (400, 404).
- **Silently wrong** (no exception, but hallucination / wrong tool / misread instruction) — not a code error; needs Lesson 3.5's RAG-triad/MAST-style stage attribution to classify.
- **Whole-request failure with no single span erroring** — often the input distribution underneath the agent has shifted; see drift detection below, often invisible at single-trace level.

---

## Anomaly and drift detection: the automated "something's wrong" trigger

- **What it does** — metrics and traces both assume you already know something's wrong. Drift detection tells you *that* something is wrong before any individual request looks broken.
- **Data drift** (Evidently AI) — "the input data has changed... the trained model is not relevant for this new data" though it would "still perform well" on data resembling the old distribution. Example: a propensity model trained mostly on paid-search traffic degrades once a campaign shifts most traffic to Facebook — the model didn't get worse, the traffic mix did.
- **Concept drift** — the opposite: input distributions may stay the same, but "the relationships between the model inputs and outputs change" — the world changed underneath familiar-looking inputs.
  - Three flavors: **gradual** (expected, slow — competitors, macro shifts), **sudden** (discontinuity — Evidently's COVID example: yoga-pants demand surging 350% overnight), **recurring** (seasonality — "has no importance" if the model already accounts for it; a Black Friday spike isn't drift, and alerting on it is a false-positive trap).
- **Training-serving skew** — a related but distinct failure: a mismatch revealed at first deployment, not a change over time (Evidently's example: a model trained on clean invoice images "surprised by the diversity" of real ones on day one).
- **Exam-likely: data drift ≠ concept drift** — data drift is a change in *P(inputs)*; concept drift is a change in the input→output relationship. Different fixes: retrain on recent data vs. redesign/re-label because the target itself changed meaning.
- **Agent mapping** — data drift = query/context distribution shifts (new product line the retriever's index never covered, a new language). Concept drift = subtler and more dangerous: the query distribution looks identical but the *correct answer* changed (an upstream API's schema changed, a policy was updated, a tool's behavior silently changed after a vendor deploy) — the agent keeps confidently giving the old, now-wrong answer.
- **Data-quality signals that feed drift alerts** (Evidently): **call volume** over time (a sudden drop to ~0 is usually a wiring failure, not drift — "an easy way to catch when something is very wrong"); **schema validation** (are expected fields still present/typed correctly); **missing-data rate** per field; **feature value range/statistics**, including flatlines (a "broken sensor" that stays in-range but stops varying — agent analog: a tool/retriever silently returning the same cached response regardless of query).
- A metric anomaly (8.1) is worth investigating as real specifically when it correlates with one of these distribution signals moving — not a one-off blip or known seasonal pattern.

---

## Root-cause diagnosis workflow: alert to first broken span

1. **Alert fires** — a metric anomaly (8.1) or a drift detector crossing a threshold flags a time window or traffic segment.
2. **Find the affected trace(s)** — use correlation/trace ID, time window, and any segment filter (language, cohort) to pull the actual flagged requests.
3. **Walk the span tree backward to the first failing step**, not the step where the symptom appeared. A wrong final answer with a healthy generation span is almost always caused earlier (bad retrieval, wrong tool, a silently absorbed tool exception) — the same "trace backward before touching a prompt" discipline from Lesson 3.5, applied to one live trace instead of a batch.
4. **Classify the first failure** — bad retrieval, wrong tool, hallucinated argument/answer despite good context, unhandled tool exception, or a drift signal.
5. **Hand off** — transient exception/malformed output → Lesson 2.4 (retry/circuit-breaker/fallback); systematic quality failure (recurring wrong-tool selection, retrieval gap) → Lesson 3.5 (error-analysis taxonomy); genuine drift → retraining, re-indexing, or updating reference data.

- Step 3 only works if instrumentation exists — a root-cause investigation with "an alert but no trace" dead-ends immediately, which is the whole reason structured logging and span-per-step tracing matter.

---

## Trade-offs and pitfalls

- **Logging everything isn't free** — IBM: comprehensive logging "can bury important information under... 'noise,'" and high-volume structured logs "significantly increase data volume," requiring sophisticated log-management tooling. Argues for sampling and bounding full-content capture (prompts, completions, retrieved docs) to a fraction of traffic.
- **Full request/response logging is a PII risk**, not just a cost one — capturing `gen_ai.input.messages`/`output.messages` or full retrieved documents by default puts every user query and retrieved context in a log store; this is why OTel's GenAI content capture is opt-in.
- **Traces can't be retrofitted** — instrumentation must exist *before* the failure; there's no reconstructing a span tree for a request that already completed without a span. An un-instrumented service makes an incident unrecoverable for root-cause purposes regardless of diagnostic skill.
- **Correlation IDs break silently if any hop drops them** — every system in the chain must continue the trace instead of starting a new one; one un-instrumented service or one async hop (queue, background job) that doesn't forward the header fractures "one trace" into two disconnected ones.
- **Drift alerts have real false-positive risk** — confusing known/expected patterns (seasonality) with genuine drift is the sharpest version; the reference window a live distribution is compared against (last week? last quarter? original training set?) directly determines whether a benign shift trips the alarm.
- **Non-determinism defeats simple repro** — per Lesson 4.1, an agent can take a different reasoning path on the same input across runs, so you often can't just "run it again" to confirm a diagnosis — the actual failing run's trace may be the only evidence that will ever exist.
- **The whole chain fails if the trace was never captured** — sampling, an exporter outage, or an uninstrumented service at incident time all produce the same dead end.

---

## The NVIDIA angle, read critically

- **NeMo Agent Toolkit (NAT)** ships real plumbing mapping directly onto the mechanisms above: an event-driven `IntermediateStepManager` publishes `IntermediateStep` events to a reactive stream (function execution boundaries, LLM calls, tool usage), with **span exporters** converting events into spans for distributed tracing, plus OpenTelemetry exporters for OTLP backends — all processed asynchronously, "off the hot path."
- Logging config mirrors the log-level discipline described above: `console`/`file` providers, `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL`, configurable per exporter.
- **Real answer to fractured multi-workflow traces**: when one NAT workflow calls another (HTTP or programmatically), a `parent_id`/`parent_name` (via `session.run()` or `workflow-parent-id`/`workflow-parent-name` headers) attaches the child's spans under the parent's trace, so backends "show a single, connected tree instead of separate traces."
- **Read critically:** none of this is a novel diagnosis technique — OpenTelemetry is a CNCF standard, and NAT's contribution is conformance + multi-backend fan-out (Phoenix, Langfuse, LangSmith, Weave, from one config), not a better localization method than "walk the span tree."
- NAT's **troubleshooting docs are thin by design**: three short bullet categories (workflow-registration issues; one runtime bullet on `429 Too Many Requests` recommending pausing and retrying; dependency-installation issues). No documented root-cause workflow, no span-tree-walking guidance, and — notably — **no drift or anomaly detection capability anywhere in the toolkit**. NAT gives excellent raw spans/logs; both the reading discipline (this lesson) and drift monitoring (Evidently-style, not in NAT) have to come from outside the toolkit. Treat NAT as an instrumentation layer, not a diagnosis product.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Three pillars, split by lesson | Metrics = aggregate "what/when" (8.1); Logs = discrete event "why"; Traces = causal "where/how" across steps (this lesson owns logs + traces) |
| Structured logging | JSON/machine-parseable, leveled (DEBUG→CRITICAL); alone it can't reconstruct cross-service order |
| Correlation ID vs trace ID | Correlation ID = one flat tag across a request's whole journey; trace ID = the same idea, structured as a span tree with parent/child + timing; in mature stacks the trace ID *is* the correlation ID |
| Why a trace beats a log line for agents | An agent turn is a span tree (plan → tool call → retrieval → generation, e.g. OTel's `invoke_agent`→`chat`/`execute_tool`); only the tree structure answers "which step actually broke," not just "something was slow" |
| Distributed tracing tooling | LangSmith: nested runs via `@traceable`/OpenAI wrapper; OpenTelemetry `gen_ai.*` semantic conventions = vendor-neutral standard; NAT (7.3) = OTel-based, async, cross-workflow `parent_id` stitching |
| Tracing vs. GPU-level tracing | Application/request tracing (this lesson) ≠ Nsight Systems (4.3) kernel/GPU tracing — different stack layer, different questions answered |
| Data drift vs. concept drift | Data drift = P(inputs) changes, model itself still fine on familiar inputs; concept drift = the input→output relationship changes, inputs can look identical; training-serving skew = mismatch at first deployment, not a change over time |
| Drift flavors | Gradual (expected aging), sudden (discontinuity, e.g. COVID), recurring (seasonality — not real drift if already modeled) |
| Data quality/integrity signals | Call volume, schema validation, missing-data rate, feature value range/statistics (flatlined "broken sensor" pattern) — the automated inputs to a drift/anomaly alert |
| Root-cause workflow | Alert (8.1/drift) → find the trace(s) via correlation/trace ID → walk span tree backward to the *first* failing step → classify (bad retrieval / wrong tool / hallucination / tool exception / drift) → hand off to 2.4 (fix) or 3.5 (taxonomy) |
| Biggest pitfalls | Log-everything cost/noise/PII risk; traces can't be retrofitted after the fact; correlation IDs break if any hop drops them; drift false positives from unmodeled seasonality; non-determinism defeats simple repro; the whole chain fails if the trace was never captured |

---

## Quick self-check

1. A user reports a wrong answer from an agent. The generation-step span looks completely healthy (well-formed output, no exception). What does the root-cause workflow tell you to do next, and why shouldn't you assume the generation step is the culprit?
   **A:** Walk the span tree backward to find the *first* failing step, not the step where the symptom appeared — a wrong answer with a healthy generation span is almost always caused earlier (bad retrieval, wrong tool, an absorbed tool exception).
2. Explain the difference between a correlation ID and a trace ID, and why "the trace ID functions as your correlation ID" in a mature observability stack.
   **A:** A correlation ID is a flat tag identifying one request across services; a trace ID is the same idea structured as a span tree with parent/child spans and timing. In mature stacks they're unified — the trace ID does the correlation job while also carrying the causal hierarchy.
3. Using OpenTelemetry's `gen_ai` semantic conventions example (`invoke_agent` → `chat`/`execute_tool` spans), explain why a flat set of log lines from the same agent run cannot answer "did the model hallucinate in step 3, or did retrieval return the wrong document" as reliably as a trace can.
   **A:** Flat logs don't encode which step's output fed which step's input; the span tree's parent/child structure directly mirrors the plan→tool→retrieval→generation flow, so you can inspect each step's `gen_ai.input.messages`/`output.messages` in causal order.
4. Distinguish data drift from concept drift with an agent-specific example for each, and explain why "recurring drift" (seasonality) is usually not something a drift alert should fire on.
   **A:** Data drift: users start asking about a new product line the retriever's index never covered (P(inputs) shifted). Concept drift: an upstream API's schema or a company policy changes, so the same-looking query now has a different correct answer. Seasonality is a known, expected repeating pattern already accounted for by the model, not an actual distribution change — alerting on it is a false positive.
5. Name two concrete pitfalls that can make this lesson's alert → trace → root-cause chain fail even when the diagnosis discipline itself is followed correctly.
   **A:** Traces can't be retrofitted (instrumentation must predate the failure), and correlation IDs break silently if any hop (an un-instrumented service, an async queue) fails to propagate them — either one leaves you with an alert and no usable trace.

---

## Sources used

- [Three Pillars of Observability: Logs, Metrics and Traces — IBM](https://www.ibm.com/think/insights/observability-pillars)
- [Tracing quickstart — Docs by LangChain (LangSmith)](https://docs.langchain.com/langsmith/observability-quickstart)
- [Inside the LLM Call: GenAI Observability with OpenTelemetry — OpenTelemetry Blog](https://opentelemetry.io/blog/2026/genai-observability/)
- [Correlation ID vs Trace ID: Differences & When to Use Each — Last9](https://last9.io/blog/correlation-id-vs-trace-id/)
- [Machine Learning Monitoring, Part 4: How To Track Data Quality and Data Integrity — Evidently AI](https://www.evidentlyai.com/blog/machine-learning-monitoring-how-to-track-data-quality-and-integrity)
- [Machine Learning Monitoring, Part 5: Why You Should Care About Data and Concept Drift — Evidently AI](https://www.evidentlyai.com/blog/machine-learning-monitoring-data-and-concept-drift)
- [Observe Workflows — NVIDIA NeMo Agent Toolkit docs (GitHub raw)](https://raw.githubusercontent.com/NVIDIA/NeMo-Agent-Toolkit/develop/docs/source/run-workflows/observe/observe.md)
- [NVIDIA NeMo Agent Toolkit Troubleshooting (1.2)](https://docs.nvidia.com/nemo/agent-toolkit/1.2/troubleshooting.html)

*Cross-references: Lesson 8.1 (dashboards/metrics — the "what/when" this lesson's diagnosis starts from), Lesson 3.5 (offline error analysis, failure taxonomy, RAG-triad stage attribution, the NeMo Agent Toolkit profiler — the same trace structure applied to curated eval traces rather than a live incident), Lesson 2.4 (retry/circuit-breaker/fallback — what happens after this lesson's diagnosis lands on a transient or tool-level cause), Lesson 4.1 (distributed agent fleet, non-determinism, why per-process logging alone can't localize a fleet-wide failure), Lesson 4.3 (Nsight Systems — the GPU/kernel tracing tier, distinct from this lesson's application-level tracing), Lesson 7.3 (NeMo Agent Toolkit overview — the OpenTelemetry-based observability layer used here in its root-cause-diagnosis role).*
