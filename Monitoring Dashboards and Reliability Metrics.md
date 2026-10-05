## In one line

A monitoring dashboard for a deployed agent answers, continuously and at a glance: **is this agent healthy right now, and would I know before a user tells me?** — stacking infra, model-serving, and agent-task metrics against SLO thresholds.

---

## Why this matters

- **What it is** — the *ongoing* production instrument panel, distinct from pre-prod load profiling (Lesson 4.3) and post-incident log/trace debugging (Lesson 8.2).
- **Why agents make this harder than a typical web service:**
  - An agent "request" is a variable-length **trajectory** of reasoning, tool calls, and retries — not one round trip — so request/response metrics undercount failure modes.
  - Agent "correctness" isn't a clean pass/fail like an HTTP status code — there's no "500 error" for a subtly wrong Slack message sent to the wrong channel.
  - An agent sits on a full stack (GPUs, model server, orchestration) each needing its own monitoring vocabulary; a dashboard that only shows the bottom of the stack can look green while the agent itself fails every task.

---

## Three canonical reliability-metric frameworks

- **Four Golden Signals (Google SRE book, *Monitoring Distributed Systems*)** — Latency, Traffic, Errors, Saturation.
  - **Latency:** separate successful- from failed-request latency — a fast failure (instant 500) can make average latency look artificially good.
  - **Traffic:** demand, e.g. requests/sec.
  - **Errors:** explicit (5xx), implicit (200 with wrong content), or by-policy (over-SLA counts as an error even if "successful").
  - **Saturation:** how full the most constrained resource is; systems often degrade *before* 100% utilization, and rising p99 latency is often the earliest saturation signal.
  - Checklist for a **user-facing service as a whole** — the top-level "is the product up" dashboard.
- **RED method (Tom Wilkie, Grafana Labs)** — Rate, Errors, Duration. "Basically the same as the Four Golden Signals, but without saturation" — deliberately narrower so any engineer can be on-call for a service they didn't write. Fits **per-service instrumentation** (retrieval service, tool-execution service, orchestrator).
- **USE method (Brendan Gregg)** — for every resource: **Utilization**, **Saturation**, **Errors**. Resource-oriented, not service-oriented; a systems-performance checklist Gregg says resolves "about 80% of server issues with 5% of the effort." Underlies GPU/infra monitoring (DCGM + Prometheus, Lesson 4.3).
- Wilkie's summary: **"RED is about caring about your users and how happy they are, USE is about caring about your machines."** None of the three alone tells you whether the *agent* did the right thing — that's a fourth layer this lesson adds.

---

## The metric layers of a deployed agent

- **1. Infrastructure/GPU layer** — hardware health: GPU utilization, memory, temperature, power, ECC errors via **NVIDIA DCGM → Prometheus**. A USE-method layer; owned by Lesson 4.3.
- **2. Model-serving layer** — backend health/speed: time-to-first-token (TTFT), inter-token latency (ITL), tokens/sec, queue depth, batch size. RED/Golden Signals map cleanly here (e.g., **NVIDIA Triton** exports this in Prometheus format). Also Lesson 4.3's territory — same metrics, watched continuously instead of under synthetic load.
- **3. Agent-specific layer** — did the agent do its job? Not covered by the other frameworks. NVIDIA's agent-evaluation guidance calls model eval and agent eval "fundamentally different questions" — a model benchmark score doesn't confirm the *system* completed multi-step, tool-using work. Track:
  - **Task Success Rate (TSR)** — fully resolved user intent within stated constraints, not just a plausible-looking answer.
  - **Tool Call Accuracy** — right tool, correct schema, no hallucinated parameters.
  - **Trajectory Efficiency** — steps/tokens per successful task, catching wasteful paths or retry loops.
  - IBM's AI-agent-evaluation reference adds: **success rate/task completion**, **error rate**, **cost** (tokens/compute), **latency**, plus tool-calling failure categories (**wrong function name, missing required parameters, wrong parameter type, hallucinated parameter**).
  - Dashboard panels this implies: task completion/success rate; tool-call success/error rate *per tool* (one flaky API can sink success without the model degrading); steps per task (rising step counts precede failures, like rising p99 precedes saturation); cost/tokens per request (a looping agent burns money before it burns error budget); **human-escalation rate** (rising escalation is a cheap early proxy for "agent losing confidence"); hallucination/quality proxies (drift, LLM-as-judge spot scores, citation/faithfulness checks — full grading methodology is Lesson 2.6/8.3, dashboard just surfaces the rate).

---

## SLI/SLO: the yardstick the dashboard is measured against

- **What it is** — **SLI** = the specific metric measured (e.g., "task success rate over a 5-min window"); **SLO** = its target (e.g., "≥95% over 28 days").
- **Why it matters** — dashboard panels should *be* your SLIs; alert thresholds should *be* your SLOs (or an early-warning fraction of error budget).
- **Cross-reference** — full error-budget/burn-rate mechanics belong to Lesson 4.5, not here.

---

## Building the dashboard: Prometheus, Grafana

- **Stack** — instrumented services expose metrics on an HTTP endpoint → **Prometheus** scrapes/stores as time series → **Grafana** renders panels. NVIDIA's ML-monitoring guide describes the same pairing; Triton already exports GPU/CPU/memory/latency in Prometheus format, so the serving layer plugs straight in.
- **What makes a dashboard useful under incident pressure:**
  - Answers "up / fast / erroring / saturating / task-quality-holding" in one screen, layered agent-task → model-serving → infra/GPU, so a viewer localizes the problem fast.
  - Distinguishes successful- vs failed-request latency (SRE book's warning).
  - **Links to the debugging surface** — Prometheus's alerting guidance: alerts "should link to relevant consoles"; the link target is the trace/log view owned by Lesson 8.2.
  - **Alerts on symptoms, not every cause** — the single most repeated point in both primary sources. SRE book's Bigtable case study: a team so overwhelmed by cause-level alerts they had to *dial back the SLO* just to get quiet enough to fix things properly. Prometheus's own doc: "alert on symptoms... rather than trying to catch every possible way that pain could be caused," and "only page on latency at one point in a stack." For an agent: page on *"task success rate dropped below 90% over 10 minutes"* or *"p95 latency exceeds SLO"* — not *"GPU util is at 40%"* or *"one retrieval pod restarted"* (those go on the dashboard for triage, not the pager).

---

## Trade-offs and pitfalls

- **Vanity metrics vs. actionable ones** — 99.9% uptime while task success silently drops to 60% is *worse* than no dashboard: false confidence.
- **Alerting on causes → alert fatigue** — per SRE book/Prometheus consensus; get it backwards and engineers start ignoring alerts, including real ones (the Bigtable story again).
- **Agent quality is hard to measure live** — ground truth is rarely available in real time; NVIDIA's guide says "proxies such as prediction drift will suffice." Same for agent task success: usually needs a human check or LLM-as-judge pass, both imperfect. Treat live quality panels as signals to sample and investigate, not ground truth (deeper methodology: Lesson 2.6/8.3).
- **Dashboards that show infra health but miss agent-task failure** — the most agent-specific failure mode: GPUs healthy, TTFT low, zero server errors, yet the agent fails most tasks (wrong tool, looping, misread instructions). Off-the-shelf GPU/inference tooling stops at model-serving; the agent-specific layer must be deliberately instrumented — it doesn't fall out of infra monitoring for free.
- **Monitoring overhead has a cost** — every panel/alert is something a human reasons about and the pipeline computes/stores. SRE book's principle: "as simple as possible, no simpler" — remove signals nothing actually uses.

---

## The NVIDIA angle, read critically

- NVIDIA's stack matches the above: **DCGM** (GPU) + **Triton** (model-serving) → Prometheus → Grafana (Lesson 4.3 owns mechanics). **NeMo Agent Toolkit** plugs into existing agent frameworks to add evaluation/observability — task outcomes, trajectories, tool calls — without a full rebuild.
- **Caveat:** NVIDIA's tooling is strong exactly where a GPU vendor's tooling would be strong — infra and model-serving metrics the hardware/server emit natively. The agent-specific layer (task success, tool-call accuracy, trajectory efficiency) is *not* automatic instrumentation — NVIDIA's own guidance says you must deliberately design it in from day one (your own trajectory logging, success criteria). The vendor dashboard will happily say GPUs are healthy and tokens are flowing while the agent quietly fails most tasks, unless you build the top layer yourself.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Four Golden Signals | Latency, Traffic, Errors, Saturation — Google SRE's checklist for a **user-facing service as a whole**; separate successful-request latency from failed-request latency |
| RED method | Rate, Errors, Duration — Tom Wilkie's **per-service** pattern for microservices; a subset of Golden Signals minus saturation |
| USE method | Utilization, Saturation, Errors — Brendan Gregg's **per-resource** (CPU/GPU/disk) checklist; underlies infra/GPU monitoring (Lesson 4.3) |
| Metrics unique to an agent, beyond infra/serving | Task completion/success rate, tool-call success/error rate, steps per task, cost/tokens per request, human-escalation rate — not produced by GPU or model-serving instrumentation automatically |
| SLI vs. SLO on a dashboard | SLI = the plotted metric; SLO = the target threshold that turns a panel into an alert; error-budget math belongs to Lesson 4.5, not here |
| "Alert on symptoms, not causes" | Page for user-facing SLO breaches (e.g., task success rate, latency); route cause-level signals (one pod restart, a GPU blip) to dashboards/tickets to avoid alert fatigue |
| Dashboard vs. this domain's other lessons | This lesson = ongoing production dashboard + metric definitions; Lesson 8.2 = logs/traces/root-cause debugging; Lesson 4.3 = pre-prod profiling under load |

---

## Quick self-check

1. What are the four Golden Signals, and which one does the RED method deliberately drop?
   **A:** Latency, Traffic, Errors, Saturation. RED drops **Saturation**, keeping Rate, Errors, Duration as a leaner per-service pattern.
2. A teammate wants to page on-call whenever any single GPU's utilization dips below 50%. Using the SRE book's symptom-vs-cause framing, why is this likely to cause alert fatigue, and what should the alert be based on instead?
   **A:** GPU utilization is a *cause*-level signal, not something users feel — paging on it (like Bigtable's team over-alerting on SLO-approach causes) leads to fatigue and ignored alerts. Page instead on a user-facing symptom, e.g. task success rate or p95 latency breaching its SLO.
3. Name three agent-specific metrics that a GPU/model-serving dashboard would *not* surface on its own, and explain why each matters.
   **A:** Task Success Rate (did the agent actually resolve intent), Tool Call Accuracy (right tool/schema, no hallucinated params), and Trajectory Efficiency or human-escalation rate (catches wasteful/looping paths or eroding agent confidence) — none of these come from GPU or serving-layer instrumentation.
4. What is the difference between an SLI and an SLO, and which lesson owns the full error-budget mechanics?
   **A:** SLI is the measured metric (e.g., task success rate over 5 minutes); SLO is its target (e.g., ≥95% over 28 days). Full error-budget/burn-rate mechanics belong to Lesson 4.5.
5. Why does NVIDIA's ML-monitoring guide recommend "proxies" like prediction drift instead of always comparing against ground truth?
   **A:** True labels are rarely available in real time, so drift or LLM-as-judge proxies are used instead — treated as signals to investigate, not unimpeachable pass/fail.

---

## Sources used

- [Monitoring Distributed Systems — Google SRE Book, Chapter 6](https://sre.google/sre-book/monitoring-distributed-systems/)
- [The RED Method: How to Instrument Your Services — Grafana Labs (Tom Wilkie)](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)
- [The USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html)
- [A Guide to Monitoring Machine Learning Models in Production — NVIDIA Technical Blog](https://developer.nvidia.com/blog/a-guide-to-monitoring-machine-learning-models-in-production/)
- [What is AI Agent Evaluation? — IBM](https://www.ibm.com/think/topics/ai-agent-evaluation)
- [Mastering Agentic Techniques: AI Agent Evaluation — NVIDIA Technical Blog](https://developer.nvidia.com/blog/mastering-agentic-techniques-ai-agent-evaluation/)
- [Alerting — Prometheus Best Practices](https://prometheus.io/docs/practices/alerting/)
