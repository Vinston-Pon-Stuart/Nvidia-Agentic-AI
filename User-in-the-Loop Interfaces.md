## In one line

**User-in-the-loop** design makes the human a participant *during* an agent's task execution — approving steps, editing plans, redirecting mid-task, or answering clarifying questions — rather than someone who only reviews a finished answer.

---

## Why it's a separate design problem

- Lesson 1.1's "intuitive interaction" (memory, grounding, latency, modality) all describe a human **receiving** something from the agent.
- User-in-the-loop is about a human **doing** something alongside the agent while a task is still running — a different design problem.
- The two pull in opposite directions: an intuitive receiving-interface minimizes friction; a user-in-the-loop interface has to *introduce* deliberate pauses at specific moments.
- Get the pause wrong (too rare, too frequent, too vague, too late) → either errors slip through unchecked, or reviewing eats the time the agent was meant to save.
- Converging sources: Microsoft's CHI 2019 HAI guidelines, LangGraph's `interrupt()` primitive, Smashing Magazine's Feb 2026 piece on agentic UX control/consent patterns.

---

## The core idea — four patterns + one supporting thread

**User-in-the-loop** = the human engages at specific, *designed* decision points during the task — not reviewing every token, not absent until the end. OneReach.ai: "humans and agents working hand in hand," staying involved "at key moments such as checking outputs, fixing errors, and guiding learning."

- **Approval / confirmation before an action** — the agent proposes a specific, consequential step and waits for yes/no. LangGraph's most common interrupt use case (e.g., approving an API call or database change).
- **Editable plans and drafts** — the user modifies what the agent is about to do or has generated, not just approve/reject. LangGraph calls this "review and edit state." Smashing's **Intent Preview** pattern: a plan shown as plain-language steps with "Proceed with this Plan" / "Edit Plan" / "Handle it Myself" buttons.
- **Steering / redirection mid-task** — the user changes course without rejecting the whole plan. Smashing's **Autonomy Dial**: a per-task-type setting ("Observe & Suggest" → "Plan & Propose" → "Act with Confirmation" → "Act Autonomously").
- **Clarifying-question turns** — the agent asks instead of guessing at ambiguous intent. Smashing's **Escalation Pathway** (e.g., "You said 'next Tuesday' — Sept 30 or Oct 7?").
- **Progressive disclosure of reasoning** — ties the other four together. An approval gate with no context on *what* is being approved is a rubber stamp, not real oversight.

---

## Microsoft's Human-AI Interaction Guidelines (CHI 2019), applied

- **Source**: Amershi, Vorvoreanu, Horvitz + 10 co-authors — 18 guidelines distilled from 150+ recommendations, validated by 49 practitioners across 20 production AI products. Predates agents (examples: AutoReplace, Excel Ideas) but maps directly onto agentic checkpoints.
- **G7 / G8 (efficient invocation / dismissal)** — the approval gate and the "Handle it Myself" escape hatch each need to be one interaction, not buried in a menu.
- **G9 (efficient correction)** — licenses the editable-plan pattern directly; correction is first-class, not a fallback.
- **G10 (scope when in doubt)** — licenses clarifying questions. Microsoft's own AutoReplace example (multiple correction options) is structurally the same move as an agent asking "did you mean X or Y?"
- **G11 (explain why)** — underlies progressive disclosure; without it, an approval request is a black box.
- **G3 (time services to context)** — *when* an interruption fires matters as much as whether it fires (see design tension below).
- **G15 (encourage granular feedback)** — every approve/edit/reject decision is granular feedback worth logging (picked up by Lesson 10.2's feedback loop).
- **Why it matters**: none of these were written with agents in mind — evidence that the *shape* of user-in-the-loop interaction (invoke, dismiss, correct, disambiguate, explain, collect feedback) is general good HCI, not an agentic invention. What agents add is a concrete mid-task execution point where these guidelines must fire, before an irreversible step.

---

## Mechanics: interrupt-and-resume

- **LangGraph `interrupt()`** — called inside a graph node: suspends execution at that point, persists full graph state via a checkpointer, surfaces a payload to the caller (for UI rendering), and waits indefinitely for `Command(resume=...)`. The resume value becomes the `interrupt()` call's return value.
- **One primitive, three patterns** — an approval gate passes a yes/no question and resumes with a boolean; a review-and-edit gate passes generated content and resumes with the user's edit; a tool-level interrupt pauses inside a tool call (e.g., before `send_email` fires) so a human can approve/edit/cancel with the exact arguments visible.
- **Caveat**: a node **restarts from its beginning** on resume — any side effect placed before `interrupt()` must be idempotent, or a naive approval gate double-sends on every re-approval.
- **NVIDIA NeMo Agent Toolkit** — `simple_calculator_hitl` example wires a `hitl_approval_function` into a ReAct agent; when the agent hits its tool-call recursion limit, it synchronously prompts "...Respond with 'yes' or 'no'." "Yes" continues with a larger iteration budget; "no" stops the agent. Same gate-and-resume shape as LangGraph, but a **blocking CLI prompt + boolean**, not a rich resumable interrupt with a structured payload.

---

## UX principles for low-friction, well-timed intervention

- **Surface uncertainty (Confidence Signal)** — expose the agent's confidence (%, scope statement, or a simple icon) so users know which approvals deserve a close read vs. a quick click; "surfacing uncertainty helps prevent automation bias." Without it, every approval request looks identical.
- **Make undo cheap (Action Audit & Undo)** — reversibility is "the single most powerful mechanism for building user confidence... dramatically lowers the perceived risk of granting autonomy." Practically: a chronological log of agent actions with visible undo on each, plus an explicit window when an action becomes irreversible (e.g., "Undo available for 15 minutes"). Implements G9 at the interface-artifact level.
- **Show the plan up front (Intent Preview)** — plain-language, sequential steps before execution starts. Gives the "conversational pause before action" (an informed-consent moment) and a reference point to check progress against. A preview phrased `"Cancel flight AA123 to San Francisco"` is legible; `"cancel_booking(id: 4A7B)"` is not.

---

## The design tension: control vs. friction

- **Central trade-off**: too little user-in-the-loop → errors slip through, trust quietly erodes; too much → approval fatigue, kills the productivity gain the agent existed to deliver.
- **Under-control failure**: without a checkpoint, "users feel ambushed by the agent's actions and will disable the feature to regain control" (Smashing) — no calibrated trust develops; users either over-trust by default or abandon the feature at the first surprise.
- **Over-control failure**: OneReach.ai names it directly — **"Scalability Bottleneck"** (oversight limits automation speed/scale) and **"Cost of Continuous Oversight"** (reviewer labor cost can offset automation savings). An agent that asks permission for everything isn't user-in-the-loop — it's a form filled out by two parties.
- **Smashing's representative metrics** (not a universal standard, but shows the tension is measurable):
  - **Override Frequency** ("Handle it Myself" clicks) **>10%** → flag for model/plan-quality review.
  - **Escalation Frequency** (clarifying-question requests) healthy at **5–15%**.
  - **Reversion Rate** (undone actions) **>5%** for a task type → disable autonomous execution, fall back to confirmation.
- Mirrors alert fatigue (Lesson 8.1) and escalation overload (Lesson 9.4): too many low-value interruptions make reviewers faster and less discriminating, turning every gate into the rubber stamp it was built to prevent.

---

## Trade-offs and pitfalls

- **Interruption/decision fatigue** — the most common real-world failure mode: not a missing safety gate, but too many gates training users to click through without reading.
- **Uniform confirmation trains automation bias instead of preventing it** — asking the same way for every action regardless of stakes removes the confidence signal users need to know when to look closely; functionally equal to no gate, just slower.
- **Poorly timed interventions** — violates G3 directly: a clarifying question asked too early, or an approval gate that fires only after most of a task has already run, undermines the checkpoint's purpose. Timing is inseparable from *whether* to interrupt.
- **A confusing plan preview is worse than no plan** — Smashing's own metric, **Recall Accuracy** (% of test users who can list the plan's steps 10 seconds after the preview is hidden), concedes a preview can be present yet fail if too dense or jargon-heavy.
- **Accessibility of the controls** — approval buttons, editable-plan fields, undo affordances need full keyboard/screen-reader support; a mouse-only gate excludes exactly the users who most need the extra checkpoint.
- **Control ≠ oversight if the user can't evaluate what they're approving** — OneReach.ai: **"Skill Gap in Workforce: Employees lack AI literacy to effectively review and validate AI decisions."** A well-built approval gate in front of a user who can't judge the domain is a UI success and an oversight failure at once. Whether the approver is authorized/competent is Lesson 10.4's territory — this lesson only covers making the ask legible.

---

## NVIDIA angle (read critically)

- NeMo Agent Toolkit's `hitl_approval_function` is a genuine, working interrupt-and-resume instance — gates a well-scoped decision (extend iteration budget or not) behind a synchronous prompt; the docs correctly frame the general need for human input during workflow execution.
- **Read critically**: the example is deliberately minimal — blocking CLI yes/no, no Intent Preview, no Confidence Signal, no undo. It demonstrates the *mechanism* (a tool call gated on human input), not the interface-design work the practitioner literature treats as load-bearing. Same "instrumentation, not paradigm" pattern as NVIDIA's other agent tooling (Lesson 1.1).
- It only demonstrates the narrowest pattern (approval/confirmation), even though the underlying `user_input_manager` primitive is general enough to support editable plans or clarifying questions too.
- **Bottom line**: treat the shipped example as proof the plumbing exists, not a reference implementation of a well-designed approval/edit interface.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| User-in-the-loop, defined | The human participates *during* the agent's task execution — approving, editing, steering, or being asked — not just receiving a finished answer (OneReach.ai: "humans and agents working hand in hand") |
| Four interaction patterns | Approval/confirmation before an action; editable plans/drafts; steering/redirection mid-task; clarifying-question turns — all four map onto LangGraph's `interrupt()`/`Command(resume=...)` primitive with a different payload/resume shape each |
| Microsoft's guidelines that apply | G7/G8 (efficient invocation/dismissal), G9 (efficient correction — licenses editable plans), G10 (scope when in doubt — licenses clarifying questions), G11 (explain why — licenses progressive disclosure), G3 (time interruptions to context), G15 (granular feedback) |
| Mechanism | LangGraph `interrupt()` suspends a node, checkpoints state, surfaces a payload, and resumes on `Command(resume=...)`; side effects before an interrupt must be idempotent since the node restarts from the top on resume |
| NVIDIA instance | NeMo Agent Toolkit's `hitl_approval_function` — a working but minimal (blocking, yes/no) instance of the same gate mechanic; infrastructure, not a finished UI-design answer |
| Low-friction UX principles | Surface confidence/uncertainty so users know when to scrutinize; make undo cheap and time-bounded; show the plan up front (Intent Preview) so a long task stays legible |
| The design tension | Too little control → errors slip through, users feel "ambushed," abandon the feature; too much → scalability bottleneck, decision fatigue, approvals become reflexive rubber stamps — Smashing's own targets: Override Frequency >10% flags a model review, Escalation Frequency healthy at 5–15%, Reversion Rate >5% should disable autonomy for that task |
| Biggest pitfalls | Decision/approval fatigue; uniform confirmation requests train automation bias instead of preventing it; badly timed interruptions; confusing-but-present plan previews; inaccessible controls; a granted approval isn't real oversight if the user lacks the domain literacy to evaluate it (10.4/8.5 territory) |

---

## Quick self-check

1. Explain the difference in scope between Lesson 1.1's "intuitive human-agent interaction" and this lesson's "user-in-the-loop interfaces." What kind of interaction does each one cover that the other doesn't?
   **A:** Lesson 1.1 covers the human **receiving** something from the agent (memory, grounding, latency, modality surfaced honestly); this lesson covers the human **doing** something alongside the agent mid-task — approving, editing, steering, or being asked — while the task is still in flight.
2. Name the four user-in-the-loop interaction patterns this lesson covers, and for each, describe how LangGraph's `interrupt()`/`Command(resume=...)` mechanism could implement it.
   **A:** Approval/confirmation (pass a yes/no question, resume with a boolean); editable plans/drafts (pass generated content, resume with the user's edited version); steering/redirection (an Autonomy-Dial-style setting adjusting how/when interrupts fire); clarifying questions (interrupt surfaces the ambiguous point, resume supplies the disambiguating answer).
3. Pick three of Microsoft's 18 Guidelines for Human-AI Interaction discussed here and explain, in your own words, how each one applies specifically to an agentic approval or editable-plan interface.
   **A:** G9 (efficient correction) licenses letting users edit an agent's plan rather than only approve/reject it; G10 (scope when in doubt) licenses the agent asking a clarifying question instead of guessing; G11 (explain why) licenses showing the reasoning behind a proposed action before asking for approval.
4. Describe the control-versus-friction design tension in your own words. What concrete, measurable signals (per Smashing Magazine's proposed metrics) would tell a team they've erred too far in either direction?
   **A:** Too little control lets errors through and erodes trust (users feel "ambushed" and disable the feature); too much creates approval fatigue and a scalability/cost bottleneck. Signals: Override Frequency >10%, Escalation Frequency outside the 5–15% band, Reversion Rate >5%.
5. Why does surfacing an agent's confidence or uncertainty help prevent automation bias, and what happens to that protective effect if an interface asks for the same kind of confirmation on every action regardless of stakes or confidence?
   **A:** Confidence signals let users allocate scrutiny unevenly — reading closely on low-confidence actions instead of rubber-stamping everything. If every action gets the same confirmation regardless of stakes, that signal disappears and the gate becomes functionally identical to no gate at all, just slower.

---

## Sources used

- [Guidelines for human-AI interaction design (Amershi, Vorvoreanu, Horvitz et al., CHI 2019) — Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/)
- [Interrupts — LangGraph / LangChain Docs](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Human-in-the-Loop (HitL) Agentic AI Systems: Balancing Autonomy and Oversight for High-Stakes Use Cases — OneReach.ai](https://onereach.ai/blog/human-in-the-loop-agentic-ai-systems/)
- [Designing For Agentic AI: Practical UX Patterns For Control, Consent, And Accountability — Victor Yocco, Smashing Magazine (Feb 2026)](https://www.smashingmagazine.com/2026/02/designing-agentic-ai-practical-ux-patterns/)
- [AI Agents With Human In The Loop — Cobus Greyling, Medium](https://cobusgreyling.medium.com/ai-agents-with-human-in-the-loop-f910d0c0384b)
- [Simple Calculator - Human in the Loop (examples/HITL/simple_calculator_hitl) — NVIDIA NeMo Agent Toolkit, GitHub](https://github.com/NVIDIA/NeMo-Agent-Toolkit/blob/develop/examples/HITL/simple_calculator_hitl/README.md)

*Cross-references: Lesson 1.1 (intuitive-UI fundamentals — memory, grounding, latency, modality; not re-taught here), Lesson 10.2 (the feedback loop that turns granular in-the-moment corrections into model/system improvement), Lesson 10.3 (transparency and explainability of agent reasoning as its own design problem, beyond the progressive-disclosure sketch given here), Lesson 10.4 (human-in-the-loop control design for accountability and oversight — who is authorized and competent to approve, not just how the approval UI is built), Lesson 2.5 (real-time streaming and feedback UX), Lesson 8.1 (alert fatigue), Lesson 8.5 (calibrated trust and automation bias), Lesson 9.4 (escalation overload).