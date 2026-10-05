## In one line

**Logic trees, prompt chains, and stateful orchestration** are three ways to hard-code *some* of an LLM system's control flow in advance — so the set of possible paths is knowable and debuggable — as opposed to a free-running agent that decides its own next move at runtime.

---

## Why it's needed

- Anthropic's hard line: **workflows** = control flow *predefined in code*; **agents** = the *LLM itself* directs what happens next. Advice: find the simplest solution, only add agentic autonomy when a workflow demonstrably can't do the job.
- All three patterns here hard-code some structure (which path, what order, what persists) to get a **predictable reasoning path** — inspectable and testable before you ever run it. That predictability is what a fully autonomous agent trades away for flexibility.
- **Scope:** single-agent control flow only. Multi-agent orchestration is a separate lesson (see Multi-Agent Workflows and Coordination). Stateful orchestration also appears in the Cognition domain as a runtime capability; here it's the **architecture-level design choice** made up front.

---

## 1. Logic / decision trees — routing and branching

- **What it does:** classifies an incoming request, sends it down a specialized path. Anthropic: routing "allows for separation of concerns, and building more specialized prompts. Without this workflow, optimizing for one kind of input can hurt performance on other inputs."
- **The classifier doesn't have to be an LLM** — Anthropic explicitly allows "an LLM or a more traditional classification model/algorithm." A plain if/else on a keyword or confidence score still counts as a logic tree.
- **Also a cost/latency lever**, not just a quality one: cheap model for easy/common questions, capable model for hard/unusual ones.
- **Examples:** split customer-service queries into billing/technical/general handlers, each with its own prompt+tools; gate on document length before deciding to summarize directly vs. chunk-and-map-reduce.
- **Key distinction:** a decision tree is **one-shot dispatch** — branch chosen once, rest of the path fixed. Different from stateful orchestration below, where the routing decision can re-fire at every step.

---

## 2. Prompt chains — an ordered sequence of LLM calls

- **Definition (memorize):** prompt chaining "decomposes a task into a sequence of steps, where each LLM call processes the output of the previous one" (Anthropic). Optional programmatic **gates** — checks on intermediate output — catch a chain that's gone off track.
- **When to use:** task decomposes cleanly into fixed subtasks, and you're trading latency for higher per-step accuracy. Examples: generate marketing copy → translate it; write outline → check outline against criteria → write full document.
- **Worked example** (Prompt Engineering Guide): two-step document-QA chain — prompt 1 extracts relevant quotes from a long document, prompt 2 composes the final answer from those quotes + the document. Each step is simpler/more reliable than one combined call, and the intermediate output is inspectable — why chaining "boosts transparency... increases controllability, and reliability."
- **Not the same as chain-of-thought (CoT).** NVIDIA's prompt-engineering primer: CoT is a **single-call** technique (few-shot step-by-step examples, or "let's think about this logically") — one response, reasoning and answer together. Prompt chaining is **multiple separate LLM calls/round trips**, with real state handed between calls and code insertable in between.
- **Branching instead of linear — Tree of Thoughts (ToT).** Yao et al.'s ToT generalizes CoT into a *search tree* over intermediate reasoning "thoughts": the model generates multiple candidate next-thoughts, self-evaluates, can look ahead or backtrack. **Game of 24: GPT-4 + plain CoT solved 4%; GPT-4 + ToT solved 74%.** Distinction from a logic tree: a decision tree branches on *input category* to pick a *handler*; ToT branches on *candidate reasoning content* to search for a *better answer* to the same problem — both are "trees," different problems, don't conflate on the exam.

---

## 3. Stateful orchestration — the workflow as a graph

- **When you need it:** once a workflow needs **loops** (retry until a check passes), **re-evaluated routing** (branch again after every step, not just once), or **durability** (survive a crash, pause for a human, resume across sessions) — chains and routers alone aren't enough.
- **LangGraph** is the reference implementation. Three primitives:
  - **State** — shared data structure (schema) every node reads/writes.
  - **Nodes** — functions taking current state, returning an update (LLM call or plain code).
  - **Edges** — decide the next node. *Normal* edges are fixed (A→B); **conditional** edges call a routing function reading current state and returning the next node name — the logic-tree pattern from section 1, generalized to fire **at any point in the graph**.
  - Docs: "nodes do the work, edges tell what to do next." Because it's a graph, **cycles are legal** — a node can route back to an earlier node for retry-until-valid or iterative refinement.
- **Persistence and checkpointing** — the real payoff of "stateful," beyond "the model remembers the conversation":
  - Compiling with a **checkpointer** (`graph.compile(checkpointer=...)`) snapshots full graph state at every super-step, keyed by `thread_id`.
  - **Fault tolerance** — if a node fails mid-run, completed sibling nodes' outputs ("pending writes") are preserved; resume doesn't re-run finished work.
  - **Human-in-the-loop** — a graph can pause (`interrupt`) at a checkpoint for human inspection/edit, then resume from that exact point.
  - **Conversational memory** — follow-ups to the same `thread_id` continue with full prior state; a separate cross-thread **Store** interface handles memory persisting across different conversations for the same user (stable preferences), distinct from per-thread checkpoints.
  - **Time travel / debugging** — replay from any past checkpoint, or fork an alternate trajectory from a specific step.
  - **Cost:** writing a full state snapshot after every node has overhead — LangGraph offers three **durability modes** (`sync`, `async`, `exit`) trading consistency against performance.
- The planning-survey literature (Huang et al.) categorizes agent planning into task decomposition, plan selection, external modules, reflection, memory. **Prompt chaining = task decomposition; routing and ToT = plan selection.** Stateful orchestration is the substrate that can host any of these.

---

## How they compose

- These three nest rather than compete. Anthropic: "a routing pattern often feeds into a prompt chain," and a chain implemented as a graph can loop via conditional edges. LangGraph can *implement* both routing (conditional edge) and chaining (sequence of nodes), layering persistence on top of either.
- **Contrast with a free-running agent (ReAct):** there the LLM decides the next action itself at runtime, no predefined path — right for genuinely open-ended tasks where steps can't be enumerated in advance, but you pay in unpredictability, harder testing, higher cost from unconstrained iteration.
- **The trade the exam tests:** logic trees, chains, and state graphs keep the **set of possible paths knowable ahead of time** (even if a graph's conditional edges make the *exact* path runtime-dependent) — predictability/debuggability vs. autonomy/flexibility.

---

## Trade-offs and pitfalls

- **Rigidity vs. autonomy** — chains/routers work only when the task "can be easily and cleanly decomposed into fixed subtasks" (Anthropic's own qualifier). An input that fits no branch, or steps unknowable in advance, breaks a hard-coded workflow — the signal to consider an agent instead.
- **Error compounding across chain steps** — each call has some failure rate; N sequential dependent calls compound errors rather than average them out (step 3's input is step 2's possibly-wrong output). Exactly why Anthropic recommends programmatic gates between steps.
- **Latency** — more calls (chaining, multi-hop routing) = more round trips. Anthropic is explicit: the "main goal" of prompt chaining "is to trade off latency for higher accuracy" — a deliberate cost, not a side effect.
- **State bloat** — LangGraph checkpoints the full value of every state channel at each super-step by default; long threads with heavy accumulation (full message history) grow storage over time. (`DeltaChannel`, storing incremental deltas instead of full state, exists specifically to address this.)
- **Framework abstraction risk** — Anthropic's caution: frameworks like LangGraph "can obscure the underlying prompts and responses, making them harder to debug," tempting unneeded complexity. Start with direct LLM calls; reach for a graph only when you need loops, persistence, or human-in-the-loop.

---

## NVIDIA-native angle

- NVIDIA doesn't ship a competing state-machine abstraction — it plugs into and instruments the frameworks above.
- **NeMo Agent Toolkit** is framework-agnostic: adds observability, profiling, evaluation on top of workflows built in LangChain/LangGraph, LlamaIndex, CrewAI, Semantic Kernel, Google ADK, or plain Python. A workflow is declared in `workflow.yml` (LLMs, functions/tools, workflow type); the profiler traces per-invocation usage (tokens, time between calls, number of LLM calls) end-to-end — useful for spotting the latency/error-compounding costs above.
- **NIM + LangGraph** — NVIDIA's worked example of stateful orchestration: NIM microservices expose an OpenAI-compatible tool-calling API, get bound as LangChain tools, get called from nodes inside a LangGraph `StateGraph` — same graph/checkpoint/conditional-edge machinery, with NIM-served models behind the nodes. NVIDIA points developers to LangGraph specifically for "stateful, multi-actor applications" (customer support, coding assistants, advanced RAG).
- **Read critically:** the NVIDIA-native story here is mostly *instrumentation and serving*, not a novel orchestration paradigm — NVIDIA has no proprietary graph/state-machine engine replacing LangGraph.

---

## How to think about it for the exam

| Technique | What it does / when to use |
|---|---|
| **Logic / decision tree (routing)** | Classifies a request, dispatches to a specialized handler; classifier can be an LLM *or* plain code. Use for distinct input categories, or to tier cost/latency. |
| **Prompt chain** | Decomposes one task into an ordered sequence of LLM calls, each consuming the prior output; optional gates between steps. Use when a task cleanly decomposes into fixed subtasks and you'll trade latency for accuracy. Not the same as chain-of-thought (single-call). |
| **Tree of Thoughts (branching search)** | Generalizes CoT into a tree: multiple candidate next-thoughts, self-evaluation, lookahead/backtrack. Use for problems needing exploration, not linear decomposition. |
| **Stateful orchestration (e.g. LangGraph)** | Nodes + edges over shared state; conditional edges re-route at any step; cycles allow retries/loops; checkpointing gives fault tolerance, human-in-the-loop, cross-turn memory, replay. Use when a workflow needs loops, durability, or resumability. |
| **Free-running agent (ReAct)** | LLM decides the next action itself at runtime — no predefined path. Use for genuinely open-ended tasks where step count/order can't be known ahead of time. |

---

## Quick self-check

1. A support system sends billing questions to one prompt and technical questions to another. Which pattern is this, and does the classifier have to be an LLM?
   **A:** Routing (a logic/decision tree). No — Anthropic explicitly allows a traditional classification model/algorithm, not just an LLM.
2. You implement "write outline → check outline against criteria → write full document." Name the pattern, and identify what role the criteria-check plays in it.
   **A:** Prompt chaining. The criteria-check is a programmatic **gate** — it verifies the intermediate output before letting the chain continue, catching a chain that's gone off track.
3. Why does Tree of Thoughts count as branching/tree reasoning but is *not* the same thing as a routing decision tree used to dispatch a request to a handler?
   **A:** A routing decision tree branches on *input category* to select a *handler*, chosen once. ToT branches on *candidate reasoning content* to search for a *better answer* to the same problem, exploring and backtracking — different purpose despite both being "trees."
4. Name two capabilities a LangGraph checkpointer gives you that a plain (non-stateful) prompt chain does not.
   **A:** Any two of: fault tolerance (resume without re-running completed work), human-in-the-loop pause/resume, cross-turn conversational memory via `thread_id`, or time-travel/replay from a past checkpoint.
5. Give two concrete costs of chaining or routing, compared to a single LLM call, that this lesson calls out as trade-offs.
   **A:** Latency (more LLM calls = more round trips, a deliberate trade for accuracy) and error compounding (each step's failure rate compounds across a sequential chain rather than averaging out).

---

## Sources used

- [Building Effective AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents)
- [Prompt Chaining — Prompt Engineering Guide (DAIR.AI)](https://www.promptingguide.ai/techniques/prompt_chaining)
- [Tree of Thoughts: Deliberate Problem Solving with Large Language Models — arXiv 2305.10601 (Yao et al., 2023)](https://arxiv.org/abs/2305.10601)
- [Graph API overview — LangGraph docs (Docs by LangChain)](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [Persistence — LangGraph docs (Docs by LangChain)](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Understanding the Planning of LLM Agents: A Survey — arXiv 2402.02716 (Huang et al., 2024)](https://arxiv.org/abs/2402.02716)
- [An Introduction to Large Language Models: Prompt Engineering and P-Tuning — NVIDIA Technical Blog](https://developer.nvidia.com/blog/an-introduction-to-large-language-models-prompt-engineering-and-p-tuning/)
- [NVIDIA/NeMo-Agent-Toolkit — GitHub README](https://github.com/NVIDIA/NeMo-Agent-Toolkit)
- [Building AI Agents with NVIDIA NIM Microservices and LangChain — NVIDIA Technical Blog](https://developer.nvidia.com/blog/building-ai-agents-with-nvidia-nim-microservices-and-langchain/)
