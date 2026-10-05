## In one line

**Data quality/augmentation/preprocessing** is the discipline of making a RAG corpus or fine-tuning set clean, deduplicated, PII-safe, and — where thin — deliberately filled in, because retrieval and fine-tuning are only as trustworthy as the data underneath them.

---

## Why it's needed

- An agent's retrieval mechanics and fine-tuning recipe can be flawless and it will still fail if the corpus is wrong, stale, duplicated, or leaks PII — a RAG answer is only as trustworthy as the document it was retrieved from.
- **Scope discipline** — three neighboring lessons look similar and aren't:
  - **3.3** = collecting/structuring *user feedback* (ratings, corrections) for RLHF/DPO — raw material is human judgments about model outputs.
  - **This lesson** = the underlying knowledge corpus or training set itself, before any feedback loop touches it.
  - **6.3** = ETL/ingestion *pipeline structure* (extraction, chunking, loading); this lesson is what determines whether what flows through that pipeline is trustworthy.
  - Toxicity/bias as an ongoing *safety* policy concern = Domain 9. Here, filtering unsafe content is a one-time data-hygiene step at ingestion.
- **Stakes**: per IBM/Forrester, over a quarter of data/analytics professionals say poor data quality costs their org more than $5M/year, 7% report $25M+ losses — and IBM projects agentic AI workflows increasing eightfold by 2026, multiplying today's data-quality debt across far more automated decisions.

---

## The data-quality dimensions

- Traces to Wang & Strong's 1996 paper (originally 15 dimensions); practice has converged on **six core dimensions** (per IBM):
  - **Accuracy** — does data correctly represent the real-world entity/event? *Check*: cross-reference against a trusted source, periodic audits.
  - **Completeness** — are required fields/values present? *Check*: null/missing-value scans, required-field coverage.
  - **Consistency** — uniform, no contradictions across systems? *Check*: cross-system reconciliation, schema validation.
  - **Timeliness** — available when needed, reflects current state? *Check*: freshness/staleness checks, refresh-cadence SLA monitoring.
  - **Validity** — conforms to defined format/business rule? *Check*: schema/format validation, range/type checks.
  - **Uniqueness** — no duplication? *Check*: exact hash matching to near-duplicate similarity search.
- These map directly onto RAG/fine-tuning failure modes: an accuracy failure = a wrong fact the agent confidently repeats; a completeness failure = missing document categories users will ask about; a timeliness failure = a corpus still reflecting last year's policy; a **uniqueness failure** (duplication) gets the most attention below because for LLMs specifically it changes model behavior, not just storage.

---

## Preprocessing and cleaning

- Basic normalization comes first. NVIDIA's NeMo Curator: `DocumentModifier` fixes inconsistent quotation marks; `UnicodeReformatter` (built on `ftfy`) resolves encoding/mojibake issues — a validity-dimension problem that, left unfixed, degrades every downstream embedding or training example.
- At scale, NVIDIA's **Nemotron-CC** pipeline (now merged into NeMo Curator) applies to Common Crawl: **HTML-to-text extraction** (jusText, strips markup/boilerplate) → **language ID** (FastText) → **28 heuristic filters** (non-alphanumeric ratios, numerical/URL ratios, whitespace inconsistencies, repeating n-grams) → **perplexity filter** (pretrained KenLM models — low perplexity = fluent/predictable text, high = strange/incoherent, discard). NeMo Curator's TinyStories tutorial shows the toy-scale version: `WordCountFilter` + `IncompleteStoryFilter` chained via a `Sequential` pipeline.
- **Why heuristics aren't enough**: Nemotron-CC team: "traditional data curation methods are limited due to the heavy use of heuristic filtering that can't assess semantic quality." Their fix: **model-based quality labeling** — an ensemble of three classifiers (a FastText quality classifier + two FineWeb-Edu-style classifiers), each scoring documents on a 0–19 scale, combined by taking the max score, bucketed into five quality tiers. Heuristics catch mechanical noise; classifiers catch content that's mechanically clean but substantively low-value.

---

## Deduplication: why it specifically matters for LLMs

- **Uniqueness** in practice — but for LLMs it's more than storage optimization. Lee et al., "Deduplicating Training Data Makes Language Models Better" (arXiv:2107.06499, ACL 2022): "existing language modeling datasets contain many near-duplicate examples and long repetitive substrings," and as a result "over 1% of the unprompted output of language models... is copied verbatim from the training data." One 61-word sentence appeared over 60,000 times in C4 before their tooling caught it.
- Deduplicating let them "train models that emit memorized text ten times less frequently and require fewer train steps to achieve the same or better accuracy."
- **Evaluation validity**: dedup "reduce[s] train-test overlap, which affects over 4% of the validation set of standard datasets" — undeduplicated benchmarks can make a model look better than it actually generalizes, because it's effectively already seen part of the test set.
- Two tiers, very different cost profiles:
  - **Exact deduplication** — hashes each full document (NeMo Curator's `ExactDuplicates`, MD5, GPU-accelerated via RAPIDS cuDF), drops all but one copy per hash. Cheap, parallelizable, blind to anything differing by even one character.
  - **Near-duplicate/fuzzy deduplication** (Nemotron-CC) — MinHash signatures + locality-sensitive hashing (LSH) to find documents whose Jaccard similarity exceeds a threshold — catches templated pages, syndicated articles, boilerplate-with-minor-edits. Meaningfully higher compute cost (approximate nearest-neighbor search, not a hash lookup).

---

## PII and unsafe-content filtering

- The same discipline 3.3 applies to feedback logs applies here — arguably higher stakes — because once sensitive text is embedded into a shared index or baked into fine-tuned weights, removing it is far harder than catching it at ingestion.
- NVIDIA's NeMo Curator ships a `PiiModifier`, built on Microsoft's open-source **Presidio** framework: detects configurable entity types (e.g., `PERSON`), applies a configurable action (replace, redact) in batches — same underlying tool as 3.3's feedback-log scrubbing, applied here to RAG documents and training corpora.
- Dropping toxic/unsafe content is the parallel step for content *type* rather than content *sensitivity*. Deeper bias/toxicity-as-ongoing-safety-policy treatment → Domain 9; here it's narrowly a pipeline stage that runs before ingestion, not an afterthought.

---

## Data augmentation: filling coverage gaps

- Cleaning/filtering only work with data you already have. When a corpus has real gaps (rare edge cases, underrepresented languages/domains, too few examples of a failure mode), **augmentation** manufactures more.
- **Classic NLP augmentation** (Feng et al. survey, arXiv:2105.03075, ACL Findings 2021) predates LLM-generated synthetic data: back-translation (translate then translate back for a natural paraphrase), rule-based token-level perturbations — exist because text's discrete nature makes augmentation harder than in vision (can't just rotate/crop and trust the label holds).
- **LLM-based synthetic data generation** is the newer, more powerful tool for training-set-scale gaps:
  - **Nemotron-4 340B**: Instruct model generates synthetic text; a companion Reward model (scores 5 attributes — helpfulness, correctness, coherence, complexity, verbosity; ranked #1 on Hugging Face RewardBench at release) filters generated data for quality — an explicit generate-then-grade loop.
  - **Nemotron-CC**: two synthetic strategies by document quality. **Low-quality docs** rewritten with a Wikipedia-style prompt to recover useful info heuristic filtering would discard — restores "up to 90 percent" of content eliminated by filtering in datasets like DCLM/FineWebEdu. **High-quality docs** run through four LLM prompts generating diverse QA pairs, condensed passages, extracted-knowledge rewrites, structured knowledge lists — adds ~2 trillion synthetic tokens on top of the filtered real corpus.
  - Reported results (NVIDIA self-benchmarked against DCLM/FineWebEdu/stock Llama 3.1, vendor-chosen baselines, not independently replicated): **5.6-point MMLU improvement** for an 8B model on 1T Nemotron-CC tokens vs. DCLM; **5-point MMLU boost** from long-horizon (15T-token) pretraining incorporating Nemotron-CC vs. stock Llama 3.1.
- **The risk: model collapse.** Shumailov et al., "The Curse of Recursion" (arXiv:2305.17493): training on model-generated content "causes irreversible defects in the resulting models, where tails of the original content distribution disappear" — **Model Collapse**, demonstrated across VAEs, GMMs, and LLMs. Warning: as LLM output increasingly populates the open web future models train on, "the value of data collected about genuine human interactions... will be increasingly valuable" precisely because generation-on-generation training erodes it.
- **The complication**: Gerstgrasser et al., "Is Model Collapse Inevitable?" (arXiv:2404.01413) confirms collapse when synthetic data *replaces* real data generation-over-generation, but shows **accumulating** successive generations of synthetic data alongside the original real data avoids collapse — test error stays bounded rather than climbing without limit, across model sizes/architectures/hyperparameters and across LLMs/diffusion models/VAEs.
- **Practical upshot**: both Nemotron-4's reward-filtered generation and Nemotron-CC's rephrase-real-documents approach are structurally closer to the safe "accumulate alongside real data" pattern than to "replace with pure model output" — synthetic data anchored to real source text, or graded by a human-trained model, is the more defensible version.

---

## Trade-offs and pitfalls

- **Over-filtering removes useful diversity** — Nemotron-CC's own account: heuristic-only filtering discards up to 90% of content in datasets like DCLM/FineWebEdu, "unsuitable for long-horizon pretraining" at modern token counts — quality filtering is a real quantity-vs-quality trade, not a free improvement (their fix: rephrase-and-reincorporate rather than filter less).
- **Quality filters encode their own bias** — a classifier trained to detect "educational"/"high-quality" content reflects its training data/labels; it can systematically down-rank writing styles, dialects, or domains outside its training distribution, invisibly narrowing the corpus.
- **Near-dup detection cost vs. false-negative risk** — exact hashing is cheap but blind to near-dups; MinHash/LSH catches more but trades recall for tractability at trillion-token scale. Per Lee et al., missed near-duplicates directly drive train-test contamination — a measurable evaluation-validity cost, not just storage.
- **Dedup vs. legitimate repetition** — legal boilerplate, code idioms, FAQ templates, standardized disclaimers are *supposed* to repeat. Blind aggressive dedup risks treating structural signal as noise; thresholds need domain-aware tuning.
- **Synthetic data amplifies the generator's own errors/biases** — a reward model or quality classifier grading synthetic data has its own blind spots (e.g., Nemotron-4's Reward model trained on a specific five-attribute rubric over a comparatively small human-labeled set) — a partial safeguard, not a complete one.
- **Model collapse when synthetic data replaces rather than accumulates alongside real data** (Shumailov et al.) — mitigated by keeping real data in the mix across generations (Gerstgrasser et al.).
- **Freshness vs. stability** — aggressive re-scraping/re-embedding for timeliness trades against pipeline cost/stability; under-investing lets a knowledge base silently answer with outdated facts.
- **Augmentation that drifts from real usage** — synthetic/paraphrased examples that don't match production query/document distribution can inflate eval metrics without improving real-world performance — the same Goodhart's-law risk 3.3 documents for reward modeling, here applied to training-data augmentation.

---

## NVIDIA angle (read critically)

- **NeMo Curator**: GPU-accelerated (RAPIDS cuDF/cuML/cuGraph + Dask) pipeline framework — cleaning (`Modify`), filtering (`ScoreFilter`), deduplication (`ExactDuplicates`, MinHash+LSH fuzzy dedup), PII redaction (`PiiModifier`, Presidio-based), chained via a `Sequential` class. Concrete, checkable architecture — the tutorial walks through working code for each stage.
- **Nemotron-CC**: the more sophisticated recent iteration — classifier-ensemble quality labeling + synthetic-data rephrasing to recover content pure heuristic filtering would discard. A real architectural advance independent of whether you trust NVIDIA's specific benchmark numbers.
- **Read performance claims skeptically**: the "16x faster processing" dedup claim and MMLU improvements (5.6 points over DCLM, 5 points on long-horizon pretraining) are NVIDIA's own comparisons against baselines (DCLM, FineWebEdu, stock Llama 3.1) NVIDIA selected — documented in a companion paper (arXiv:2412.02595, not independently fetched/verified here), but still internally-run, self-reported figures.
- **The more credible half**: Nemotron-4 340B's Reward model "first place on RewardBench" is externally checkable (a third-party leaderboard) — meaningfully stronger evidence than a purely internal benchmark.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Data-quality dimensions | Six core (Wang & Strong 1996 origin; IBM's current framing): accuracy, completeness, consistency, timeliness, validity, uniqueness — each with concrete checks (trusted-source cross-reference, null checks, cross-system reconciliation, freshness/staleness checks, schema/format/range validation, duplicate detection) |
| Preprocessing/cleaning | Normalize encoding/formatting (Unicode fixes, quotation unification) → strip boilerplate (HTML extraction, language ID) → heuristic filters (length, n-gram repetition, perplexity) → model-based quality classifiers where heuristics can't assess semantic quality |
| Why dedup matters for LLMs | Lee et al.: near-dups drive verbatim memorization (>1% of unprompted output) and train-test contamination (>4% of standard validation sets) — dedup cuts memorized-output rate ~10x and reduces training steps needed for equal/better accuracy |
| Exact vs. near-dup | Exact: hash match (e.g., MD5), cheap, misses near-duplicates. Near-dup/fuzzy: MinHash + LSH on Jaccard similarity, catches templated/boilerplate variants, real compute cost at scale |
| PII/unsafe filtering | Detect + redact PII (Presidio-based) and drop unsafe content before ingestion/training, not after — same tooling as 3.3's feedback scrubbing, applied to the corpus itself; toxicity-as-ongoing-safety-policy is Domain 9 |
| Augmentation | Classic NLP techniques (back-translation, rule-based perturbation — Feng et al. survey) plus LLM-based synthetic data generation (Nemotron-4 generate-then-reward-filter; Nemotron-CC rephrase-low-quality / multi-prompt-generate-from-high-quality) to fill coverage gaps |
| Model collapse | Shumailov et al.: training on model-generated content causes irreversible loss of distribution tails ("Model Collapse"), shown in VAEs/GMMs/LLMs. Gerstgrasser et al.: avoidable if synthetic data accumulates alongside real data across generations rather than replacing it |
| Scope boundary | This lesson = corpus/dataset quality itself (RAG documents, training/eval sets). 3.3 = structured user feedback collection for RLHF/DPO. 6.3 = ETL/ingestion/chunking pipeline structure. Domain 9 = bias/toxicity as an ongoing safety concern |
| NVIDIA angle | NeMo Curator = GPU-accelerated, code-checkable dedup/quality-filter/PII pipeline; Nemotron-CC = classifier ensembling + synthetic rephrasing; throughput and MMLU figures are vendor-run against vendor-chosen baselines, flagged as such; RewardBench leaderboard placement is the more independently-checkable claim |

---

## Quick self-check

1. Name the six core data-quality dimensions and give one concrete check example for each.
   **A:** Accuracy (cross-reference a trusted source), completeness (null/missing-value scans), consistency (cross-system reconciliation), timeliness (freshness/staleness checks against last-modified timestamps), validity (schema/format/range validation), uniqueness (duplicate detection, exact-to-near-duplicate).
2. Per Lee et al. (arXiv:2107.06499), deduplication isn't just a storage optimization for LLMs. Name the two specific harms they document from undeduplicated training data, with the concrete figures they report.
   **A:** Verbatim memorization — over 1% of unprompted model output is copied verbatim from training data (a single 61-word sentence appeared 60,000+ times in C4). Train-test contamination — undeduplicated overlap affects over 4% of the validation set of standard datasets, inflating apparent benchmark performance.
3. What's the mechanical difference between exact and near-duplicate detection, and why is near-dup detection meaningfully more expensive at scale?
   **A:** Exact dedup hashes full documents (e.g., MD5) and drops repeat hashes — cheap but blind to any single-character difference. Near-dup/fuzzy dedup uses MinHash signatures + LSH to find documents above a Jaccard-similarity threshold — catches templated/boilerplate variants but requires approximate nearest-neighbor search instead of a hash lookup, costing far more compute at trillion-token scale.
4. What is "model collapse" per Shumailov et al., and how does Gerstgrasser et al.'s follow-up finding change the practical advice for using LLM-generated synthetic data safely?
   **A:** Model collapse is the irreversible loss of the tails of the original content distribution when models are trained on model-generated content generation after generation. Gerstgrasser et al. show collapse happens when synthetic data *replaces* real data over generations, but is avoidable if synthetic data *accumulates alongside* real data — so the safe practice is anchoring synthetic data to real source text rather than recursively retraining on pure model output.
5. A colleague says "6.4 and 3.3 are basically the same topic — both are about improving training data." Explain precisely what distinguishes them, and where PII/toxicity filtering fits relative to Domain 9.
   **A:** 3.3 covers collecting/structuring human feedback (ratings, corrections) for RLHF/DPO — its raw material is judgments about model outputs. This lesson covers the underlying knowledge corpus or training set itself, before any feedback loop touches it. PII/unsafe-content filtering here is a one-time data-hygiene step at ingestion; toxicity/bias as an ongoing safety-guardrail policy is Domain 9's territory.

---

## Sources used

- [What Are Data Quality Dimensions? — IBM](https://www.ibm.com/think/topics/data-quality-dimensions)
- [Deduplicating Training Data Makes Language Models Better — arXiv:2107.06499](https://arxiv.org/abs/2107.06499)
- [Curating Custom Datasets for LLM Training with NVIDIA NeMo Curator — NVIDIA Technical Blog](https://developer.nvidia.com/blog/curating-custom-datasets-for-llm-training-with-nvidia-nemo-curator/)
- [Building Nemotron-CC, A High-Quality Trillion Token Dataset for LLM Pretraining from Common Crawl Using NVIDIA NeMo Curator — NVIDIA Technical Blog](https://developer.nvidia.com/blog/building-nemotron-cc-a-high-quality-trillion-token-dataset-for-llm-pretraining-from-common-crawl-using-nvidia-nemo-curator/)
- [NVIDIA Releases Open Synthetic Data Generation Pipeline for Training Large Language Models — NVIDIA Blog](https://blogs.nvidia.com/blog/nemotron-4-synthetic-data-generation-llm-training/)
- [The Curse of Recursion: Training on Generated Data Makes Models Forget — arXiv:2305.17493](https://arxiv.org/abs/2305.17493)
- [Is Model Collapse Inevitable? Breaking the Curse of Recursion by Accumulating Real and Synthetic Data — arXiv:2404.01413](https://arxiv.org/abs/2404.01413)
- [A Survey of Data Augmentation Approaches for NLP — arXiv:2105.03075](https://arxiv.org/abs/2105.03075)
