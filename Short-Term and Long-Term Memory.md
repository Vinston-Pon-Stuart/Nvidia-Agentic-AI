## In one line

**Agent memory** is an engineering decision about what text gets put back into the prompt on the next LLM call — since the model itself is stateless, "short-term" memory is what fits in the context window right now, and "long-term" memory is everything stored outside it and pulled back in on demand.

---

## Why it's needed

- An LLM call is **stateless** — it only knows what's in the prompt tokens sent to it. Everything an agent "remembers" is really something re-inserted into the prompt.
- **Scope of this lesson:** the architecture treatment — storage tiers, memory-type taxonomy, retrieval/eviction mechanics that wire memory into the control loop. (A sibling Cognition-domain lesson covers memory's role in *reasoning/planning*, not how it's *built*.)

---

## Short-term memory: the context window as working memory

- **What it is:** whatever fits in the model's **context window** — system prompt, conversation history, tool outputs the model sees on this call.
- **Key trait:** fast (no retrieval step) but strictly bounded — once full, something must give.
- **MemGPT** (Packer et al., 2023) formalizes this as **main context**: read-only system instructions + a writeable **working context** block + a **FIFO queue** of rolling history. Only what's in main context is visible to the LLM during inference. OS analogy: main context = RAM.
- **Caveat:** no persistence — close the session or evict the queue, and it's gone unless something copied it out first.

---

## Long-term memory: external and persistent

- **What it is:** lives **outside** the context window — database, knowledge graph, or vector store — pulled in only when relevant. Survives across sessions, enabling personalization and continuity.
- **MemGPT's terms:** **external context** = archival storage (arbitrary-length text, vector search) + recall storage (full message-history DB). The LLM never sees external context directly — only via **function calls** that page data in, mirroring OS paging between RAM and disk.
- IBM's framing: long-term memory is "designed for permanent storage, often implemented using databases, knowledge graphs or vector embeddings," and RAG is "one of the most effective techniques for implementing" it.
- **Letta** (MemGPT's successor) makes the split literal: **memory blocks** ("core memory") = always-visible, pinned prompt text; **archival memory** = a general-purpose vector DB that "cannot be pinned to the context window, and must be queried on-demand via tools."

---

## Memory-type taxonomy: working, episodic, semantic, procedural

From cognitive science, formalized for language agents in **CoALA** (Sumers, Yao, Narasimhan & Griffiths, 2023):

- **Working memory** — active, in-context info for the current step. = short-term memory above.
- **Episodic memory** — records of specific past events. **Generative Agents** (Park et al., 2023) logs every perception as a timestamped natural-language **observation** in a "memory stream" (e.g., *"Isabella Rodriguez is setting out the pastries"*).
- **Semantic memory** — general facts/world knowledge, decoupled from any single event (user preferences, domain facts). Usually embeddings or a knowledge graph.
- **Procedural memory** — skills/rules for *how* to do something; usually in weights, system prompts, or tool/code definitions — not retrieved as text.

**Exam angle:** a logged event with a timestamp = episodic; a stored fact with no occasion attached = semantic; "the agent knows how to format a SQL query" = procedural; "the last three turns of this chat" = working/short-term.

---

## Retrieval memory: embeddings + vector search

- **Standard mechanism:** embed text → vectors → store → at query time, pull nearest neighbors by similarity.
- This is the **RAG** mechanism (Lewis et al., 2020): combines **parametric memory** (facts in the seq2seq model's weights) with **non-parametric memory** (a dense vector index, accessed via a learned neural retriever). RAG showed this beats parametric-only on knowledge-intensive tasks — why "retrieve embeddings, inject into prompt" became the default long-term-memory pattern generally, not just for document QA.
- **Generative Agents' scoring formula** — ranks memory objects on three signals, each normalized [0,1] and summed:

```
score = α_recency · recency + α_importance · importance + α_relevance · relevance
```

  - **Recency** — exponential decay since last access (decay factor 0.995 in the paper).
  - **Importance** — LLM-generated 1–10 "poignancy" score, assigned **at creation time**.
  - **Relevance** — cosine similarity between memory embedding and query embedding.
  - **Why it matters:** retrieval memory in production is *not* pure nearest-neighbor lookup — relevance is weighted against recency and salience so old-but-important and recent-but-mundane facts are both handled sensibly.
- **Is RAG "memory"?** Same mechanism (embed → index → similarity search → inject) whether the corpus is a static external document set or the agent's own history. It's *memory* specifically when the indexed corpus is the agent's own past interactions/facts about the user. The 2024 LLM-agent-planning survey (Huang et al.) calls this "RAG-based memory," distinct from "embodied memory" (experience baked into model parameters via fine-tuning rather than retrieved at inference).

---

## Managing memory: write, read, forget, compaction

- **Write** — what gets stored, when. Generative Agents assigns an importance score at creation so trivial events (brushing teeth) and pivotal ones (a breakup) aren't weighted equally later. Letta's memory blocks are agent-editable via dedicated tools; NVIDIA's **NeMo Agent Toolkit** exposes the same idea as first-class tools — `add_memory_tool`, `get_memory_tool`, `delete_memory_tool` — making write/read/forget explicit callable operations, not implicit prompt engineering.
- **Read** — retrieval on demand, not automatic inclusion. MemGPT's archival-storage function calls and Letta's `archival_memory_search` are pull-based: the agent must decide to query, gets back a paginated, ranked subset.
- **Forget / evict** — what happens when short-term memory fills up. MemGPT's queue manager tracks prompt-token usage against two thresholds: **~70%** inserts a "memory pressure" warning so the LLM can proactively save anything important; **~100%** **evicts** a chunk of the FIFO queue (e.g., oldest 50%), generates a **recursive summary** (new evictions + prior summary), stores it in the first queue slot — while raw evicted messages stay permanently searchable in recall storage. This is the general "context compaction" pattern in any agent framework: summarize what's dropped, keep a pointer/gist in-window, keep the full record retrievable elsewhere.

**Hierarchical / virtual context management (MemGPT)**
- MemGPT's core contribution: naming this pattern **virtual context management**, modeled on OS virtual memory — main context (RAM) ↔ external context (archival + recall storage, disk) ↔ self-generated function calls as the paging mechanism.
- Because *editing and retrieval decisions are made by the LLM itself* ("memory edits and retrieval are entirely self-directed"), the agent manages its own memory hierarchy rather than following a fixed, hand-coded summarization schedule. Know this cold for "what is MemGPT" questions.

---

## NVIDIA-native angle: NeMo Retriever + a vector database

- NVIDIA's retrieval-memory blocks map directly onto the pattern above. **NeMo Retriever**: ingest → embed → retrieve-and-rerank → generate. Documents (or stored interaction records) are chunked, embedded by a **Nemotron embedding model served as a NIM microservice**, stored in a vector DB (NVIDIA's stack uses a cuVS-accelerated index; a reranker reorders top candidates before the generating LLM sees them).
- The **NeMo Agent Toolkit** confirms this is a wired-up integration: `nat.retriever.milvus` (Milvus as a supported vector-DB backend) + `nat.embedder.nim_embedder` (NIM-served embeddings). Pairing a NIM embedding endpoint with Milvus is the standard way to stand up long-term/retrieval memory in an NVIDIA-centric stack — functionally the same embed-then-search substrate as MemGPT's archival storage or Letta's archival memory, just NVIDIA-accelerated.
- **Caveat:** this is architecture description, not a benchmark claim — treat NVIDIA's own throughput/accuracy figures for NeMo Retriever as vendor-reported and unverified here.

---

## Trade-offs and pitfalls

- **Staleness** — long-term facts can go out of date (a stored preference that later changed); most retrieval systems have no automatic invalidation, only overwrite-on-write.
- **Retrieval errors** — embedding similarity is approximate; pulling the wrong/irrelevant memory into context can introduce contradictions or reinforce hallucination rather than prevent it.
- **Cost and latency** — every retrieval call adds an embedding computation + search round-trip; storing everything "just in case" instead of curating by importance makes retrieval slower and noisier — IBM flags this directly as a core design tension.
- **Context overflow** — short-term memory is hard-bounded; without eviction/compaction, an agent hits a hard token-limit failure or pays for expensive, uncontrolled truncation.
- **Privacy** — persistent long-term memory means user data outlives the session; cross-session, cross-user stores need namespacing/access-control (scoped by user ID or session ID) so one user's history can't leak into another's context.

---

## How to think about it for the exam

|Idea|Mechanism / takeaway|
|---|---|
|Short-term / working memory|The context window itself — system prompt + rolling history (FIFO), no persistence beyond the session|
|Long-term memory|External store outside the context window (DB, vector store, knowledge graph) — persists across sessions|
|Episodic memory|Specific timestamped past events (Generative Agents' "observations")|
|Semantic memory|General facts/knowledge, not tied to one event|
|Procedural memory|Skills/how-to, usually in weights, prompts, or tool code — not retrieved text|
|Retrieval memory|Embed → vector-store → similarity search → inject into context on demand; the RAG mechanism|
|Is RAG "memory"?|Same mechanism either way; it's *memory* specifically when the indexed corpus is the agent's own history/facts|
|Hierarchical / virtual context|MemGPT's OS-paging analogy: main context (RAM) ↔ external context (disk), moved by self-directed function calls|
|Eviction / compaction|Summarize + evict low-priority/oldest content near the context limit; keep the summary in-context, keep raw text retrievable externally|
|NVIDIA substrate|NIM-served Nemotron embeddings + a vector DB (e.g., Milvus via NeMo Agent Toolkit) = the retrieval-memory backend|

---

## Quick self-check

1. In MemGPT's terms, what is "main context" analogous to in an OS, and what is "external context" analogous to? Name the mechanism that moves data between them.
   **A:** Main context ≈ RAM; external context (archival + recall storage) ≈ disk. Data moves between them via the LLM's self-generated **function calls** — the paging mechanism.
2. Match each example to a memory type (working, episodic, semantic, procedural): (a) "the last two messages in this chat," (b) "the user mentioned on March 3rd that they're vegetarian," (c) "the agent always formats dates as YYYY-MM-DD," (d) "the company's return policy is 30 days."
   **A:** (a) working, (b) episodic (timestamped event), (c) procedural (a rule for how to do something), (d) semantic (a general fact, no specific occasion).
3. Generative Agents' retrieval function combines three scores into one. Name all three, and say which one is computed at memory-creation time rather than at query time.
   **A:** Recency, importance, relevance. **Importance** (the 1–10 poignancy score) is assigned at creation time; recency and relevance are computed at query time.
4. When does RAG's retrieve-and-inject mechanism count as "memory" specifically, versus just being retrieval over a static knowledge base?
   **A:** When the indexed corpus is the agent's own accumulated past interactions or facts about the user, rather than a fixed external document set — the mechanism is identical either way.
5. Name two pitfalls of long-term memory that don't apply to short-term memory, and explain why the context window is naturally immune to them.
   **A:** Staleness and privacy/cross-session leakage. Short-term memory doesn't persist beyond the session (no session, no stale data or leaked history to worry about) — the context window is wiped when the session ends.

---

## Sources used

- [MemGPT: Towards LLMs as Operating Systems — arXiv:2310.08560](https://arxiv.org/abs/2310.08560)
- [Generative Agents: Interactive Simulacra of Human Behavior — arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks — arXiv:2005.11401](https://arxiv.org/abs/2005.11401)
- [Cognitive Architectures for Language Agents (CoALA) — arXiv:2309.02427](https://arxiv.org/abs/2309.02427)
- [Understanding the planning of LLM agents: A survey — arXiv:2402.02716](https://arxiv.org/abs/2402.02716)
- [What Is AI Agent Memory? — IBM Think](https://www.ibm.com/think/topics/ai-agent-memory)
- [Archival memory — Letta Docs](https://docs.letta.com/guides/core-concepts/memory/archival-memory)
- [Memory blocks (core memory) — Letta Docs](https://docs.letta.com/guides/core-concepts/memory/memory-blocks)
- [NVIDIA NeMo Retriever — NVIDIA Developer](https://developer.nvidia.com/nemo-retriever)
- [Retrievers — NVIDIA NeMo Agent Toolkit (1.3) Documentation](https://docs.nvidia.com/nemo/agent-toolkit/1.3/workflows/retrievers.html)
- [MCP for RAG and Agent Memory: How They Work Together (and How They Differ) — GetKnit](https://www.getknit.dev/blog/powering-rag-and-agent-memory-with-mcp)
