## In one line

An agent needs two different access methods — exact query languages (SQL/API/knowledge graph) for structured data, semantic RAG for unstructured text — plus a routing layer that decides per query which source(s) to hit, fuses the results, and keeps the whole system honest about how current its answer is.

---

## Why it's needed

- Enterprise knowledge isn't one shape. **Structured** data (a customer's Q3 revenue, an order's shipping status) lives in rows/columns with exact semantics, queryable via SQL/API/graph traversal. **Unstructured** data (contracts, wikis, tickets, PDFs) is queryable only approximately, via the semantic similarity search 6.1 covers as RAG.
- A single question often needs both: "summarize the contract terms for the customer with the highest Q3 revenue" requires an exact aggregation over structured sales data *and* a semantic read of an unstructured contract — neither access method alone answers it.
- A second axis this lesson owns: **freshness**. A text-to-SQL query against a live production DB is real-time by construction. A RAG query against a vector index is only as fresh as the last rebuild — 6.1's "non-parametric memory" framing quietly assumes someone keeps it updated.
- Scope: this lesson is about **unifying** the two axes — how an agent decides which source(s) to hit per query, combines structured + unstructured results, and keeps reads current. Vector RAG mechanics → 6.1; vector-index internals → 6.2; ingestion/CDC pipeline → 6.3; knowledge graphs as a structured-reasoning substrate → 1.7.

---

## The structured/unstructured split

- **Structured** (tables, warehouses, APIs, knowledge graphs) has exact semantics — a column has a defined type, `SUM(revenue) WHERE quarter='Q3'` has one correct answer. Natural access = a **query language** (SQL, graph query, typed API) returning exact rows, not a ranked "similar" list.
- **Unstructured** (documents, wikis, transcripts) has no schema. Natural access = **semantic retrieval** (6.1's dense/sparse/hybrid RAG) — no column to `SELECT`, no join to write.
- **Key caveat**: using the wrong method on the wrong data shape fails by design, not by bug. A vector index asked "what was Q3 revenue for our top customer" returns *chunks that talk about* revenue, not the number — embedding similarity has no notion of aggregation. A SQL engine asked to "summarize what this contract says about termination" doesn't parse — no schema for prose.
- An agent needing both capabilities needs both access methods available as tools it can choose between.

---

## Text-to-SQL: an exact, live query

- **What it is**: an LLM translates a natural-language question into SQL, executes it against the live database, reasons over the returned rows.
- LangChain's SQL-agent sequence: fetch available tables/schemas → decide which tables are relevant → fetch schemas for those tables → generate a query from the question + schema info → self-check the query with an LLM → execute → correct errors surfaced by the DB engine until it succeeds → formulate a response.
- The agent grounds itself in the *actual* schema first — guessing column names from a table's name alone is exactly how hallucinated-schema failures happen. This whole loop (inspect schema → generate → execute as tool call → read back errors → retry) is a direct application of tool calling (2.3): each SQL operation is its own callable tool the agent invokes and observes.
- **Why it's needed over RAG**: a vector search over "revenue by customer" documents returns text *about* revenue — it can't compute `MAX()` or `GROUP BY` over rows it never holds as structured data. Text-to-SQL executes against the live table — the cleanest "real-time" access in this lesson: no index to go stale, because there's no index, just a live query.
- **Why it's genuinely hard**: the **Spider** benchmark (Yu et al., arXiv:1809.08887) — "10,181 questions and 5,693 unique complex SQL queries on 200 databases... 138 different domains" — was built to stress generalization to a database schema the model has never seen (train/test schemas deliberately don't overlap). Headline result: their best contemporary model "achieves only 12.4% exact matching accuracy on a database split setting." Writing valid SQL is the easy part; correctly grounding a query in an unfamiliar schema (which column is `qty` vs. `quantity`, which join path connects two tables) is where it breaks.
- LangChain's own query-checker (run "before executing") targets this failure class by name: NOT IN with NULL values, UNION vs. UNION ALL, exclusive BETWEEN ranges, data-type mismatches, unquoted identifiers, wrong function argument counts, bad casts, wrong join columns — all plausible-looking queries that silently return wrong or empty results, the SQL equivalent of RAG's "confidently wrong" failure mode.
- **Caveat — security**: LangChain flags this directly: "Make sure that your database connection permissions are always scoped as narrowly as possible for your agent's needs." An agent with live SQL access is a live read (or write) path into production data unless permissions are deliberately narrowed. Depth → Domain 9.

---

## Agentic / router RAG: deciding which source to hit

- A **router** (or **agentic RAG**, when the deciding logic is itself an LLM reasoning step) picks which source — vector store, SQL database, knowledge graph, live web search — gets queried for a given question, instead of every query hitting one hard-coded retriever.
- LlamaIndex's router module: "Routers are modules that take in a user query and a set of 'choices'... and returns one or more selected choices" — for selecting the right data source, deciding summarization vs. semantic search, or trying multiple choices and combining results. Each candidate source is wrapped as a `QueryEngineTool` with a natural-language description; a **selector** (LLM-prompt-based, or Pydantic/function-calling-based) reads the query and picks which tool(s) to invoke. This is tool calling (2.3) generalized to *retrieval sources themselves*.
- NVIDIA's agentic-RAG reference architecture names the router node explicitly: "Decides if chunks need to be retrieved from the local retriever... Alternatively, the agent can be programmed to do a web search or simply answer with an 'I don't know.'" Companion nodes: a **query decomposer** (breaks a question into smaller sub-questions needing multiple documents), a **grader** (checks retrieved passages are actually relevant), a **hallucination checker** (checks the LLM's generation is grounded in the retrieved chunk).
- **Contrast with static RAG**: static = always retrieve from the same index, generate once. Agentic/router = decides *whether* to retrieve, *where* from, *whether the result was good enough*, and *whether to retry* with a different source or rewritten query. NVIDIA's own framing: "Traditional RAG: Simple – query, retrieve, generate... Agentic RAG: Dynamic – agent queries, refines, uses RAG as a tool, manages context over time."

---

## Combining structured and unstructured results

- The harder problem after source selection: fusing genuinely different result shapes (a table row and a paragraph of prose) into one coherent answer — this doesn't merge like two ranked chunk lists (6.1's hybrid fusion).
- **HybridRAG** (Sarmah et al., arXiv:2408.04948) is a concrete, evaluated instance for financial-document Q&A: combines Knowledge-Graph-based RAG (GraphRAG) and VectorRAG. On earnings-call transcripts with ground-truth Q&A pairs, "HybridRAG which retrieves context from both vector database and KG outperforms both traditional VectorRAG and GraphRAG individually... in terms of retrieval accuracy and answer generation."
  - **Read narrowly**: one paper, one domain (financial transcripts), combining graph + vector specifically — not a general proof every structured+unstructured combination wins. But the mechanism (retrieve from both, let generation draw on both contexts) is the same shape text-to-SQL + RAG requires.
- **NVIDIA's AI-Q Blueprint** — production-scale illustration: NeMo Retriever extraction is built to "ingest and index structured, semi-structured, and unstructured data" from one pipeline, integrating "ERP, CRM, data warehouses, documents, images, and chat logs" as first-class, side-by-side inputs. Mechanically, the router pattern applied to an entire multi-step research workflow: a reasoning model "iteratively plans, reflects, and refines," pulling from the vector-indexed KB *and* web search as needed, then handing assembled evidence to a report-generation step. Combination = giving generation context from every source the router queried, not a special structured/unstructured merge algorithm.

---

## Real-time and freshness

- A stale index gives confidently wrong answers on time-sensitive questions — same failure shape as any RAG failure, except here the ground truth exists and is queryable; the system just isn't looking at the current version.
- NVIDIA names the underlying problem: agents relying on static training data suffer "Hallucinations... Stale Information... Knowledge Gaps... Security: Data permissions may change over time, or previously available data can become confidential."
- **Two structurally different ways to deliver freshness**:
  1. **Streaming/CDC re-ingestion** (6.3's mechanism) — Change Data Capture and incremental re-extraction feed changed documents/rows into the index continuously; a query is fast but only as current as the last successful ingestion cycle. NVIDIA's AI-Q Blueprint: "enterprise data is continuously extracted, embedded, and indexed so that the system always operates using the most current information." Staleness window is bounded by pipeline latency, not eliminated.
  2. **Query the live source directly, at request time** — text-to-SQL is the clean case: no index to fall behind, the query reads the table as it exists right now. Cost is architectural: every query pays live-query latency and load on a production system, and combining a live read with a vector-indexed source in one answer means fanning out to multiple live systems, each with its own latency/failure mode.
- A **cache** sits between the two and reintroduces the same trade-off at smaller scale: it speeds up repeat queries, but a cached answer is a small, fast-to-go-stale index — inherits the same staleness risk RAG's "swap the index, not the weights" argument was meant to solve, just at a shorter refresh interval.
- **Bottom line**: freshness costs either ingestion-pipeline engineering (streaming/CDC) or query-time latency and load (live source access) — the right choice is workload-specific (a monthly-changing policy doc is fine behind a periodic re-index; a stock price or account balance is not).

---

## Trade-offs and pitfalls

- **Text-to-SQL hallucinates schema, not just syntax** — Spider's own 12.4% exact-match accuracy on unseen schemas shows the hard part is grounding a query in unfamiliar tables/columns; wrong joins and invented column names are the dominant failure, not malformed syntax.
- **A router can pick the wrong source, and the failure is silent** — a misclassifying selector sends a query to a vector store when it needed SQL (or vice versa), or misses that a question needs *both* — no error thrown, just an answer built from wrong or incomplete evidence.
- **Fan-out latency is real when a query needs multiple live sources** — hitting a live SQL DB, a vector index, and web search in one request multiplies both latency and independent failure points vs. a single-source lookup.
- **Freshness vs. cost is a real budget line** — streaming/CDC trades engineering complexity for bounded, cheap-per-query staleness; live source queries trade zero staleness for per-query latency/load; picking the wrong point for a given data type wastes engineering effort or latency budget.
- **Structured and unstructured sources can disagree, and nothing reconciles that automatically** — e.g., a contract PDF says net-30 while a billing database shows net-45 actually enforced; fusing both into a generated answer without surfacing the conflict produces one confident answer that silently picked a side.
- **Security/permissions compound across live sources** — LangChain's "scope database connection permissions as narrowly as possible" applies to every live structured source; NVIDIA names permission drift explicitly ("data permissions may change over time"). An agent with router-driven access to multiple live enterprise systems inherits the access-control surface of all of them at once. Depth → Domain 9.

---

## NVIDIA angle (read critically)

- NVIDIA's story is architectural consolidation, not a single new technique: NeMo Retriever's extraction stage ingests "structured, semi-structured, and unstructured data" through one pipeline; the AI-Q Blueprint composes a Llama Nemotron reasoning model, NeMo Retriever (vector RAG), and web search, connecting to ERP/CRM/data-warehouse (structured) sources alongside documents/chat logs (unstructured); the NeMo Agent toolkit provides router/tool-orchestration plumbing (first-party LangChain, LlamaIndex, CrewAI plugins). A real, coherent architecture — the pieces map onto this lesson's mechanisms, not marketing abstractions over nothing.
- **Read specific claims more skeptically than the architecture**: "enterprise data is continuously extracted, embedded, and indexed so that the system always operates using the most current information" is a design description, not a measured freshness SLA — it says the pipeline is built for continuous updates, not what the actual end-to-end staleness window is in a given deployment.
- Case studies (Pangaea Data's "98% accuracy in retrieving critical data points" combining structured/unstructured patient data; VAST Data's "real-time, multimodal AI agent intelligence") are partner-reported through NVIDIA's own channel, not independently audited — directional evidence the pattern is deployed in practice, not a guarantee of the same numbers elsewhere.
- **The strongest, most independently checkable claim in this lesson's sources**: HybridRAG's own reported result — combining KG and vector retrieval beat either alone on both retrieval accuracy and answer generation, on a specific financial-document evaluation — is a real, arXiv-published, peer-reviewable result, not a NVIDIA-blueprint talking point.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Structured vs. unstructured access | Structured (SQL/API/KG) → exact query semantics, aggregation, live reads; unstructured (RAG, 6.1) → semantic similarity search over free text; using the wrong method on the wrong data shape fails by design, not by bug |
| Text-to-SQL mechanism | LLM inspects schema → generates query → self-checks → executes as a tool call (2.3) → reads back DB errors → retries; gives exact/aggregate answers RAG structurally cannot |
| Text-to-SQL failure mode | Schema grounding, not syntax — Spider's own models hit only 12.4% exact-match accuracy on unseen database schemas; wrong joins/hallucinated columns are the dominant error, per LangChain's own query-checker list |
| Agentic / router RAG | A selector (LLM-prompt or function-calling/Pydantic) picks which source-as-tool to query per question — vector store, SQL DB, KG, web search — and can decide *whether* to retrieve at all, not just *what* to retrieve |
| Contrast with static RAG | Static: fixed query→retrieve→generate. Agentic: reasons about retrieval — decomposes questions, grades retrieved chunks, checks for hallucination, can retry with a different source or rewritten query |
| Combining structured + unstructured | HybridRAG (KG + vector) beat either alone on retrieval accuracy and answer generation in its own evaluation; NVIDIA's AI-Q pattern: router-selected sources all feed one generation step, not a special merge algorithm |
| Real-time / freshness — two strategies | (1) Streaming/CDC re-ingestion keeps an index fresh with bounded lag (6.3); (2) live source queries (text-to-SQL) have zero staleness but pay per-query latency/load; caching reintroduces the same trade-off at smaller scale |
| Pitfalls | Wrong-source routing is silent; live multi-source fan-out costs latency; structured/unstructured sources can disagree with no automatic reconciliation; live DB/API access inherits real permission risk (Domain 9) |
| NVIDIA angle | NeMo Retriever + AI-Q Blueprint architecturally combine structured/unstructured ingestion, reasoning, and live web search; the architecture is real, but freshness/accuracy claims and partner case studies are vendor-reported and should be read as directional |

---

## Quick self-check

1. Using the "summarize the contract terms for the customer with the highest Q3 revenue" example, explain why neither a pure vector-RAG system nor a pure text-to-SQL system alone can answer it, and what has to happen for an agent to answer it correctly.
   **A:** Vector RAG can only return text *about* revenue, not compute `MAX()`/aggregation over rows it doesn't hold structurally; text-to-SQL can't parse or summarize unstructured contract prose. The agent needs both access methods available as tools and must combine an exact SQL aggregation with a semantic read of the matching contract document.
2. Spider's own benchmark models achieved only 12.4% exact-match accuracy on a database-split test set. What specifically does that number reveal about *where* text-to-SQL systems actually fail — is it query syntax or something else? Name two concrete mistakes LangChain's own query-checker tool is designed to catch.
   **A:** It reveals the failure is schema grounding, not SQL syntax — correctly identifying the right tables, columns, and joins in an unfamiliar schema is the hard part. LangChain's checker catches things like using NOT IN with NULL values, UNION vs. UNION ALL confusion, exclusive BETWEEN ranges, and wrong join columns.
3. Walk through how a router (per LlamaIndex's own framing) decides which of several data sources to query for a given question, and name at least two selector mechanisms it can use to make that decision.
   **A:** Each candidate source is wrapped as a `QueryEngineTool` with a natural-language description of what it's useful for; a selector reads the incoming query against those descriptions and picks which tool(s) to invoke. Two selector mechanisms: an LLM selector (dumps choices into a prompt, uses text completion to decide) and a Pydantic/function-calling selector (passes choices as schemas into a function-calling endpoint).
4. Explain the two structurally different ways a system can deliver "real-time" freshness (per this lesson), what each one costs, and why a cache doesn't actually escape the trade-off.
   **A:** (1) Streaming/CDC re-ingestion keeps an index updated with bounded lag — cost is pipeline engineering and a staleness window equal to ingestion latency. (2) Querying the live source directly (text-to-SQL) has zero staleness but costs per-query latency and load on production systems. A cache doesn't escape this — it's just a smaller, faster-to-go-stale index, inheriting the same staleness risk at a shorter refresh interval.
5. An agent has live access to both a production SQL database and a company's contract-document vector index, and the two sources disagree about a customer's payment terms. What does this lesson say happens if the agent just fuses both into one generated answer, and why is that a governance-relevant risk rather than just an accuracy one?
   **A:** Nothing reconciles the conflict automatically — the agent produces one confident answer that silently picked one source over the other, without surfacing the disagreement. It's governance-relevant, not just an accuracy issue, because it compounds with the permission/access-control risk of live multi-source access (Domain 9) — a wrong silent choice on something like payment terms has compliance and audit implications, not just a wrong fact.

---

## Sources used

- [Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task — arXiv:1809.08887](https://arxiv.org/abs/1809.08887)
- [HybridRAG: Integrating Knowledge Graphs and Vector Retrieval Augmented Generation for Efficient Information Extraction — arXiv:2408.04948](https://arxiv.org/abs/2408.04948)
- [Build a SQL agent — LangChain Documentation](https://docs.langchain.com/oss/python/langchain/sql-agent)
- [Routers — LlamaIndex Documentation](https://developers.llamaindex.ai/python/framework/module_guides/querying/router/)
- [Traditional RAG vs. Agentic RAG—Why AI Agents Need Dynamic Knowledge to Get Smarter — NVIDIA Technical Blog](https://developer.nvidia.com/blog/traditional-rag-vs-agentic-rag-why-ai-agents-need-dynamic-knowledge-to-get-smarter/)
- [Build an Agentic RAG Pipeline with Llama 3.1 and NVIDIA NeMo Retriever NIMs — NVIDIA Technical Blog](https://developer.nvidia.com/blog/build-an-agentic-rag-pipeline-with-llama-3-1-and-nvidia-nemo-retriever-nims/)
- [Chat With Your Enterprise Data Through Open-Source AI-Q NVIDIA Blueprint — NVIDIA Technical Blog](https://developer.nvidia.com/blog/chat-with-your-enterprise-data-through-open-source-ai-q-nvidia-blueprint/)
