## In one line

A **knowledge graph** stores entities and their relationships as explicit, queryable data (nodes + edges), so an agent can traverse relationship paths and answer multi-hop or "connect the dots" questions that flat vector similarity search structurally can't.

---

## Why it's needed

- **Vector RAG's blind spot:** it retrieves chunks *semantically similar* to a query — great for single-hop fact lookup, but it has no notion of how facts **connect**. "What did entity X do, given X's relationships to Y and Z" may have zero relevant chunks — the answer must be *assembled by traversing relationships*, not read off one passage.
- Knowledge graphs (KGs) store those relationships explicitly as first-class data, so an agent walks node to node and reasons over paths.
- **Scope:** what a KG is, why explicit structure enables multi-hop/relational reasoning, how agents query/build graphs (Cypher/SPARQL, GraphRAG), and NVIDIA-native tooling (cuGraph, NeMo/NIM, PyG). Vector RAG mechanics belong to a separate domain — here only as a contrast point.

---

## What a knowledge graph is

Per Neo4j, three components:

- **Nodes (entities/vertices)** — people, places, products, concepts; each has a label (type) + optional properties.
- **Relationships (edges)** — connect two nodes, labeled with a relationship *type*, optionally with their own properties.
- **Organizing principles** — a schema or **ontology** layered over nodes/relationships for conceptual structure. Not mandatory — Neo4j's own guidance: add one only when the use case demands it, since ontologies are costly to define and maintain.

The atomic unit is the **triple**: *(subject, predicate, object)* — e.g., `(Ivermectin, TARGETS, CYP3A4)`. Two dominant storage approaches:

- **RDF triple stores** (queried with **SPARQL**) — express everything as subject-predicate-object triples. Rigid: relationships can't natively carry properties; modeling "since when" or "with what confidence" needs workarounds (**reification**, **singleton properties**) that bloat the model and hurt performance.
- **Labeled property graphs (LPG)** (e.g., Neo4j, queried with **Cypher**/GQL) — both nodes *and* relationships carry key-value properties directly. Maps more naturally to real data, avoids reification, and traverses faster than relational joins or RDF workarounds (relationships are physical pointers, not re-derived), especially for many-to-many multi-hop patterns.

For agentic use cases, **property graphs are what you'll see in practice** (Neo4j, GraphRAG's internal graph index, cuGraph's node/edge model).

---

## Why graphs enable multi-hop reasoning

- **Vector RAG's failure mode** (Microsoft Research's own framing): baseline RAG "struggles to connect the dots" when an answer requires **traversing disparate pieces of information through shared attributes**, and performs poorly on **global, whole-dataset questions** ("top 5 themes in this dataset") — no single semantically-similar chunk exists to retrieve, so vector search returns noise keyed on surface words.
- **Explicit path traversal** — a drug targeting an enzyme implicated in a disease is two edges: `Drug -[TARGETS]-> Enzyme -[IMPLICATED_IN]-> Disease`, traversed directly. NVIDIA/Neo4j's **G-Retriever** benchmark used exactly this kind of multi-hop biomedical question ("what drugs target the CYP3A4 enzyme and treat strongyloidiasis?") — requires chaining relationships plus node properties, something a single vector lookup can't reliably do.
- **Global/structural questions** — GraphRAG pre-computes clusters ("communities") of closely-related entities and summarizes each. A "top themes" question becomes map-reduce over community summaries, not a similarity search — the graph's *structure* is the answer material.
- **Provenance** — retrieval is a subgraph (specific entities/relationships), so GraphRAG can cite exact entity/relationship IDs behind a claim — auditable grounding, not "here's a similar-sounding passage."

---

## Querying a KG as an agent tool

- Like calling a search API or calculator in ReAct, an agent can call a graph query language as a tool.
- **Cypher** (property graphs) — pattern-matching syntax mirroring the graph visually: `(node)-[:RELATIONSHIP]->(otherNode)`. Basic read: `MATCH (p:Person)-[:LIKES]->(t:Technology) RETURN p, t`. Labels (`:Person`) filter types; properties matched in `{}`; direction shown with `->`/`<-`, or undirected when unknown.
- **SPARQL** (RDF triple stores) — equivalent query language, common in semantic-web/linked-data contexts.
- **In an agent loop:** the LLM decides a graph query is needed, generates Cypher/SPARQL (or calls a pre-built retrieval function), the database executes it, the returned subgraph becomes the next Observation — same tool-calling pattern as ReAct.

---

## GraphRAG: build the graph index first, then retrieve

- **GraphRAG** (Edge et al., Microsoft Research, arXiv:2404.16130) targets exactly the failure mode above: a "what are the main themes" question is actually **query-focused summarization** over an entire corpus, not retrieval.
- **Two stages, run before any query arrives:**
  1. **Build an entity knowledge graph** from source documents — an LLM extracts entities and relationships from text chunks.
  2. **Pregenerate community summaries** — bottom-up clustering (graph community detection) groups related entities; the LLM writes a summary for each community, hierarchically, at multiple abstraction levels.
- **At query time:** each relevant community summary generates a partial response; partial responses get summarized again into one final answer — map-reduce over graph structure, not raw chunks.
- **Results:** substantial gains over conventional RAG on **comprehensiveness and diversity** for "global sensemaking" questions over corpora in the 1M-token range — note this is the specific problem class GraphRAG targets, not a universal RAG replacement.
- **Worked example** (Microsoft Research blog): "What is Novorossiya?" / "What has Novorossiya done?" — baseline RAG fails the second, connect-the-dots question because no single chunk mentions the entity in that context; GraphRAG answers by walking the extracted graph and cites specific entity/relationship IDs as evidence.

---

## Construction: entity/relation extraction, and why it's finicky

- Per NVIDIA's technical blog: an LLM is prompted to output structured triples — `(subject, subject_type, relation, object, object_type)` — from each chunk, constrained to a defined schema of entity types (ORG, PERSON, GPE, PRODUCT, EVENT, etc.) and relation verbs.
- Three things determine whether it works:
  - **Schema/ontology definition** — relation types and entity categories must be constrained up front, or extraction is inconsistent.
  - **Entity consistency (disambiguation)** — "America," "USA," "US," "United States" must resolve to one node, or the graph fragments into duplicates.
  - **Enforced structured output** — JSON mode, function calling, or post-processing/re-prompting to force parseable triples; malformed output otherwise breaks the pipeline.
- **NVIDIA's own experiment:** triplet-extraction accuracy jumped from **54%** (Llama-2-70B, zero-shot) to **98%** (LoRA-fine-tuned Llama-3-8B + prompt "tricks") on a 100-document internal news test set — a specific benchmark on a specific dataset, not a guarantee any KG-construction pipeline hits 98%.

---

## NVIDIA-native tooling for graphs

- **cuGraph (RAPIDS)** — GPU-accelerated graph analytics; drop-in NetworkX backend (same code, GPU-accelerated), scales to multi-billion-edge graphs across multi-GPU systems; ships shortest path, PageRank, community detection (Louvain/Leiden). In a KG-RAG pipeline, cuGraph makes multi-hop search and community-summary generation tractable at scale.
- **NeMo Framework + NIM microservices** — used in NVIDIA's reference pipeline to run and LoRA-fine-tune the extraction LLM, then serve it via TensorRT-LLM-optimized NIM endpoints.
- **PyTorch Geometric (PyG) + GNNs** — a step beyond symbolic traversal: NVIDIA/Neo4j's **G-Retriever** represents domain knowledge as a graph, retrieves a relevant subgraph via similarity search + the Prize-Collecting Steiner Tree algorithm, then fine-tunes a GNN (GAT) jointly with an LLM for multi-hop questions over that subgraph. **STaRK-Prime biomedical benchmark:** more than doubled hits@1 vs. baseline (**32% vs. ~15%**) — one benchmark, not a universal multiplier.
- **VectorRAG vs. GraphRAG vs. HybridRAG** — NVIDIA's comparative evaluation (graded by a Nemotron reward model) found GraphRAG led on correctness/quality on a dataset synthetically built for multi-hop reasoning, while noting **HybridRAG** (vector + graph combined) would likely win in most real-world scenarios — a hype-tempering admission from a vendor blog.

---

## Trade-offs and pitfalls

- **Construction cost and brittleness** — extraction runs an LLM over every corpus chunk; malformed triples, missing entities, inconsistent relation naming are the norm before tuning, not the exception.
- **Entity resolution overhead** — without disambiguation, the same real-world entity becomes multiple nodes, silently degrading traversal and retrieval.
- **Schema/ontology maintenance** — an over-engineered ontology up front makes the graph brittle to new data; Neo4j's advice: start simple, evolve the schema as needs change.
- **Staleness** — GraphRAG's community summaries are pre-computed *before* query time; new documents require re-running extraction and re-clustering, not a cheap incremental update. NVIDIA lists "dynamic information updates" as unsolved future work.
- **RDF-specific cost** — triple-store workarounds (reification, singleton properties) for anything beyond a bare fact add complexity, hurt query performance vs. property graphs.
- **When vector RAG is simpler and sufficient** — single-hop lookup, paraphrase/semantic-similarity search, or small corpora with no "connect the dots" or global-summarization need. A graph adds real engineering cost not justified unless multi-hop/relational/global reasoning is actually required.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Knowledge graph | Nodes (entities) + relationships (edges), both optionally with properties, plus an organizing principle (schema/ontology) — built from triples |
| Property graph vs. RDF | Property graphs (Cypher/GQL) let relationships carry properties natively; RDF triple stores (SPARQL) are pure subject-predicate-object and need reification for the same thing |
| Why multi-hop works on graphs | Relationships are explicit, stored data — traversal follows real edges instead of hoping vector search surfaces a chunk mentioning both ends of the chain |
| GraphRAG | LLM builds an entity graph + hierarchical community summaries *before* query time; answers use map-reduce over community summaries, not raw-chunk retrieval |
| Agent query interface | Cypher (property graph) or SPARQL (RDF), invoked as a tool in a ReAct-style loop, same as any other external system call |
| Construction | LLM-driven entity/relation extraction; needs schema definition, entity disambiguation, enforced structured output (JSON mode / re-prompting) |
| NVIDIA angle | cuGraph = GPU-accelerated traversal/community-detection at scale; NeMo/NIM = extraction-model serving; PyG/GNN (G-Retriever) = learned multi-hop retrieval over subgraphs |
| When to skip it | Single-hop lookup or small corpora — plain vector RAG is cheaper and sufficient; add a graph only when relational/global reasoning is the actual requirement |

**KG/GraphRAG vs. plain vector RAG**

| | Vector RAG | KG / GraphRAG |
|---|---|---|
| Retrieval unit | Top-k semantically similar chunks | Traversed paths / retrieved subgraph / community summaries |
| Strong at | Single-hop fact lookup, paraphrase matching | Multi-hop, relational "connect the dots," global/whole-dataset summarization |
| Weak at | Chains of facts spread across documents, "what are the themes" questions | Construction cost, extraction errors, staleness, schema upkeep |
| Grounding shown | Cited passage(s) | Cited entities/relationships (explicit provenance through the graph) |

---

## Quick self-check

1. What are the three components Neo4j uses to define a knowledge graph, and how does an "organizing principle" (e.g., an ontology) differ from the instance data itself?
   **A:** Nodes (entities), relationships (edges), and an organizing principle (schema/ontology). The organizing principle is a conceptual layer defining types/structure, not itself an instance of data — it's optional and should be added only when the use case demands it.
2. A user asks an agent to "connect the dots" between two entities several relationships apart. Why does plain vector RAG likely fail here, and what specifically does a knowledge graph provide that fixes it?
   **A:** Vector RAG retrieves chunks by semantic similarity to the query, but no single chunk may mention both entities together — the connection isn't written down anywhere. A KG provides explicit, traversable edges so the agent can walk `A -[rel]-> B -[rel]-> C` directly.
3. What two structures does GraphRAG build *before* any query arrives, and how do they let it answer a "what are the top themes in this dataset" question that baseline RAG cannot?
   **A:** An entity knowledge graph (extracted by an LLM) and hierarchical community summaries (from clustering related entities). At query time, relevant community summaries are map-reduced into an answer — since the graph's clustering structure itself captures dataset-wide themes, unlike a single retrieved chunk.
4. What's the practical difference between a labeled property graph (queried with Cypher) and an RDF triple store (queried with SPARQL) when you need a relationship itself to carry a property?
   **A:** Property graphs let relationships carry key-value properties natively. RDF triple stores are pure subject-predicate-object and need workarounds (reification, singleton properties) to attach a property to a relationship, which bloats the model and hurts performance.
5. Name two real costs of building a knowledge graph (on the construction/extraction side) and one situation where you'd choose plain vector RAG instead of a KG/GraphRAG approach.
   **A:** Costs: entity disambiguation overhead and schema/ontology maintenance (or extraction brittleness/malformed triples). Choose vector RAG for single-hop fact lookup or small corpora with no connect-the-dots or global-summarization need.

---

## Sources used

- [What is a knowledge graph? — Neo4j](https://neo4j.com/blog/knowledge-graph/what-is-knowledge-graph/)
- [What is Cypher? — Neo4j Getting Started docs](https://neo4j.com/docs/getting-started/cypher/)
- [From Local to Global: A Graph RAG Approach to Query-Focused Summarization — arXiv:2404.16130](https://arxiv.org/abs/2404.16130)
- [GraphRAG: Unlocking LLM discovery on narrative private data — Microsoft Research Blog](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)
- [Insights, Techniques, and Evaluation for LLM-Driven Knowledge Graphs — NVIDIA Technical Blog](https://developer.nvidia.com/blog/insights-techniques-and-evaluation-for-llm-driven-knowledge-graphs/)
- [Boosting Q&A Accuracy with GraphRAG Using PyG and Graph Databases — NVIDIA Technical Blog](https://developer.nvidia.com/blog/boosting-qa-accuracy-with-graphrag-using-pyg-and-graph-databases/)
- [cuGraph Introduction — RAPIDS cugraph-docs](https://docs.rapids.ai/api/cugraph/stable/basics/cugraph_intro/)
