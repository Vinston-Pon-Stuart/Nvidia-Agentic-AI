## In one line

A **structured feedback loop** is the designed interaction that turns a user's approval, edit, or correction into a signal that's captured, routed, and eventually produces a visible improvement — not just a one-time transaction that vanishes the moment an interrupt resumes.

---

## Why this is its own topic

- Lesson 10.1 built the interrupt-and-resume primitives that let a human approve, edit, or redirect an agent mid-task — but explicitly left open what happens to those decisions *afterward*.
- An approval or correction that disappears the instant the interrupt resumes isn't a feedback loop, it's a one-time transaction.
- **Adjacent territory this lesson does NOT re-teach:**
  - **Lesson 2.5** — real-time *capture* mechanics: thumbs, ratings, streaming UI; explicit feedback (unambiguous, ~1–3% response rate) vs. implicit signals (rephrase, abandon, regenerate — full coverage, ambiguous).
  - **Lesson 3.3** — turning captured feedback into *training* data: collection-quality discipline, preference pairs, RLHF/DPO/SFT pipelines.
  - **Lesson 8.4** — the automated retraining loop that data feeds into.
  - **Lesson 8.3** — the GO/NO-GO evaluation gate a retrained candidate must clear before promotion.
  - **Lesson 5.5** — how an agent adapts its reasoning *within* a run, no weight update.
- **What none of those own**: the *shape* of the loop as a user experiences it — the interface that solicits structured feedback, what makes people bother to give it, and where a given piece of feedback gets routed. That's this lesson.

---

## The core idea — five stages

A closed loop: **user interacts → gives feedback at a designed touchpoint carrying more than a click → it's collected and routed → the destination produces an improvement → the user perceives the improvement**, which is what makes them give feedback again. Break any stage and the loop doesn't close — it just generates unused data, indistinguishable from not collecting it at all.

- **Holistic AI's framing**: "Continuous Human Involvement" across the AI lifecycle, plus a named "Continuous Feedback Loop" property — "human involvement continues through ongoing monitoring and refinement... allowing the AI to adapt to new data and challenges."
- **NVIDIA's Data Flywheel** gives this a concrete six-step production shape: data processing → model customization → model evaluation → guardrail implementation → deployment → "enterprise data refinement" (inference logs + human/AI feedback feed back into data processing).
- **Scope boundary**: the flywheel's six steps *start* at data processing — after feedback has already been solicited, structured, and captured. Everything upstream (how the ask is designed, why users answer, which of two destinations a piece of feedback should route to) is this lesson's subject, not the flywheel's.

---

## Signal-quality spectrum: thumbs vs. granular feedback

- **The trade-off**: every feedback interface costs the user something to answer, and carries some amount of information — the two move in opposite directions as structure increases.
- **Binary thumbs** — costs one click, which is why it only pulls **1–3% response** (Lesson 2.5), skewed toward the delighted or furious. Cheap to give, but a single bit tells a downstream system nothing about *what* was good/bad, *which part* of a response, or *what the user wanted instead*.
- **Microsoft's G15 (encourage granular feedback)** — "enable the user to provide feedback indicating their preferences during regular interaction." Example: Excel's "Ideas" asks "Is this helpful?" **per suggestion**, not once per session — tying a rating to a specific unit of output is what makes it "granular" rather than merely "explicit."
- **LangSmith's feedback API** (Agent Server runs) — a concrete mechanism: feedback attaches to a specific run via `feedback_keys` set at run creation; the response returns a pre-signed URL per key to post to. Schema is flexible by design — boolean keys, a numeric score, or a richer rubric. Crucially includes a `correction` field alongside `score`/`value`/`comment` — a slot for "not just whether this was wrong, but what it should have been."
- **NN/g's guidance**: "**aim for quality over quantity**" — 2–3 minute surveys are already too long; target ~1 minute; prefer closed-ended (ratings, multiple choice) over open-ended. Their compromise: structured, low-friction options first, then an *optional* open-ended field at the end — buys some granularity without the full response-rate cost.
- **Exam point**: coverage and signal trade off as structure increases; no design eliminates the tension, only chooses where to sit on it.

### Edits as feedback — the highest-signal case

- **What it is**: a user editing an agent's draft instead of scoring it. The edit doesn't just say "wrong" — it shows exactly what "right" looks like, free, as a byproduct of work already happening.
- Licensed by **Microsoft's G9 (support efficient correction)** — same design move Lesson 10.1 uses for editable-plan interfaces, but this lesson's concern is what happens to the correction *after* it's made.
- **Thoughtworks example (Rahul Garg)** — a "feedback flywheel" for AI coding assistants: a developer fixes a missing authorization check, then logs one line ("Authorization checks on new endpoints not enforced by generation instruction") into a shared doc future sessions draw on. "The generation instruction did not change. The priming context changed. The system learned. That is the flywheel." No model weights touched.
- **Principle to generalize**: the cheapest way to get high-signal feedback is to make giving it a side effect of a correction the user is already motivated to make — not a separate ask competing for attention.

---

## Making the loop visible: why users bother

- **Microsoft's G16 (convey the consequences of user actions)** — "immediately update or convey how user actions will impact future behaviors of the AI system." A user who corrects an agent and sees no change has learned, correctly, that feedback doesn't do anything — which compounds badly with the already-thin 1–3% baseline: an unclosed loop doesn't just fail to raise that number, it erodes it further as burned users stop bothering.
- **NN/g's request-design guidance** (the front-end mirror of G16): use microcopy explaining how feedback will be used; "task, then ask" (right after a completed task, never mid-task or before); keep it short; offer flexible formats.
- **Cost of not closing the loop**: Garg — a practice with no visible payoff "will be the first thing cut when the team is busy — which is precisely when learning matters most." Any structured feedback practice that doesn't visibly change something is optimized to be abandoned, not sustained.
- **Fastest close** — apply the correction in-context on the very next turn (Lesson 5.5's territory): a directly perceivable signal ("it remembered what I told it"), unlike a training run that might help everyone in three weeks but shows nothing to the person who reported the problem. This timing gap is why feedback often needs to flow to two destinations at once.

---

## Two destinations, two speeds

- **In-context reuse, without retraining** — a correction/preference is stored and reused on later turns/sessions; no weight update, just a memory/context store (Lesson 5.5's mechanism). **Fast**: the reporting user sees it applied, often same session. **Narrow**: scoped to what was stored and where retrieved, not a model-wide change.
- **The training and evaluation flywheel** — feedback clearing Lesson 3.3's collection-quality bar (volume, inter-annotator agreement, structured preference pairs/gold examples) becomes RLHF/DPO/SFT data. NVIDIA's Data Flywheel names the cycle: curate → customize (LoRA/DPO/SFT) → evaluate → guardrail → redeploy, with "Human-in-the-loop and AI systems" reviewing generated institutional knowledge and adjusting the database. **Lesson 8.3's** GO/NO-GO gate decides promotion; **Lesson 8.4** owns the automated retrain mechanics. **Slow** (a release cycle) and **diffuse** (helps everyone, not visibly the one person who filed the correction).
- **Why both are often needed together**: apply immediately in-context so the reporting user sees the loop close today, *and* log to the flywheel so the model improves for everyone next cycle. LangSmith's `correction` object supports exactly this split — informs the running session immediately while remaining available for later eval/training pulls. Closing the loop honestly means closing it **twice**, on two clocks, for two audiences.

---

## Trade-offs and pitfalls

- **Signal vs. coverage is positionable, not solvable** — more structure always raises signal and lowers response count; NN/g's "quality over quantity, ~1 minute" is a deliberate choice of position, not an escape from the trade-off.
- **Feedback bias compounds with added friction** — explicit feedback already skews to the two extremes (Lesson 2.5); a longer, more structured ask selects even harder for the unusually invested, underrepresenting the "it was fine" middle.
- **Unclosed loops kill motivation, not just miss an opportunity** — Microsoft's G16 and Garg's "first thing cut" describe the same decay from two domains: no visible consequence → people correctly conclude it isn't worth their time.
- **An uncurated loop is an attack surface** — anyone who knows corrections feed the flywheel can submit bad-faith feedback to steer future behavior; guarded against by Lesson 3.3's collection-quality discipline and Lesson 6.4's data-curation territory.
- **Over-personalizing from sparse in-context feedback** — a single correction (Lesson 5.5's mechanism) is easy to over-generalize, turning one user's one-off preference into an unrequested behavior change applied too broadly.
- **Neither destination substitutes for the other** — in-context is fast but narrow and un-audited; the flywheel is thorough (passes 8.3's gate) but slow and diffuse. Doing only one leaves a real gap: either the same mistake recurs for every new user until the next release, or a correcting user never sees it matter before then.

---

## NVIDIA angle (read critically)

- NVIDIA's Data Flywheel page is a genuinely coherent systematization of the **slow-closing destination** — six-step cycle, explicit human-in-the-loop review role, credit-worthy account of feedback → retrained, redeployed model at scale.
- **Precondition it assumes, doesn't teach**: the page admits "without a centralized system for feedback and logging, it's difficult to track and analyze system performance" — implicitly assuming well-designed feedback capture already exists. Its diagram's first step ("data processing") comes *after* the work Microsoft's guidelines, NN/g's request design, and LangSmith's feedback schema actually cover. Adopting the flywheel without first designing the feedback interface optimizes six well-engineered steps on input that may be mostly noise or silence.
- **Two claims worth skepticism**: (1) "over 98% savings in inference costs — all without compromising accuracy" — no methodology, benchmark, or attribution on the public page; a vendor marketing claim, not an audited result. (2) The role table collapses "Human-in-the-loop and AI systems" into a single row for reviewing/adjusting — flattening what this lesson shows are two distinct jobs (designing the ask so people answer it, and curating what they submit before it trains anything).
- **Bottom line**: NVIDIA's flywheel is a real, useful description of the pipeline — not a description of the interaction design this lesson covers, which sits upstream of where its diagram begins.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Closed feedback loop, five stages | Interact → structured feedback at a touchpoint → collected/routed → drives an improvement → user perceives it, which sustains future feedback; capture (2.5) alone is not a loop unless it closes |
| Signal-vs-coverage spectrum | Binary thumbs = cheap, ~1–3% response (2.5), low signal; granular (category tags, per-item ratings, a `correction` field) = higher signal, lower response rate unless kept short (NN/g); Microsoft G15 "encourage granular feedback" |
| Edits as the highest-signal case | A user's correction to a draft shows the *right* answer, not just "wrong" — Microsoft G9 "support efficient correction"; LangSmith's `correction` field; Garg's example of a fix logged as a one-line learning-doc entry |
| Making the loop visible | Microsoft G16 "convey the consequences of user actions"; NN/g: ask right after a real task, explain how feedback is used, keep it short; unclosed loops don't stay flat, they decay (Garg: "the first thing cut") |
| Two destinations | (a) in-context reuse without retraining — fast, narrow, closes today (bridges to 5.5); (b) training/eval flywheel — feedback cleared by 3.3's collection-quality bar → 8.3's GO/NO-GO gate → 8.4's automated retrain/promote, slow, diffuse (NVIDIA's Data Flywheel) |
| Why both destinations are often needed together | In-context closes the loop for the individual now; the flywheel closes it for everyone later — neither substitutes for the other |
| Biggest pitfalls | Signal/coverage trade-off is positional, not solvable; more structure worsens feedback-extremes bias; unclosed loops actively erode participation; uncurated feedback is an attack surface (3.3/6.4); sparse in-context corrections can over-generalize; latency mismatch between the two destinations |

---

## Quick self-check

1. Explain the five stages of a closed feedback loop as an interaction-design mechanism, and explain specifically why a feedback *capture* mechanism (Lesson 2.5's territory) is not automatically a closed *loop*.
   **A:** Interact → feedback at a touchpoint → collect/route → produce an improvement → user perceives it. Capture alone only covers stage two — without routing, an improvement, and a perceivable result, the data just sits unused, which is functionally the same as never collecting it.
2. Using the signal-vs-coverage trade-off, explain why a binary thumbs-up/down button and a `correction` field asking the user to rewrite the agent's draft sit at different points on the same spectrum — what does each gain, and what does each give up, relative to the other?
   **A:** Thumbs cost one click and get ~1–3% response, but the single bit carries almost no information about what was wrong or what was wanted instead. A `correction` field carries far more signal (the actual right answer) but costs more effort, so fewer people fill it in — coverage falls as signal rises.
3. Why is an edit a user makes to an agent's draft considered higher-signal feedback than a star rating? Connect your answer to Microsoft's "support efficient correction" guideline and the Thoughtworks feedback-flywheel example of a logged fix.
   **A:** An edit shows exactly what "right" looks like, not just that something was wrong — it's G9 (efficient correction) applied to feedback rather than just editing UI. Garg's example: a developer's authorization-check fix, logged as one line, changed the team's shared priming context with near-zero extra effort — the correction rode along with work already being done.
4. Name the two destinations structured feedback can flow to after it's collected, explain why they close the loop at different speeds, and explain why a well-designed system often needs to route the same piece of feedback to both.
   **A:** (1) In-context reuse without retraining — fast, closes within a session, but narrow. (2) The training/eval flywheel — slow (a release cycle), but improves the model for everyone. A system needs both so the reporting user sees an immediate fix while the correction also improves the agent broadly at the next cycle.
5. What does it mean to "make the loop visible," per Microsoft's "convey the consequences of user actions" guideline and NN/g's feedback-request guidance — and what specifically happens to a user's willingness to keep giving feedback if the loop never visibly closes?
   **A:** It means showing the user that their feedback changed something — immediately or eventually — and explaining upfront how it will be used. If the loop never visibly closes, users correctly conclude feedback doesn't matter, and participation doesn't stay flat, it decays.

---

## Sources used

- [Guidelines for human-AI interaction design (Amershi, Vorvoreanu, Horvitz et al., CHI 2019) — Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/)
- [Data Flywheel: What it is and how it works — NVIDIA Glossary](https://www.nvidia.com/en-us/glossary/data-flywheel/)
- [How to collect user feedback for Agent Server runs — LangSmith / LangChain Docs](https://docs.langchain.com/langsmith/agent-server-feedback)
- [Human in the Loop AI: Keeping AI Aligned with Human Values — Holistic AI](https://www.holisticai.com/blog/human-in-the-loop-ai)
- [Feedback Flywheel — Rahul Garg, martinfowler.com / Thoughtworks (April 2026)](https://martinfowler.com/articles/reduce-friction-ai/feedback-flywheel.html)
- [User-Feedback Requests: 5 Guidelines — Anna Kaley, Nielsen Norman Group](https://www.nngroup.com/articles/user-feedback/)

*Cross-references: Lesson 2.5 (real-time feedback capture UX — thumbs, implicit signals, streaming, the 1–3% explicit-response baseline), Lesson 3.3 (turning captured feedback into RLHF/DPO/SFT training data — collection-quality discipline), Lesson 8.3 (the GO/NO-GO evaluation gate a retrained candidate must clear), Lesson 8.4 (the automated retraining and promotion pipeline), Lesson 5.5 (in-context adaptation of reasoning from feedback, without retraining), Lesson 10.1 (user-in-the-loop interaction interfaces — the touchpoints where this loop's feedback actually gets solicited), Lesson 10.4 (human oversight and escalation authorization design — not yet covered in this vault), Lesson 6.4 (data curation and the model-collapse risk of uncurated feedback loops).