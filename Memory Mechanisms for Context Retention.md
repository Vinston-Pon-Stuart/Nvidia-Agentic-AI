## In one line

**Memory mechanisms** are the plumbing that feeds an LLM agent's reasoning loop the right material at the right time — **working memory** holds the live reasoning state, while **episodic**, **semantic**, and **procedural** memory (a taxonomy borrowed from cognitive-architecture theory, formalized for LLM agents by **CoALA**) get retrieved into it to ground, inform, or skip reasoning entirely.

---

## Historical basis: not invented for LLMs

- **CoALA** (Cognitive Architectures for Language Agents — Sumers, Yao, Narasimhan & Griffiths, 2023) is the spine of this lesson: it separates memory *types* from memory *actions* (reasoning, retrieval, learning) and shows how they compose into a decision cycle.
- Its four-way memory split is lifted from **Soar**, a pre-LLM cognitive architecture for modeling human problem-solving.
- Soar's own definitions: **working memory** = current goals, perceptual input, intermediate reasoning results; **long-term memory** splits into **procedural** (the rule/production system driving behavior), **semantic** (facts about the world), **episodic** (sequences of past behavior).
- CoALA's contribution: this same structure maps cleanly onto LLM agents, with the **LLM itself replacing Soar's hand-written production rules**.

---

## Working memory: the active reasoning workspace

- **What it is:** the live workspace for the current decision cycle — not just "whatever fits in the prompt this turn," but a **data structure that persists across LLM calls**.
- **What it does:** holds perceptual input, active goals, and results of reasoning/retrieval. Each call synthesizes a subset into the prompt; the LLM's output is parsed back into working-memory variables that drive the next action. CoALA calls it "the central hub connecting different components of a language agent."
- **Why it matters:** every intermediate step of multi-step reasoning — a partial plan, a running hypothesis, a sub-goal — has to live here to be usable by the next call.
- **Key caveat:** context-window limits on working memory are a **cognitive** constraint, not just an engineering one (see below).

---

## Reasoning actions: the mechanism that does the "thinking"

- **Retrieval:** reads from long-term memory into working memory — read-only.
- **Reasoning:** reads **and writes** working memory — an LLM call that summarizes an observation, derives the next plan step, or distills an insight; its output feeds back into the same pool the next call reads from.
- **What it's for:** results can support **learning** (written into long-term memory) or **decision-making** (used as context for the next LLM call).
- **Why it matters:** this is the actual mechanism behind "the agent is thinking" — not a separate cognitive module, just LLM calls chained through working memory as both input and output.

---

## Episodic memory: recalling experience to change the next decision

- **What it is:** specific past experience from earlier decision cycles.
- **What it does:** retrieved into working memory during planning to shape the current choice — learning from what happened before, without touching model weights.
- **Generative Agents' reflection mechanism** (Park et al., 2023) goes beyond raw memory-stream retrieval: periodically prompts the LLM with recent observations, asking it to infer higher-level insights and cite supporting memories.
  - Example: observations "Klaus Mueller is writing a research paper" + "...enjoys reading a book on gentrification" → reflection **"Klaus Mueller is dedicated to his research (because of 1, 2, 8, 15)."**
  - Reflections can be reflected on again, forming **trees of reflections** — leaf nodes = base observations, higher nodes = more abstract thoughts.
- **Why it matters:** an agent using only raw observations picks whoever it interacted with most; an agent that retrieves its own reflections picks who shares its actual interest — reasoning has already distilled the *pattern*, not just logged events.
- **Cross-ref:** this retrieve-then-reason loop is exactly what later reflection/self-correction techniques (5.5) build on.

---

## Semantic memory: grounding reasoning (and building it from experience)

- **What it is:** general facts about the world/self that reasoning needs but shouldn't rely on model weights or working memory to hold reliably.
- **What it does:** standard implementation = **RAG's mechanism exactly** — embed a corpus, pull nearest matches at query time. Semantic-memory retrieval and RAG retrieval are the *same operation*, just applied to a knowledge base instead of conversation history.
- **Key nuance:** not necessarily read-only — agents can **write** new knowledge from LLM reasoning into semantic memory.
  - **Reflexion example:** after a failed episode, the agent reasons about what went wrong and stores the conclusion (e.g., "there is no dishwasher in kitchen") as semantic knowledge to ground future reasoning in the same environment.
- Same reason-then-write pattern as episodic memory, just landing in a different store.

---

## Procedural memory: skills that don't need re-deriving

- **What it is:** encodes *how to do something* so the agent doesn't reason through it from scratch each time.
- **Two forms (CoALA):** implicit knowledge in **LLM weights**, and explicit knowledge in **code** — prompt templates, parsers, tool/action definitions.
- **Key caveat:** unlike episodic/semantic memory (can start empty), procedural memory **must be initialized by the designer** — some compiled skill (even just "call the LLM, parse output") is needed before the agent can do anything.
- **Analogy — SwiftSage** (planning survey): dual-process theory — slow thinking (complex LLM reasoning) vs. fast thinking (instinctive, pre-trained response); switches to slow reasoning only when the fast path fails. Procedural memory is the fast path.

---

## The decision cycle: retrieve, reason, act, write, repeat

- CoALA's loop ties all four memory types together: **retrieve** relevant episodic/semantic memory into working memory → **reason** over working memory to propose and evaluate options → **select** → **execute** (an external grounding action or an internal learning write) → **observe** → repeat.
- No memory type does anything alone — the reasoning step reading/writing working memory is what connects them.

---

## Context window as a cognitive constraint, not just an engineering limit

- Because working memory holds all multi-step reasoning, its size caps how much an agent can hold coherently in one pass.
- Planning survey: "for highly complex tasks that are decomposed into dozens of sub-tasks, the planning is constrained by the context length of the LLM, leading to the forgetting of the planning trajectories."
- **Key point:** overflow isn't just lost storage — the agent cognitively loses track of earlier reasoning it needs to stay consistent with later steps. Same eviction/compaction pressure as MemGPT's queue eviction (mechanics covered in 1.4); here the cost is **reasoning state**, not just chat history.

---

## Long context vs. retrieval: two fixes for the same constraint

- **Retrieval (1.4's approach):** keep working memory small, page relevant content in from external long-term memory.
- **Long context:** make working memory itself bigger so more of the trajectory fits without eviction.
- **Jamba-1.5** (AI21, hybrid Transformer-Mamba mixture-of-experts): two sizes — **Large (94B active params)**, **Mini (12B active params)** — both ship **256K effective context**, the largest among open-weight models at release. Mechanism: interleaving **Mamba** (state-space, linear-time) with Transformer attention avoids attention's quadratic cost at long sequence lengths.
- **Trade-off, not a strict upgrade:**
  - **Retrieval** — cheap per call, scales to corpora far larger than any window, persists indefinitely; but approximate — a bad similarity match hands reasoning a false premise with no way to detect it.
  - **Long context** — sidesteps retrieval error, lets attention decide relevance; but costs more compute/memory per call, doesn't persist across sessions, and having something *in* context isn't the same as the model reliably *using* it. Even NVIDIA's own Nemotron 3 Super design keeps dedicated attention layers because "pure SSMs can struggle with precise associative recall."

---

## NVIDIA angle (read critically)

- **Nemotron 3 Super:** hybrid Mamba-Transformer MoE (**120B total / 12B active**), native **1M-token** context window.
- NVIDIA's own problem framing: "Multi-agent systems generate up to **15x** the tokens of standard chats, re-sending history, tool outputs, and reasoning steps at every turn... this 'context explosion' causes goal drift" — an accurate description of the working-memory-overflow problem above.
- Architecture mirrors Jamba: Mamba-2 layers for linear-time complexity, interleaved Transformer attention layers to protect precise recall.
- **Skeptical read:** NVIDIA calls the 1M-token window giving "agents long-term memory for aligned, high-accuracy reasoning" — a **conflation**. A bigger context window is still **working** memory (in-context, ephemeral, gone at session end), not the persistent, cross-session external memory (a database, vector store, or Letta memory block) that "long-term memory" denotes per CoALA/1.4. It genuinely reduces eviction/compaction pressure within one long-running task — useful for the goal-drift problem — but doesn't survive past the session.
- Throughput/accuracy figures ("**5x throughput**," "**85.6%**" on a named benchmark) are vendor-reported and unverified here — treat as marketing pending independent evaluation, consistent with how 1.4 treats NVIDIA's NeMo Retriever figures.

---

## Trade-offs and pitfalls

- **Context-window limits force eviction of the reasoning state itself**, not just old chat history — the agent can no longer fully see its own plan.
- **Retrieval errors corrupt reasoning, not just retrieval accuracy** — a wrong/irrelevant memory becomes a false premise the next LLM call reasons *from*, potentially compounding rather than correcting error.
- **Stale episodic memory misleads decisions** — a reflection is a snapshot; reasoning over an outdated one gives a confidently wrong decision.
- **Reflection trees propagate their own errors** — an early faulty inference can be inherited and amplified by later, higher-level reflections built on top of it.
- **Long context vs. retrieval is a real cost trade-off** — bigger windows cut retrieval error and eviction pressure but raise per-call compute/memory cost and still face "needle in a haystack" recall limits even in architectures purpose-built for it.

---

## How to think about it for the exam

|Idea|Cognitive function|
|---|---|
|Working memory|The active reasoning workspace for the current decision cycle — perceptual input, active goals, and results of reasoning/retrieval; the hub every other memory type feeds into|
|Reasoning action|Reads *and writes* working memory (unlike retrieval, which only reads long-term memory *into* working memory) — this is the actual "thinking" step|
|Episodic memory|Specific past experience, retrieved and reasoned over to change the current decision (Generative Agents' reflection tree: episodic → higher-level insight); bridges to reflection/self-correction learning (5.5)|
|Semantic memory|General world/domain knowledge that grounds reasoning; same retrieval mechanism as RAG; can be read-only or incrementally written from reasoning over episodic memory (Reflexion)|
|Procedural memory|Encoded skills — implicit (LLM weights) or explicit (code/prompts/tools) — executed without re-deriving them via reasoning each time|
|Decision cycle|Propose (reason + retrieve) → evaluate → select → execute → observe → repeat; the loop that ties all memory types to cognition|
|Context window as cognitive constraint|Bounds how much reasoning state can be held at once; overflow causes literal "forgetting of planning trajectories," not just lost chat history|
|Long context vs. retrieval|Two ways to relieve the same constraint: expand working memory (Jamba-1.5, Nemotron 3 Super — hybrid Mamba/SSM + Transformer for near-linear-cost long context) vs. page from external store (RAG/vector DB, per 1.4)|
|NVIDIA framing to scrutinize|A large context window is still working memory, not persistent cross-session long-term memory — read "long-term memory" claims about context length critically|

---

## Quick self-check

1. In CoALA's terms, what is the difference between a "retrieval" action and a "reasoning" action, in terms of which memory each one reads from and writes to?
   **A:** Retrieval only reads from long-term memory into working memory. Reasoning reads *and writes* working memory — it processes working-memory contents to generate new information, which can then be written to long-term memory (learning) or used as context for the next call (decision-making).
2. Walk through Generative Agents' reflection example: what raw episodic observations feed into the reflection "Klaus Mueller is dedicated to his research," and why does having that reflection available change which action the agent picks later?
   **A:** Observations like "Klaus Mueller is writing a research paper" and "Klaus Mueller enjoys reading a book on gentrification" get reasoned over into the higher-level reflection. With that reflection retrievable, the agent later picks who to spend time with based on shared underlying interest rather than raw interaction frequency, because reasoning already distilled the pattern out of the episodes.
3. Explain, in cognitive rather than storage terms, why a context-window overflow during a long multi-step plan is worse than just "losing some old chat history."
   **A:** Working memory is where all multi-step reasoning has to live, so evicted tokens are lost reasoning state — a partial plan, hypothesis, or sub-goal the agent needs to stay consistent with later steps — not just discarded storage. This is literal "forgetting of the planning trajectories."
4. What is the core trade-off between expanding an agent's context window (e.g., Jamba-1.5 or Nemotron 3 Super) versus relying on external retrieval memory for the same goal — name one advantage and one cost of each.
   **A:** Expanding the window avoids retrieval error and lets attention decide relevance, but costs more compute/memory per call and doesn't persist across sessions. Retrieval is cheap per call and scales to huge, persistent corpora, but is approximate — a bad similarity match feeds reasoning a false premise with no way to detect it.
5. Why is describing a large context window as giving an agent "long-term memory" a conflation, using the working/long-term memory distinction from 1.4?
   **A:** A large context window is still *working* memory — ephemeral, in-context, gone once the session ends — not the persistent, cross-session external memory (a database or vector store) that "long-term memory" denotes in CoALA's and 1.4's taxonomy.

---

## Sources used

- [Cognitive Architectures for Language Agents (CoALA) — arXiv:2309.02427](https://arxiv.org/abs/2309.02427)
- [Generative Agents: Interactive Simulacra of Human Behavior — arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
- [Understanding the planning of LLM agents: A survey — arXiv:2402.02716](https://arxiv.org/abs/2402.02716)
- [Jamba-1.5: Hybrid Transformer-Mamba Models at Scale — arXiv:2408.12570](https://arxiv.org/abs/2408.12570)
- [What Is AI Agent Memory? — IBM Think](https://www.ibm.com/think/topics/ai-agent-memory)
- [Introducing Nemotron 3 Super: An Open Hybrid Mamba-Transformer MoE for Agentic Reasoning — NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nemotron-3-super-an-open-hybrid-mamba-transformer-moe-for-agentic-reasoning/)
