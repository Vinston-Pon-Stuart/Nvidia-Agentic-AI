## In one line

Tuning model parameters means deliberately picking a point on the **accuracy/latency/cost trade-off surface** for a specific workload — not defaulting to whatever the SDK ships with — by adjusting decoding knobs (temperature, top-p/top-k, max tokens, penalties) and serving knobs (batch size, quantization, KV-cache, model size).

---

## Scope: two knob families

- **Decoding/sampling parameters** — control how a *single model instance* turns its next-token probability distribution into an actual token (temperature, top-p, top-k, max tokens, stop sequences, penalties). Shape quality/diversity, and directly bound cost/latency per request.
- **Serving/efficiency parameters** — control how *many* requests you can serve, how fast, on what hardware, at what accuracy cost (quantization, batch size, KV-cache management, model-size selection).
- The deep engine mechanics of the second family (TensorRT-LLM compilation, Triton's scheduler internals) are Domain 7 territory; this lesson stays at "which knob to turn to hit a target."

---

## Decoding parameters: shaping the next token

**Temperature**
- **What it does**: scales logits before softmax. Divide logits by T: T→0 collapses toward near-greedy decoding; T=1 samples the raw learned distribution; T>1 flattens the distribution, raising diversity — and error rate — together.
- OpenAI's API: 0–2 scale, default 1; recommends altering *either* temperature *or* top-p, not both — they act on the same distribution in overlapping ways.

**Top-p (nucleus) vs. top-k**
- Both truncate the distribution before sampling, but differently, and the difference matters.
- Holtzman et al., ["The Curious Case of Neural Text Degeneration"](https://arxiv.org/abs/1904.09751) (arXiv:1904.09751): likelihood-maximizing decoding (greedy/beam search) produces "bland and strangely repetitive" text — the highest-probability continuation is disproportionately likely to be a repetitive loop, not something a human would write.
- **Top-p (nucleus sampling)**: samples from the smallest set of tokens whose cumulative probability exceeds p — "the dynamic nucleus" — so it adapts to the distribution's shape: few candidates when the model is confident, more when genuinely uncertain.
- **Top-k**: always keeps exactly k candidates regardless of shape, which can be too small on a flat distribution (killing diversity) or too large on a peaked one (letting implausible tokens back in).
- **Why top-p is generally preferred**: its adaptivity is a direct mechanical consequence of what it truncates on (cumulative probability) versus top-k's fixed count.

**Max tokens and stop sequences**
- The most direct latency/cost levers — they bound *how much* gets generated, not the sampling distribution.
- Generation is autoregressive (one forward pass per token), typically the dominant latency contributor; a hard `max_tokens` is a direct ceiling on worst-case latency and — since output tokens often cost more per-token — a direct cost ceiling.
- Stop sequences end generation the instant a defined string appears, avoiding wasted tokens that would just be discarded.

**Frequency and presence penalties**
- Both discourage repetition by modifying logits based on generation history, differently: **frequency penalty** scales with *how many times* a token has appeared (fights word-level repetition loops); **presence penalty** applies a flat, one-time penalty to any token that's appeared at all (pushes toward new topics). OpenAI: −2.0 to 2.0 scale, compose additively.

**The reliability angle**
- For an agent that must emit a syntactically valid tool call every turn, low temperature (near 0) minimizes variance — there's one correct tool name and JSON shape, no benefit to "creative" alternatives.
- **Key caveat**: this collides with **self-consistency** (Wang et al., arXiv:2203.11171), which improves reliability by sampling *multiple diverse* reasoning paths and majority-voting — that mechanism requires temperature > 0. At temperature 0, every sample collapses to the same greedy path, so self-consistency degenerates into running the same deterministic call N times for no benefit. Low temperature and self-consistency are opposite reliability strategies; choosing which a given agent step needs is itself a tuning decision.

---

## Serving and efficiency parameters: the throughput/latency/cost surface

**Batch size**
- Trades throughput against per-request latency directly, with **no accuracy cost** — a pure scheduling decision.
- Grouping requests lets a GPU amortize the cost of loading weights across more concurrent work, since single-request autoregressive generation is largely memory-bandwidth-bound.
- NVIDIA's worked example: on a single A100, pushing batch size to 64 increased throughput **~14x** — but increased per-request latency **~4x** in the process.
- Continuous/in-flight batching, KV-cache-aware scheduling, and PagedAttention push batch size up without wasting memory, but the underlying tension doesn't disappear — the right batch size for a low-latency interactive agent isn't the right batch size for a high-throughput batch pipeline.

**Quantization**
- Reduces numeric precision of weights (and sometimes activations/KV-cache), trading accuracy for memory footprint and speed.
- NVIDIA TensorRT-LLM's decision tree: try **FP8** first (native hardware support on Hopper/Blackwell, "typically offers the best performance and accuracy"); if not enough, **INT8** (commonly via SmoothQuant, which redistributes quantization difficulty between weights and activations); only then **INT4** (AWQ, GPTQ) if more compression is needed.
- Hardware matters: A100/L40S predate hardware-native FP8, so INT8 is the practical ceiling there even if FP8 is available in software.
- Batch regime matters: weight-only quantization (INT4 AWQ) helps most at small batch sizes (memory-bound); combined weight-and-activation quantization matters more at large batch sizes (compute-bound).
- **KV-cache quantization** is a separate, complementary lever: switching FP16→FP8 KV cache lets a system run a **2–3x larger batch** on the same H100, for roughly a **1.5x** realized throughput gain.
- **Accuracy cost is not uniform across task types** — this is the exam's likeliest probe. Lotfi et al. (2026, hf.co/papers/2606.00206): aggressive quantization on reasoning models reduces accuracy *while increasing* chain-of-thought length, and in up to **52%** of quantized failures, the model reached the correct answer in its intermediate reasoning but failed to output it as the final answer. Liu et al. (hf.co/papers/2504.04823): "lossless quantization can be achieved with W8A8 or W4A16," but "lower bit-widths introduce significant accuracy risks," with **model size, model origin, and task difficulty** flagged as the critical variables — the same scheme safe on a large model or easy task can be unsafe on a small model or hard one. Quantization concentrates its damage on exactly the hard, multi-step reasoning tasks agents are often built to handle; a level validated on a general benchmark can silently fail on your agentic workload.

**Model-size selection**
- The coarsest-grained lever, decided before any of the above knobs get tuned.
- A smaller, fine-tuned/distilled model (trained on curated traces from a larger "teacher" — the data-flywheel/distillation pattern) can hit a latency/cost target a large general-purpose model can't reach at any batch size or quantization level, because it needs less memory bandwidth and fewer FLOPs per token.
- **Key caveat**: only works if the target task distribution is narrow enough for the smaller model to match teacher accuracy — a distilled model excellent at one agent's tool-calling format is not a general substitute for the model it was distilled from.

---

## The metrics you're actually tuning against

NVIDIA's LLM inference benchmarking framing, the standard set:

- **Time to first token (TTFT)** — latency to the first generated token; dominates perceived responsiveness for interactive/streaming agents.
- **Inter-token latency (ITL)** — latency between subsequent tokens; governs how fast a stream feels once it starts.
- **End-to-end latency** — TTFT plus the sum of per-token latencies; what a user or orchestrating agent actually waits on.
- **Throughput** — tokens/sec or requests/sec the *system* processes in aggregate; determines how much GPU capacity a given traffic volume needs.
- **Cost per request/token** — derived from throughput and hardware cost; the business-facing number the other four feed into.

These pull in different directions: the operator wants maximum fleet-wide throughput; the end user of any single request wants low TTFT and ITL. Pushing batch size or concurrency up raises throughput but raises individual-request latency too, tracing out a **Pareto frontier** — tuning is choosing where on that frontier to sit, not finding one setting that maximizes everything.

---

## Trade-offs and pitfalls

- **Temperature = 0 does not guarantee fully deterministic output.** GPU floating-point non-associativity and batch-size-dependent reduction kernels in fused attention/normalization mean the same prompt can produce different token probabilities across runs, because a request can be dispatched under different dynamic batch sizes on a shared server. Thinking Machines Lab's ["Defeating Nondeterminism in LLM Inference"](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/): fixing this required building batch-invariant kernels for RMSNorm, matmul, and attention — not just setting temperature to zero.
- **Over-quantizing tanks accuracy disproportionately on hard tasks, not uniformly.** Riskiest on multi-step reasoning, math, and coding; safest on conversational or easy tasks. Validating a level on the wrong benchmark hides the failure until production.
- **Maximizing throughput hurts the metric an interactive user actually feels.** The ~14x throughput/~4x latency A100 batch-size example is the clearest version — tuning purely for cost-per-token can silently degrade the latency an interactive agent needs.
- **Tuning to a benchmark, not your workload.** A config validated on a public eval suite reflects that suite's task mix and difficulty, not necessarily yours — task difficulty is a critical determinant of quantization accuracy loss, and the same caution applies to sampling parameters.
- **Chasing one metric in isolation.** A Goodhart-flavored trap: a config that wins on one axis (throughput, accuracy, TTFT) while silently regressing another (fast+cheap+wrong; accurate but unusably slow) isn't actually an improvement. Measure accuracy, latency, and cost jointly, on your own workload, before shipping a parameter change.

---

## NVIDIA angle (read critically)

- NVIDIA's stack exposes both families end to end: **TensorRT Model Optimizer** implements FP8/INT8/INT4 quantization (SmoothQuant, AWQ, GPTQ) and hardware-precision matching; **Triton Inference Server**/**NIM** continuous/dynamic batching implements the throughput/latency trade-off; NIM's KV-cache handling and FP8 KV-cache support is the memory-side lever.
- The mechanisms match independent literature (native FP8 hardware on Hopper/Blackwell, continuous batching reducing idle time, KV-cache quantization freeing memory for larger batches), but NVIDIA's headline *multipliers* are vendor-reported for a specific model/hardware configuration, not universal guarantees. Example: NVIDIA's NIM blog reports **2.5x throughput, 4x faster TTFT, 2.2x faster ITL** for its Llama 3.1 8B Instruct NIM vs. unspecified "best open-source alternatives" — treat the *existence and direction* of the gain as credible (consistent with the mechanisms), the specific multiples as marketing framing for one config, not a number that transfers to your model/hardware/traffic.
- The engine-compilation/scheduler mechanics behind these numbers (how TensorRT-LLM builds a quantized engine, how Triton's batcher dispatches requests) are Domain 7 material; here, these are just more knobs on the same accuracy/latency/cost surface as temperature and top-p, operating at the serving layer instead of the sampling layer.

---

## How to think about it for the exam

| Knob / technique | What it actually controls | Push toward reliability/accuracy | Push toward latency/cost |
|---|---|---|---|
| Temperature | Sharpness of the next-token distribution | → 0 for deterministic-ish, low-variance output (tool calls, factual answers) | Not a direct latency lever, but low temp + tight max_tokens reduces retries/re-generation cost |
| Top-p (nucleus) | Adaptive cumulative-probability cutoff | Lower p narrows the sampled set, reducing off-distribution errors | No direct latency effect; interacts with temperature, not a replacement for it |
| Top-k | Fixed-size cutoff | Adaptivity is the weakness vs. top-p (fixed k can over- or under-truncate) | Same — no direct latency effect |
| Max tokens / stop sequences | Hard ceiling on output length | Prevents runaway or truncated-mid-thought generation if set too low | Direct, near-linear latency and cost ceiling — the most direct cost lever of the family |
| Frequency / presence penalty | Logit adjustment based on repeat/appear history | Reduces repetition loops and topic-staleness | Marginal — mostly a quality lever, not a latency lever |
| Batch size | How many requests share one GPU pass | No accuracy effect | Throughput up, per-request latency up — the core serving-layer trade-off (NVIDIA A100: ~14x throughput / ~4x latency at batch 64) |
| Quantization (FP16→INT8→FP8→INT4) | Numeric precision of weights/activations/KV-cache | FP8 ≈ best accuracy retention; INT4 riskiest, esp. on hard reasoning tasks | Smaller memory footprint, faster arithmetic, larger batches fit in memory |
| KV-cache management | Memory used to avoid recomputing past tokens | Cache quantization (e.g., FP8) trades a little accuracy for cache size | Frees memory for larger batch size / longer context at fixed hardware |
| Model-size selection | Which model you deploy at all | A distilled/fine-tuned smaller model can match teacher accuracy on a narrow task | Smallest model that clears the accuracy bar is usually the cheapest, fastest option |

---

## Quick self-check

1. Why does setting temperature to 0 break the self-consistency technique (sampling N diverse reasoning paths and majority-voting), and separately, why doesn't temperature 0 even guarantee bit-for-bit identical output across runs?
   **A:** At temperature 0 every sample collapses to the same greedy path, so there's nothing diverse to vote across — self-consistency degenerates into running the same deterministic call N times. Separately, GPU floating-point non-associativity and batch-size-dependent reduction kernels mean the same prompt can still produce different token probabilities across runs, since a request can be dispatched under different dynamic batch sizes on a shared server.
2. Mechanically, what's the difference between top-k and top-p truncation, and why does Holtzman et al.'s finding about greedy/beam search degeneration motivate nucleus sampling specifically?
   **A:** Top-p samples from the smallest token set whose cumulative probability exceeds p, adapting to the distribution's shape; top-k always keeps exactly k candidates regardless of shape, which can over- or under-truncate. Holtzman et al. found greedy/beam search (likelihood-maximizing) produces bland, repetitive text, which is exactly what motivates nucleus sampling's adaptive, non-greedy alternative.
3. Using NVIDIA's batch-size-64 example on an A100 (~14x throughput, ~4x latency), explain why an interactive agent and a high-volume batch pipeline would deliberately choose different batch sizes on the same hardware.
   **A:** Batch size 64 raises throughput ~14x but raises per-request latency ~4x. An interactive agent needs low per-request latency, so it uses a smaller batch; a batch-processing pipeline cares about aggregate throughput per GPU-hour and would push batch size up even at the cost of individual-request latency.
4. Why does aggressive (e.g., INT4) quantization hurt reasoning-heavy tasks more than conversational tasks, and what does that imply about validating a quantization scheme against a benchmark that doesn't match your actual workload?
   **A:** Studies found quantization accuracy loss concentrates on hard, multi-step reasoning/math/coding tasks, with model size, model origin, and task difficulty as critical variables — conversational/easy tasks are comparatively safe. A quantization level validated on a general or easy benchmark can silently fail once applied to a harder, reasoning-heavy agentic workload.
5. Name the five core metrics (TTFT, ITL, end-to-end latency, throughput, cost per request) and explain, with an example, why optimizing hard for just one of them can produce a worse deployment overall.
   **A:** Pushing batch size up to maximize throughput raises per-request latency, which can make an interactive agent feel unusably slow even though the system is cheaper and more efficient in aggregate — the metrics trade off along a Pareto frontier, so optimizing one in isolation can silently regress another.

---

## Sources used

- [The Curious Case of Neural Text Degeneration — arXiv:1904.09751](https://arxiv.org/abs/1904.09751)
- [Quantized Reasoning Models Think They Need to Think Longer, but They Do Not — Hugging Face Papers (2606.00206)](https://hf.co/papers/2606.00206)
- [Quantization Hurts Reasoning? An Empirical Study on Quantized Reasoning Models — Hugging Face Papers (2504.04823)](https://hf.co/papers/2504.04823)
- [Self-Consistency Improves Chain of Thought Reasoning in Language Models — arXiv:2203.11171](https://arxiv.org/abs/2203.11171)
- [Mastering LLM Techniques: Inference Optimization — NVIDIA Technical Blog](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
- [LLM Inference Benchmarking: Fundamental Concepts — NVIDIA Technical Blog](https://developer.nvidia.com/blog/llm-benchmarking-fundamental-concepts/)
- [Optimizing Inference Efficiency for LLMs at Scale with NVIDIA NIM Microservices — NVIDIA Technical Blog](https://developer.nvidia.com/blog/optimizing-inference-efficiency-for-llms-at-scale-with-nvidia-nim-microservices/)
- [Optimizing LLMs for Performance and Accuracy with Post-Training Quantization — NVIDIA Technical Blog](https://developer.nvidia.com/blog/optimizing-llms-for-performance-and-accuracy-with-post-training-quantization/)
- [Speed up Inference with SOTA Quantization Techniques in TensorRT-LLM — NVIDIA TensorRT-LLM Docs](https://nvidia.github.io/TensorRT-LLM/blogs/quantization-in-TRT-LLM.html)
- [Create chat completion (API Reference) — OpenAI Platform Docs](https://platform.openai.com/docs/api-reference/chat/create)
- [Defeating Nondeterminism in LLM Inference — Thinking Machines Lab](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/)
