## In one line

An agent adapts **at run time** by turning its own failures, tool errors, and past experience into new text that conditions its next attempt — with **no weight update** — and this lesson is about that in-context mechanism only, not training-time methods (3.3) or the developer's eval loop (2.6).

---

## Scope note

- **2.6:** the *developer's* evaluation loop — LLM-as-judge, self-consistency, evaluator-optimizer — scoring/improving an agent from the outside, during development. Not covered here.
- **1.2:** places Reflexion inside the ReAct family as a reasoning-and-acting loop variant. Not repeated here.
- **3.3:** structured feedback becoming training data for RLHF/DPO — a permanent weight change. Not covered here.
- **This lesson's question:** what is the actual mechanism by which an agent's *in-context* behavior changes because of its own prior experience/feedback, and where does that mechanism run out of road?

---

## Two kinds of adaptation — this lesson covers only one

- Exactly two ways an agent's behavior changes after a mistake: **someone retrains it** (gradient descent updates weights — SFT, DPO, RLHF, RLVR — 3.3), or **the wrong attempt becomes new text fed into the next prompt**, changing what the same frozen model produces next.
- Reflexion's authors frame the second path as the only option for systems built on massive pretrained models: "such approaches have been so far limited to using in-context examples as a way of teaching the agents, since more traditional optimization schemes like reinforcement learning with gradient descent require substantial amounts of compute and time" (Shinn et al., arXiv:2303.11366). Reflexion is designed "to reinforce language agents not by updating weights, but instead through linguistic feedback."
- **Self-Refine** (Madaan et al., arXiv:2303.17651) makes the same claim: it "does not require any supervised training data, additional training, or reinforcement learning," improving outputs "at test time" with the same frozen model as generator, critic, and refiner.
- **The whole lesson in one line:** run-time adaptation changes what goes into the next prompt; training-time adaptation changes the weights that process every future prompt.

---

## Mechanism 1: self-reflection loops — the "policy update" is a paragraph of text

- **Reflexion's architecture:** three roles — an **Actor** (generates text/actions), an **Evaluator** (scores output), a **Self-Reflection model** (turns the score into language).
- After a trial, given a sparse reward (success/fail), the trajectory, and persistent memory *mem*, the self-reflection model generates specific feedback — e.g., inferring a specific action led to two subsequent wrong actions and stating "it should have taken a different action... which would have resulted in" a better outcome.
- That reflection is appended to a **persistent memory buffer**; the *next* attempt's Actor call is conditioned on it. Formally, Reflexion's policy is θ = {LLM parameters, *mem*} — the memory buffer, not the weights, is what actually updates between trials. The paper calls the resulting text **"a 'semantic' gradient signal"** — no backprop, but the reflection points the next attempt in a concrete direction the way a gradient would.
- **Results:** 91% pass@1 on HumanEval (vs. 80% GPT-4 baseline), +22 points absolute on AlfWorld over 12 iterative trials, +20 points on HotPotQA.
- **Self-Refine** is the same mechanism stripped down: one model alternates **feedback** and **refine** steps on a *single* generation, "repeated either for a specified number of iterations or until [the model] determines that no further refinement is necessary" — 5–40% absolute improvement across seven tasks (up to 13 points on code generation).
- **The exam-relevant distinction:** Reflexion's own comparison table marks Self-Refine with a check for "self-refine" but an explicit **✗ for "memory."** Self-Refine iterates within one generation and stops; Reflexion persists a reflection *across separate trials* via an explicit episodic buffer. **Self-Refine adapts within an attempt; Reflexion adapts between attempts.**

---

## Mechanism 2: learning from past trajectories via episodic memory (bridge to 5.1)

- 5.1 covers the mechanics (CoALA's taxonomy, Generative Agents' reflection-tree construction). **Here:** the adaptation angle — retrieving a specific past experience and reasoning over it is how an agent changes its *current* decision without touching weights.
- Generative Agents (Park et al., arXiv:2304.03442): retrieved memories "are also used to form longer-term plans and create higher-level reflections, both of which are entered into the memory stream for future use" — each act of reasoning over experience produces new material available to *later* reasoning, a run-time learning loop that compounds within a session purely through what gets written back into memory.
- The paper's controlled study isolated this effect, showing breakdowns from "improper memory retrieval" — evidence the memory-and-reflection loop measurably changes agent behavior, not just decoration.
- **Distinction from Mechanism 1:** Reflexion's buffer holds reflections about *this specific task*, replayed on retries of the same task. Episodic-memory recall pulls in a *different, related* past experience to inform a *new* decision — same underlying operation (retrieve text about the past, condition next generation on it), broader scope.

---

## Mechanism 3: in-context learning from live feedback signals

- Not every adaptation signal comes from a full post-trial reflection — some arrive mid-trajectory, one step at a time: a tool error, an ambiguous search result, a human correction typed into chat — all become the next observation in context, and the agent's next reasoning step reacts to it.
- Reflexion is explicit its framework is signal-agnostic: "flexible enough to incorporate various types (scalar values or free-form language) and sources (external or internally simulated) of feedback signals."
- **Distinct from 2.4** (resilience engineering — retries, circuit breakers around a failing tool call) and **2.6** (measurement — a judge scoring the trajectory): here the frame is what the agent's *reasoning* does once the error/score is in context — conditions the next Thought on it, exactly as 1.2's ReAct trace shows a bad Observation ("one of two distinct land regions") triggering a reformulated Action.
- Single-step reaction and Reflexion's cross-trial buffer are the **same underlying move** — condition the next generation on feedback already in context — at different timescales: one step vs. one full retry.

---

## Mechanism 4: self-improvement loops, and the limit that governs all of them

- Chaining feedback → reflect → retry into a repeatable loop is what Self-Refine and Reflexion both are: self-improvement without a training step.
- **The central exam trap:** self-reflection reliably improves an agent **only when it has an external or verifiable signal to reflect on.** Reflexion's own limitations section: the technique "does have the disadvantages of relying on the power of the LLM's self-evaluation capabilities (or heuristics) and not having a formal guarantee for success."
- **With ground truth** (Reflexion's programming setting — failed unit tests, compiler errors): the reflection has something real to fix.
- **Without it** (Reflexion's decision-making setting, self-graded by a separate LLM classifier as Evaluator): the model has only its own opinion — a confidently wrong model can talk itself into a confidently wrong reflection just as easily as a correct one.
- Mirrors 2.6's judge-reliability problem from the other side: it's not just judging that degrades without ground truth — an agent's own *adaptation* degrades the same way, because the critique step has no more access to truth than the generation step did.

---

## Where run-time adaptation hits its ceiling (pointer to 3.3)

- Run-time reflection fixes *this* attempt, *this* session. It does **not** durably fix a model that repeats the same mistake across thousands of unrelated sessions — that's a training-time problem.
- NVIDIA's agentic-RL guidance states the boundary as a decision rule: "If the agent repeats tool-call mistakes, fails in long workflows, formats outputs incorrectly, or chooses the wrong strategy, you need a training signal. That is where RL fits" (NVIDIA Technical Blog, "Mastering Agentic Techniques: AI Agent Reinforcement Learning," 2026). Its decision matrix maps long-horizon, multi-tool-call failures to "Environment-based RL with trajectory-level rewards" — not to "add a reflection step."
- **The clean contrast:** RL's "policy update" is "the training step that increases the probability of better outputs" by adjusting weights; every mechanism above leaves weights untouched and adjusts only what's in context.
- Bottom line: a single bad trajectory is a reflection problem; a systematic failure pattern across many trajectories is a training-data problem (3.3).

---

## Trade-offs and pitfalls

- **No external signal, no reliable improvement.** The central exam trap: self-critique with nothing to check against can reinforce an error as easily as fix it.
- **Context bloat from accumulated reflections.** Every stored reflection/retrieved episode is more tokens the next call carries; a long-running agent that keeps writing reflections without pruning hits the same context-window ceiling as 5.1.
- **Overfitting to the most recent feedback.** Heavy conditioning on the latest reflection can cause overcorrection — abandoning a sound strategy because of one unusual failure, the in-context analogue of recency bias.
- **Run-time adaptation is ephemeral by default.** A reflection lives in the current context window/session memory; unless explicitly written to a persistent long-term store (5.1's semantic/episodic memory), it disappears when the session ends.
- **Knowing when to stop reflecting and start retraining.** Persistent, cross-session failure patterns signal handing structured feedback to the training pipeline (3.3), not layering more prompt-level reflection on a model that keeps making the same mistake.

---

## NVIDIA angle (read critically)

- NVIDIA's agentic-RL decision matrix is useful for what it **doesn't** recommend: it runs RAG → prompting → SFT → DPO → RLVR/GRPO → full environment-based RL for long-horizon failures — **no row for "have the agent reflect harder."** Read plainly, NVIDIA's own practitioner guidance implicitly agrees with this lesson's central claim: run-time reflection is a within-session patch; a durable, systematic fix is a training problem in NVIDIA's own playbook, not a prompting one.
- **Corrective example (1.2):** NeMo Agent Toolkit's `reasoning_agent` generates an execution plan up front and, per NVIDIA's own docs, explicitly "does not revisit the plan to revise strategy during execution" — the **opposite** of the Reflexion/Self-Refine pattern. Useful reminder: not every agent framework marketed as "reasoning" is actually adapting from feedback mid-run — some are static plans wearing a reasoning-shaped prompt.

---

## How to think about it for the exam

| Mechanism | What changes | Scope | Needs external signal? |
|---|---|---|---|
| Self-Refine (feedback→refine loop) | The current generation, iteratively | Within a single attempt; no persistence | Not required, but weaker without one |
| Reflexion (self-reflection + episodic buffer) | Text conditioning the *next* attempt | Across retries of the same task, via *mem* | Strongly — reliable gains depend on a verifiable signal (tests, environment reward) |
| Episodic memory recall (bridge to 5.1) | Which past experience informs the current decision | Across tasks/sessions, if written to long-term memory | The past outcome itself is the signal |
| In-context feedback (tool error, observation, human/AI correction) | The very next reasoning step | Single step, mid-trajectory | The observation/error is the external signal |
| Training-time adaptation (3.3: SFT/DPO/RLHF/RLVR) | Model weights, permanently | Every future session | Yes — reward, preference, or verifier signal required |

---

## Quick self-check

1. In one sentence, what is the difference between run-time adaptation and training-time adaptation, and which one does Reflexion's own paper explicitly say it is *not* doing?
   **A:** Run-time adaptation changes what goes into the next prompt (frozen weights); training-time adaptation changes the weights themselves. Reflexion explicitly says it works "not by updating weights, but instead through linguistic feedback" — it is explicitly not doing the weight-update kind.
2. Using Reflexion's own comparison table, what is the one mechanical feature that distinguishes Reflexion from Self-Refine, and why does that feature let Reflexion improve *across* trials while Self-Refine only improves *within* one?
   **A:** Reflexion has a persistent episodic memory buffer (*mem*); Reflexion's own table marks Self-Refine with an explicit ✗ for "memory." Self-Refine iterates feedback→refine within one generation and stops, so gains don't carry over; Reflexion's buffer carries the reflection into the next retry of the same task.
3. Why does an agent's self-reflection loop need an external or verifiable signal to reliably help — what goes wrong when it doesn't have one, and which of Reflexion's own experimental settings (programming vs. decision-making) illustrates each case?
   **A:** Without ground truth, the model has only its own opinion to improve on and can talk itself into a confidently wrong reflection. The programming setting (failed unit tests/compiler errors) gives real ground truth to reason from; the decision-making setting (self-graded by a separate LLM classifier) has no external check, so reflection quality depends entirely on the model's own self-evaluation.
4. A production agent makes the same tool-call mistake across thousands of unrelated user sessions. Is more run-time reflection the right fix? What does NVIDIA's own agentic-RL decision matrix suggest instead, and why?
   **A:** No — that's a systematic cross-session failure pattern, not a single bad trajectory. NVIDIA's guidance: "If the agent repeats tool-call mistakes... you need a training signal. That is where RL fits" — its decision matrix routes this to environment-based RL with trajectory-level rewards, not added reflection.
5. Explain why a reflection generated mid-session can simply vanish by the next session, and what would need to happen (per 5.1) for it to persist.
   **A:** By default a reflection lives only in the current context window or session memory. For it to survive, it must be explicitly written to a persistent long-term store — 5.1's semantic or episodic memory — otherwise it disappears when the session ends and the next session starts from zero.

---

## Sources used

- [Reflexion: Language Agents with Verbal Reinforcement Learning — arXiv:2303.11366](https://arxiv.org/abs/2303.11366)
- [Self-Refine: Iterative Refinement with Self-Feedback — arXiv:2303.17651](https://arxiv.org/abs/2303.17651)
- [Cognitive Architectures for Language Agents (CoALA) — arXiv:2309.02427](https://arxiv.org/abs/2309.02427)
- [Generative Agents: Interactive Simulacra of Human Behavior — arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
- [Understanding the planning of LLM agents: A survey — arXiv:2402.02716](https://arxiv.org/abs/2402.02716)
- [Mastering Agentic Techniques: AI Agent Reinforcement Learning — NVIDIA Technical Blog](https://developer.nvidia.com/blog/mastering-agentic-techniques-ai-agent-reinforcement-learning/)
