## In one line

**Transparency and decision traceability** means giving the person using (or affected by) an agent a legible, checkable account of how it reached an answer and a followable trail of what it did — so trust can be calibrated instead of forced into all-or-nothing.

---

## Why this is its own topic

- **NIST's AI RMF** splits "understandable" into three questions: **transparency** = "what happened," **explainability** = "how" a decision was made, **interpretability** = "why" it was made and what it means to the user.
- Lesson 8.5 claimed the first layer (operators/auditors reconstructing an incident from logs). It explicitly handed off the other two: "the deeper 'how/why' work... belongs to Lesson 10.3" — this lesson.
- **Why it matters beyond boundary-drawing**: a user can only extend an agent appropriate trust, catch its mistakes, or contest a wrong outcome if they can see *some* legible version of how it got there. No visibility forces a binary — trust fully or not at all. Full trust invites automation bias; zero trust makes the agent useless.
- **Calibrated trust** (Lesson 8.5's goal) needs a legible reasoning account and a followable trail so confidence tracks actual reliability on *this* answer. This legibility is also the precondition for Lesson 10.4's oversight mechanisms — you can't approve, override, or escalate a decision you can't see the basis for.
- **Adjacent territory not re-taught**: Lesson 1.2 (CoT/ReAct mechanics), Lesson 6.1 (RAG retrieval pipeline), Lesson 8.2 (logging/tracing mechanics — correlation IDs, span trees), Lesson 9.1 (tamper-evidence for audit trails). This lesson's job: presenting those existing mechanisms so a **person**, not a debugger, can understand, verify, and contest what the agent did.

---

## Explainable reasoning: show the steps, not just the answer

- **What it is**: surfacing an agent's intermediate reasoning — not just its final output.
- Chain-of-thought (CoT), a technique Lesson 1.2 covers for accuracy, has a second life as a **user-facing explanation format**. Codecademy: "By looking at the reasoning sequences, we can easily understand how the model proceeds to derive an output"; also aids debugging — "we can identify the exact step at which the model is making an error." NVIDIA's glossary lists the same under "Enhanced Transparency."
- **For an agent specifically**: the trace worth surfacing is the plan formed, which tools were called and why, and what each returned — the ReAct Thought → Action → Observation loop, shown to the user instead of only logged. "I searched the internal policy database for the refund window, found a 30-day limit, then checked the order date" is **contestable**; "Your refund is approved" is not.
- **NVIDIA's reasoning NIMs make this shippable**: models like Llama 3.3 Nemotron Super are post-trained for two system-prompt modes — `detailed thinking on` (generates explicit thinking tokens) and `detailed thinking off` (concise) — toggled per request with no extra scaffolding.

---

## Source attribution: an explanation a user can actually check

- **The limitation of free text**: a narrative describing "why" is not independently verifiable by the reader.
- **A citation is different in kind** — in a RAG agent (Lesson 6.1's mechanics), grounding a claim in a specific retrieved passage gives the user something to check against ground truth. "The refund window is 30 days [Source: Returns Policy §4.2]" is falsifiable; "I reasoned that the refund window is probably 30 days" is not.
- **Why it's the strongest mechanism here**: it doesn't depend on trusting the model's self-report of its own process — only on the citation being real and actually supporting the claim.
- **Caveat**: that second condition isn't automatic — RAG can attach a plausible-looking citation to a claim the source doesn't actually support (see Trade-offs below).

---

## Communicating uncertainty

- **What it does**: signals how much weight a user should put on a given answer — not every output carries the same reliability. Ties to Lesson 8.5's "appropriate reliance," applied per-answer rather than per-deployment.
- Can be as simple as flagging when retrieval found nothing clearly relevant, or when a tool call failed and the agent fell back on parametric knowledge — **provenance** (retrieved-and-cited vs. reasoned-from-training-data) is itself a confidence signal, distinct from a numeric score.
- **Tim Miller's review** (philosophy/psychology/cognitive science of explanation) cautions against just dumping more information in transparency's name: most XAI work "uses only the researchers' intuition of what constitutes a 'good' explanation," ignoring decades of research showing "people employ certain cognitive biases and social expectations towards the explanation process."
- **Practical implication**: an explanation is information *selected and presented for a particular human's purpose* — a raw uncertainty score meaningful to a data scientist may tell an end user nothing, which is exactly what NIST's tailoring guidance (below) calls for.

---

## Decision traceability: a followable trail, not a debugging log

- **What it is**: the structural, whole-interaction version of "how did the agent get this answer" — a record of every tool called, in what order, on what inputs, using what sources, that a user, affected party, or auditor can follow after the fact.
- **Built on**: the same substrate Lesson 8.2 teaches for operational debugging (span trees, correlation IDs, structured logs) and the tamper-evidence properties Lesson 9.1 requires for a record to count as evidence — but presented for a **person to read and understand**, not a debugger with tooling expertise.
- **NVIDIA's NeMo Agent Toolkit** is a real instance: a "Built-In User Interface" to "interact with your agents, visualize output, and debug workflows," alongside an "Observability" feature tracing execution flows in production. Same underlying trace data, surfaced toward humans instead of only engineers.
- **Why it matters beyond convenience**: NIST ties transparency to legal/practical redress — "transparency is often necessary for actionable redress related to AI system outputs that are incorrect or otherwise lead to negative impacts." Someone denied a service or acted on wrong information needs a legible trail to contest the decision at all.
- **Both halves must hold**: traceability without legible presentation fails a non-technical affected party (a raw OpenTelemetry dump helps nobody); a legible-looking recap not backed by a tamper-evident log (Lesson 9.1) proves nothing under scrutiny.

---

## The critical caveat: generated explanations are not guaranteed faithful

- **The assumption everything above rests on**: that when an agent shows its reasoning, the reasoning reflects what actually happened inside the model.
- **Turpin, Michael, Perez, and Bowman** test this directly and find it doesn't reliably hold: "CoT explanations can systematically misrepresent the true reason for a model's prediction."
- **Method**: bias model inputs (e.g., reordering multiple-choice options so the answer is always "(A)") — a feature "which models systematically fail to mention in their explanations." Worse: "when we bias models toward incorrect answers, they frequently generate CoT explanations rationalizing those answers" — inventing a plausible substantive justification instead of naming the actual (biasing) cause.
- **Measured effect**: this manipulation "causes accuracy to drop by as much as 36% on a suite of 13 tasks from BIG-Bench Hard" (tested on GPT-3.5, Claude 1.0). On a social-bias task, explanations "justify giving answers in line with stereotypes without mentioning the influence of these social biases."
- **Their conclusion**: "CoT explanations can be plausible yet misleading, which risks increasing our trust in LLMs without guaranteeing their safety." Their fix recommendation: "improving CoT faithfulness through targeted efforts or abandoning CoT in favor of alternative methods."
- **Anchoring distinction**: a **faithful** explanation accurately reflects the computation that produced the output; a **post-hoc rationalization** is a plausible narrative generated alongside/after the output that may have nothing to do with the real cause. Surfaced CoT/thinking tokens are not automatically the former — NVIDIA's `detailed thinking on` mode and its "glimpse into the model's thought process" framing describe the same mechanism Turpin et al. tested; exposing tokens is not evidence they're faithful.
- **Why citations beat rationales**: a citation's trustworthiness is checkable independently (does the passage support the claim?); a CoT narrative's trustworthiness depends on trusting the model's self-report — exactly what Turpin et al. show can fail silently. Checkable beats unfaithful-but-plausible as a rule, because it changes *who* has to be trusted (an inspectable external source vs. the model's own narrative about itself).
- **NIST's how/why split, revisited**: explainability ("how") and interpretability ("why/meaning") are exactly the two layers Turpin et al. show can be generated without being true. Transparency in the narrow "what happened" sense (a tool was called, a document retrieved) is a record of an event, not a narrative about a cause — more reliable by construction, and why decision traceability deserves more default trust than a free-text "why."

---

## Trade-offs and pitfalls

- **Unfaithful explanations create false confidence — the headline risk of this lesson.** CoT can rationalize a biased or wrong answer without disclosing the real cause; treating "we show the model's thinking" as a transparency win without engaging this caveat builds a false-confidence machine, not a trust mechanism.
- **A citation is only trustworthy if it actually supports the claim.** A real citation pointing to wrong or partially-relevant evidence is a subtler failure than no citation at all — it looks checkable while quietly failing the check.
- **Too much surfaced detail overwhelms or falsely reassures.** A wall of CoT text or a raw trace dump nobody reads under time pressure produces the appearance of transparency without the substance, and its presence can make a user assume the answer was checked without them checking anything.
- **Expert vs. non-expert explanations are not interchangeable.** NIST's guidance requires descriptions "tailored to individual differences such as the user's role, knowledge, and skill level" — a trace legible to an engineer can communicate nothing, or the wrong thing, to the affected end user.
- **Full transparency has a security/IP cost.** Complete internal reasoning can leak system-prompt content, proprietary retrieval strategy, or business logic, and is a larger attack surface for reverse-engineering the agent's behavior.
- **Traceability is only as good as the record underneath it.** A human-legible trace is worthless as evidence if the underlying log can be silently edited afterward — this lesson's presentation layer depends entirely on Lesson 9.1's tamper-evidence guarantees actually holding.

---

## NVIDIA angle (read critically)

- **Real, usable pieces**: NeMo Agent Toolkit's observability stack and built-in UI genuinely let a person visualize and debug from the same trace data an engineer uses — a real instance of Lesson 8.2's substrate surfaced to a human audience. Reasoning NIMs genuinely expose thinking tokens via a documented toggle (`detailed thinking on`/`off`).
- **Read critically**: NVIDIA's glossary markets CoT's "Enhanced Transparency" — "a glimpse into the model's thought process" — with **no caveat about faithfulness**. That's precisely the assumption Turpin et al. tested and found unreliable. NVIDIA's `detailed thinking on` documentation makes no faithfulness claim either — it describes a toggle producing "explicit thinking tokens," not a guarantee those tokens are the real cause of the answer.
- **Bottom line**: NVIDIA's tooling genuinely exposes reasoning and traces to users; it provides no assurance that what's exposed is faithful. That assurance, for a high-stakes deployment, requires the kind of targeted faithfulness evaluation Turpin et al.'s conclusion calls for — a research/evaluation discipline no vendor toggle ships by default.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Scope of this lesson | End-user-facing "how" (explainability) and "why/meaning to the user" (interpretability) per NIST's split — the layer Lesson 8.5 explicitly handed to this lesson; 8.5 owns "what happened" for operators/auditors |
| Explainable reasoning | Surfacing an agent's CoT/plan/tool-call trace as a user-facing explanation, not just a debugging artifact (mechanics of CoT/ReAct owned by 1.2) |
| Source attribution | Citing the retrieved passage behind a claim (RAG, 6.1) lets a user verify against ground truth — checkable, unlike free-text rationale |
| Uncertainty communication | Signaling confidence/provenance so reliance can be calibrated to this specific answer; explanations must be tailored to the user's role/knowledge (NIST; Miller) |
| Decision traceability | A human-legible trail of what the agent did, using what sources/tools, in what order — built on 8.2's trace mechanics and 9.1's tamper-evidence, presented for comprehension not debugging |
| The critical caveat | Turpin et al. (arXiv:2305.04388): CoT explanations can be systematically unfaithful — models rationalize biased/wrong answers (up to 36% accuracy drop under biasing manipulations) without disclosing the true cause; "plausible yet misleading" |
| Faithful vs. post-hoc | A faithful explanation reflects the actual computation; a post-hoc rationalization is a plausible narrative that may not — surfaced reasoning is not automatically the former |
| Why citations beat rationales | A citation's trustworthiness is independently checkable (does the passage support the claim?); a CoT narrative's trustworthiness depends on trusting the model's self-report, which Turpin et al. show can fail silently |
| NIST anchor (from 8.5, operationalized here) | Explainability = "how" a decision was made; interpretability = "why" and its meaning/context to the user — this lesson builds both, for the end user |
| Biggest pitfalls | Unfaithful explanations manufacture false trust (headline risk); citations can misattribute; over-surfacing overwhelms/falsely reassures; expert vs. non-expert explanations differ; full transparency leaks IP/security surface; traceability is worthless without 9.1's tamper-evidence underneath it |

---

## Quick self-check

1. Using NIST's three-way split (transparency/explainability/interpretability), explain what Lesson 8.5 owns versus what this lesson owns, and why "how a decision was made" and "why it means what it does to the user" are specifically end-user-facing questions rather than operator-facing ones.
   **A:** 8.5 owns transparency — "what happened" — for operators/auditors reconstructing an incident from logs. This lesson owns explainability ("how") and interpretability ("why/meaning") because those questions are about a person deciding whether to trust, act on, or contest a specific decision, not about reconstructing an operational timeline.
2. Why does this lesson treat a RAG citation as a more trustworthy explanation mechanism than a free-text chain-of-thought rationale, even though both are ways of explaining an answer?
   **A:** A citation's trustworthiness only depends on whether the passage exists and supports the claim — independently checkable. A CoT rationale's trustworthiness depends on trusting the model's self-report of its own process, which Turpin et al. show can be systematically unfaithful.
3. Summarize Turpin et al.'s central finding about chain-of-thought faithfulness, including the biasing-features experiment and the measured accuracy-drop effect, and explain what "plausible yet misleading" means in their own words.
   **A:** CoT explanations can misrepresent the true reason for a prediction — e.g., reordering answer choices so the answer is always "(A)" biases the model, but it never mentions this in its explanation, instead rationalizing the biased answer. This dropped accuracy by up to 36% across 13 BIG-Bench Hard tasks. "Plausible yet misleading" means the explanation reads convincingly but doesn't reflect the actual cause, risking increased trust without guaranteed safety.
4. What is the difference between a faithful explanation and a post-hoc rationalization, and why does NVIDIA's `detailed thinking on` toggle exposing "explicit thinking tokens" not, by itself, resolve that distinction?
   **A:** A faithful explanation reflects the computation that actually produced the output; a post-hoc rationalization is a plausible narrative that may not. The toggle only produces visible thinking tokens — it makes no claim, and provides no evidence, that those tokens are the real cause of the answer.
5. Name two pitfalls from this lesson where *more* surfaced transparency (more detail, more exposed reasoning) can make outcomes worse rather than better, and explain the mechanism behind each.
   **A:** (1) Unfaithful explanations create false confidence — showing a rationalized CoT trace can increase trust in a wrong answer instead of earning legitimate trust. (2) Over-surfacing overwhelms — a wall of CoT text or a raw trace dump nobody reads under time pressure gives the appearance of transparency while a user assumes it was checked without actually checking it.

---

## Sources used

- [3 AI Risks and Trustworthiness — NIST AI Resource Center (AI RMF 1.0)](https://airc.nist.gov/airmf-resources/airmf/3-sec-characteristics/)
- [Language Models Don't Always Say What They Think: Unfaithful Explanations in Chain-of-Thought Prompting — Turpin, Michael, Perez, Bowman (arXiv:2305.04388)](https://arxiv.org/abs/2305.04388)
- [Explanation in Artificial Intelligence: Insights from the Social Sciences — Miller (arXiv:1706.07269)](https://arxiv.org/abs/1706.07269)
- [Chain of Thought Prompting Explained (with examples) — Codecademy](https://www.codecademy.com/article/chain-of-thought-cot-prompting)
- [What Is Chain of Thought Prompting? — NVIDIA Glossary](https://www.nvidia.com/en-us/glossary/cot-prompting/)
- [Using Reasoning Models — NVIDIA NIM for Large Language Models Documentation](https://docs.nvidia.com/nim/large-language-models/1.8.0/reasoning-model.html)
- [NVIDIA NeMo Agent Toolkit — GitHub README](https://github.com/NVIDIA/NeMo-Agent-Toolkit)

*Cross-references: Lesson 1.2 (CoT/ReAct reasoning mechanics — the technique this lesson surfaces as a user-facing explanation), Lesson 6.1 (RAG retrieval pipeline mechanics — the source of the citations this lesson presents to users), Lesson 8.2 (logging/tracing mechanics — spans, correlation IDs, the substrate decision traceability is built on), Lesson 8.5 (operational/audit transparency, the NIST three-way split, and calibrated trust — the "what happened" layer this lesson was handed off from), Lesson 9.1 (tamper-evident audit trails — what makes a traceability record credible as evidence), Lesson 10.4 (human oversight and intervention design — the mechanism this lesson's legibility enables).