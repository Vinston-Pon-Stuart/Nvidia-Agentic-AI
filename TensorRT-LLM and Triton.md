## In one line

**TensorRT-LLM** compiles a model into a hardware- and precision-specific inference engine; **Triton Inference Server** loads that engine (or any other model artifact) and serves the actual traffic — NVIDIA's own summary: "build TensorRT engines that contain the model optimizations, then deploy the optimized models with Triton in production." **Compile, then serve.**

---

## Why it's needed / scope

- **What it explains** — every latency number an agent's user feels (time to first token, streamed response speed, concurrent-user capacity) bottoms out in these two pieces of software under whatever product (a NIM, a custom deployment) sits on top.
- **What's covered elsewhere, not here**: decoding-parameter/quantization choice and batch-size latency/throughput trade-offs (**3.4**); measuring actual throughput/TTFT/ITL under load — GenAI-Perf, percentiles, goodput (**4.3**); the queue:compute-ratio Kubernetes autoscaling pattern (**4.4**); dynamic batching as a general scalability lever (**1.8**); NIM as the product packaging this whole stack (**7.2**).

---

## Division of labor: compile once, serve many

- **TensorRT-LLM** — "an open-sourced library for optimizing LLM and Visual Gen inference," providing custom kernels (attention, GEMMs, MoE), algorithmic runtime optimizations (Prefill-Decode disaggregation, Wide Expert Parallelism, Speculative Decoding), for NVIDIA GPUs. Classic workflow: ahead-of-time compiles a model at a chosen precision into an **engine** — a serialized TensorRT "plan" tied to that model, precision, and GPU architecture. Build-time flags (`gpt_attention_plugin`, `paged_kv_cache`, `gemm_plugin`, `quantization`) are baked in, not runtime-adjustable.
- **Runtime loop** — the `PyExecutor`: **Scheduler** (which requests run this step) + **KVCacheManager** (alloc/free KV cache) + **ModelEngine** (GPU forward pass) + **Sampler** (logits → tokens via greedy/top-k/top-p/beam). Same loop whether invoked via TensorRT-LLM's Python `LLM` API or through Triton.
- **Triton** — generic server: requests arrive via HTTP/REST, gRPC, or the C API, route to a per-model scheduler, optionally batch, dispatch to "the backend corresponding to the model type." Supports TensorRT, TensorFlow, PyTorch, ONNX; runs across cloud/data center/edge/embedded. **Doesn't care how a model was produced** — a TensorRT-LLM engine is one of several artifact types it serves; **Triton is not TensorRT-LLM-exclusive**.

---

## TensorRT-LLM: compiling the optimized engine

- **Kernel fusion** — fewer, larger GPU kernels cut launch overhead and memory round-trips (FlashAttention-style: fuse layers, minimize memory reads/writes, group same-data calculations). TensorRT-LLM's custom attention/GEMM/MoE kernels apply this at build time. **CUDA Graphs** add a runtime-layer fix: capture a whole GPU-op sequence as one graph, relaunch with one API call — "up to a 22% end-to-end throughput increase on certain models and hardware" (vendor-reported).
- **Quantization** — FP8/INT8/INT4 baked into the compiled engine (`quantization`, `gemm_plugin` flags); low-precision GEMMs are themselves faster fused kernels. Accuracy trade-offs and the precision decision tree are **3.4**'s; here it's just: quantization is a build-time property, not a per-request knob.
- **In-flight (continuous) batching — the headline LLM-serving win.** Static batching waits for the whole batch to finish (every request waits for the *longest* one). In-flight batching "immediately evicts finished sequences from the batch [and] begins executing new requests while other requests are still in flight." TensorRT-LLM's Triton backend implements this in `inflight_batcher_llm` (C++). Two competing scheduler policies via `batch_scheduler_policy`: **`MAX_UTILIZATION`** (packs new requests as aggressively as KV-cache allows) vs. **`GUARANTEED_NO_EVICT`** (an admitted request won't be evicted) — predictability vs. packing density.
- **Paged KV cache** — unmanaged KV cache is over-provisioned to max sequence length and "tied up for the lifetime of the request," wasting memory to internal + external fragmentation. **PagedAttention** (credited to vLLM) partitions KV cache into fixed-size blocks stored non-contiguously, fetched via a block table — "significantly limits memory wastage, enabling larger batch sizes." Build flag: `paged_kv_cache`. Monitored live via Triton metric `nv_trt_llm_kv_cache_block_metrics` (`fraction`, `tokens_per`, `used`, `free`, `max`).
- **Tensor / pipeline parallelism (multi-GPU)** — **Tensor parallelism** splits individual layers horizontally (MLP weight matrix split across devices; attention heads split naturally). **Pipeline parallelism** splits the model vertically into per-device layer-stacks run in sequence, introducing idle "bubbles" (mitigated, not eliminated, by microbatching). Coordinated via MPI: **Leader Mode** (one Triton process per GPU, rank 0 leads — Slurm-friendly) vs. **Orchestrator Mode** (single orchestrator spawns per-GPU workers on demand — multi-model, single-node). Set via `tensor_parallel_size`/`pipeline_parallel_size`/`moe_expert_parallel_size`. Cross-GPU comms overhead is measured, not fixed, at **4.3**'s layer.
- **Speculative decoding** — parallelizes sequential autoregressive generation: a cheap draft model proposes several tokens; the target model verifies them in parallel; matching tokens are accepted, generation resumes after the first mismatch. Not one algorithm — a family: TensorRT-LLM supports **Medusa**, **ReDrafter**, **Lookahead**, **Eagle**, plus standard top-k/top-p/beam decoding.

---

## Triton: serving whatever engine you hand it

- **Backends** — extensible via a backend C API. Documented list: **TensorRT-LLM**, **vLLM**, **Python**, **PyTorch**, **ONNX Runtime**, **TensorRT**, **FIL**, **DALI** — one server process running an LLM engine, a vision model, and an ONNX preprocessing step side by side. This is the concrete scope difference from TensorRT-LLM: TensorRT-LLM optimizes one model class; Triton serves anything with a backend.
- **Dynamic batching** — combines separately-arriving requests into a batch automatically ("Creating a batch of requests typically results in increased throughput"). Knobs: `preferred_batch_size`, `max_queue_delay_microseconds` (the latency-for-throughput dial), `priority_levels`, per-priority queue timeouts. (Trade-off magnitude = **3.4**; general mechanism intro = **1.8**.)
- **Exam-relevant distinction**: dynamic batching combines whole *requests* at a queue boundary and isn't aware individual LLM requests finish at wildly different times. True LLM-style **continuous/iteration-level batching** is a different mechanism — "reusing batch slots as soon as they are free without waiting for all requests in a batch to complete." Exposed generically via `sequence_batching { iterative_sequence: true }`; the TensorRT-LLM backend's `inflight_batcher_llm` is exactly this pattern, purpose-built for LLM decoding.
- **Concurrent model execution / instance groups** — run multiple copies of a model in parallel instead of serializing on one instance. `instance_group` config sets `count`, `kind` (`KIND_GPU`/`KIND_CPU`), can pin to specific GPUs, attach a rate-limiter. **Exception**: ensembles "do not support `instance_group`... since there is no physical instance associated with an ensemble model" — composing models scale independently via their own settings.
- **Model ensembles/pipelines** — a *pipeline* of models with input/output tensors connected between them (e.g. preprocess → inference → postprocess), avoiding intermediate-tensor transfer overhead. Each `step` in `ensemble_scheduling` has an `input_map`/`output_map`; the ensemble scheduler is "event-driven... almost never the bottleneck." **Backpressure**: `max_inflight_requests` caps queued intermediate requests between a fast upstream step and a slower downstream one — "without backpressure, requests accumulate... eventually leads to out-of-memory errors."
- **The scheduler** — **Default** (spreads across instances, no batching logic), **Ensemble** (mandatory/exclusive for ensembles, routes tensors between steps), **Sequence Batcher** (pins related requests to the same instance for stateful models; its "iterative sequences" extension underlies LLM continuous batching).

---

## How they combine, and where NIM fits

- TensorRT-LLM compiles the engine (kernel fusion, quantization, paged KV cache, parallelism at build time) → placed in a Triton model repository → the **TensorRT-LLM backend** loads it, runs `inflight_batcher_llm`'s iteration-level scheduling → layered under Triton's **instance groups** and, if mixed with pre/post-processing, an **ensemble**.
- **Fully instrumented, not a black box**: `nv_trt_llm_request_metrics` (`waiting`, `context`, `scheduled`, `max`, `active`), `nv_trt_llm_inflight_batcher_metrics` (`micro_batch_id`, `generation_requests`, `total_context_tokens`).
- **Newer path**: a streamlined **PyTorch Backend (LLM API)** launches directly from a Hugging Face model ID via `model.yaml` mapping to `LLM()` constructor args (KV cache, quantization, parallelism) — blurring the classic offline-compile/online-serve line, though the same mechanisms make either path fast.
- **Downstream lessons on top of this stack**: **4.3** measures throughput/TTFT/ITL under load (GenAI-Perf, `trtllm-bench`); **4.4** turns Triton's `nv_inference_queue_duration_us`/`nv_inference_compute_infer_duration_us` into a Prometheus queue:compute ratio driving K8s HPA; **NIM (7.2)** packages the whole stack (engine + Triton or Dynamo + OpenAI-compatible API) so teams get in-flight batching and paged KV cache without hand-building any of it.

---

## Trade-offs and pitfalls

- **Engine is model-, precision-, and GPU-architecture-specific** — not portable across GPU families without rebuilding (Triton's `cc_model_filenames` exists precisely because TensorRT plans are "tied to a specific compute capability").
- **Quantization's accuracy cost is real and workload-dependent** — doesn't disappear just because it's applied at compile time (specifics = **3.4**).
- **Batching trades throughput for per-request latency** — `max_queue_delay_microseconds` and `MAX_UTILIZATION` vs. `GUARANTEED_NO_EVICT` are exactly this dial; magnitude (NVIDIA's ~14x throughput/~4x latency A100 batch-64 example) is **3.4**'s.
- **Parallelism adds cross-GPU comms overhead invisible in a single-GPU trace** — all-reduce per sharded layer (tensor parallelism), bubble idle time (pipeline parallelism); measured via Nsight Systems/NCCL tracing at **4.3**.
- **Speculative decoding only pays off with a well-matched draft model** — mismatched drafts get rejected by verification, turning "free" parallel compute into pure overhead.
- **Ensemble backpressure is a real failure mode** — a fast preprocess/retrieval step feeding a slower LLM decode step can exhaust memory without `max_inflight_requests` configured (directly relevant to RAG-style Triton ensembles).
- **Large, interacting tuning surface** — batch size, queue delay, instance count, scheduler policy, chunked-context size, KV-cache block size all interact; Triton's own docs point to its **Model Analyzer** tool for automated search rather than hand-tuning.
- **Vendor numbers are configuration-specific** — the CUDA Graph "22%" figure and various "3x"/"3.6x" speculative-decoding claims are reported on specific model/GPU/settings; treat as evidence the mechanism works, not transferable numbers.

---

## NVIDIA angle (read critically)

- **Genuine, inspectable engineering** — concrete file structures (`inflight_batcher_llm`), scheduler policies, Prometheus metrics, MPI coordination modes are installable/configurable/inspectable today, not abstractions. TensorRT-LLM is "now fully open-source, with developments moved to GitHub" (March 2025). Triton is genuinely general-purpose — serves vLLM/PyTorch/ONNX through the same server/scheduling/metrics, not a walled garden built only for TensorRT-LLM (same openness pattern as NeMo Guardrails hosting third-party safety models).
- **Read skeptically**: "up to 22%" (CUDA Graphs) and "3x"/"3.6x" (speculative decoding) are vendor-reported on NVIDIA's own chosen model/GPU/config combos — directionally consistent with the mechanism, not guaranteed to transfer. The operational complexity of the full stack (MPI multi-node, Slurm scripts, custom container images, per-compute-capability engine files) is real overhead a "build and serve in three commands" framing tends to understate.

---

## How to think about it for the exam

| Product / mechanism | What it actually does |
|---|---|
| TensorRT-LLM | Compiles a model + precision + GPU-architecture-specific inference engine (kernel fusion, quantization, parallelism, paged KV cache, speculative decoding all applied at/around build time) |
| Triton Inference Server | General-purpose server: loads engines/models from any backend, schedules and batches incoming requests, returns responses — not TensorRT-LLM-exclusive |
| Kernel fusion | Fewer, larger GPU kernels → less launch overhead and memory traffic; TensorRT-LLM custom kernels (attention/GEMM/MoE) + CUDA Graphs (up to 22% throughput, vendor-reported) |
| Quantization (FP8/INT8/INT4) | Baked into the compiled engine at build time; accuracy trade-offs and precision selection = Lesson 3.4, not re-taught here |
| In-flight (continuous) batching | Evicts finished sequences, admits new ones mid-batch instead of waiting for the whole batch — the headline LLM throughput win; TensorRT-LLM backend's `inflight_batcher_llm`; scheduler policy `MAX_UTILIZATION` vs `GUARANTEED_NO_EVICT` |
| Paged KV cache | Non-contiguous, block-based KV-cache allocation (PagedAttention-style) — eliminates fragmentation, fits more concurrent sequences; monitored via `nv_trt_llm_kv_cache_block_metrics` |
| Tensor / pipeline parallelism | Shard a too-big model across GPUs (horizontal layer-split vs. vertical stage-split); MPI Leader/Orchestrator mode coordinates it; comms overhead measured in Lesson 4.3 |
| Speculative decoding | Small draft model proposes tokens, target model verifies in parallel; TensorRT-LLM supports Medusa, ReDrafter, Lookahead, Eagle; only helps with a well-matched draft model |
| Triton backends | TensorRT-LLM, vLLM, Python, PyTorch, ONNX Runtime, TensorRT, FIL, DALI — one server, many model types |
| Dynamic batching (Triton) | Server-side batching at a request-queue boundary, tuned via `preferred_batch_size`/`max_queue_delay_microseconds`; generic, not LLM-token-aware — distinct from in-flight batching; cross-ref Lessons 1.8/3.4 |
| Concurrent model execution / instance groups | Multiple model instances per GPU (or across GPUs) run requests in parallel instead of serializing on one instance |
| Ensembles / pipelines | Chain preprocess→model→postprocess as one served unit; event-driven ensemble scheduler; `max_inflight_requests` backpressure prevents OOM on speed-mismatched steps |
| The scheduler | Default (spread across instances, no batching logic) vs. Ensemble (route tensors between pipeline steps) vs. Sequence/iterative (stateful models; also underlies continuous batching) |
| How they combine | TensorRT-LLM builds the engine → Triton's TensorRT-LLM backend loads and serves it with in-flight batching + instance groups on top; NIM (7.2) packages the whole stack behind one API |

---

## Quick self-check

1. In one sentence each, state what TensorRT-LLM does and what Triton Inference Server does, and explain why "TensorRT-LLM compiles, Triton serves" is the right way to remember the division of labor.
   **A:** TensorRT-LLM compiles a model into a hardware- and precision-specific inference engine; Triton loads that engine (or any other model artifact) and manages incoming traffic. The phrase captures that compilation happens once, offline, while serving handles ongoing, varying request traffic.
2. What is the difference between Triton's generic dynamic batching and in-flight (continuous) batching? Why is the second specifically necessary for LLM workloads and not, say, an image classifier?
   **A:** Dynamic batching groups whole requests at a queue boundary and waits for the batch to finish together. In-flight batching evicts finished sequences and admits new ones mid-batch. LLM requests finish generating at wildly different times (different output lengths), unlike a single-pass image classifier, so waiting for the "longest" request wastes GPU capacity — in-flight batching fixes that specifically.
3. Explain how paged KV cache solves a memory problem that a naively over-provisioned KV cache creates, and name the OS-inspired mechanism it borrows from.
   **A:** Unmanaged KV cache reserves space for the maximum possible sequence length per request, causing internal and external fragmentation. PagedAttention partitions the cache into fixed-size, non-contiguous blocks fetched via a block table — the same approach an OS uses for virtual memory paging.
4. A team wants to use speculative decoding to cut latency. Under what condition does it actually help, and under what condition does it become pure overhead?
   **A:** It helps when the draft model's proposed tokens closely match what the target model would generate — verification accepts most tokens. It becomes overhead when the draft diverges often, since verification rejects most speculative tokens and the extra compute buys no latency win.
5. Why can't an ensemble model itself have an `instance_group`, and how do its composing models scale instead?
   **A:** An ensemble has no physical instance of its own — it's just a pipeline definition routing tensors between other models. Each composing model scales independently via its own `instance_group` setting.

---

## Sources used

- [NVIDIA/TensorRT-LLM — README (GitHub)](https://raw.githubusercontent.com/NVIDIA/TensorRT-LLM/main/README.md)
- [Architecture Overview — TensorRT-LLM Developer Guide (GitHub)](https://raw.githubusercontent.com/NVIDIA/TensorRT-LLM/main/docs/source/developer-guide/overview.md)
- [TensorRT-LLM Backend — README (GitHub, triton-inference-server/tensorrtllm_backend)](https://raw.githubusercontent.com/triton-inference-server/tensorrtllm_backend/main/README.md)
- [Triton Architecture — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/architecture.html)
- [Model Configuration — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html)
- [Batchers — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/batcher.html)
- [Schedulers — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/scheduler.html)
- [Ensemble Models — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/ensemble_models.html)
- [Mastering LLM Techniques: Inference Optimization — NVIDIA Technical Blog](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
- [Scaling LLMs with NVIDIA Triton and NVIDIA TensorRT-LLM Using Kubernetes — NVIDIA Technical Blog](https://developer.nvidia.com/blog/scaling-llms-with-nvidia-triton-and-nvidia-tensorrt-llm-using-kubernetes/)
