## In one line

A **vector database / vector index** is the layer under a RAG pipeline that finds the nearest neighbors to a query embedding among millions-to-billions of stored vectors in milliseconds, by trading a small amount of accuracy for large gains in speed and memory.

---

## Why it's needed

- Lesson 6.1 covers the RAG *pipeline* (chunking, embedding choice, dense/sparse/hybrid retrieval design). None of that is fast unless the layer underneath — the vector index — can actually find nearest neighbors quickly at scale.
- This lesson goes one level down: how approximate nearest neighbor (**ANN**) search works internally, the index structures, the metrics, the tunable parameters, and NVIDIA's GPU acceleration layer.
- Out of scope here: what to embed, how to chunk, when to rerank (→ 6.1). The question here is purely: given a query vector and a stored collection, how do you find nearest neighbors fast?

---

## Why exact search doesn't scale

- **FLAT / brute-force** = compare the query to every stored vector, compute distance to each, keep the closest k. Cost scales with N (stored vectors) × d (dimensionality) per query.
- **What it is**: the only method that *guarantees* the correct answer — which is why it's the ground-truth baseline every ANN index is measured against.
- **When it's fine**: small corpora or low query volume — FAISS's own guidance: if you're only running a handful of searches, "the index building time will not be amortized by the search time... direct computation is the most efficient option."
- **Caveat**: once a collection hits millions-to-billions of vectors and queries need single-digit-millisecond latency, brute force is off the table.
- **ANN** = build an index structure (clustering, graph, compression) that lets a query skip most of the collection and compare against a small, well-chosen subset. Milvus: ANNS "improves retrieval efficiency by sacrificing accuracy within an acceptable range."
- **Recall** = the fraction of true nearest neighbors an ANN search actually returns, measured against exact/FLAT results — the metric every configuration choice below trades against speed and memory.

---

## The trade-off triangle: recall, latency/throughput, memory

Every index decision is a point on this triangle — pick two, no setting maximizes all three:

- **FLAT** — 100% recall guaranteed, slowest search, no memory savings.
- **HNSW** — very fast, high recall, but memory-hungry (Milvus: needs "large memory resources").
- **PQ-based/quantized indexes** — small memory footprint, fast, real recall hit from compression.
- **IVF and IVF-PQ** — slide along the triangle rather than commit to one point (via `nprobe` at query time, compression params at build time).

---

## The index families

- **FLAT** — exact, no training, no parameters beyond distance metric, 100% recall. Used for small (≈million-scale or below) collections, or as the recall yardstick every ANN config is benchmarked against.
- **IVF (Inverted File)** — clusters the dataset into `nlist` groups via k-means at build time; at query time compares to the `nlist` centroids, picks the `nprobe` closest, and only searches vectors in those clusters.
  - `nprobe` up → more clusters searched → recall climbs, latency climbs too. Down → faster, but risks skipping the cluster holding the true nearest neighbor (classic failure: a vector near a cluster boundary).
  - At `nprobe = nlist`, recall converges toward exact — at which point FAISS says just use brute force directly.
  - FAISS sizing guidance for `nlist`: under 1M vectors, "K is 4·√N to 16·√N"; structured values like IVF65536/262144/1048576 (paired with an internal HNSW graph for cluster assignment) as the dataset passes 1M/10M/100M.
- **HNSW (Hierarchical Navigable Small World)** — a multi-layer graph (Malkov & Yashunin); most vectors live only in the bottom dense layer, progressively fewer in sparser upper layers (exponentially decaying probability). Search starts at the top, greedily walks toward the query, drops a layer when it can't get closer — gives logarithmic complexity instead of linear scan.
  - **M** — max edges per node per layer. Higher M = denser/more accurate graph, direct memory cost: FAISS puts per-vector memory at *(d×4 + M×2×4)* bytes — scales linearly with M.
  - **efConstruction** — candidate-list size while *building*. Higher = better graph, longer build time.
  - **efSearch / ef** — candidate-list size at *query* time; the primary recall/latency dial post-build. Milvus: "Higher ef leads to more accurate but slower search."
  - No training step needed; per FAISS, "the best option" when memory isn't binding and the dataset fits in RAM. Supports sequential inserts but **not** vector removal.
- **PQ (Product Quantization) / scalar quantization (SQ)** — compress vectors, not just the search. PQ decomposes each vector into `m` low-dimensional sub-vectors, quantizes each sub-space separately (Milvus), each finding `2^nbits` centroids via k-means — a full-precision vector becomes an `m`-byte (or `m×nbits`-bit) code. Lossy by construction (FAISS's `ProductQuantizer` API is built around measuring reconstruction error). SQ is simpler: quantize each dimension independently into fixed discrete levels — Milvus's IVF_SQ8 converts each 4-byte float to 1 byte, a 70–75% memory cut.
- **IVF-PQ** — combines both: cluster into `nlist` cells + prune with `nprobe` (like IVF), but store vectors inside each cell as PQ codes instead of full-precision floats. Milvus recommends it for "high-speed query, limited memory resources, accepts minor compromise in recall rate." NVIDIA's cuVS frames the same trade via `pq_dim` (compressed dimensionality) and `pq_bits` (4–8 bits per element); cuVS also supports **refinement** — over-fetch more neighbors than needed from the compressed index, then re-rank exactly against uncompressed originals, clawing back some accuracy.
- **DiskANN and CAGRA** — two escape hatches when a collection doesn't fit in memory at all. **DiskANN** (Microsoft Research) keeps a graph-based ANN index on SSD instead of RAM ("Disk provider, for larger than memory support"), tracing to NeurIPS'19 billion-point single-node NN search. **CAGRA** (NVIDIA, part of open-source cuVS) builds and searches a k-NN graph entirely on GPU, optimized for single/small-batch interactive queries. Both are graph-based like HNSW but solve different constraints: DiskANN removes the RAM ceiling, CAGRA removes the CPU throughput ceiling.

---

## Distance metrics: match what the embeddings were trained on

- Three metrics matter for dense float embeddings (Milvus): **Euclidean/L2** (straight-line distance, unnormalized continuous data), **inner product (IP)** (sum of elementwise products, sensitive to magnitude *and* angle), **cosine similarity** (angle only, magnitude-independent, range [-1, 1]).
- **Load-bearing fact**: "If you apply the IP distance metric to normalized embeddings, the result will be equivalent to calculating the cosine similarity between the embeddings" (Milvus) — IP and cosine are the same computation once vectors are unit-normalized, which is why many embedding models/vector-DB defaults ship with implicit normalization.
- **Key caveat — fails silently, not with an error**: Milvus's own FAQ for why a top-1 IP-search result isn't the query vector itself boils down to un-normalized vectors. The metric must match how the embedding model was trained/evaluated (most modern text-embedding models are optimized under cosine similarity) — a mismatch doesn't crash, it just quietly returns worse or badly wrong neighbors.

---

## Configuring an index in practice

1. **Pick the index family** from dataset size, recall target, memory budget. FAISS's decision tree: need guaranteed-exact or only run a handful of searches → Flat. Memory not constrained, small-to-medium dataset → HNSW. Moderate memory concern → cluster first, uncompressed inside clusters (IVF+Flat). Serious memory constraint → add PQ compression on top of clustering (IVF-PQ), tuning `m`/`nbits` (or cuVS's `pq_dim`/`pq_bits`) against tolerable accuracy loss. Milvus's scenario table matches: FLAT for "requires 100% recall," HNSW for "very high-speed query... large memory resources," IVF_PQ for "high-speed query, limited memory resources, accepts minor compromise in recall rate."
2. **Tune the recall/latency dial empirically**, against a representative query set. HNSW's `efSearch` and IVF's `nprobe` move recall and latency together — the right value is workload-specific, not a documentation default (Milvus defaults: IVF `nprobe`=8 against `nlist`=128; HNSW requires `ef` ≥ `top_k` — starting points only). Reliable method: sweep the parameter against a held-out query set with known (or FLAT-computed) ground truth, plot recall vs. latency, pick the point meeting the latency budget at highest available recall.

---

## GPU-accelerated vector search

- Two things get faster on GPU: brute-force search (embarrassingly data-parallel), and ANN index build/search (clustering, quantization, graph traversal all parallelize well).
- **FAISS's GPU paper** (Johnson, Douze & Jégou) is the foundational reference: identifies **k-selection** (finding the top-k smallest distances among candidates) as the real GPU bottleneck, proposes a design reaching "up to 55% of theoretical peak performance," yielding a NN implementation "8.5x faster than prior GPU state of the art." Their implementation built an accurate k-NN graph over 95M images in 35 minutes, and connected 1B vectors "in less than 12 hours on 4 Maxwell Titan X GPUs."
- **NVIDIA's cuVS** (open source, part of RAPIDS) is the current-gen continuation, shipping three index types: **IVF-Flat** (same `n_lists`/`n_probes` as CPU IVF, GPU-resident, so the full index must fit in GPU memory), **IVF-PQ** (adds `pq_dim`/`pq_bits` compression + the optional exact-refinement pass), **CAGRA** (GPU-native graph tuned for single/small-batch queries; `graph_degree`/`intermediate_graph_degree` control density, built on an initial k-NN graph from IVF-PQ; `itopk_size`/`search_width` control the query-time recall/speed trade-off — functionally like HNSW's `ef`).
- NVIDIA's benchmark (DEEP-100M dataset, H100 GPU vs. Intel Xeon Platinum 8480CL running HNSW): cuVS-based indexes showed higher QPS than CPU HNSW at matched recall for single queries, gap widening sharply with batch size — at 10,000-query batches, CAGRA "outperforms all other indexes by far."
- Milvus, FAISS, and other vector DBs can opt into cuVS as a GPU backend rather than reimplementing GPU kernels.

---

## NVIDIA angle (read critically)

- cuVS and CAGRA are real, actively developed open-source projects (`rapidsai/cuvs` on GitHub) — the parameter docs match what actually ships.
- **Caveat**: every performance number above (55% of theoretical peak k-selection, 8.5x faster, DEEP-100M throughput comparisons) is vendor-reported, on hardware/datasets NVIDIA (or Meta/FAISS in 2017) chose, not independently reproduced here.
- GPU acceleration only pays off once query volume and batch size amortize GPU memory/transfer costs — NVIDIA's own charts show the CPU/GPU gap is small at single-query workloads and only becomes dramatic at large batch sizes. Meaningful caveat for any application with mostly single interactive queries rather than large batches.

---

## Trade-offs and pitfalls

- **Over-tuning recall kills latency, and vice versa** — pushing `nprobe`/`efSearch`/`itopk_size` up chases recall at a real, often steep, latency cost; no setting is simultaneously fast and exhaustive.
- **Wrong distance metric silently wrecks results** — IP on unnormalized vectors, or a metric the embedding model wasn't trained under, doesn't error, it returns confidently wrong neighbors (Milvus's IP-normalization FAQ documents this).
- **Build time/memory is a separate trade-off from query speed** — higher `efConstruction`/`M` (HNSW) or `nlist` (IVF) buys better search quality but costs real build time and, for `M` specifically, higher per-vector memory.
- **Quantization always costs some accuracy** — compression is lossy by construction; IVF_SQ8's 70–75% memory savings come from the same trade.
- **Inserts and updates aren't free** — HNSW supports sequential inserts but not removals; IVF/IVF-PQ centroids are learned from a training sample and drift stale as new data arrives outside the original distribution, degrading recall until retrained.
- **Benchmarking on the wrong query distribution gives false confidence** — recall measured against synthetic or off-distribution queries doesn't predict recall on real production queries.
- **Vendor throughput numbers don't transfer** — NVIDIA's, FAISS's, Milvus's published benchmarks run on specific hardware/datasets (DEEP-100M, SIFT1B); treat every number here as directional, not a guarantee elsewhere.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Exact vs. approximate NN | FLAT/brute-force = exhaustive scan, cost scales with N×d, 100% recall, doesn't scale; ANN trades a small, controllable recall loss for large speed/memory gains |
| The trade-off triangle | Recall, latency/throughput, and memory footprint — no index or parameter setting maximizes all three at once |
| IVF | Cluster into `nlist` cells at build time; search the `nprobe` closest cells at query time — nprobe up = recall up, speed down |
| HNSW | Multi-layer navigable small-world graph; `M` = graph degree (accuracy vs. memory), `efConstruction` = build-time quality, `efSearch`/`ef` = query-time recall/latency dial |
| PQ / quantization | Compress vectors into short codes (product quantization) or per-dimension levels (scalar quantization) to cut memory at a real recall cost |
| IVF-PQ | IVF's cluster pruning + PQ's compression combined — fast *and* memory-efficient, at the largest recall compromise of the mainstream options |
| DiskANN / CAGRA | Two different scale-out answers: DiskANN keeps a graph index on SSD past the RAM ceiling; CAGRA is NVIDIA's GPU-native graph index (cuVS), tuned for high-throughput/low-latency batch search |
| Distance metric | Must match how the embeddings were trained (cosine ≈ normalized inner product); wrong metric fails silently, not with an error |
| GPU acceleration | FAISS-GPU and NVIDIA cuVS (IVF-Flat, IVF-PQ, CAGRA) parallelize both brute-force k-selection and ANN index build/search; gains are largest at high batch size, and published numbers are vendor-reported |

---

## Quick self-check

1. Why doesn't exact (FLAT/brute-force) nearest-neighbor search scale to large collections, and under what circumstances is it still the right choice?
   **A:** Cost scales with N×d per query, so every stored vector adds fixed work to every future search — infeasible at millions-to-billions of vectors with millisecond latency targets. It's still right for small corpora, low query volume (per FAISS, when index-build time can't be amortized), or whenever a guaranteed-exact answer is required.
2. Explain the recall–latency–memory trade-off triangle using two specific index types as examples of different points on it.
   **A:** FLAT sits at 100% recall with no memory savings and the slowest search. HNSW sits at high recall/high speed but is memory-hungry. No index or parameter setting maximizes all three at once — you pick two.
3. What do HNSW's `M`, `efConstruction`, and `efSearch` each control, and which one is the primary dial for trading recall against query latency after the index is already built?
   **A:** `M` = max edges per node per layer (accuracy vs. memory, fixed at build). `efConstruction` = candidate-list size while building (graph quality vs. build time). `efSearch`/`ef` = candidate-list size at query time — the primary post-build recall/latency dial.
4. A production RAG system starts returning irrelevant top-1 results right after switching embedding models, even though the index configuration didn't change. What's the most likely distance-metric-related cause, and why does this fail silently instead of erroring out?
   **A:** Most likely the new embedding model's outputs aren't normalized the way the index's distance metric (e.g., IP) expects, or the model wasn't trained/evaluated under that metric (e.g., it expects cosine). It fails silently because IP/cosine mismatch doesn't throw an error — the search still returns *a* nearest neighbor, just a wrong one.
5. What problem does IVF-PQ solve that plain IVF doesn't, and what does it give up to solve it? Separately, what's the difference between what DiskANN and NVIDIA's CAGRA each do to scale past a single machine's RAM?
   **A:** IVF-PQ adds PQ compression on top of IVF's cluster pruning, cutting memory footprint further (Milvus's smallest index file among these) at the cost of additional recall loss from quantization. DiskANN keeps a graph index on SSD to remove the RAM ceiling; CAGRA builds/searches the graph entirely on GPU to remove the CPU throughput ceiling — different constraints, both graph-based like HNSW.

---

## Sources used

- [Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs — arXiv:1603.09320](https://arxiv.org/abs/1603.09320)
- [Billion-scale similarity search with GPUs — arXiv:1702.08734](https://arxiv.org/abs/1702.08734)
- [In-memory Index — Milvus Documentation](https://milvus.io/docs/index.md)
- [Similarity Metrics — Milvus Documentation](https://milvus.io/docs/metric.md)
- [Guidelines to choose an index — FAISS Wiki](https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index)
- [Faiss building blocks: clustering, PCA, quantization — FAISS Wiki](https://github.com/facebookresearch/faiss/wiki/Faiss-building-blocks:-clustering,-PCA,-quantization)
- [Accelerating Vector Search: Fine-Tuning GPU Index Algorithms — NVIDIA Technical Blog](https://developer.nvidia.com/blog/accelerating-vector-search-fine-tuning-gpu-index-algorithms/)
- [microsoft/DiskANN — GitHub](https://github.com/microsoft/DiskANN)
