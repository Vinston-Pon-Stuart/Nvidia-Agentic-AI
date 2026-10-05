## In one line

**MLOps for agentic systems** = applying CI/CD/CT discipline (automation + monitoring across build, test, release, deploy) to an agent's own versionable artifacts — prompts, tool schemas, model versions, RAG configs, eval gold sets — plus governance controls (approval gates, audit trails, model cards, rollback) that keep that automation safe rather than reckless.

---

## Why it's needed

- A prompt tweak that fixes one test case can silently regress twenty others; a tool schema edited on a Friday can break every downstream caller; a model swap can quietly shift cost/latency/safety for live users with no record of who approved it.
- Google Cloud: MLOps is "an ML engineering culture and practice that aims at unifying ML system development (Dev) and ML system operation (Ops)" — automation and monitoring at every step (integration, testing, releasing, deployment, infra management).
- **Agents need it more**, not less — they chain together more moving, versionable parts (prompts, tool schemas, model versions, retrieval configs, eval datasets) than a single predictive model ever did.
- **Scope**: this lesson stays at the pipeline-and-governance level. Monitoring metrics/dashboards = Domain 8; retraining/versioning mechanics recur in 8.4.

---

## The CI/CD/CT maturity ladder (Google)

Applies DevOps' CI + CD to ML systems in three levels — the exam-relevant distinction is **what gets automated** at each, not "more tooling."

- **Level 0 — manual process**: every step run by hand from a notebook; "disconnection between ML and operations" (data scientist hands artifact over the wall to engineering). New versions ship "only a couple of times per year." No CI (implementation rarely changes), no CD, and "the process doesn't track or log the model predictions and actions" — no active monitoring, so degradation goes undetected.
- **Level 1 — ML pipeline automation**: goal shifts to **continuous training (CT)** — automatic retraining on fresh data. Requires **data validation** (catches schema/data-value skews that halt the pipeline or trigger retraining), **model validation** (compares the new model's metrics against the production baseline before promotion, checks segment consistency, plus a separate *online* canary/A-B step before real traffic), and an **ML metadata store** — every run records pipeline/component versions, start/end times, executor, params, artifact pointers, and "a pointer to the previous trained model if you need to roll back." Pipelines trigger on demand, schedule, new data, detected degradation, or concept drift.
- **Level 2 — CI/CD pipeline automation**: adds a full CI/CD system on top, six stages — dev/experimentation → **pipeline CI** (build + test, produce packaged components) → **pipeline CD** (deploy artifacts to target env) → automated triggering (pipeline runs, produces a trained model → **model registry**) → **model CD** (serve as prediction service) → monitoring (feeds the next cycle). CI here is "no longer only about testing... code" but also data/schemas/models; CD is "no longer a single software package" but a whole training-pipeline system.
- **Caveat**: Google's doc states it "applies primarily to predictive AI systems" — never works through a generative/agentic example. That gap is exactly why LLMOps exists as a distinct adaptation, not a relabeling.

---

## What gets versioned in an agent

- Fowler/Thoughtworks' **CD4ML** (Continuous Delivery for Machine Learning) frames change along **three axes — code, model, data** — each needing different tools/workflows. Mapped onto an agent, the list grows:
  - **Prompts/templates** — the agentic analogue of "code," and likely the highest-change-frequency artifact.
  - **Tool/function definitions and schemas** — a renamed parameter or changed return shape silently breaks every caller with zero "model" code changed.
  - **Model and adapter versions** — base LLM plus fine-tuned adapters (LoRA, etc.).
  - **RAG/retrieval configuration** — embedding model version, chunking strategy, index snapshot.
  - **Eval datasets (gold sets)** — the regression-test artifact itself (3.1); if it silently changes, score deltas become uninterpretable.
- CD4ML's concrete example of versioning non-code: **DVC (Data Science Version Control)** — Git-like semantics + external storage for large files + dependency-graph tracking (`dvc run -d ... -o ... -M results/metrics.json`, reproduced via `dvc repro`) + Git-branch integration for parallel experiments. The tool is dated (2019), but the discipline — every stage declares dependencies/outputs, content-addressed, reproducible — is what an agent's artifacts need regardless of current tooling.

---

## CI: the eval suite *is* the test suite

- Google's CI stage: unit-test feature-engineering logic and model methods, test training convergence (no NaNs), test each component's expected artifact, test integration between components. For an agent most of this maps to ordinary software testing — except **behavioral correctness of a nondeterministic system**, which has no classical analogue. That's where 3.1's eval pipeline becomes the CI test suite: an eval-gated deployment runs the regression gold set on every prompt/tool/model change and blocks promotion below a threshold.
- **NVIDIA NeMo Evaluator SDK**: `nel gate` gives one **GO / NO-GO / INCONCLUSIVE** decision across multiple benchmarks from an explicit policy file (`nel gate ./results/baseline ./results/candidate --policy gate_policy.yaml --strict --output gate_report.json`). `nel compare` runs paired comparisons between two runs, reporting "score deltas, flips, and statistical evidence" rather than one number — a tooled answer to 3.1's warning that a small gold set can't reliably distinguish a real regression from run-to-run noise. `INCONCLUSIVE` is an honest admission that nondeterministic evals sometimes can't support an automated decision.
- **Amdocs case study** (NVIDIA, telecom vendor): pipeline runs regression tests on standard benchmarks (**GSM8K, SQuAD, GLUE, SuperGLUE**) comparing a fine-tuned model against its base ("a critical regression test to ensure the new model's general capabilities haven't been negatively impacted"), then a **business-specific LLM-as-judge** eval on a custom dataset, then **final human evaluation** by domain experts on top candidates before deployment — a working example of graduated CI (automated regression gate → automated judge gate → human gate), not full unattended automation.

---

## CD: automated, governed rollout

- Once a change passes the eval gate, delivery is about *how*, not *whether*, it's correct.
- Google's CD checklist: verify infra compatibility (packages, memory, compute, accelerators), call the prediction API to confirm request/response shape, load-test QPS/latency, and a staged rollout — **automated** (dev-branch push → test env), **semi-automated** (reviewed merge to main → pre-production), **manual** (production, only after several successful pre-production runs) — deliberately keeping a human decision point at the highest-stakes step.
- CD4ML's model deployment patterns: **embedded model** (packaged as an app dependency — simple, but couples model and app release cadence), **model as a separate service** (independently releasable, at the cost of inference-time network latency), **model published as data** (app subscribes to new-version events and hot-swaps at runtime — enables blue-green/canary for models).
- Two agent-relevant scenarios CD4ML names: **shadow models** (new version runs on production traffic without serving results, purely to gather comparison data) and **competing models** (explicit A/B or multi-armed-bandit split, needs enough traffic/time for statistical significance).
- Rollout-strategy sizing mechanics = 4.5 (recurs in Domain 8). The point here: CD for an agent is the same staged, gated rollout logic applied to prompts/tools/models as first-class deployable artifacts.

---

## Monitoring hooks (pointer, not the destination)

- Google's stage 6 ("collect statistics on model performance... a trigger to execute the pipeline or a new experiment cycle") and CD4ML's feedback loop converge on the same shape: monitoring isn't bolted on after deployment — it decides whether the *next* CI/CD cycle starts.
- Microsoft's "AI Agents for Beginners" production lesson frames an explicit loop: **evaluate offline → deploy → monitor online → collect new failure cases → add to offline dataset → refine agent → repeat** — online evaluation "capture[s] things you might not anticipate in a lab setting."
- Depth (metrics, dashboards, retraining/versioning automation) = Domain 8 / 8.4; this lesson just names the hook.

---

## Governance: gates, audit trails, model cards, lineage, rollback

Automation without governance is how a bad prompt reaches every user before anyone notices. CD4ML: manual approval stages belong in the pipeline **by design, not as a failure of automation** — they "allow humans to decide if they should be promoted or not... model a Machine Learning governance process and introduce checks for model bias, model fairness, or... explainability."

- **Approval gates** — CD4ML's manual promotion stage, plus Google's semi-automated (reviewer-approved merge) / manual (production) steps: cheap transitions automate fully; a new model version or a policy-relevant prompt change gets a named human sign-off.
- **Audit trails / lineage** — Google's **ML metadata store** records, per run: pipeline/component versions, executor, params, artifact pointers, and "a pointer to the previous trained model." One structure answers both "who/what produced this" and "what to roll back to." Databricks' LLMOps framing: "track model and pipeline lineage and versions... through their lifecycle" — for LLMs this also covers base-model version, fine-tuning data, hyperparameters.
- **Model cards** — Mitchell et al., "Model Cards for Model Reporting" (2019): "short documents... provide benchmarked evaluation in a variety of conditions... disclose the context in which models are intended to be used, details of the performance evaluation procedures," proposed "as a step towards the responsible democratization of machine learning." For an agent: a per-release summary (prompt diff, tool diff, model version, eval scores) attached to the promotion decision.
- **Rollback** — CD4ML names it a distinct orchestration concern: "define a process for rollbacks, in case a deployed model turns out to perform poorly or incorrectly in production" — only works if the lineage/metadata layer already points to the last-known-good version.

---

## How LLMOps differs from classic MLOps (Databricks)

- **No mandatory retraining loop** — classical MLOps' central automation target is CT (retrain from scratch). Agent iteration instead changes prompts/RAG config or fine-tunes a small adapter — Databricks' **transfer learning**: LLMs "start from a foundation model and are fine-tuned with new data... using less data and fewer compute resources." CI/CD (gate every change through eval) still applies; the CT half of "CI/CD/CT" is often simply absent.
- **Different performance metrics** — classical ML has "clearly defined performance metrics, such as accuracy, AUC, F1 score" (cheap, unambiguous). LLM eval leans on BLEU/ROUGE or LLM-as-judge (3.1) — inherently more subjective, needing "extra consideration."
- **Prompt engineering as a risk-bearing artifact** — "prompt engineering can reduce the risk of model hallucination and prompt hacking, including prompt injection, leaking of sensitive data and jailbreaking" — a prompt change carries the same safety-review weight as code touching an auth boundary.
- **Human feedback as a first-class signal** — beyond RLHF at training time: "human feedback from your application's end users is often critical for evaluating LLM performance... simplifies evaluation and provides data for future fine-tuning."
- **Chains/pipelines over single models** — "LLM application development often focuses on building these pipelines [of multiple LLM calls plus external systems like vector databases], rather than building new LLMs" — precisely the agent framing: the deployable unit is an orchestration graph, not a single model.

---

## Trade-offs and pitfalls

- **Gating CI on a nondeterministic signal** — an eval suite is noisier than a unit test (3.1). Treating one run's average score as ground truth produces false-positive blocks and false-negative passes unless variance is accounted for — exactly what `nel gate`'s `INCONCLUSIVE` and `nel compare`'s statistical evidence are built to surface, not paper over.
- **Prompt changes shipped without versioning** — tempting to treat a prompt edit as "just text" outside pipeline discipline, but per CD4ML's three-axes framing, an unversioned prompt change is as unreproducible, unauditable, and unrevertible as an unversioned code change.
- **Governance theater vs. real controls** — a model card generated once and never updated, or a rubber-stamped approval gate, gives the appearance of governance without the substance. Amdocs' structure (automated regression gate → automated judge gate → human eval only on survivors) works precisely because the human step is reserved for what matters most, not asked to review every commit.
- **Over-automating high-risk changes** — Google's own ladder keeps a manual step at final production promotion even at level 2 ("manual deployment... after several successful runs on the pre-production environment"); zero-human-sign-off full automation is not the goal of "level 2 maturity," and treating it as one is a common misreading.

---

## NVIDIA angle (read critically)

- Most concrete, verifiable contribution: **`nel gate` / `nel compare`** — a real, open-source, policy-driven quality gate turning a benchmark run into a ternary CI decision plus a paired-comparison report. Genuine engineering, not marketing.
- **Amdocs/NeMo case study** (Curator → Customizer → Evaluator → Guardrails → NIM, GitOps via ArgoCD/Argo Workflow) — real, named example of a multi-stage gated pipeline, with numbers: LoRA-fine-tuned Llama 3.1 8B scored **0.83 vs. base model's 0.74** on a business task; a TriviaQA regression check confirmed no degradation (0.6 both). **Caveat**: a co-authored vendor/customer blog promoting NVIDIA's AI Blueprint reference architecture, not an independent audit — a 50-example fine-tuning dataset and one benchmark comparison is a modest evidentiary base for the headline claim.
- **Thinnest area — governance itself**: model cards, formal audit-trail schemas, and role-based approval workflows aren't given anything like the detail Google's ML metadata store or CD4ML's manual-governance-stage discussion give them. NVIDIA is strong on evaluation/deployment plumbing (gates, benchmarks, GitOps) and safety filtering (NeMo Guardrails), but a team would still need to bring its own model-card practice, audit-log retention policy, and approval-workflow tooling from elsewhere.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| MLOps definition | Unifies ML dev + ops via automation and monitoring across integration, testing, releasing, deployment, infra management (Google Cloud) |
| CI/CD/CT maturity levels | Level 0 = manual, no CI/CD, infrequent releases; Level 1 = automated ML pipeline + CT, with data/model validation and an ML metadata store; Level 2 = full CI/CD automation of the pipeline itself (6 stages: dev → pipeline CI → pipeline CD → automated trigger → model CD → monitoring) |
| What's versioned in an agent | Prompts/templates, tool/function schemas, model + adapter versions, RAG/retrieval config, eval gold sets — CD4ML's "code, model, data" 3-axes framing extended |
| CI = eval-as-test | Regression gate against the Lesson 3.1 eval suite; NeMo Evaluator's `nel gate` = GO/NO-GO/INCONCLUSIVE from a policy file; `nel compare` = paired score-delta + statistical evidence |
| CD = governed rollout | Staged automated → semi-automated (reviewed) → manual (production) deployment (Google); embedded/service/data-published model patterns, shadow + competing models (CD4ML); canary/blue-green mechanics = Lesson 4.5 / Domain 8 |
| Monitoring | Feeds the next CI/CD cycle (Google stage 6; CD4ML feedback loop); offline→deploy→online→new failure cases→offline dataset loop (Microsoft); metrics depth = Domain 8 |
| Governance | Approval gates (manual promotion stages), audit trails + lineage (ML metadata store: versions, executor, params, artifact + rollback pointers), model cards (Mitchell et al. 2019), rollback as an explicit orchestration concern |
| LLMOps vs. MLOps | Often no CT/retrain-from-scratch loop (fine-tune/adapt instead); subjective metrics (BLEU/ROUGE, LLM-judge) vs. accuracy/F1/AUC; prompt = risk-bearing artifact; human feedback as first-class signal; deployable unit = a chain/pipeline, not one model |
| Biggest pitfalls | Gating CI on noisy nondeterministic scores without variance handling; unversioned prompt changes; governance theater (rubber-stamped reviews); over-automating the one step (final prod promotion) that should keep a human gate |

---

## Quick self-check

1. Name Google's three MLOps maturity levels and, for each, state what specifically becomes automated that wasn't at the level before.
   **A:** Level 0 (manual) — everything by hand, no CI/CD, no monitoring. Level 1 (ML pipeline automation) — adds **CT**: automated data validation, model validation, and an ML metadata store, with pipelines auto-triggered. Level 2 (CI/CD pipeline automation) — adds a full CI/CD system around the pipeline itself: pipeline CI/CD, automated triggering into a model registry, model CD, and monitoring that feeds the next cycle.
2. List at least four distinct artifact types an agentic system needs to version (beyond just "the model"), and explain why an unversioned prompt change is as risky as an unversioned code change.
   **A:** Prompts/templates, tool/function schemas, model + adapter versions, RAG/retrieval config, and eval gold sets. Per CD4ML's code/model/data framing, a prompt is the agentic analogue of "code" — leaving it unversioned makes it just as unreproducible, unauditable, and unrevertible as unversioned code.
3. What does NeMo Evaluator's `nel gate` command decide, and why is an `INCONCLUSIVE` outcome a meaningful third option rather than a design gap?
   **A:** It produces one **GO / NO-GO / INCONCLUSIVE** decision across multiple benchmarks from an explicit policy file. `INCONCLUSIVE` matters because nondeterministic evals sometimes genuinely can't support an automated pass/fail call — it's an honest admission instead of forcing a binary decision on noisy data.
4. Give three concrete governance mechanisms this lesson covers (not counting monitoring dashboards) and, for each, name the source that describes it.
   **A:** Approval gates (CD4ML's manual promotion stage; Google's semi-automated/manual deployment steps), audit trails and lineage (Google's ML metadata store; Databricks' LLMOps lineage requirement), and model cards (Mitchell et al., 2019). Rollback (CD4ML) is a fourth.
5. Name two specific ways LLMOps differs from classical MLOps per Databricks' framing, and explain why one of them (the retraining loop) changes what "CI/CD/CT" even means for an agent.
   **A:** LLMOps has no mandatory retraining loop (fine-tuning/adapters via transfer learning instead of training from scratch) and uses more subjective metrics (BLEU/ROUGE, LLM-as-judge) instead of accuracy/F1/AUC. Because most agent iteration just changes prompts or fine-tunes a small adapter rather than retraining from scratch, the CT leg of "CI/CD/CT" is often simply absent — the discipline reduces to gated CI/CD over prompts/tools/models.

---

## Sources used

- [MLOps: Continuous delivery and automation pipelines in machine learning — Google Cloud Architecture Center](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning)
- [Continuous Delivery for Machine Learning (CD4ML) — Martin Fowler / Thoughtworks](https://martinfowler.com/articles/cd4ml.html)
- [AI Agents in Production: Observability & Evaluation — Microsoft "AI Agents for Beginners" (GitHub source)](https://raw.githubusercontent.com/microsoft/ai-agents-for-beginners/main/10-ai-agents-production/README.md)
- [What is LLMOps? — Databricks](https://www.databricks.com/blog/what-is-llmops)
- [Fine-Tuning LLMOps for Rapid Model Evaluation and Ongoing Optimization — NVIDIA Technical Blog](https://developer.nvidia.com/blog/fine-tuning-llmops-for-rapid-model-evaluation-and-ongoing-optimization/)
- [NVIDIA-NeMo/Evaluator — README (nel gate, nel compare)](https://raw.githubusercontent.com/NVIDIA-NeMo/Evaluator/main/README.md)
- [Model Cards for Model Reporting — Mitchell et al., arXiv:1810.03993](https://arxiv.org/abs/1810.03993)
