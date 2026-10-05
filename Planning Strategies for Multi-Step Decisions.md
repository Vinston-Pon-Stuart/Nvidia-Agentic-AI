## In one line

**Planning** = mapping a goal to an ordered sequence of actions under uncertainty, where each action changes the environment and constrains what's possible next; Huang et al.'s 2024 survey names **five strategies** an LLM agent can use to turn a decomposed goal into a working course of action.

---

## Formal shape

- Given time step $t$, environment $E$, action space $A$, goal $g$: $p = (a_0, a_1, \dots, a_t) = \text{plan}(E, g; \Theta, \mathcal{P})$ — $\Theta$ = LLM parameters, $\mathcal{P}$ = prompt.
- **Key idea:** a plan is a *trajectory* through a state space, not a single answer — same shape as the classical state-action formulation behind MDPs, though LLM-agent planning isn't typically trained as an RL policy.
- Source: **Huang et al., "Understanding the Planning of LLM Agents"** (2024, arXiv:2402.02716) — this lesson's backbone. Its five strategies: task decomposition, multi-plan selection, external planner-aided planning, reflection and refinement, memory-augmented planning. Decomposition is covered in depth in 5.2; here it's one entry, with the other four covered in depth, plus a plan-then-execute vs. interleaved framing that cuts across all five.

---

## Why not classical planning or RL, straight up?

- **Symbolic methods** (e.g., PDDL): "require conversion from flexible natural language-described problems into symbolic modeling, which may require human experts' efforts" — and poor error tolerance; a few mistakes and the whole formalization fails.
- **RL-based policy learning:** needs large numbers of environment interactions, "impractical or costly" when data collection is expensive.
- LLMs offer a third path — using world knowledge/language understanding directly as the planning module — but bring their own failure modes, which is exactly why several of the five strategies exist to borrow back guarantees from symbolic/RL methods.

---

## Strategy 1: Task decomposition (pointer to 5.2)

- Divide goal into sub-goals, plan each in turn: $g_0, \dots, g_n = \text{decompose}(E, g)$, then $p^i = \text{sub-plan}(E, g_i)$ per sub-goal.
- **Decomposition-first** (commit to the full breakdown upfront — HuggingGPT, Plan-and-Solve) vs. **interleaved decomposition** (reveal one or two sub-goals at a time — CoT family, ReAct).
- 5.2 owns the depth. Note here: decomposition-first vs. interleaved is the *same tension* as plan-then-execute vs. interleaved-plan-and-act below, just at the sub-goal level instead of the whole-plan level.

---

## Strategy 2: Multi-plan selection — generate many, pick the best

- **What it is:** a single LLM plan is often suboptimal or infeasible (decoding is inherently uncertain), so generate several candidates and select. Formally: $P = p_1, \dots, p_n = \text{plan}(E, g; \Theta, \mathcal{P})$, then $p^* = \text{select}(E, g, P; \Theta, \mathcal{F})$.
- **Generating candidates:** **self-consistency** (Wang et al., 2022) samples multiple reasoning paths via temperature/top-k — "the solutions for complex problems are rarely unique." **Tree of Thoughts** (Yao et al., 2023, arXiv:2305.10601 — deep in 1.6) generalizes with *sample* and *propose* generation modes, organized into a tree.
- **Selecting the winner:** self-consistency uses majority vote. ToT runs classical search (BFS, DFS) using the LLM as evaluator. LLM-MCTS and RAP use the LLM as a heuristic policy inside Monte Carlo Tree Search; LLM-A* borrows A*, using distance-to-goal as the cost heuristic.
- **Trade-off:** "a broader exploration of potential solutions in the expansive search space" — but multiplies token cost/latency, which matters "particularly in scenarios where resource constraints are a significant factor, such as ... online service." And selection quality depends on an LLM whose "performance in ranking tasks is still under scrutiny," with stochastic outputs adding randomness even on identical inputs.

---

## Strategy 3: External module-aided planning — let a solver own the guarantees

- **Why:** LLMs are weak at hard constraints — struggle "when confronted with environments featuring intricate constraints, such as mathematical problem-solving or generating admissible actions."
- **What it does:** stop asking the LLM to plan; have it *translate* instead: $h = \text{formalize}(E, g; \Theta, \mathcal{P})$, then $p = \text{plan}(E, g, h; \Phi)$ — $\Phi$ = the external planning module.
- **Symbolic planners — LLM+P** (Liu et al., 2023, arXiv:2304.11477): LLM translates a natural-language problem into **PDDL**, hands it to a classical solver (Fast-Downward in the paper), translates the solver's output back to natural language. Motivation, in the paper's words: "LLMs cannot reliably solve long-horizon planning problems. By contrast, classical planners, once a problem is given in a formatted way, can use efficient search algorithms to quickly identify correct, or even optimal, plans." **Headline result:** LLM+P "is able to provide optimal solutions for most problems, while LLMs fail to provide even feasible plans for most problems" on their benchmark — not marginally better, LLM+P succeeds where bare LLM planning often fails to produce even a *feasible* plan.
  - Follow-ons: **LLM-DP** (re-formalizes PDDL after each observation, for dynamic settings), **LLM+PDDL** (adds human verification on the generated PDDL model, uses the LLM's plan as a search heuristic).
- **Neural planners:** small domain-specific models (DRRN modeling planning as an MDP; Decision Transformer cloning demonstrated trajectories) — fast and effective in-domain, generalize poorly outside it. **SwiftSage** hybrid: fast neural planner for routine steps, LLM "slow thinking" fallback on errors — a dual-process design from cognitive psychology.
- **Exam framing:** this strategy is the direct answer to "LLMs make invalid/infeasible plans" — outsource the actual search to a system with correctness guarantees, keep the LLM for parsing messy language into a formal spec and reasoning about the result.

---

## Strategy 4: Reflection and refinement (pointer to 5.5)

- **What it does:** closes the loop on a plan that turned out wrong: $p_0 = \text{plan}(E, g)$, then $r_i = \text{reflect}(E, g, p_i)$ (LLM critiques its own trajectory), then $p_{i+1} = \text{refine}(E, g, p_i, r_i)$ — repeat.
- **Reflexion** (Shinn et al., 2023, arXiv:2303.11366) is the flagship instance (full depth in 1.2 and 5.5): extends a ReAct-style rollout with an evaluator, stores verbal self-reflections in an episodic memory buffer to improve later attempts.
- **Key caveat (the survey's own):** resembles RL conceptually (agent = policy, environment feedback triggers an "update"), but the update is **textual, not a weight change** — and "the convergence of this textual form of update currently lacks a guaranteed proof." No theoretical guarantee repeated reflection actually converges toward the goal, unlike a proven RL update rule.

---

## Strategy 5: Memory-augmented planning (pointer to 5.1)

- **What it does:** retrieves relevant stored information before planning, instead of planning cold from the goal alone: $m = \text{retrieve}(E, g; \mathcal{M})$, then $p = \text{plan}(E, g, m; \Theta, \mathcal{P})$.
- **RAG-based memory** — commonsense knowledge/past experience/domain facts stored externally (text, embeddings, tables, knowledge graph), retrieved by similarity at plan time. Cheap and real-time-updatable, but only as good as retrieval.
- **Embodied memory** — experience baked into model *parameters* via fine-tuning (often LoRA), not retrieved at inference. Larger effective capacity, but expensive to update and prone to losing fine-grained detail.
- Survey's framing: RAG-based = "real-time, low-cost external memory updates ... but rely[s] on the accuracy of retrieval"; fine-tuning "provides a larger memorization capacity through parameter modifications but has high memory update costs." (5.1 owns storage-tier/retrieval mechanics; here memory is just a planning *input*.)

---

## Plan-then-execute vs. interleaved plan-and-act

- **The cross-cutting choice:** commit to a plan before acting, or replan at every step as new information arrives.
- **Plan-then-execute (upfront):** generate the full plan first, then carry it out. Advantage: "a stronger correlation between the sub-tasks and the original tasks, reducing the risk of task forgetting and hallucinations" — reasoned about all at once with full context. Cost: "since the sub-tasks are predetermined at the beginning, additional mechanisms for adjustment are required[,] otherwise one error in some step will result in failure." Cheaper (fewer round-trips) and more coherent, but goes stale the moment reality doesn't match assumptions. **Canonical instances:** ReWOO, Plan-and-Solve, Plan-and-Execute/ToT (1.6), NAT's `reasoning_agent`.
- **Interleaved plan-and-act:** replan after every action using the freshest observation. **ReAct** (1.2) is the standard example: Thought → Action → Observation → Thought, so a surprising result immediately reshapes the next step. Better fault tolerance/adaptivity, at the cost of a fresh LLM call (full context re-read) every step — and it's **not immune to drift either**: "excessively long trajectories may lead to LLM ... deviating from the original goals during subsequent sub-tasks."
- **Neither dominates:** upfront planning wins in predictable environments with a well-understood tool set; interleaved wins when observations are likely to surprise the agent mid-task. Multi-plan selection and external-planner-aided planning are naturally upfront (need the whole plan before selecting/handing to a solver); reflection-and-refinement and memory-augmented planning can attach to either shape.

---

## Trade-offs and pitfalls

- **Hallucinated / infeasible plans.** LLMs "often suffer from hallucinations, leading to irrational plans, unfaithfulness to task prompts, or failing to follow complex instructions" — including actions referencing items that don't exist in the environment. Core motivation for Strategy 3: a symbolic solver can't emit an action violating its formal preconditions.
- **Plan efficiency is usually ignored.** Most LLM-agent planning is "greedily based on generated plans from LLM output, without considering [plan] efficiency" — a *working* plan can still be needlessly long, because nothing in generation optimizes for path length the way classical search does.
- **Long-horizon error compounding and context limits.** Decomposing into "dozens of sub-tasks" means planning is "constrained by the context length of the LLM, leading to the forgetting of the planning trajectories" — more steps means more likely uncorrected early-error propagation, and a harder-to-hold full trajectory.
- **Plan-selection cost.** Multi-plan selection multiplies LLM calls and depends on an LLM (unproven ranking reliability) to judge candidates — real cost for uncertain benefit.
- **Overhead on simple tasks.** Decomposing or generating multiple candidates for a task that didn't need it "requires more reasoning and generation, incurring additional time and computational costs" for no accuracy gain.
- **Upfront plans go stale.** A plan-then-execute agent has no built-in mechanism to notice broken assumptions unless reflection/refinement or replanning is explicitly wired in.

---

## NVIDIA angle

- **NeMo Agent Toolkit (NAT)** ships YAML-configurable implementations of several strategies above — engineering instances, not novel research.
- `react_agent` / `rewoo_agent` (1.2) = the interleaved-vs-upfront pair. `reasoning_agent` = a third pattern mapping to plan-then-execute: wraps an underlying agent (`augmented_fn`, e.g. a `react_agent`) and, via a reasoning-capable LLM, generates a full step-by-step plan *before* invoking it — default prompt asks the model to "provide a detailed step-by-step plan" identifying "the key components of the input," the "most suitable tools," and "the sequence of actions to be taken." Documented limitation: "performs reasoning up front, and does not revisit the plan to revise strategy during execution like a ReAct agent does" — a vendor-acknowledged instance of the plan-then-execute staleness trade-off.
- **`nat.experimental.test_time_compute`** — a four-stage multi-plan-selection pipeline: **Search** (`single_shot_multi_plan` samples *n* plans at different temperatures from one model; `multi_llm_plan` queries several LLMs in parallel and pools outputs) → **Editing** (optional critique/refine) → **Scoring** (LLM judges each plan 1–10) → **Selection** (`best_of_n_selector` keeps top scorer; `llm_based_plan_selector` lets an LLM choose/merge). The `plan_select_execute_function` wires it end-to-end. Near-literal implementation of the survey's $P = \text{plan}(\dots)$, $p^* = \text{select}(\dots)$ — generate-many via sampling/multiple models, then LLM-as-judge selection, functionally similar to self-consistency/ToT.
- **Read critically:** lives under `nat.experimental` — NVIDIA's own naming signals less-stable, not a core guarantee. None of these are new planning algorithms: `reasoning_agent` is an applied instance of upfront/Plan-and-Solve-style planning; test-time-compute's Search/Selection is an applied instance of self-consistency/ToT-style generate-and-judge search, inheriting the same LLM-ranking-reliability caveat the survey raises. Contribution = a configurable, NIM-integrated harness for established strategies, not a new method.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Planning, formally | $p = (a_0, \dots, a_t) = \text{plan}(E, g; \Theta, \mathcal{P})$ — a sequence of actions toward a goal; each action changes the environment and constrains future choices (MDP-adjacent state-action framing, high level only) |
| The survey's five strategies | Task decomposition (5.2), multi-plan selection, external planner-aided planning, reflection & refinement (5.5), memory-augmented planning (5.1) — interconnected, not mutually exclusive |
| Multi-plan selection | Generate candidates (sampling / self-consistency / ToT's sample-or-propose) → select via search (majority vote, BFS/DFS, MCTS, A*) — costs more compute, depends on unproven LLM ranking ability |
| External planner-aided planning | LLM formalizes the problem (e.g., to PDDL); a symbolic solver (Fast-Downward) or trained neural planner does the actual search — LLM+P: LLMs "fail to provide even feasible plans for most problems" while LLM+P finds optimal ones |
| Reflection & refinement | plan → reflect → refine, repeating; Reflexion-style; textual "policy update" with no convergence guarantee (survey's own caveat) |
| Memory-augmented planning | Retrieve relevant knowledge/experience before planning; RAG-based (cheap, retrieval-error-prone) vs. embodied/fine-tuned (costly, larger capacity) |
| Plan-then-execute vs. interleaved | Upfront: cheaper, more globally coherent, brittle to surprises (ReWOO, Plan-and-Solve, reasoning_agent). Interleaved: adapts per step, costlier, still risks drift on long trajectories (ReAct, 1.2) |
| Biggest pitfalls | Hallucinated/infeasible plans, ignored plan efficiency, long-horizon context/error compounding, plan-selection cost, over-planning simple tasks, stale upfront plans |
| NVIDIA's role | NAT's `reasoning_agent` = plan-then-execute wrapper; `nat.experimental.test_time_compute` (Search/Selection) = configurable multi-plan-selection pipeline — conformant implementations of the survey's strategies, not new algorithms, and explicitly marked experimental |

---

## Quick self-check

1. Name the survey's five LLM-agent planning strategies, and say in one line what problem each one is solving.
   **A:** Task decomposition — break a goal into subgoals, plan each (5.2). Multi-plan selection — generate several candidate plans, pick the best, since a single LLM plan is often unreliable. External planner-aided planning — hand a formalized problem to a symbolic/neural solver for correctness guarantees the LLM lacks. Reflection & refinement — critique and revise a plan after it fails (5.5). Memory-augmented planning — retrieve relevant past knowledge/experience before planning, instead of planning cold (5.1).
2. Why does LLM+P route a planning problem through PDDL and a classical solver instead of asking the LLM to plan directly? What did the paper find about plain-LLM feasibility versus LLM+P's results?
   **A:** Because "LLMs cannot reliably solve long-horizon planning problems," while a classical planner given a properly formatted problem can search efficiently for correct or optimal plans. The paper found LLM+P "provide[s] optimal solutions for most problems, while LLMs fail to provide even feasible plans for most problems."
3. What's the structural difference between plan-then-execute and interleaved plan-and-act, and what specific failure mode does each one remain vulnerable to (per the survey's own discussion)?
   **A:** Plan-then-execute commits to the full plan before acting — cheaper and more coherent, but "one error in some step will result in failure" without added adjustment mechanisms (goes stale). Interleaved plan-and-act replans after every action — better fault tolerance, but "excessively long trajectories may lead to LLM ... deviating from the original goals" — it drifts too, just differently.
4. In multi-plan selection, name two ways candidate plans get generated and two ways the best one gets selected. What's the survey's stated concern about using an LLM as the selector?
   **A:** Generated via self-consistency sampling (temperature/top-k) or ToT's sample/propose modes. Selected via majority vote (self-consistency) or LLM-guided search like BFS/DFS/MCTS/A* (ToT, LLM-MCTS, LLM-A*). Concern: "LLM's performance in ranking tasks is still under scrutiny," plus stochastic outputs add randomness to selection even on identical inputs.
5. The survey says Reflexion-style reflection resembles reinforcement learning conceptually. What's the one guarantee that a real RL policy update has that a textual "reflect and refine" update does not?
   **A:** A real RL update has a proven convergence guarantee; the survey states the textual update "currently lacks a guaranteed proof" that repeated reflection actually converges the agent toward the goal.

---

## Sources used

- [Understanding the Planning of LLM Agents: A Survey — arXiv:2402.02716 (Huang et al., 2024)](https://arxiv.org/abs/2402.02716)
- [LLM+P: Empowering Large Language Models with Optimal Planning Proficiency — arXiv:2304.11477 (Liu et al., 2023)](https://arxiv.org/abs/2304.11477)
- [Tree of Thoughts: Deliberate Problem Solving with Large Language Models — arXiv:2305.10601 (Yao et al., 2023)](https://arxiv.org/abs/2305.10601)
- [ReAct: Synergizing Reasoning and Acting in Language Models — arXiv:2210.03629 (Yao et al., 2022)](https://arxiv.org/abs/2210.03629)
- [Reflexion: Language Agents with Verbal Reinforcement Learning — arXiv:2303.11366 (Shinn et al., 2023)](https://arxiv.org/abs/2303.11366)
- [About Reasoning Agent — NVIDIA NeMo Agent Toolkit (1.2) docs](https://docs.nvidia.com/nemo/agent-toolkit/1.2/workflows/about/reasoning-agent.html)
- [Test Time Compute With NVIDIA NeMo Agent Toolkit — NVIDIA NeMo Agent Toolkit (1.2) docs](https://docs.nvidia.com/nemo/agent-toolkit/1.2/reference/test-time-compute.html)
