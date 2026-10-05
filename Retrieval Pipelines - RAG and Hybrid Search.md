## In one line

**RAG (Retrieval-Augmented Generation)** grounds an LLM's answer by retrieving relevant passages from an external, updatable index at query time and handing them to the model as context — instead of relying only on facts frozen in its weights.

---

## Why it's needed

- An LLM's knowledge lives in its weights, frozen at training time — it can't know your internal wiki, today's news, or a doc from five minutes ago without retraining (expensive, slow, still bounded by one checkpoint).
- **RAG** = pull the specific passages relevant to the current query from an external, updatable store and hand them to the model before it generates.
- Lewis et al. (2020, arXiv:2005.11401) frame this as combining **parametric memory** (the model's weights) with **non-parametric memory** (an explicit, inspectable index the model queries) — grounds generation in retrievable, provenance-bearing evidence instead of pure recall.
- Scope note: this lesson covers pipeline mechanics — dense vs. sparse vs. hybrid retrieval, reranking. Vector-index internals (ANN, HNSW, IVF) → 6.2; chunking/embedding quality → 6.3/6.4; knowledge graphs/GraphRAG → 1.7.

---

## What RAG actually is

- Pretrained LMs "store factual knowledge in their parameters" but "their ability to access and precisely manipulate knowledge is still limited" — and lag task-specific retrieve-and-extract architectures on knowledge-intensive tasks (Lewis et al.).
- Two structural problems follow: no **provenance** (can't say which fact it drew on) and no cheap **updating** (can't refresh knowledge without retraining).
- RAG's fix: pair a pretrained seq2seq model (parametric) with a dense vector index of Wikipedia accessed via a trained neural retriever (non-parametric), fine-tuned jointly.
- Two formulations in the paper:
  - **RAG-Sequence** — conditions the whole output on one fixed set of retrieved passages.
  - **RAG-Token** — can draw a different passage per generated token.
- Result: state-of-the-art on three open-domain QA benchmarks at the time, beating both parametric-only baselines and extractive architectures — and produced "more specific, diverse and factual language" than a parametric-only baseline, not just better short-answer accuracy.

---

## The end-to-end pipeline

**Offline** (build the index, then update incrementally):
1. **Ingest and chunk** documents into retrievable units (chunk-boundary quality → 6.3/6.4).
2. **Embed** each chunk (dense) or index it lexically (sparse) and store it (index/ANN structure → 6.2).

**Online** (per query):
1. **Embed the query** with the same model as the corpus (dense), and/or tokenize for lexical matching (sparse).
2. **Retrieve top-k** — cheap, recall-oriented: cast a wide net, don't miss the relevant passage.
3. **Optionally rerank** — expensive, precision-oriented pass over the smaller candidate set.
4. **Assemble context** — join surviving top-N chunks into the prompt.
5. **Generate** — LLM conditions its answer on retrieved context + query.

**Key point:** retrieval quality is a hard ceiling on generation quality — the LLM can only ground in what actually made it into the assembled context.

---

## Dense retrieval: dual encoders

- **Dense Passage Retrieval (DPR)** (Karpukhin et al., arXiv:2004.04906) is the canonical dense retriever and RAG's own retriever (shared co-author).
- **Dual-encoder framework**: one encoder embeds passages, another embeds queries, into the same vector space; relevance = approximate nearest-neighbor search (cosine/dot product), not string matching.
- Trained on (question, relevant-passage) pairs → embeddings capture **semantic similarity**, so paraphrases land near each other — exactly what lexical matching can't do.
- Headline result: DPR "outperforms a strong Lucene-BM25 system... by 9%-19% absolute in terms of top-20 passage retrieval accuracy" across open-domain QA benchmarks, driving new SOTA end-to-end QA.
- **Caveat**: needs a trained embedding model; quality is bounded by how well its training distribution matches your actual documents and queries.

---

## Sparse retrieval: BM25

- **BM25** is the lexical counterpart — DPR's own abstract calls sparse models like BM25/TF-IDF "the de facto method" it was built to beat.
- Per Elastic's breakdown of the Lucene/BM25 formula, a document's score sums, over each query term:
  - **IDF (inverse document frequency)** — rarer terms score higher; common words ("the") are down-weighted.
  - **Term frequency with saturation** — controlled by constant **k1** (default 1.2): the 4th occurrence of a term adds much less than the 1st.
  - **Length normalization** — controlled by constant **b** (default 0.75): a match in a short doc counts more than the same match in a long one.
- No training or embedding model needed — a closed-form statistical formula.
- Strong on rare tokens, exact IDs, product codes, error messages, acronyms.
- **Caveat**: blind to paraphrase/synonymy — scores near zero if query and passage don't share tokens, even when they mean the same thing.

---

## Why hybrid usually wins

- Dense and sparse fail in **complementary** ways: dense misses exact-token queries (part numbers, undertrained proper nouns); sparse misses semantic queries ("cancel my order" vs. a passage that only says "return policy").
- **Hybrid retrieval** = run both, fuse results, so the query benefits from whichever mode matches.
- Fusion is non-trivial: dense (cosine similarity) and sparse (BM25) scores sit on incompatible scales (NVIDIA's NeMo Retriever docs make this point directly).
- Two fusion strategies:
  - **Score normalization + weighted combination** — requires calibrating two incompatible scales.
  - **Reciprocal Rank Fusion (RRF)** — fuses on *rank*, not raw score: `RRFscore(d) = Σ 1/(k + r(d))` summed over rankings, k=60 (empirically near-optimal, "not critical" per the paper). No calibration or training data needed. (Cormack, Clarke & Büttcher, SIGIR 2009.) Beat Condorcet Fuse and CombMNZ, and beat every individual input ranking by ~4–5% MAP on average in TREC/LETOR experiments.
- Real numbers (NVIDIA NeMo Retriever, avg recall@5 across FiQA, HotpotQA, BEIR, NQ, TechQA, top-k=100): **0.5699 dense-only → 0.5856 dense+sparse** — a real but modest gain from fusion alone (reranking's gain is much bigger — see below).

---

## Reranking: a second, precision-oriented stage

- Retrieval and reranking optimize for different things: a first-stage retriever (lexical or dense bi-encoder) "has to be efficient for large document collections... [but] might return irrelevant candidates" (Sentence-Transformers docs).
- **Cross-encoder** reranker: query and document passed *together* into one transformer, outputs a single relevance score — captures relevance far more precisely than comparing two independently-computed embeddings, but too slow to run over a whole corpus.
- Two-stage pattern: cheap recall-oriented retrieval narrows millions of docs to ~100 candidates → expensive precision-oriented reranking picks the few that go into the prompt.
- NVIDIA NeMo Retriever numbers on the same benchmark suite: reranking takes avg recall@5 from **0.5699 → 0.7070** (dense-only base) and **0.5856 → 0.7137** (dense+sparse base). Reranking's gain is much larger than fusion's gain alone, and it helps even with a single retrieval source.
- **Cost**: ~1,750 ms to rerank 500 passages on an H100 GPU — real latency to budget against the recall gain.

---

## Trade-offs and pitfalls

- **Chunking quality dominates retrieval quality**, not the retrieval algorithm — no dense/sparse/hybrid/rerank sophistication recovers info split across a bad chunk boundary (→ 6.4).
- **Embedding-model/domain mismatch** — a dense retriever trained on general/QA data may embed domain jargon or internal terminology poorly, exactly where BM25 keeps working — another argument for hybrid in specialized domains.
- **Latency vs. recall** — wider top-k, hybrid fusion, and reranking all trade latency for quality (the ~1.75s/500-passage figure above is a concrete instance).
- **Over-retrieval + position bias** — Liu et al.'s "Lost in the Middle" (arXiv:2307.03172): LM performance on finding relevant info "is often highest... at the beginning or end of the input context, and significantly degrades... in the middle." Stuffing in more chunks "just in case" costs tokens *and* can bury the one relevant passage where the model is worst at using it.
- **RAG can't fix a bad or incomplete knowledge base** — generation is conditioned on whatever the index contains; retrieval can only surface what was ingested.

---

## RAG vs. fine-tuning

- Same root problem: LLM knowledge "is limited to training... the models cannot easily access company-specific information... unless specifically trained with it" (Towards Data Science / Schürch, 2025).
- **RAG** leaves the model unchanged and extends it at inference time. **Fine-tuning** adjusts weights so new knowledge is stored directly in the model.
- Rule of thumb: **fine-tune for behavior** (style, tone, task-specific reasoning patterns baked into weights); **use RAG for knowledge** (content too extensive or dynamic to bake into weights).
- Not mutually exclusive: **RAFT (Retrieval Augmented Fine-Tuning)** — fine-tune first so the model learns domain terminology/structure, then extend with RAG for current facts. Per the article, this combo "is actually the most common in real production systems."

---

## NVIDIA angle (read critically)

- **NeMo Retriever** = separately deployable **Embedding NIM** + **Reranking NIM** microservices, each a GPU-accelerated container behind an OpenAI-compatible API.
- Embedding NIM handles dense encoding (offline corpus + online query). Reranking NIM performs the cross-encoder second-stage pass, explicitly positioned as necessary "especially when the retrieval pipeline involves citations from different datastores that each have their own algorithms for measuring similarity" — i.e., NVIDIA frames reranking as the fix for hybrid score-disparity, not just a generic accuracy booster.
- Read the recall@5 numbers (0.57 → 0.71 with reranking; ~1.75s/500 passages on H100) as a dated benchmark on one dataset mix and one model pairing — directional, not a universal guarantee.
- NVIDIA's DLI course "Building RAG Agents With LLMs" (LangChain + FAISS + NVIDIA endpoints) — the course-detail page didn't render readable content on direct fetch, so this rests on third-party summaries, not a firsthand read. Treat specifics as unverified.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| RAG (Lewis et al.) | Parametric memory (LLM weights) + non-parametric memory (retrievable index); grounds generation, adds provenance, updates without retraining; RAG-Sequence (one passage set per output) vs. RAG-Token (per-token passage) |
| Pipeline order | Ingest/chunk/embed (6.3/6.4) → index (6.2) → embed query → retrieve top-k → optional rerank → assemble context → generate |
| Dense retrieval (DPR) | Dual-encoder embeddings, ANN similarity search; captures semantic/paraphrase matches; needs training data and a domain-matched embedding model; beat BM25 by 9–19% absolute top-20 accuracy in DPR's own benchmarks |
| Sparse retrieval (BM25) | Term-frequency (saturating via k1) × inverse document frequency, length-normalized (via b); no training required; strong on rare terms, exact IDs, keywords; blind to synonyms/paraphrase |
| Hybrid retrieval | Runs dense + sparse, fuses results; fixes each retriever's blind spot; fusion by score (needs normalization) or by rank (RRF: `1/(k+rank)`, k=60, no calibration needed) |
| Reranking | Second-stage cross-encoder scores query+doc jointly (higher precision, too slow for the full corpus); applied to the small candidate set a cheap first-stage retriever already narrowed down |
| Why rerank helps most | NVIDIA's own benchmark: reranking's recall@5 gain (dense 0.57→0.71) is larger than hybrid fusion's gain alone (dense 0.57→dense+sparse 0.59) — cost is real latency (~1.75s/500 passages on H100) |
| RAG vs. fine-tuning | Fine-tune for behavior (style, task reasoning baked into weights); RAG for knowledge (dynamic, extensive, or private content); RAFT combines both |
| Pitfalls | Chunking quality is the real ceiling (6.4); embedding/domain mismatch; latency vs. recall; over-retrieval + "lost in the middle" positional bias; RAG can't fix a bad knowledge base |
| NVIDIA angle | NeMo Retriever = Embedding NIM (dense encoding) + Reranking NIM (cross-encoder second stage), explicitly framed around fixing hybrid score-disparity; read vendor benchmark numbers as directional, not universal |

---

## Quick self-check

1. In Lewis et al.'s own terms, what is the difference between "parametric memory" and "non-parametric memory" in a RAG system, and what two problems (provenance, updating) does non-parametric memory solve that a parametric-only model can't?
   **A:** Parametric memory is the LLM's weights (facts baked in, no way to point to a source). Non-parametric memory is an explicit, inspectable index queried at inference. It solves provenance (you can point to the retrieved passage) and updating (swap the index instead of retraining).
2. A user query contains an exact product SKU that never appears verbatim near a paraphrased description of it. Explain, using dense retrieval's and BM25's respective mechanisms, why each retriever might fail on this query alone, and why running both and fusing results helps.
   **A:** Dense retrieval embeds meaning, not exact strings, so a rare/undertrained SKU token may not land near the right passage. BM25 matches exact terms, so it fails if the SKU string doesn't co-occur with the paraphrased description. Fusing lets whichever retriever actually matches surface the passage — that's the complementary-failure argument for hybrid.
3. Why does Reciprocal Rank Fusion combine rankings by rank rather than by raw score, and what specific problem in hybrid dense+sparse retrieval does that solve (per NVIDIA's own framing)?
   **A:** Dense (cosine similarity) and sparse (BM25) scores sit on incompatible scales, so a simple reordering by raw score isn't reliable (NVIDIA's own framing). RRF sidesteps this by fusing on rank position (`1/(k+rank)`), which needs no score calibration or training data.
4. Walk through why a cross-encoder reranker isn't used as the *only* retrieval step over an entire multi-million-document corpus — what does a first-stage retriever provide that makes the pipeline practical?
   **A:** A cross-encoder scores each query-document pair jointly, which is far too slow to run over millions of documents. A cheap first-stage retriever (lexical or dense bi-encoder) narrows the corpus to a small candidate set (e.g., ~100) that the reranker can then afford to score precisely.
5. Per the "fine-tune for behavior, use RAG for knowledge" framing, would you reach for RAG, fine-tuning, or both (RAFT) if you needed a model to (a) stay current on a rapidly changing internal policy document and (b) consistently use correct legal terminology and citation style? Justify each choice.
   **A:** (a) RAG — the content is too dynamic to bake into weights and needs live updating. (b) Fine-tuning — terminology/citation style is a behavior pattern best baked into weights. If both are needed together, RAFT combines them.

---

## Sources used

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks — arXiv:2005.11401](https://arxiv.org/abs/2005.11401)
- [Dense Passage Retrieval for Open-Domain Question Answering — arXiv:2004.04906](https://arxiv.org/abs/2004.04906)
- [Practical BM25 — Part 2: The BM25 Algorithm and its Variables — Elastic Blog](https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables)
- [Reciprocal Rank Fusion outperforms Condorcet and Individual Rank Learning Methods — Cormack, Clarke & Büttcher, SIGIR 2009 (PDF)](https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf)
- [Retrieve & Re-Rank — Sentence Transformers documentation](https://sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)
- [Overview of NVIDIA NeMo Retriever Reranking NIM](https://docs.nvidia.com/nim/nemo-retriever/text-reranking/latest/overview.html)
- [Overview of NVIDIA NeMo Retriever Embedding NIM](https://docs.nvidia.com/nim/nemo-retriever/text-embedding/latest/overview.html)
- [Lost in the Middle: How Language Models Use Long Contexts — arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- [How to Make Your LLM More Accurate with RAG & Fine-Tuning — Towards Data Science](https://towardsdatascience.com/how-to-make-your-llm-more-accurate-with-rag-fine-tuning/)
- [Building RAG Agents with LLMs — NVIDIA DLI course (search-sourced; course-detail page did not render readable content on direct fetch)](https://learn.nvidia.com/courses/course-detail?course_id=course-v1%3ADLI+S-FX-15+V1)
