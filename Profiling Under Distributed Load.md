## In one line

**Profiling under distributed load** = testing a deployment under **many concurrent requests, not one**, to find the throughput/latency **knee** — because contention, queuing, and tail latency are properties of *load* that simply don't exist at concurrency 1.

---

## Why it's needed

- 3.4 (latency/throughput trade-offs) and 3.5 (profiling a single run) measure **one request against one fixed config**. Production runs behind a load balancer across **many GPU replicas absorbing many concurrent users**.
- A config that's fast in a single-request benchmark can fall apart under concurrency — **contention, queuing, and batching dynamics only exist once multiple requests are in the system at once**.
- At concurrency 1 there is no queue, no shared batch, no contention for KV-cache memory or bandwidth. So the config with the lowest single-request TTFT is often **not** the right production choice.
- **Load testing ≠ performance benchmarking** (NVIDIA's split): load testing = "can it handle real-world concurrent traffic" (capacity, autoscaling, network); performance benchmarking = "how fast is the model itself" (throughput, latency, token metrics). A single-request benchmark can only ever do the second.

---

## Metric glossary — what each one actually means

**Latency (how long you wait):**
- **TTFT — Time To First Token** — the wait from hitting enter to the **first word appearing**. The "did it freeze?" number. Includes queue time + prompt processing (prefill) + network.
- **ITL / TPOT — Inter-Token Latency / Time Per Output Token** — the **gap between each streamed word** after the first. How fast the answer "types itself out." `ITL = (e2e − TTFT) / (output_tokens − 1)`.
- **e2e latency — end-to-end** — **total time**, request → last token. `e2e = TTFT + (ITL × output tokens)`.

**Throughput (how much work gets done):**
- **System TPS** — total output **tokens/sec across the whole GPU/fleet**, all users combined. The **operator's** number (capacity, cost-per-token). `system TPS = total_output_tokens / benchmark_duration`.
- **Per-user TPS** — tokens/sec for **one** request's stream ≈ `1/ITL`. The **single user's** felt speed.
- **RPS** — how many **full requests** complete per second.
- **Goodput** — throughput counting **only requests that met your SLA** (e.g. TTFT < 300 ms). "Throughput that was actually usable." **Can fall while raw throughput rises.**

**Shape of the results:**
- **Percentiles (p50 / p90 / p95 / p99)** — p99 = "the **slowest 1%** take at least this long." **Tail latency** = the worst-case a real user hits. The **mean hides it.**

**The two knobs you turn:**
- **Concurrency** — how many requests are **in flight at the same time**.
- **Batch size** — how many requests the GPU crunches **together in one pass**.
- **Saturation point / "the knee"** — the concurrency where **more users stop adding throughput and only add latency** — the elbow of the curve, and the number a load test exists to find.

**Under the hood — the two phases that explain every metric:**
- **Prefill** — processing your **input prompt**. Runs **in parallel**, **compute (FLOPs)-heavy** → drives **TTFT**.
- **Decode / generation** — producing **output tokens one at a time**. **Sequential**, **memory-bandwidth-heavy** (reloads weights + KV cache every token) → drives **ITL**.
- **KV cache** — GPU memory holding attention state for every token so far. **Grows with context length**, and it's usually what caps how many / how long requests fit at once (and what OOMs first).

> **Rule of thumb:** **input length → prefill → TTFT.** **output length + concurrency → decode + memory → ITL & throughput.**

---

## What moves what — cause → effect

*(↑ = rises, ↓ = falls, ~ = roughly unchanged; "↑↑" = strongly)*

| You change… | TTFT | ITL (per-token) | e2e latency | System throughput | Memory / KV-cache pressure |
|---|---|---|---|---|---|
| **Longer input (ISL ↑)** | **↑↑** (more prefill) | ~ | ↑ (via TTFT) | slight ↓ | ↑ (bigger cache per request) |
| **Longer output (OSL ↑)** | ~ | ~ | **↑↑** (more tokens × ITL) | ↓ (each request holds the GPU longer) | ↑ (cache grows every step) |
| **More users (concurrency ↑)** — *below saturation* | ↑ (queuing) | ↑ (shared GPU) | ↑ | **↑↑** | ↑ |
| **More users** — *past saturation* | **↑↑↑** (queue unbounded) | ↑ | ↑↑ | **plateau / ↓** | ↑↑ (OOM risk) |
| **Larger batch size ↑** | ↑ | ↑ | ↑ | **↑↑** (amortizes weight loads) | ↑ (more caches held at once) |
| **More replicas** | ~ (until they fill) | ~ | ~ | **↑↑** (more concurrency before saturation) | — (spread across GPUs) |
| **Quantize (lower precision)** | ↓ | ↓ | ↓ | ↑ | ↓ (smaller weights + cache) |

**How to read it:**
- **Input length is a TTFT story; output length is an "ITL × token count" story.** They hit different phases (prefill vs decode), so they show up in different metrics.
- **Concurrency and batch size both buy system throughput at the cost of each individual request's latency.** That trade-off *is* the whole lesson: `System TPS ↑` and `per-user TPS ↓` move in **opposite directions** as you push load up.
- **Almost everything ↑ raises memory pressure**, and memory is usually what fails first — either a hard OOM or a silent decode slowdown (KV-cache bandwidth).

---

## Diagnostic cheat sheet — symptom → likely cause → check / fix

| Symptom | Likely cause | Check / fix |
|---|---|---|
| **TTFT high, but tokens stream fine (ITL ok)** | Prefill or **queuing** — long inputs, or concurrency > batch slots | Check ISL; check concurrency vs (max batch size × replicas); add replicas or trim the prompt |
| **Tokens stream slowly (ITL high)** | **Decode bottleneck** — KV-cache / memory-bandwidth pressure, or batch too big | Check KV-cache usage + memory bandwidth; quantize the KV cache; lower batch size |
| **System TPS flat but latency still climbing** | You're **past the saturation knee** | Stop adding concurrency; add replicas or reduce batch size |
| **Throughput went up, but users are complaining** | **Goodput down** — over-batching pushed requests past the SLA | Measure **goodput**, not raw TPS; lower batch/concurrency until requests meet the SLA |
| **Mean latency looks fine, but some users are furious** | **Tail (p99)**, amplified by agent **fan-out** | Report **p99/p95**, not the mean; reduce parallel fan-out or hedge the slow calls |
| **GPU utilization ~100% but tokens are slow** | **Memory-bandwidth-bound**, hiding behind SM utilization | Don't trust `gpu_utilization` alone; check ITL + memory-bandwidth metrics |
| **Single-GPU trace looks clean, but the cluster is slow** | **Cross-node network** — NCCL / InfiniBand congestion | Use Nsight multi-node flags (`--nccl-trace`, `--nic-metrics`), not a single-node trace |
| **Latency creeps up over a long run / won't recover after load drops** | **Memory leak, KV fragmentation, or a stuck queue** | Run a **soak test**; watch whether metrics return to baseline once load ends |
| **Outstanding requests grow without bound** | **Fixed request-rate > saturation throughput** | Switch to **concurrency-controlled** load; add capacity |

**Which tool tier answers which question:**
- Numbers (TTFT / ITL / throughput / goodput) under a load pattern → **GenAI-Perf** *(inference-server tier)*.
- "Why is this kernel slow / is the network the bottleneck?" → **Nsight Systems** *(system/GPU tier, incl. NCCL/InfiniBand)*.
- "Which node/GPU saturated, fleet-wide, over time?" → **DCGM + Prometheus/Grafana** *(cluster tier)*.
- "Where in the agent's own logic did the time go?" → **NeMo Agent Toolkit profiler** *(app tier)*.

---

## The tail-at-scale trap (why agents are extra exposed)

- **Percentiles, not means.** Dean & Barroso, *"The Tail at Scale"* (Google, CACM 2013): a server with 10 ms typical / 1 s p99 has only 1-in-100 slow requests — but **fan out to 100 such servers and wait on all of them, and 63% of requests take >1 s**, because the odds that *at least one* parallel call hits its tail grow fast with fan-out.
- Real Google measurement: root p99 = 10 ms, but p99 for *all* sub-requests in a big fan-out = 140 ms — "the slowest 5% is responsible for half of total 99th-percentile latency."
- **Why it matters for agents:** an agent firing several tool/sub-agent calls per turn **reproduces this fan-out exactly** — user-facing p99 is set by the **slowest** call, not the average. Reporting mean latency systematically understates the real experience.

---

## Load-testing methodology: finding the knee

- **Sweep concurrency N** (keep exactly N in flight) from **1 up to ~max batch size**. Throughput "saturates around the maximum batch size while latency keeps steadily increasing" — that saturation point is the **knee**, and finding it is the whole point.
- **Prefer concurrency control over fixed request-rate** for reliability testing: a fixed rate above real throughput makes "outstanding requests grow unbounded" — useful to *provoke* failure, dangerous if mistaken for a normal benchmark.
- **Use realistic ISL/OSL** for your use case — wrong shape measures the wrong bottleneck (TTFT-bound vs decode-bound scale completely differently). NVIDIA's shapes: translation (~500–2,000 both ways), generation (short in, long out), summarization (long in, short out), **reasoning (short in, very long out — O(1,000–10,000) from CoT)**.
- **Soak past warm-up** — GenAI-Perf uses a stable sliding window, excluding warm-up/cool-down; long runs also surface slow memory leaks / GPU-memory fragmentation.
- **`ignore_eos`** — force every request to generate the intended OSL so early-stopping doesn't contaminate comparisons across concurrency levels.

---

## Profiling tooling across four tiers

Pick the wrong tier and you chase a bottleneck the tool literally can't see.

1. **Application / agent** — NeMo Agent Toolkit profiler (3.5's per-run bottleneck attribution, aggregated across concurrent runs).
2. **Inference-server** — **GenAI-Perf** (concurrency/rate sweeps against OpenAI-compatible / Triton / TensorRT-LLM / NIM endpoints; reports TTFT/ITL/throughput/goodput) and **`trtllm-bench`** (throughput straight against the engine).
3. **System / GPU** — **Nsight Systems** (system-wide timeline: CPU threads, CUDA calls, GPU kernels, memory; projects NVTX regions onto the GPU timeline). `TLLM_PROFILE_START_STOP=A-B` captures a chosen iteration slice so the trace stays analyzable. Multi-node flags: `--nccl-trace`, `--mpi-impl`, `--nic-metrics`, `--ib-switch-congestion-devices` — cross-node network is a first-class target.
4. **Cluster telemetry** — **DCGM Exporter** (GPU power/util/memory/temp/clock on `/metrics`, port 9400) → **Prometheus** (`kube-prometheus-stack`, scrapes fleet-wide via K8s service discovery) → **Grafana** (standard dashboard `12239`) so a saturation event traces to *which* node/GPU.

---

## Reliability under load

- **Saturation cliff** — system TPS rises with concurrency until GPU compute / max batch size is exhausted, then plateaus and "will possibly decrease," while per-request latency keeps climbing (everything queues).
- **Unbounded queue growth** — fixed rate > saturation throughput → outstanding requests grow without bound → TTFT rises until something breaks (client timeout, OOM, operator restart).
- **Tail amplification under fan-out** — see the tail-at-scale section; checking only the *average* agent turn misses it.
- **Failure + recovery** — a saturated system degrades (rising tail) then fails hard (timeouts, dropped connections, OOM). Reliability testing tracks the **failed/timed-out rate as concurrency rises**, and whether the system **recovers cleanly once load drops** (vs staying degraded — a leak/stuck-queue/fragmentation tell).

---

## Trade-offs & pitfalls (beyond the cheat sheet)

- **Vendor numbers don't transfer.** DGX Cloud Benchmarking's headlines (e.g. 97% training-time cut for +2.6% cost scaling Llama 3 70B) are NVIDIA's own numbers, on NVIDIA's own stack, for one model/precision/framework — evidence the *mechanism* works, not a number that transfers to your model/ISL-OSL/network.
- **Benchmarks are only comparable if identical.** GenAI-Perf's own docs note other tools (e.g. LLMPerf) define TTFT/ITL/TPS with subtly different formulas — a number is only comparable to another from the same tool, definitions, and load pattern.

---

## NVIDIA angle (read critically)

- **Genuinely integrated stack:** GenAI-Perf + `trtllm-bench` (sweeps), Nsight Systems (system/GPU/network, native NCCL/InfiniBand), DCGM → Prometheus/Grafana (always-on cluster telemetry). Real value: goodput as a first-class flag, percentile reporting on by default, iteration-scoped Nsight capture.
- **Read the numbers more skeptically than the tooling:** DGX Cloud Benchmarking is framed around *training* throughput/TCO, not concurrent-inference load — good for the general "communication overhead grows with scale" principle, not itself an inference load test.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Why load-profiling differs | Queuing, batch-slot contention, tail-latency distributions are **emergent** — invisible at concurrency 1, dominant near saturation |
| Load testing vs perf benchmarking | Load testing = capacity/autoscaling/network under concurrency; benchmarking = the model's own throughput/latency/token metrics |
| Input vs output length | Input (ISL) → prefill → **TTFT**; output (OSL) → decode → **ITL × token count → e2e** |
| Concurrency / batch trade-off | Both raise **system throughput** but raise **per-request latency**; System TPS and per-user TPS move in opposite directions |
| Goodput vs throughput | Goodput = throughput meeting an SLA; a system can raise throughput (over-batching) while goodput **falls** |
| Percentiles vs mean | p99, not average, is the real user experience; tail amplifies fast under agent fan-out (63% >1s from 100 parallel calls) |
| Methodology | Sweep concurrency 1→~max batch to find the **knee**; prefer concurrency over unbounded request-rate; realistic ISL/OSL; soak past warm-up |
| Tooling tiers | Agent (NAT profiler) → inference-server (GenAI-Perf) → system/GPU (Nsight, incl. NCCL/InfiniBand) → cluster (DCGM+Prometheus+Grafana) |
| Reliability | Saturation → latency cliff + throughput plateau; unbounded rate → unbounded queue → OOM/timeouts; recovery-after-load matters |
| Biggest pitfalls | Wrong ISL/OSL; mean-only reporting; single-node profiling of a cross-node bottleneck; GPU-util hiding a memory-bandwidth bound; vendor numbers not transferring |

---

## Quick self-check

1. Why can a config that looks optimal in a single-request benchmark be the wrong choice under real concurrency? Name two effects that only emerge under load.
   **A:** Contention/queuing/tail effects are **emergent** — invisible at concurrency 1. Two examples: **queuing** (TTFT rises once concurrency exceeds what the engine serves immediately) and **batch-slot contention** (requests wait once concurrency exceeds max batch size × replicas).
2. What is goodput, and how can raw throughput rise while goodput falls?
   **A:** Goodput = throughput counting only requests that met an SLA (e.g. `request_latency:300`). Over-batching raises raw throughput but pushes more requests past the SLA, so goodput falls.
3. Using "The Tail at Scale," why is a fan-out agent especially exposed to tail latency even if each call rarely runs slow?
   **A:** Waiting on the slowest of many parallel calls makes the odds that *at least one* hits its tail grow fast with fan-out — 63% of requests exceed 1 s when fanning out to 100 servers each with a 1% chance of a 1 s response. An agent's parallel tool calls reproduce this, so its p99 is set by the slowest call, not the average.
4. Name the four profiling tiers, one NVIDIA tool each, and what each sees that the others can't.
   **A:** App/agent (NeMo Agent Toolkit profiler) — time inside the agent's logic. Inference-server (GenAI-Perf) — what the serving layer delivers under a load pattern. System/GPU (Nsight Systems) — kernel/network-level hardware activity. Cluster (DCGM + Prometheus/Grafana) — the same GPU metrics aggregated fleet-wide over time.
5. What is the saturation point, what happens to throughput vs latency on either side, and why prefer concurrency control over a fixed request rate for reliability?
   **A:** It's where system throughput stops rising with added concurrency (plateaus/declines) while per-request latency keeps climbing (extra requests queue). Prefer concurrency control because a fixed rate above real throughput lets the queue grow unbounded — good for provoking failure, dangerous if mistaken for a normal benchmark.

---

## Sources used

- [LLM Inference Benchmarking: Fundamental Concepts — NVIDIA Technical Blog](https://developer.nvidia.com/blog/llm-benchmarking-fundamental-concepts/)
- [GenAI-Perf — NVIDIA Triton Inference Server documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/genai-perf/README.html)
- [Performance Analysis (Nsight Systems integration) — TensorRT-LLM documentation](https://nvidia.github.io/TensorRT-LLM/performance/perf-analysis.html)
- [User Guide — NVIDIA Nsight Systems documentation](https://docs.nvidia.com/nsight-systems/UserGuide/index.html)
- [Setting up Prometheus — NVIDIA GPU Telemetry documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/kube-prometheus.html)
- [Measure and Improve AI Workload Performance with NVIDIA DGX Cloud Benchmarking — NVIDIA Technical Blog](https://developer.nvidia.com/blog/measure-and-improve-ai-workload-performance-with-nvidia-dgx-cloud-benchmarking/)
- [The Tail at Scale — Dean & Barroso, Communications of the ACM (2013)](https://cacm.acm.org/research/the-tail-at-scale/)
