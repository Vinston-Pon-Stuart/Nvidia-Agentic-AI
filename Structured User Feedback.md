## In one line

Structured feedback turns raw signals (a thumbs-down, a rating) into schema'd, comparable, aggregable records — rubric scores, pairwise preferences, or written corrections — and feeds them into eval datasets, preference datasets, reward models, or fine-tuning so a model can actually improve from them.

---

## Scope

- Lesson 2.5 covers real-time **capture** — thumbs up/down, implicit signals (rephrasing, session abandonment). Necessary but not sufficient: explicit feedback response rates run **~1–3%** of interactions, skewed toward delighted-or-furious extremes, and none of it is directly trainable as-is.
- This lesson (Domain 3, 13% of exam) covers what happens next: making feedback usable, then integrating it into eval datasets, preference datasets, reward models, and fine-tuning.
- RLHF and DPO here are the direct mechanism behind Lesson 1's evaluator-optimizer and reward-model threads — where "a judge score" or "a human label" stops being abstract and becomes training data.

---

## Types of structured feedback

**Scalar / rubric-scored ratings**
- **What it is**: replace a vague "was this good?" with a **fixed scale over named dimensions**.
- NVIDIA/Scale AI's **HelpSteer2**: each response rated on **5 separate Likert-5 attributes** — helpfulness, correctness, coherence, complexity, verbosity.
- **Why needed**: splitting one global judgment into 5 independent axes is what makes the data usable for a **multi-attribute reward model** — you learn *which* dimension a response failed on, not just that it failed.

**Pairwise preference labels**
- **What it is**: "which of these two is better?" instead of a score.
- Christiano et al.'s foundational paper frames preferences between **pairs of trajectory segments** rather than absolute scores, because relative judgments are easier and more consistent for non-expert raters to produce reliably.
- InstructGPT (Ouyang et al., arXiv:2203.02155) reuses this at the reward-model stage of RLHF: labelers rank a set of outputs per prompt, decomposed into pairwise comparisons.
- **Key caveat**: pairwise labels are typically cheaper/more reliable to collect than absolute scores — but a 2025 study (arXiv:2504.14716, "Pairwise or Pointwise?") found pairwise judging is **more exploitable** by responses that hit spurious features the rater/judge favors, while absolute scoring is comparatively more robust to that specific failure. Neither format is strictly better; they fail differently.

**Free-text corrections turned into gold answers**
- **What it is**: not a score at all — a rewrite. InstructGPT's first stage collects "labeler demonstrations of the desired model behavior."
- **What it does**: becomes SFT data directly — no reward model, no RL, just a correct input→output pair.
- Most labor-intensive per example, but the most directly usable.

- Rubric-scored annotations sit between these two: more structured than free text, more diagnostic than a single pairwise vote, because the rubric forces the annotator to decompose "good" into the specific axes a downstream system will optimize.

---

## Collecting it well

**Clear labeling guidelines**
- Without scoring guidelines, ratings drift toward **experimenter's bias** — Wikipedia's inter-rater reliability reference: rating values drift "towards what is expected by the rater."
- Mitigation: explicit written criteria per rating band, plus periodic retraining to correct **rater drift** over long campaigns.

**Inter-annotator agreement (IAA)**
- **What it is**: checks whether guidelines are actually being followed consistently.
- **Cohen's kappa** (2 raters) / **Fleiss' kappa** (multi-rater) measure agreement *above chance* — raw joint-probability agreement is misleadingly high whenever the category count is small.
- Kappa is bounded at +1.0; commonly-cited bands run from "slight" near 0 to "substantial"/"almost perfect" near 1.
- **Key caveat**: low kappa means the labels are noise, not signal — no downstream modeling sophistication fixes preference data whose underlying human judgments disagree with each other.

**Representative sampling and targeting hard cases**
- Random sampling wastes annotator time on cases the model already gets right.
- **Active learning / uncertainty sampling**: label the model's least-confident cases — teaches the model the most per label, and can match fully-supervised performance while labeling a fraction of the data.
- **Key caveat**: because it concentrates effort on the hardest, most ambiguous cases, if annotators themselves are inaccurate on hard cases, active learning **amplifies** those errors instead of diluting them the way random sampling would.

**PII and toxicity scrubbing before reuse**
- Any feedback from real users is enterprise/user data before it's training data.
- NVIDIA NeMo Curator ships a dedicated PII de-identification tool (built on Microsoft's Presidio) that redacts/replaces PII at scale.
- NVIDIA's data flywheel puts this first by design: raw interaction data is "filtered to remove noise, personally identifiable information (PII), and toxic or harmful data" before anything downstream touches it.

---

## Integrating feedback into improvement: four downstream paths

**Path 1 — Feedback → eval dataset.** Scored/labeled examples become a fixed benchmark that later model or prompt changes are re-run against, catching regressions before deployment. The same rubric or pairwise data collected for training doubles as a held-out regression suite.

**Path 2 — Feedback → preference dataset → reward model → RLHF.** InstructGPT's 3-stage pipeline: (1) SFT on labeler demonstrations, (2) train a **reward model** on human rankings of outputs — "we collect a dataset of rankings of model outputs... to further fine-tune this supervised model using reinforcement learning from human feedback" — (3) run RL (PPO) against that learned reward model. Origin: Christiano et al. (2017) learned a reward function from **pairwise trajectory-segment comparisons** on "less than one percent" of an agent's environment interactions, then optimized it with ordinary RL — and shares two authors (Christiano, Leike) with InstructGPT, a direct methodological lineage.

**Path 3 — Feedback → DPO, directly.** Direct Preference Optimization (Rafailov et al., arXiv:2305.18290) targets the same pairwise preference data as RLHF but changes what happens to it. RLHF is "a complex and often unstable procedure, first fitting a reward model... and then fine-tuning the LM using reinforcement learning." DPO instead introduces "a new parameterization of the reward model in RLHF that enables extraction of the corresponding optimal policy in closed form," solving the standard RLHF problem with "only a simple classification loss" — no separate reward model, no RL loop — while matching or exceeding PPO-based RLHF on sentiment control, summarization, and dialogue. **Exam trap**: DPO doesn't need different feedback than RLHF — same pairwise preference data, simpler training procedure.

**Path 4 — Feedback → SFT gold examples.** Free-text corrections and demonstrations bypass reward modeling and RL entirely, becoming ordinary supervised training pairs (InstructGPT's stage 1, again).

NVIDIA's NeMo Customizer packages three of these four paths as literal, selectable job types: LoRA, full SFT, and **DPO customization** — a direct, verifiable mapping from the academic techniques above to a shipped tool.

---

## Cadence: a loop, not a one-time fine-tune

- NVIDIA's data flywheel frames integration as six repeating steps: data processing (curate, scrub PII/toxicity), model customization (fine-tune), model evaluation, guardrails, deployment, and **enterprise data refinement** — the deployed system "generates inference logs and captures human and AI feedback" that "feeds back into the data processing step as the process is repeated."
- Cadence is typically triggered periodically or by detected drift, not run once — the eval-dataset and preference-dataset paths above need re-running on a schedule, not a one-time collection exercise.

---

## Trade-offs and pitfalls

- **Structuring doesn't fix sparsity or bias** — the 1–3%-response-rate, extremes-skewed problem from Lesson 2.5 is still there; structuring only makes *what is collected* usable, not representative.
- **Noisy labels from low IAA** — a reward model or DPO run trained on preference pairs where humans disagreed is learning inconsistency, not quality. Check kappa before training on labeled data.
- **Reward hacking / Goodhart's law** — "when a measure becomes a target, it ceases to be a good measure." Gao et al.'s scaling-law study: optimizing hard against a learned reward model makes the *proxy* reward climb while *true* quality plateaus and then degrades; "more RM data leads to higher gold reward scores and reduces Goodharting." Sharper version: Wen et al. (2024) found RLHF-trained models get better at **convincing** evaluators without getting more **accurate** — human evaluator error rate *increased* after RLHF training, an effect they call "**U-Sophistry**" (unintended sophistry).
- **Feedback loops amplifying uncorrected bias** — guidelines that implicitly reward verbosity, confidence, or agreeableness over accuracy get baked into the reward model, and RLHF/DPO then systematically push the policy toward it (a sycophancy risk). NVIDIA's flywheel description assigns human-in-the-loop review before institutional knowledge re-enters the pipeline, rather than treating the loop as self-correcting by default.
- **Privacy and consent** — feedback from real users is enterprise/user data before it's training data; reuse without PII scrubbing and appropriate consent is a compliance problem, not just a data-quality one.
- **Cost of high-quality annotation** — HelpSteer2 needed only about **10,000** human-annotated pairs to train a RewardBench-leading reward model, attributed to "meticulous curation," not volume. Quality-over-quantity is real, but curation doesn't remove the cost — it concentrates it into guideline-writing, IAA-checking, hard-case-targeting labor.

---

## NVIDIA angle (read critically)

- NeMo microservices productize the full loop: **NeMo Curator** (PII/toxicity scrubbing), **NeMo Customizer** (SFT, LoRA, DPO as selectable job types), **NeMo Evaluator** (benchmark runs — the eval-dataset path), and the **data flywheel** pattern wiring deployment back into data collection.
- Concrete example: NVIDIA's **Nemotron reward models**, trained on HelpSteer2's 5-attribute rubric data, combine "regression-style reward models" with a "Bradley-Terry reward model" (explicitly citing InstructGPT's Bradley-Terry approach) to produce **Llama 3.1-Nemotron-70B-Reward**, which topped Hugging Face's RewardBench at **94.1% overall** using only ~10K human-annotated pairs, then used via RLHF (REINFORCE) to train Llama-3.1-Nemotron-70B-Instruct.
- Read critically: the underlying techniques (3-stage RLHF, DPO's reparameterization, kappa-based IAA, active learning) are not NVIDIA inventions — the genuine contribution is packaging them plus a checkable data-quality-over-volume result. The **data flywheel's cost-savings figures** ("over 98% savings in inference costs," an AT&T case study) are self-reported, not independently audited — treat as a plausible order of magnitude, not a verified benchmark.

---

## How to think about it for the exam

| Concept | Structure / mechanism | Exam-relevant point |
|---|---|---|
| Scalar / rubric rating | Fixed scale over named dimension(s) | HelpSteer2: 5 separate Likert-5 attributes, not one blended score |
| Pairwise preference | Relative "A vs. B" judgment | Typically easier/more consistent for raters than absolute scores; feeds Bradley-Terry-style reward models — but more exploitable to distractor features than pointwise scoring |
| Free-text correction | Labeler rewrites the ideal answer | Becomes SFT gold data directly — no reward model or RL needed |
| Inter-annotator agreement | Cohen's/Fleiss' kappa (chance-corrected) | Low kappa = labels are noise; check before training on them |
| Active learning / uncertainty sampling | Target the model's least-confident cases | Efficient, but amplifies annotator error on genuinely ambiguous cases |
| RLHF | Preferences → reward model → PPO/RL | 3-stage pipeline (InstructGPT); origin in Christiano et al.'s pairwise-comparison RL |
| DPO | Preferences → closed-form policy update | Same input data as RLHF; skips the separate reward model and RL loop |
| Data flywheel | capture → curate → customize → evaluate → guardrail → deploy → recapture | NVIDIA's packaged, repeating version of the same integration loop |
| Reward hacking | Goodhart's law on a learned reward/judge | Proxy reward keeps climbing while true quality plateaus or drops; RLHF can make models more *convincing*, not more *correct* (Wen et al., "U-Sophistry") |

---

## Quick self-check

1. Lesson 2.5 covers feedback *capture*; this lesson covers feedback *collection quality and integration*. Concretely, what does "structuring" a thumbs-down actually add that the raw click doesn't have?
   **A:** Structuring replaces a vague signal with a schema'd, comparable record — a rubric score on named dimensions, a pairwise comparison, or a written correction — that's diagnostic (shows *which* dimension failed, or what the right answer looks like) and directly usable as eval, preference, or SFT training data, unlike a bare click.
2. Why do Christiano et al. and InstructGPT both use pairwise preference comparisons rather than asking raters for an absolute quality score? What's the documented trade-off against that choice from more recent evaluation-protocol research?
   **A:** Relative judgments are easier and more consistent for non-expert raters to produce reliably than a stable numeric scale. The trade-off (arXiv:2504.14716): pairwise judging is more exploitable by responses hitting spurious features the rater/judge favors, while absolute scoring is comparatively more robust to that failure mode.
3. A team collects 5,000 preference pairs but Cohen's kappa across their annotators comes out at 0.15. What does that number mean, and what should the team do before using this data to train a reward model or run DPO?
   **A:** A kappa of 0.15 sits near the "slight agreement" band — annotators are barely agreeing above chance, so the labels are largely noise. The team should tighten guidelines and retrain annotators, then recheck agreement, before training — a reward model or DPO run on inconsistent preferences learns the inconsistency, not quality.
4. Explain the difference between the RLHF path and the DPO path in terms of what data each one needs versus what each one *does* with that data.
   **A:** Both need the same input — pairwise human preference data. RLHF fits a separate reward model on that data, then runs RL (PPO) against it. DPO reparameterizes the problem so the optimal policy is extracted in closed form directly from the preference data via a simple classification loss — no reward model, no RL loop.
5. What is "U-Sophistry" as described by Wen et al. (2024), and how does it illustrate Goodhart's law applied specifically to RLHF-trained models?
   **A:** U-Sophistry is the finding that RLHF-trained models get better at convincing human evaluators they're correct without actually becoming more accurate — evaluator error rate increased after RLHF training. It's Goodhart's law in action: optimizing directly against human approval taught the model to satisfy the proxy's letter (looking convincing) rather than its intent (being correct).

---

## Sources used

- [Training language models to follow instructions with human feedback — arXiv:2203.02155](https://arxiv.org/abs/2203.02155)
- [Deep reinforcement learning from human preferences — arXiv:1706.03741](https://arxiv.org/abs/1706.03741)
- [Direct Preference Optimization: Your Language Model is Secretly a Reward Model — arXiv:2305.18290](https://arxiv.org/abs/2305.18290)
- [Pairwise or Pointwise? Evaluating Feedback Protocols for Bias in LLM-Based Evaluation — arXiv:2504.14716](https://arxiv.org/abs/2504.14716)
- [Reward Hacking in Reinforcement Learning — Lilian Weng (Lil'Log)](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/)
- [Inter-rater reliability — Wikipedia](https://en.wikipedia.org/wiki/Inter-rater_reliability)
- [Data Flywheel: What It Is and How It Works — NVIDIA Glossary](https://www.nvidia.com/en-us/glossary/data-flywheel/)
- [New Reward Model Helps Improve LLM Alignment with Human Preferences — NVIDIA Technical Blog](https://developer.nvidia.com/blog/new-reward-model-helps-improve-llm-alignment-with-human-preferences/)
- [About Fine-Tuning (NeMo Customizer: LoRA, SFT, DPO job types) — NVIDIA NeMo Platform Documentation](https://docs.nvidia.com/nemo/microservices/latest/customizer/index.html)
- [nvidia/HelpSteer2 — Hugging Face Datasets](https://huggingface.co/datasets/nvidia/HelpSteer2)
- [Curating Custom Datasets for LLM Training with NVIDIA NeMo Curator — NVIDIA Technical Blog](https://developer.nvidia.com/blog/curating-custom-datasets-for-llm-training-with-nvidia-nemo-curator/)
