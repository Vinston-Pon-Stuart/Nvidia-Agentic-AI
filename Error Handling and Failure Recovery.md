## In one line

**Error handling and failure recovery** is the classic distributed-systems resilience toolkit (retries, circuit breakers, timeouts, fallbacks) applied to agents, which compound failure risk because a chain of tool calls has roughly N times the failure surface of one call — plus a new failure class where the reasoning layer itself can fail (bad JSON, wrong tool, invented arguments).

- Azure's Architecture Center frames the base case as **transient fault handling**: cloud-connected apps must tolerate faults that are "typically self-correcting" if retried after a delay.

---

## Transient vs. non-transient — the fork in the road

- **Transient faults** self-correct: "momentary loss of network connectivity... temporary unavailability of a service... timeouts when a service is busy" (Azure Retry pattern doc). **Signal:** HTTP **429** and **5xx** are typical retry candidates; most **4xx** (400, 401, 403, 404) indicate a retry won't help.
- **The single most exam-relevant judgment call: retry only transient faults.** Retrying non-transient faults (malformed requests, auth failures, business-logic errors, a resource that genuinely doesn't exist) wastes time, burns cost, and can mask a real bug that needed fixing, not resending.

---

## Retry pattern: strategies, backoff, jitter

- Four strategies: **cancel** (non-transient — stop, report), **retry immediately** (rare one-off faults, e.g., a corrupted packet — but "never do an immediate retry more than once"), **fixed delay**, **increasing delay** (incremental or exponential), up to a max attempt count.
- **Exponential backoff with jitter** — know this cold. Wait a short time, then double the delay each retry (2s, 4s, 8s...); **jitter** = a small random delay added to each interval, to stop many clients from syncing their retries and spiking load together.
  - Marc Brooker (AWS Architecture Blog): plain exponential backoff still produces synchronized retry clusters under contention; jitter spreads them into a roughly constant rate. Simulation with 100 contending clients: jittered backoff **reduced call count by more than half** and improved completion time.
  - Three variants: **Full Jitter** (uniform random up to the exponential cap), **Equal Jitter** (half fixed, half jittered — avoids very short sleeps), **Decorrelated Jitter** (like Full, but next cap depends on the previous draw). Full and Equal perform similarly; Decorrelated does more work with different completion-time behavior.
- **Idempotency is the hard requirement.** Azure: "if [the operation is] not [idempotent], retries could cause the operation to be executed more than once, with unintended side effects" (e.g., a response is lost after a server-side success, so a naive retry duplicates the action). LangGraph's `RetryPolicy` encodes this: retries `ConnectionError`, 5xx, and other transient categories, but **never** retries `ValueError`, `TypeError`, `RuntimeError` — "almost always programming bugs."
- **Caps matter as much as the curve.** Need a max-attempts ceiling and, ideally, a **retry budget** (e.g., "no more than 60 retries per minute" across all requests to a dependency) — per-request limits alone don't stop many concurrent callers from collectively overwhelming a struggling service. "Never implement an endless retry mechanism."

---

## Circuit Breaker pattern

- Retry answers "how do I eventually succeed?" Circuit Breaker answers "when do I stop trying altogether?" Azure: "Retry... expect[s] eventual success. Circuit Breaker prevents an application from performing an operation that's likely to fail." **They combine** — retry logic should stop once the breaker signals the fault isn't transient (a favorite exam trap).
- **Three states — know cold:**
  - **Closed** — normal flow, counts recent failures; threshold exceeded within a window → flips to Open, starts a timeout timer.
  - **Open** — fails immediately without attempting the call ("fail fast"), returns an exception or default/cached value. Protects the caller and the failing dependency from further load.
  - **Half-Open** — after timeout expires, a limited number of trial requests pass through. All succeed → back to Closed, counter resets. Any fail → back to Open, timer restarts. Prevents a recovering service from being flooded right as it comes back online.
- **Why bother:** a call blocked on a slow dependency until timeout holds critical resources (memory, threads, DB connections) that can starve unrelated parts of the system — a circuit breaker rejects immediately once a dependency is known unhealthy.
- **When not worth it:** local in-memory resources (overhead, no benefit), a substitute for basic exception handling, message/event systems that already route failures to a dead-letter queue, or when plain retry algorithms are sufficient.

---

## Timeouts and bulkheads

- A circuit breaker only works if calls are bounded — one with a long timeout can still block a thread before the breaker even registers the failure. **Set timeouts on every outbound call before implementing retry logic** — "a retry strategy is only as effective as the timeouts that govern each individual attempt." Too-long timeouts let threads/connections pile up during an outage; too-short causes premature failure on calls that would have succeeded.
- LangGraph's `TimeoutPolicy`: `run_timeout` (hard wall-clock ceiling on one attempt) + `idle_timeout` (resets on any progress signal — streamed tokens, heartbeat — so long-but-working calls aren't killed, but truly hung ones are).
- **Bulkheads** — isolate resource pools (connections, threads, per-tool concurrency limits) so one failing dependency can't exhaust resources needed elsewhere. Structural partner to timeouts.

---

## Fallbacks and graceful degradation

- Azure's worked example: an app hitting a Cosmos DB throughput ceiling (`429`) trips its breaker, "gracefully degrades by returning default or cached responses" while informing the user of temporary degradation — stays up in reduced-functionality mode, and Azure Monitor alerts ops in parallel.
- LangGraph's `error_handler` node runs only *after* retries are exhausted — cleanup, alerting, dead-letter writes, or fallback paths (cheaper model, cached last-known-good answer, canned apology).
- **SAGA pattern** — for multi-step actions with real-world side effects (book a seat → charge a card → issue a ticket): retry each step individually; if a step's retries are exhausted, run **compensating actions** that undo only the steps that actually completed, in reverse order. Graceful degradation applied to non-idempotent, stateful workflows.

---

## Agent-specific failure recovery

- Three failure modes that don't exist in a plain API call:
  - **Malformed/unparseable LLM output** — model returns text that doesn't match the expected tool-call schema. **Fix: validate, then re-prompt** — feed the parse error back and ask the model to self-correct instead of crashing the run. NVIDIA's ReAct agent: `retry_agent_response_parsing_errors` (default `True`), `parse_agent_response_max_retries` (default `1`) — capped explicitly "to prevent the agent from getting into infinite hallucination loops."
  - **Tool-call exceptions** — a tool throws (bad args, downstream failure, timeout). **Fix: catch and feed the exception back as an observation**, so the agent self-corrects on the next Thought (matches the ReAct loop, 1.2). LangGraph's `ToolNode` (`handle_tool_errors`): the error is returned as the tool result instead of crashing — "the LLM sees the error and can decide to retry with different arguments." NVIDIA's identical knobs: `pass_tool_call_errors_to_agent` (default `True`), `tool_call_max_retries` (default `1`, capped to prevent "infinite tool call loops").
  - **Runaway loops** — a Thought→Action→Observation loop has no natural stopping point if the agent keeps deciding it needs "one more call." **Fix: hard step/loop budget.** NVIDIA's `max_tool_calls` (default `15`) is an absolute ceiling regardless of the agent's own reasoning — a circuit-breaker-like guard on the control flow itself.
- **Throughline:** agent-specific recovery is retry-pattern discipline (bounded attempts, feed the error back as new information) applied one layer up — to the reasoning loop, not just the network call underneath it.

---

## Trade-offs and pitfalls

- **Retrying non-idempotent actions** — Azure flags directly: retrying a command that changes business data (e.g., charging a card) can execute it twice. An agent that retries a tool call blindly, without checking idempotency, can double-book, double-charge, or double-send.
- **Retry storms / cascading retries** — layered retries multiply: a retry count of 3 at two layers adds up to 9 total attempts. Uncoordinated retries at every layer amplify load on an already-struggling dependency — exactly what a retry budget and circuit breaker prevent.
- **Masking real bugs** — indiscriminate retries hide non-transient bugs behind "eventually worked." Azure's own logging guidance: log early retries as informational, only the final failure as an error, so persistent (not occasional) retries stay visible as a signal.
- **Infinite self-correction loops** — the agent-specific analogue of an endless retry mechanism: re-prompting on malformed output or retrying a failing tool forever. Exactly why NVIDIA hard-caps `parse_agent_response_max_retries`, `tool_call_max_retries`, and `max_tool_calls`.
- **Cost** — every retry in an agent context is a real LLM inference call, not a free network resend, and errors compound across a multi-step chain. A five-step agent at 20% per-step transient failure with generous retries can multiply latency/token spend well beyond a happy-path demo estimate.

---

## The NVIDIA angle — read critically

- **Concrete, verifiable:** NeMo Agent Toolkit's ReAct agent config fields — `retry_agent_response_parsing_errors`, `parse_agent_response_max_retries`, `tool_call_max_retries`, `max_tool_calls`, `pass_tool_call_errors_to_agent` — map directly onto "validate & re-prompt," "feed tool errors back as observation," and "loop/step-budget guard." Real, documented, worth citing by name.
- **Thinner:** the toolkit's troubleshooting docs handle the classic distributed-systems case — LLM rate limiting (`429`) — with a single informal bullet ("pause briefly and retry a few times"). **No documented backoff algorithm, no jitter guidance, no circuit breaker anywhere** in the toolkit's public docs at time of writing — contrast with Azure's precisely specified strategies and state machines. **Treat NVIDIA's agent-level retry knobs as real, but don't assume the toolkit ships a production-grade resilience layer** (backoff curves, jitter, circuit breakers) underneath — that discipline still has to come from surrounding application code or a framework (LangGraph, Polly, Resilience4j), exactly as Azure's own guidance recommends.

---

## How to think about it for the exam

| Concept | What to remember |
|---|---|
| **Transient vs. non-transient** | Retry only transient faults (timeouts, 429/5xx, connectivity blips). Never retry non-transient faults (400/401/403/404, business-logic errors) — fail fast instead. |
| **Retry pattern** | Cancel / immediate (once only) / fixed delay / exponential backoff. Requires idempotency. Cap max attempts + use a retry budget. |
| **Jitter** | Random delay added to backoff to desynchronize retrying clients and prevent thundering-herd load spikes (Full / Equal / Decorrelated Jitter). |
| **Circuit Breaker** | Three states: Closed (normal, counts failures) → Open (fail fast, no calls sent) → Half-Open (trial requests) → Closed or back to Open. Purpose: stop hammering a failing dependency, prevent cascading resource exhaustion. |
| **Retry + Circuit Breaker combo** | Retry logic must respect the breaker — stop retrying once the breaker reports the fault isn't transient. |
| **Timeouts / bulkheads** | Set timeouts before adding retries; isolate resource pools so one failing dependency can't starve others. |
| **Fallback / graceful degradation** | Cached/last-known-good response, cheaper model, canned safe reply, or escalate to a human — triggered once retries + breaker both say "stop." |
| **Agent-specific recovery** | Malformed output → validate & re-prompt (bounded). Tool exception → feed error back as an observation so the agent self-corrects. Runaway loop → hard step/tool-call budget. |
| **Key pitfalls** | Retrying non-idempotent actions, retry storms, masking real bugs, infinite self-correction loops, compounding cost across multi-step chains. |

---

## Quick self-check

1. An agent's tool call returns HTTP 404. Should the agent retry it? Why or why not, and what should happen instead?
   **A:** No — 404 is a non-transient fault (the resource doesn't exist), and a retry won't fix it. It should fail fast/cancel and surface the error rather than retry.
2. Name the three circuit-breaker states and explain what triggers each transition between them.
   **A:** Closed (normal, counts failures) → exceeds threshold in a time window → Open (fails fast, no calls sent) → timeout expires → Half-Open (trial requests) → all succeed → Closed, or any fail → back to Open.
3. Why does the Retry pattern doc say retry logic should be "sensitive to" the circuit breaker's exceptions? What goes wrong if it isn't?
   **A:** Because retrying after the breaker has already determined a fault isn't transient wastes time and effort and can further hammer a dependency the breaker is deliberately protecting — the two patterns need to cooperate, not work independently.
4. Explain, in one sentence each, why exponential backoff alone isn't enough at scale and what jitter fixes.
   **A:** Plain exponential backoff still produces synchronized clusters of retries across many contending clients; jitter adds randomized delay so those clusters spread into a roughly constant rate instead of spiking together.
5. Give the three agent-specific failure modes covered in this lesson and, for each, name the concrete recovery mechanism (not just "retry it").
   **A:** Malformed LLM output → validate and re-prompt with the parse error (bounded retries). Tool-call exceptions → catch and feed the error back to the model as an observation so it self-corrects. Runaway loops → a hard step/tool-call budget (e.g., `max_tool_calls`).

---

## Sources used

- [Retry pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [Circuit Breaker pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Best practices for transient fault handling — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transient-faults)
- [Exponential Backoff And Jitter — AWS Architecture Blog (Marc Brooker)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Fault Tolerance in LangGraph: Retries, Timeouts, and Error Handlers — LangChain Blog](https://www.langchain.com/blog/fault-tolerance-in-langgraph)
- [ReAct Agent — NVIDIA NeMo Agent Toolkit (1.2) Docs](https://docs.nvidia.com/nemo/agent-toolkit/1.2/workflows/about/react-agent.html)
- [NVIDIA NeMo Agent Toolkit Troubleshooting (1.2)](https://docs.nvidia.com/nemo/agent-toolkit/1.2/troubleshooting.html)
