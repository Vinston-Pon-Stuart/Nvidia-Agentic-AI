## In one line

**Chain-of-thought (CoT)** externalizes a model's intermediate reasoning as tokens instead of hiding it in one forward pass; **task decomposition** goes further by explicitly splitting one hard problem into ordered sub-problems *before* solving any of them. Both are reasoning paradigms, not prompt-writing tricks.

---

## Scope note

- **1.2 (Reasoning and Action Frameworks):** ReAct and the reason-plus-act family — grounding reasoning in tool calls/observations. Not repeated here.
- **2.1 (Prompt Engineering):** CoT as a *single-call prompting technique* (few-shot exemplars, "let's think step by step") plus self-consistency (sample multiple CoT paths, majority-vote). Not repeated here.
- **1.6 (Logic Trees, Prompt Chains, Stateful Orchestration):** prompt chaining and Tree-of-Thoughts as *control-flow architecture*. Not repeated here.
- **This lesson's question:** at the cognition level, *why* do intermediate steps help, *when* do they help (scale), and what does decomposition mean as a reasoning strategy — the on-ramp to planning (5.3).

---

## Chain-of-thought as a reasoning paradigm

- **What it is:** "a series of intermediate natural language reasoning steps that lead to the final output" (Wei et al., 2022, arXiv:2201.11903).
- **What it does:** gives the model a scratchpad to condition later steps on partial results, instead of collapsing a multi-step derivation into one guess. Mechanically, more tokens = more compute spent on the problem, spread across passes instead of one fixed-depth pass.
- **Key finding — emerges with scale:** "chain-of-thought prompting is an emergent ability of model scale." Tested across five model families (GPT-3 350M–175B, LaMDA 422M–137B, PaLM 8B/62B/540B, UL2 20B, Codex). **At small scale, CoT actively hurts** — "models of smaller scale produced fluent but illogical chains of thought, leading to lower performance than standard prompting." Headline win: **PaLM 540B with 8 CoT exemplars reached SOTA on GSM8K**, "surpassing even finetuned GPT-3 with a verifier."
- **Gains scale with difficulty:** larger performance gains on harder problems, but on the *easiest* MAWPS subset (SingleOp, one-step), "performance improvements were either negative or very small." CoT is not a universal accuracy boost.
- **Why scaling fixes it:** error analysis of wrong CoT answers found **46% "almost correct, barring minor mistakes"** (calculator slip, symbol error, missing step) vs. **54% "major errors in semantic understanding or coherence."** PaLM 540B (vs. 62B) "fixes a large portion of one-step missing and semantic understanding errors" — scale measurably repairs both dominant failure modes.
- **Zero-shot CoT** (Kojima et al., 2022, arXiv:2205.11916): appending **"Let's think step by step"** alone (no exemplars) induces the same reasoning trace. On `text-davinci-002`: MultiArith **17.7% → 78.7%**, GSM8K **10.4% → 40.7%**; similar magnitude on PaLM 540B. Implies the capability was **latent** in the model, not taught by few-shot demonstrations. (Mechanically this is 2.1's single-call technique; the point here is what it implies about *where* reasoning ability lives.)

---

## Task decomposition — the headline idea

- **CoT's documented weak spot:** "tends to perform poorly on tasks which requires solving problems harder than the exemplars shown in the prompts" — the **easy-to-hard generalization** problem (Zhou et al., 2022, arXiv:2205.10625). If exemplars show 4 steps and the test needs 9, plain CoT has no template to extend.
- **Least-to-most prompting** is the fix and the canonical instance of task decomposition: "break down a complex problem into a series of simpler subproblems and then solve them in sequence." Two explicit stages:
  1. **Decomposition** — prompt the model to break the target into an ordered list of subproblems.
  2. **Sequential solving** — solve each in order, appending every prior subproblem-answer pair into context before the next.
- **Headline number:** on **SCAN**, `code-davinci-002` with least-to-most reached **≥99% accuracy on every split (including the length split) with just 14 exemplars**, vs. **only 16%** for plain CoT. Specialized neural-symbolic SCAN models needed the **full 15,000+ example training set** to match it — with no gradient updates at all.
- **Why it works:** because subproblem *structure* is derived from the specific test question (not copied from a fixed-depth demonstration), the chain length isn't capped by exemplar depth.
- **Exam-critical fact:** least-to-most prompting *is* task decomposition applied to reasoning — it out-generalizes CoT because decomposition is explicit and separate from solving, not baked into one continuous chain.

---

## Decomposed Prompting: modular sub-task delegation

- **What it is** (Khot et al., 2022, arXiv:2210.02406): breaks a hard task into sub-tasks, but delegates each to a **library of prompting-based LLMs dedicated to that sub-task**, instead of solving every subproblem with the same model in the same growing context.
- **Payoff:** "this modular structure allows each prompt to be optimized for its specific sub-task, further decomposed if necessary, and even easily replaced with more effective prompts, trained models, or symbolic functions if desired."
- **Two concrete strategies:**
  - **Recursive decomposition on problem size** — "when the complexity comes from the input length, we can recursively decompose the task into the same task but with smaller inputs" (divide-and-conquer, distinct from least-to-most's step chain).
  - **Symbolic sub-task substitution** — hand a hard sub-task to a **symbolic function** instead of another LLM call (e.g., symbolic information retrieval for open-domain multi-hop QA), which the paper reports improves both long-context and open-domain multi-hop settings.
- **Distinction to hold:** least-to-most decomposes *one task* into subproblems of the same kind, solved sequentially by the same model with growing context. Decomposed Prompting decomposes a task into sub-tasks handled by **different, specialized modules** — including non-LLM symbolic functions. Least-to-most = decomposition-as-sequence; decomposed prompting = decomposition-as-architecture.

---

## Where decomposition sits: implicit vs. explicit, and the bridge to planning

- **Axis:** is the decomposition happening inside one generation, or as separate, explicit steps?
  - **Implicit, single-call:** plain CoT (2.1) — sub-problems solved implicitly within one continuous generation; no separate, inspectable decomposition step.
  - **Explicit, multi-step:** least-to-most and Decomposed Prompting — a distinct decomposition pass produces a subgoal list *before* any solving. Once decomposition is its own step, this is structurally **prompt chaining** (1.6): multiple LLM calls with state (the growing answered-subproblem list) passed forward.
- **Huang et al.'s planning survey (arXiv:2402.02716)** formalizes this at the agent-planning level: **Task Decomposition** = `g₀, g₁, ..., gₙ = decompose(E, g; Θ, 𝒫)` (split goal into ordered subgoals), then `pⁱ = sub-plan(E, gᵢ; Θ, 𝒫)` per subgoal (generate the action sequence). Same shape least-to-most/decomposed prompting use for pure reasoning; 5.3 extends it to agents generating tool-using action sequences per subgoal.
- **vs. ReAct (1.2):** ReAct doesn't decompose upfront at all — it interleaves one Thought/Action/Observation step at a time, discovering structure as it goes, replanning after every observation. **Trade-off, not a strict winner either way:** decomposition-first buys inspectability and per-subgoal tailored strategy, at the cost that a bad initial decomposition can't be cheaply revised; ReAct buys continuous adaptability at the cost of no global plan to check progress against.
- **vs. Tree of Thoughts (1.6):** ToT branches over multiple *candidate* reasoning continuations at each step and searches/backtracks — a breadth-of-exploration axis, **orthogonal** to decomposition's structure axis. The two can compose: decompose into subgoals, then run ToT-style search within a subgoal that resists direct solving.

---

## Trade-offs and pitfalls

- **CoT can rationalize a wrong answer — unfaithful reasoning.** 54% of Wei et al.'s incorrect CoT traces had major semantic/coherence errors, not just slips — a fluent trace is not proof the model reasoned correctly rather than rationalized a guess.
- **CoT only helps at sufficient scale, and can actively hurt below it** — not a soft caveat; small models produce "fluent but illogical chains of thought, leading to lower performance than standard prompting."
- **Decomposition adds calls and latency, and errors compound across sub-problems.** Both least-to-most and decomposed prompting need multiple sequential generations; because each sub-answer feeds the next, an early wrong answer propagates forward — decomposition doesn't average out mistakes, it lets them cascade.
- **Decomposition quality depends entirely on the decomposer.** A misidentified subgoal structure (wrong grouping, missing dependency) in the upfront `decompose()` step propagates to every downstream subproblem, with no built-in mechanism to notice and re-decompose. Decomposed Prompting raises the stakes further: correctness also depends on each delegated handler being well-matched to its sub-task.
- **Gains are proportional to problem difficulty, not universal.** On the simplest MAWPS subset, CoT improvements were "either negative or very small" — these are tools for genuinely multi-step problems, not a default for everything.

---

## NVIDIA angle

- **Nemotron 3 Nano's reasoning controls** are a production-facing version of the CoT-emergence finding, **not a new reasoning algorithm.** It keeps "Thinking ON/OFF modes and Thinking Budget controls," learned "directly from the chat template": **Reasoning ON** = multi-step mode, preserving/building on prior CoT within a task; **Reasoning OFF** = multi-turn mode, reasoning not carried across turns.
  - **Read critically:** Wei et al. showed CoT is *emergent and prompting-elicited* at sufficient scale; Nemotron's contribution is making it an explicit, **trained, switchable** capability (SFT + RL) with a token-budget knob — a productization of the academic finding, not a departure from it. The "thinking budget" is a direct, practical answer to the cost/latency trade-off above — an explicit dial on what was previously an implicit scaling decision.
- **NeMo Agent Toolkit's `reasoning_agent`** (detailed in 1.2) is architecturally decomposition-first: generates a full execution plan *before* invoking a wrapped agent, and per NVIDIA's docs, unlike ReAct it "does not revisit the plan to revise strategy during execution." This is exactly Huang et al.'s decompose-then-execute shape, demonstrated years earlier by least-to-most and decomposed prompting. NVIDIA's contribution: packaging this as a configurable workflow type on NIM-served, reasoning-tuned models — **integration, not a new decomposition algorithm.**

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| CoT as a reasoning paradigm | Externalizes intermediate reasoning as tokens so the model can condition later steps on earlier partial results, spreading compute across a multi-step problem instead of one collapsed guess. |
| CoT emergence with scale | Benefit is an *emergent* ability of scale (Wei et al.) — small models produce "fluent but illogical" chains and get **worse** than direct prompting; only sufficiently large models (PaLM 540B, etc.) benefit, and gains grow with problem difficulty (but shrink/vanish on trivial problems). |
| Zero-shot CoT | "Let's think step by step" alone (no exemplars) triggers the same paradigm — MultiArith 17.7%→78.7%, GSM8K 10.4%→40.7% (`text-davinci-002`, Kojima et al.) — evidence the reasoning capability is latent, not taught by the demonstrations. |
| Least-to-most prompting = task decomposition | Explicit two-stage process: decompose into ordered subproblems, then solve sequentially with prior answers feeding forward. Generalizes easy-to-hard where CoT plateaus — SCAN: ≥99% vs. 16% for plain CoT, 14 exemplars vs. 15,000+ training examples for specialized baselines. |
| Decomposed Prompting | Delegates each sub-task to a dedicated, swappable handler (prompt, trained model, or symbolic function) rather than solving every subproblem the same way in one growing context; supports recursive decomposition on input size. |
| Decomposition vs. CoT vs. ReAct vs. ToT | CoT: implicit, single-call. Decomposition (least-to-most / decomposed prompting): explicit subgoals, solved in sequence, mostly upfront. ReAct (1.2): interleaved, no upfront plan, replans every step. ToT (1.6): branches/searches over candidate reasoning at each step — a different (breadth) axis than decomposition's (structure) axis. |
| Decomposition's formal shape | Huang et al.'s planning survey: `decompose(E, g) → g₀...gₙ`, then `sub-plan(E, gᵢ) → actions` per subgoal — decompose-then-plan-per-subgoal, the direct bridge to planning (5.3). |
| NVIDIA's role | Nemotron 3 Nano's Reasoning ON/OFF + thinking budget = a trained, controllable production version of CoT's emergent behavior. NeMo Agent Toolkit's `reasoning_agent` = a decomposition-first planning wrapper that does not revise its plan mid-execution — integration of published ideas, not a new algorithm. |

---

## Quick self-check

1. Wei et al.'s paper reports that CoT's benefit is an "emergent ability of model scale." What specifically happens to CoT's accuracy on *small* models, and why does that matter more than just saying "small models don't benefit"?
   **A:** Small models produce "fluent but illogical chains of thought," giving *lower* accuracy than standard direct prompting — not just no improvement. That matters because CoT isn't a safe default to add everywhere; below a scale threshold it's actively harmful.
2. What two explicit stages make up least-to-most prompting, and what is the headline SCAN result (with numbers) that shows why it generalizes from easy to hard better than plain chain-of-thought?
   **A:** (1) Decomposition — break the problem into an ordered list of subproblems; (2) sequential solving — solve each in order, appending prior subproblem-answer pairs to context. On SCAN, `code-davinci-002` with least-to-most hit ≥99% accuracy on every split (including length) with 14 exemplars, vs. 16% for plain CoT — and specialized neural-symbolic models needed 15,000+ training examples to match it.
3. How does Decomposed Prompting's approach to sub-tasks differ from least-to-most prompting's approach to subproblems — specifically, what can a sub-task be delegated to in Decomposed Prompting that least-to-most doesn't offer?
   **A:** Least-to-most solves every subproblem with the same model in the same growing context. Decomposed Prompting delegates each sub-task to a dedicated, swappable handler — which can be a different LLM prompt, a trained model, or a **symbolic function** — none of which least-to-most offers.
4. Using Huang et al.'s formal `decompose()` / `sub-plan()` shape, explain why ReAct (1.2) is *not* an instance of task decomposition even though it also breaks a task into steps.
   **A:** Task decomposition requires an upfront `decompose(E, g) → subgoals` step before any solving begins. ReAct never does that — it interleaves Thought/Action/Observation one step at a time, discovering task structure as it goes and replanning after every observation, with no separate, inspectable decomposition step.
5. Name two concrete failure modes of relying on CoT or decomposition that this lesson attributes directly to the primary papers' own error analyses — and how does NVIDIA's Nemotron "thinking budget" address one of the cost-related trade-offs, without actually fixing the reliability one?
   **A:** (1) Unfaithful reasoning — 54% of Wei et al.'s incorrect CoT traces had major semantic/coherence errors, so a fluent trace isn't proof of correct reasoning. (2) CoT hurts small models — "fluent but illogical" chains reduce accuracy below standard prompting. The thinking budget lets developers dial down reasoning-token spend, addressing cost/latency — but it doesn't touch whether the reasoning produced is actually faithful or correct.

---

## Sources used

- [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models — arXiv:2201.11903 (Wei et al., 2022)](https://arxiv.org/abs/2201.11903)
- [Large Language Models are Zero-Shot Reasoners — arXiv:2205.11916 (Kojima et al., 2022)](https://arxiv.org/abs/2205.11916)
- [Least-to-Most Prompting Enables Complex Reasoning in Large Language Models — arXiv:2205.10625 (Zhou et al., 2022, ICLR 2023)](https://arxiv.org/abs/2205.10625)
- [Decomposed Prompting: A Modular Approach for Solving Complex Tasks — arXiv:2210.02406 (Khot et al., 2022, ICLR 2023)](https://arxiv.org/abs/2210.02406)
- [Understanding the Planning of LLM Agents: A Survey — arXiv:2402.02716 (Huang et al., 2024)](https://arxiv.org/abs/2402.02716)
- [Nemotron 3 Nano — A New Standard for Efficient, Open, and Intelligent Agentic Models — NVIDIA (Hugging Face blog)](https://huggingface.co/blog/nvidia/nemotron-3-nano-efficient-open-intelligent-models)
- [Inside NVIDIA Nemotron 3: Techniques, Tools, and Data That Make It Efficient and Accurate — NVIDIA Technical Blog](https://developer.nvidia.com/blog/inside-nvidia-nemotron-3-techniques-tools-and-data-that-make-it-efficient-and-accurate/)
- [About Reasoning Agent — NVIDIA NeMo Agent Toolkit docs](https://docs.nvidia.com/nemo/agent-toolkit/1.2/workflows/about/reasoning-agent.html)
