## In one line

**Licensing** (private contract law: am I allowed to use this model/data, on what terms?) and **regulation** (public law: what does the government require of a system like mine?) are two independent compliance gates a deployment must clear — neither is optional scaffolding, and passing one says nothing about the other.

---

## Why this matters

- A team ships the best-benchmarking open-weight model inside a product with 800M MAU, then discovers the license requires asking the publisher's permission to keep using it commercially at that scale. A separate team ships a RAG agent quoting scraped articles back verbatim, then asks — too late — whether they had the right to retrieve and republish that content.
- Both teams built something that worked; neither asked, early enough, whether they were allowed to. A model you cannot legally ship, or a deployment that violates a jurisdiction's AI law, is not a shipped product, regardless of how good the technology is.

---

## The core idea

- **Licensing** answers "am I allowed to use this model, this training data, this retrieved content, and on what terms?" — contract and IP law between you and whoever owns the weights/corpus/content. Apache 2.0 vs. Llama Community License vs. a closed API's ToS are all licensing questions.
- **Regulation** answers "what does the government where I'm operating require of a system like mine?" — public law, applies regardless of license. EU AI Act, NIST's voluntary framework, FDA medical-device rules, GDPR all sit here.
- A team can be compliant on one axis and exposed on the other: a permissive Apache-2.0 model doesn't exempt you from EU AI Act high-risk obligations; regulatory transparency compliance doesn't protect you from a licensing breach if you exceed a commercial-use threshold.
- **Scope note:** 9.2 owns privacy guardrails (runtime GDPR enforcement); 9.1 owns audit trails (the evidence a regulator would ask to see); 9.3 owns bias as a harm. This lesson is the knowledge layer underneath: which licenses/laws apply, and how to figure that out.

---

## Part A: Licensing — the spectrum from open to closed

**Truly permissive licenses**
- **Apache 2.0 / MIT** — broad grants to use, modify, redistribute, including commercial use, no field-of-use restriction, no usage-scale threshold. Apache 2.0 adds an explicit **patent grant**, foreclosing a later patent-infringement claim from the licensor. Practically unrestricted: no AUP to violate, no scale trigger, no permission needed at any size.

**"Open-weight but restricted" — the Llama Community License as the canonical case**
- Looks open (downloadable, free, commercially usable) but carries real restrictions. Meta grants "a non-exclusive, worldwide, non-transferable and royalty-free limited license... to use, reproduce, distribute, copy, create derivative works of, and make modifications." Conditions to track:
  - **Incorporated Acceptable Use Policy** — Section 1(b)(iv) makes AUP compliance a license condition (not just a ToS annoyance); prohibits illegal activity, harassment, discrimination in employment/credit/housing, unauthorized practice of licensed professions, military/weapons/nuclear use, critical-infrastructure operation, self-harm content, undisclosed AI-generated content, fake engagement.
  - **The 700-million-MAU commercial gate** — Section 2: if the licensee's products/services exceed 700M MAU in the preceding calendar month (as of the release date), "you must request a license from Meta, which Meta may grant to you in its sole discretion." A small startup never hits this; a hyperscaler-adjacent product does, and Meta decides.
  - **Attribution/naming** — "Built with Meta Llama 3" notice, Notice-file attribution; derivative models must start their name with "Llama 3."
  - **No-improvement clause** — may not use Llama Materials/outputs "to improve any other large language model" besides Llama itself or its derivatives.
- **Why it's not "open source":** the Open Source Initiative states directly this "puts restrictions on commercial use for some users (paragraph 2) and also restricts the use of the model and software for certain purposes (the Acceptable Use Policy)," violating OSD points 5 and 6 (no discrimination against persons/groups/fields of endeavor). OSI: an Open Source license gives users "sovereignty over the technology they use" without needing to engage another party — a Llama-style license removes exactly that above a certain scale or use case. **Open-weight (downloadable, inspectable parameters) ≠ open-source (OSD-compliant)** — conflating the two is the single most common licensing mistake.

**Fully proprietary / API-only models**
- No weights downloaded, interaction only via hosted API under vendor ToS. No redistribution issue, typically no AUP as elaborate as Llama's — but also no self-hosting, no visibility into what's running, and you inherit whatever the vendor's terms say about your data, outputs, and their right to change/discontinue the model.

**Training-data licensing, copyright, and RAG**
- A separate question underneath the model: do you have the right to train on, or retrieve from, a given dataset? **Genuinely unsettled area of law.** The EU AI Act's response is regulatory documentation, not resolution: GPAI providers must "establish a policy to respect the Copyright Directive" and "publish a sufficiently detailed summary about the content used for training the GPAI model."
- **Same uncertainty applies to RAG at inference time** — a pipeline that retrieves and republishes content scraped or licensed for a narrower purpose carries the same risk as training on unlicensed data, independent of the LLM's own license. A perfectly-licensed Apache-2.0 model serving RAG results from an unlicensed corpus is not a compliant deployment just because the model was.

**Output, IP, and indemnification**
- **Ownership of outputs:** NVIDIA's Open Model License: "NVIDIA claims no ownership rights in outputs. You are responsible for outputs and their subsequent uses" — the vendor disclaims outputs, so infringement risk in generated content sits with the deploying team.
- **Indemnification runs the same direction:** NVIDIA's license requires "You will indemnify and hold harmless NVIDIA from and against any claim by any third party arising out of or related to your use or distribution of the Model, Derivative Models or outputs" — language that recurs, in substance, across essentially every model license. The vendor is protecting itself contractually, not accepting your legal risk.

**Practical takeaway**
- License compliance is a real deployment gate, checked *before* a model is finalized into product architecture — a licensing blocker discovered post-integration is far more expensive than one caught during model selection.

---

## Part B: Regulation — the risk-based landscape

**The EU AI Act's four risk tiers**
- **Prohibited (unacceptable risk):** an enumerated banned list — subliminal/manipulative/deceptive techniques causing significant harm; exploiting vulnerabilities tied to age/disability/socio-economic status; biometric categorization inferring race/political opinion/sexual orientation; social scoring; crime-risk assessment based solely on profiling/personality traits; untargeted facial-image scraping for recognition databases; workplace/school emotion inference (except medical/safety); real-time remote biometric ID in public spaces for law enforcement (banned except narrow, procedurally-gated exceptions — missing persons, imminent threat to life, identifying suspects in specific serious crimes, each needing authorization and generally a fundamental-rights impact assessment).
- **High-risk:** the bulk of substantive obligations. A system is high-risk if it's a safety component/product under existing EU product law's third-party conformity assessment, or falls into an Annex III category (biometrics, critical infrastructure, education, employment/worker management, essential public/private services, law enforcement, migration/asylum/border control, justice/democratic processes) — unless a narrow carve-out applies (narrow procedural task, improving completed human work, detecting deviations without replacing human judgment, preparatory task). Providers must (Articles 8–17): run a lifecycle risk-management system; ensure representative/error-free data governance; produce technical documentation; enable automatic record-keeping; supply instructions for downstream deployers; design for human oversight; meet accuracy/robustness/cybersecurity standards; maintain a quality-management system.
- **Limited risk:** transparency-only — users must know they're interacting with AI (chatbots, disclosed/labeled AI-generated content/deepfakes).
- **Minimal risk:** unregulated — currently the majority of AI applications (spam filters, AI-enabled games), though the mix is shifting as generative AI spreads.
- Obligations fall mainly on **providers** (develop/place the system on the market); **deployers** ("users," organizations putting a system into professional use) carry lighter but real obligations. **Extraterritorial:** a non-EU provider is covered "where the high risk AI system's output is used in the EU" — same logic applies to deployers.

**GPAI and systemic-risk obligations**
- Every GPAI provider: technical documentation, downstream-integrator information, Copyright Directive compliance, training-content summary. **Free/open-licence GPAI models** (weights, architecture, usage terms all public) are exempt from the first two — *unless* deemed systemic risk.
- **Systemic risk trigger:** cumulative training compute exceeding **10²⁵ FLOPs** — crossing it requires notifying the European Commission within two weeks (provider may still argue no systemic risk despite meeting the criterion). Systemic-risk providers (open or closed) take on: documented adversarial testing/model evaluations, systemic-risk assessment and mitigation, incident tracking/reporting, cybersecurity protections.

**The compliance timeline** (measured from the Act's entry into force)
- Prohibited-practice rules: **6 months**. GPAI obligations: **12 months**. High-risk Annex III systems: **24 months**. High-risk Annex I systems: **36 months** — bans move fastest, highest-obligation systems get the longest runway.

**NIST AI RMF: the voluntary US counterpart**
- **AI RMF 1.0** (Jan 26, 2023) is explicitly **voluntary**: "intended for voluntary use and to improve the ability to incorporate trustworthiness considerations into the design, development, use, and evaluation of AI products, services, and systems." A process framework, not a legal gate, with four functions:
  - **Govern** — cross-cutting; "cultivates and implements a culture of risk management"; Govern 1.1 requires "legal and regulatory requirements involving AI are... understood, managed, and documented"; Govern 6 covers "AI risks and benefits arising from third-party software and data and other supply chain issues" — ties governance directly to this lesson's licensing question.
  - **Map** — establishes context (intended purpose, risk tolerance); Map 4.1: "approaches for mapping AI technology and legal risks of its components – including the use of third-party data or software... as are risks of infringement of a third party's intellectual property or other rights."
  - **Measure** — quantitative/qualitative TEVV (test, evaluation, verification, validation) testing on trustworthiness characteristics, including privacy risk (Measure 2.10) and fairness/bias (Measure 2.11, 9.3's territory).
  - **Manage** — allocates resources to identified risks: prioritize, respond, monitor, including incident response and decommissioning.
- Better understood as the operational discipline to *discover and document* regulatory obligations (Govern 1.1's own language) than a substitute for knowing what law applies. NIST itself flags AI RMF 1.0 is currently being revised — an evolving, not static, framework.

**Sector-specific regimes**
- **FDA / AI-ML in Software as a Medical Device (SaMD):** reviews AI-enabled devices through existing pathways (510(k), De Novo, premarket approval), while stating directly "the FDA's traditional paradigm of medical device regulation was not designed for adaptive artificial intelligence and machine learning technologies." Instead of one new law, the FDA layered AI-specific guidance onto its existing framework: 2019 modification-framework discussion paper → Jan 2021 "AI/ML SaMD Action Plan" → Oct 2021 "Good Machine Learning Practice" → 2023–2024 "Predetermined Change Control Plans" (lets a manufacturer pre-specify post-market model updates without a fresh submission each time) → June 2024 transparency principles → Jan 2025 draft lifecycle-management guidance. **Pattern:** sector regulators tend to extend existing regimes for AI rather than start over — "what regulates my agent" can be sector guidance layered on decades-old law, not a single AI statute.
- **GDPR** — the EU's foundational data-protection law; not re-taught here (9.2 owns the runtime guardrails — PII masking, access control, retention limits — that operationally enforce it). What matters here: GDPR is independent regulation from the EU AI Act — a deployment can satisfy the AI Act's transparency rules while violating GDPR's data-protection principles, or vice versa. Other regulated sectors (finance chief among them) layer their own model-risk/disclosure rules similarly; not detailed further here since doing so accurately requires sector-specific sources this lesson didn't cover.

**The practitioner's job**
- **Map your specific use case to its risk tier and applicable regime, then implement the concrete obligations.** EU AI Act: is the system prohibited, Annex III high-risk, or only limited-risk — and if high-risk, build the Articles 8–17 documentation/risk-management/human-oversight design. NIST: run Govern-Map-Measure-Manage to actually produce the "legal and regulatory requirements... documented" artifact. Sector law: check whether a regulator like the FDA already has an AI-specific regime for your domain. Across all of it: 9.1's audit trails are what you show a regulator asking for proof of oversight/incident response; 9.3's bias mitigation is itself a regulated harm under both the AI Act's data-governance requirement and NIST's Measure 2.11.

---

## Trade-offs and pitfalls

- **Open-weight is not open-source, and the gap catches teams by surprise.** Llama's 700M-MAU clause and incorporated AUP are exactly the trap: "I can download the weights for free" ≠ "I can do anything with this model" — discovered only when usage crosses a threshold or hits an unscreened AUP violation.
- **Regulation is fast-moving and jurisdiction-specific.** A deployment compliant in one jurisdiction isn't automatically compliant elsewhere — a US-only system can trigger EU obligations the moment its output reaches an EU user (extraterritorial framing). "We're compliant" is a jurisdiction-scoped claim, not a global one.
- **"We used an API model, so it's fine" ignores the deployer's own obligations.** The EU AI Act imposes obligations on deployers/users of high-risk systems, not just providers — using someone else's model via API doesn't transfer regulatory responsibility to them.
- **Over-compliance and under-compliance are both real failure modes.** Treating every system as high-risk imposes overhead that can stall shipping without proportional benefit; treating a genuinely high-risk system as minimal-risk ("it's just a chatbot") is the liability version of the same mistake. Getting the risk-tier mapping right is the actual skill.
- **NIST AI RMF is voluntary — following it isn't automatic legal compliance.** Genuinely useful operational discipline (Govern 1.1 even points at figuring out legal obligations), but creates no legal safe harbor — a process for finding/managing risk, not a substitute for the underlying law.
- **Training-data and copyright law for AI is genuinely unsettled.** The EU AI Act's Copyright Directive/training-summary obligations are a response to this uncertainty, not proof it's resolved. Treat "can I train or retrieve on this data" as an open legal question for counsel beyond clearly-licensed content, not a one-time checkbox.
- **A vendor's model license doesn't discharge your regulatory obligations.** NVIDIA's Open Model License requires keeping its safety "Guardrails" intact (bypassing without a substitute terminates the license automatically) and complying with a separate "Trustworthy AI" terms document — but complying with a vendor's contractual safety terms ≠ complying with the EU AI Act's high-risk obligations or GDPR. Using a vendor's model doesn't transfer your regulatory obligations to them; it only adds their contract terms on top of your own exposure.

---

## NVIDIA angle, read critically

- **NVIDIA's Open Model License** (Nemotron family) is a genuine improvement over Llama's structure in one respect: no MAU threshold, no separate-license requirement at any commercial scale — "NVIDIA models released under this Agreement are intended to be used permissively," "commercially usable," licensees "free to create and distribute Derivative Models," "NVIDIA does not claim ownership to any outputs." Structurally more open than Llama 3's Section 2 commercial gate.
- **But two restrictions carry the same underlying logic as an AUP, just relocated:**
  - The license is **revocable**, automatically, if a licensee bypasses/disables/"reduce[s] the efficacy of" any "technical limitation, safety guardrail... encryption, security, digital rights management, or authentication mechanism" without substituting "a substantially similar Guardrail appropriate for your use case" — a real constraint on how freely a team can fine-tune or modify the model.
  - Use "must be consistent with NVIDIA's Trustworthy AI terms" — a separate incorporated document functioning like Llama's AUP, a field-of-use restriction dressed differently.
  - Same indemnification pattern as elsewhere: the licensee, not NVIDIA, bears legal risk for their own use, outputs, distribution.
- **Read critically:** NVIDIA's marketing repeatedly frames "trust" and "compliance" as product features, but a permissive-sounding license, an AI-ethics clause, and a guardrail-preservation condition are NVIDIA managing its own legal/reputational exposure — they don't and can't satisfy the EU AI Act's high-risk obligations, GDPR's data-protection duties, or FDA device-software rules on your behalf. A well-drafted, commercially generous license is a real advantage over a Llama-style MAU gate — but it's a licensing decision, not a regulatory compliance program. **The deployer, not the model vendor, is who a regulator holds accountable** for how the deployed system actually behaves.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Licensing vs. regulation | Licensing = private contract law governing your right to use a model/data (Apache-2.0, Llama license, vendor ToS); regulation = public law governing how you may build/operate a system (EU AI Act, NIST RMF, GDPR, FDA) — both are separate compliance gates |
| Permissive licenses | Apache 2.0 / MIT — free use, modification, redistribution, including commercial, no field-of-use restriction, no usage-scale threshold |
| Open-weight ≠ open-source | Llama Community License: weights downloadable and commercially usable, but gated by an incorporated Acceptable Use Policy and a 700-million-MAU clause requiring a separate license from Meta above that threshold; OSI states this violates OSD points 5–6 (no discrimination against persons/fields of use), so it is not "open source" despite being widely called "open" |
| Training-data / RAG licensing | Whether you may train on, or retrieve and republish, a given dataset is a live, unsettled legal question — the EU AI Act's Copyright Directive and training-summary obligations are a regulatory response to that uncertainty, not a resolution of it; RAG over unlicensed content is a compliance risk independent of the model's own license |
| Output/IP and indemnification | Model licenses (e.g., NVIDIA's) typically disclaim vendor ownership of outputs and require the licensee to indemnify the vendor — legal risk for outputs sits with the deployer, not the model provider |
| EU AI Act's four risk tiers | Prohibited (banned outright, e.g. social scoring, manipulative techniques) → High-risk (Annex I/III use cases; risk management, data governance, technical docs, human oversight, etc.) → Limited-risk (transparency duties: disclose AI interaction, label AI content) → Minimal-risk (unregulated) |
| GPAI / systemic risk | All GPAI providers: technical docs, downstream info, Copyright Directive compliance, training-summary; free/open-licence models exempt from the first two unless systemic risk (>10^25 training FLOPs), which adds model evals, adversarial testing, incident reporting, cybersecurity |
| NIST AI RMF | Voluntary US framework, four functions: Govern (culture, incl. "legal and regulatory requirements... understood, managed, documented"), Map (context and legal/IP risk of components), Measure (TEVV testing incl. privacy and bias), Manage (prioritize/respond/monitor risk) — a process, not a legal safe harbor |
| Sector law | FDA regulates AI/ML medical-device software through existing premarket pathways plus AI-specific guidance (GMLP, Predetermined Change Control Plans) layered on top, rather than one new AI statute; GDPR is the EU's data-protection law, operationally enforced by 9.2's guardrails |
| The practitioner's job | Map your specific use case to its risk tier (EU AI Act) and applicable regime (NIST RMF process, sector law), then implement the concrete obligations that follow — documentation, transparency, human oversight, risk management, audit trail (9.1) |
| Biggest pitfalls | Open-weight/open-source confusion (Llama MAU trap); assuming US compliance covers the EU or vice versa; "we used an API model" doesn't transfer the deployer's own obligations; NIST RMF is voluntary, not a legal safe harbor; training-data copyright law is unsettled — don't overstate certainty; a vendor's license terms don't discharge your regulatory obligations |

---

## Quick self-check

1. Explain why the Llama 3 Community License is described as "open-weight but not open-source," citing the two specific clauses (per the license text and OSI's analysis) that disqualify it from meeting the Open Source Definition.
   **A:** The incorporated Acceptable Use Policy (field-of-use restrictions) and the 700-million-MAU commercial gate (requiring a separate license from Meta above that threshold) violate OSD points 5 and 6, which bar discrimination against persons/groups or fields of endeavor — per OSI's own statement that this structure "is not Open Source."
2. Walk through the EU AI Act's four risk tiers and give one example use case for each, including what makes an Annex III use case high-risk unless a specific carve-out applies.
   **A:** Prohibited (e.g., social scoring), High-risk (e.g., employment/worker-management systems under Annex III — high-risk unless a narrow carve-out applies, such as a preparatory task or improving a completed human activity's result), Limited-risk (e.g., a chatbot, which must disclose it's AI), Minimal-risk (e.g., a spam filter, unregulated).
3. What is the difference between the EU AI Act (as applied here) and the NIST AI RMF in terms of legal force, and what does the AI RMF's Govern function require regarding third-party data and software specifically?
   **A:** The EU AI Act is binding law with tiered obligations; NIST's AI RMF is explicitly voluntary, a process framework with no legal safe harbor. Govern 6 requires addressing "AI risks and benefits arising from third-party software and data and other supply chain issues," directly tying governance to the licensing question this lesson covers.
4. A team deploys a RAG agent that retrieves and quotes scraped web content back to users, using a model licensed under Apache 2.0. Why is this deployment not automatically compliant just because the model's license is permissive?
   **A:** The model's license only covers rights to the model itself; whether the team had the right to retrieve and republish the scraped content is a separate, unsettled copyright question about the source content, independent of what license the LLM carries.
5. Why does the lesson argue that "we used an API model, so it's fine" is a mistaken compliance argument, and how does the EU AI Act's provider/deployer distinction support that conclusion?
   **A:** The EU AI Act imposes real obligations on deployers/users of high-risk systems, not just on providers — using someone else's model through an API doesn't transfer your regulatory responsibility to them; the vendor's compliance work covers their model, not your deployment.

---

## Sources used

- [High-level summary of the AI Act — EU Artificial Intelligence Act](https://artificialintelligenceact.eu/high-level-summary/)
- [AI Risk Management Framework — NIST](https://www.nist.gov/itl/ai-risk-management-framework)
- [AI RMF — AIRC (NIST AI Resource Center)](https://airc.nist.gov/airmf-resources/airmf/)
- [AI RMF Core (Govern, Map, Measure, Manage functions) — AIRC](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/)
- [AI RMF Playbook — AIRC](https://airc.nist.gov/airmf-resources/playbook/)
- [Meta Llama 3 Community License Agreement — GitHub (meta-llama/llama3)](https://raw.githubusercontent.com/meta-llama/llama3/main/LICENSE)
- [Meta's LLaMa license is not Open Source — Open Source Initiative](https://opensource.org/blog/metas-llama-2-license-is-not-open-source)
- [Artificial Intelligence in Software as a Medical Device — FDA](https://www.fda.gov/medical-devices/software-medical-device-samd/artificial-intelligence-software-medical-device)
- [NVIDIA Open Model License Agreement — NVIDIA](https://www.nvidia.com/en-us/agreements/enterprise-software/nvidia-open-model-license/)

*Cross-references: Lesson 9.2 (privacy guardrails — the runtime enforcement mechanism for GDPR-style rules this lesson introduces as regulation), Lesson 9.1 (audit trails — the evidence artifact a regulator or auditor would ask for to verify the obligations this lesson maps), Lesson 9.3 (bias mitigation — a specific harm both the EU AI Act's high-risk data-governance requirement and NIST's Measure 2.11 treat as a regulated concern), Lesson 9.4 (the layered safety framework these compliance obligations sit alongside).
