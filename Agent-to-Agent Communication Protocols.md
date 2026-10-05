## In one line

**Agent-to-agent communication protocols** are standardized message formats, discovery mechanisms, and transports that let independently-built agents (different teams, stacks, even organizations) find each other and exchange work without sharing code, memory, or a runtime.

---

## Why it's needed

- A single agent with tools (ReAct) or a control graph (see Logic Trees, Prompt Chains, and Stateful Orchestration) still does one job. Real systems split work across **multiple, independently-built agents** — a billing agent, a research agent, a supplier's agent on someone else's infrastructure.
- **Scope of this lesson:** the wire format and handshake only — not orchestration patterns built on top (supervisor/worker, swarms — see Multi-Agent Workflows and Coordination) and not agent memory (see Short-Term and Long-Term Memory).
- **The 40-year arc:** symbolic, speech-act-based languages (KQML, FIPA ACL) gave way, in the LLM era, to a fragmented new generation (MCP, A2A, ACP, ANP, Agora) reinventing the same problems for opaque, natural-language-native agents.

---

## Three sub-problems every protocol addresses

- **Message structure** — what fields does a message carry (sender, meaning, payload)?
- **Capability discovery** — how does agent A learn what agent B can do, before sending a real request?
- **Coordination** — once messages flow, how do agents divide labor, delegate, and know when something's done?

Most exam questions about "what does protocol X provide" map onto one of these three.

---

## Classic agent communication languages: KQML and FIPA ACL

- **KQML** (Knowledge Query and Manipulation Language) — came out of DARPA's Knowledge Sharing Effort, early 1990s, led by Tim Finin. Built around **performatives**: operations like `tell` (change receiver's beliefs) or `achieve` (change receiver's goals) that make the sender's *intention* explicit. Introduced **communication facilitators** — special agents that coordinate interactions among others. Higher-level patterns (contract nets, negotiation) build on top of KQML performatives. Later superseded by **FIPA ACL**.
- **FIPA ACL** (Agent Communication Language) — standardized by the Foundation for Intelligent Physical Agents; grounded in **speech act theory** (a message is itself an action). Only *mandatory* field: the **performative** (`request`, `inform`, `query-if`, `not-understood`); most messages also carry `sender`, `receiver`, `content`. Also defines `language`/`encoding`/`ontology` (how to interpret content) and conversation-control fields — `protocol`, `conversation-id`, `reply-with`, `in-reply-to`, `reply-by` — for threading multi-message exchanges and reply deadlines.
- **Why it still matters:** modern protocols reinvent almost every one of these fields under new names — A2A's `contextId` ≈ FIPA's `conversation-id`; A2A's `Task` lifecycle ≈ what `protocol` + `reply-by` managed. The underlying problems didn't change; only the substrate (symbolic logic vs. natural language/JSON) did.

---

## Contract Net Protocol: delegation as coordination

- **Contract Net Protocol (CNP)** — Reid G. Smith, 1980. A task-sharing protocol run as an announce–bid–award cycle:
  1. A **manager** sends a **call-for-proposals** for a task to candidates.
  2. Each **contractor** replies with a **proposal** (bid) or a **refuse**.
  3. The manager sends **accept-proposal** to the winner, **reject-proposal** to the rest.
  4. On completion, the contractor sends **inform** (result), or **cancel**/failure if it can't deliver.
- Tasks can be recursively subcontracted (a contractor becomes manager for a sub-task) — exactly "A delegates to B, who delegates part to C."
- FIPA formalized CNP (plus an **Iterated Contract Net** variant letting the manager re-solicit bidders instead of committing on the first round) directly in ACL.
- **CNP is the conceptual ancestor of A2A task delegation** — a client agent sending a task to a remote agent is a simplified, non-competitive contract net.

---

## Message passing vs. shared memory (blackboard)

- Nearly every protocol below is **point-to-point message-passing**: A addresses a message directly to B.
- **Blackboard architecture** (1970s Hearsay-II speech-recognition system) is the older alternative: specialist "knowledge sources" don't message each other — they read/write a shared data structure (the "blackboard"), watching for information they're qualified to act on; a **control shell** decides who acts next. Coordination is **indirect** — agents don't need to know who else exists.
- **Exam distinction:** message passing **decouples in time but couples sender-to-receiver** (needs an address/discovery step); blackboard **decouples sender from receiver entirely but requires a shared representation** and creates a central point of contention. If agents don't address each other directly but all read/write common state, that's blackboard — not A2A or FIPA ACL.

---

## Modern LLM-agent protocols

**MCP is agent-to-tool, not agent-to-agent**
- Anthropic's **Model Context Protocol** (open-sourced November 2024) standardizes how an LLM app connects to external **tools, resources, prompts** (a database, API, file system).
- Uses **JSON-RPC 2.0** between three roles: **Host** (the LLM app), **Client** (connector inside the host, one per server), **Server** (exposes tools/resources/prompts).
- **Not** a protocol for two autonomous agents to negotiate a shared task. Per A2A's own docs: "A2A is about agents *partnering* on tasks, while MCP is more about agents *using* capabilities." — one of the highest-yield exam facts in this lesson.

**A2A — the agent-to-agent standard**
- Google released the **Agent2Agent (A2A) Protocol** in April 2025, donated it to a vendor-neutral **Agent2Agent Project under the Linux Foundation** in June 2025 (founding members: AWS, Cisco, Google, Microsoft, Salesforce, SAP, ServiceNow) — explicitly to keep it vendor-agnostic.
- Lets **opaque** (black-box) agents from different teams/stacks discover and collaborate as peers. Core building blocks:
  - **Agent Card** — JSON "digital business card": identity, endpoint, capabilities, auth requirements, skills list. A2A's answer to capability discovery.
  - **Task** — stateful unit of work, unique ID, defined lifecycle; supports long-running, multi-turn work.
  - **Message** — one turn of communication (role: `"user"` or `"agent"`), made of one or more **Parts**.
  - **Part** — content container: `text`, `raw` (inline bytes), `url` (external file), `data` (structured JSON) — modality-independent.
  - **Artifact** — tangible task output (document, image, structured data), distinct from a conversational Message.
  - **contextId** — groups related Tasks for context across a longer interaction.
- **Interaction mechanisms:** request/response with polling for long tasks, **streaming via Server-Sent Events (SSE)**, **push notifications** (webhooks) for very long or disconnected work.
- **Transport:** HTTP(S) with JSON-RPC 2.0. **Auth:** declared in the Agent Card; credentials travel in HTTP headers, separate from the payload.

**A2A vs. MCP**

| | MCP | A2A |
|---|---|---|
| Connects | Agent to a tool/resource | Agent to another agent |
| Granularity | Tool-level (function calls) | Agent-level (delegated, multi-turn tasks) |
| Discovery | Tool list with schemas | Agent Card with skills |
| Interaction style | Structured, often stateless | Stateful, multi-turn, conversational |
| Typical use | LLM calling a calculator, DB, weather API | Customer-service agent delegating to a billing agent |

- A2A's own docs describe them as complementary layers: an app uses **A2A to talk to other agents**, and each agent internally uses **MCP to talk to its own tools**.

**ACP, ANP, Agora — the fragmented middle layer**
- **Agent Connect Protocol (ACP)**, Cisco Outshift's **AGNTCY** initiative (now under Linux Foundation) — REST/OpenAPI standard interface to invoke/configure remote agents, part of a broader "Internet of Agents" effort with identity/directory components.
- **Naming collision:** a *different* "Agent Communication Protocol (ACP)," from IBM, uses RESTful APIs with structured JSON encoding goals/actions/intents, transport-agnostic. Two distinct specs share "ACP" — if the exam names it without context, don't assume which.
- **Agent Network Protocol (ANP)** — decentralized identity + semantic interoperability, using **decentralized identifiers (DIDs)** and JSON-LD; explicit lifecycle (creation, operation, update, termination); supports both structured protocols and natural-language negotiation.
- **Agora** — a **meta-coordination layer** above the others: integrates MCP, ANP, ACP; introduces **Protocol Documents (PDs)**, machine-interpretable specs agents exchange to select/construct the actual protocol for an interaction.
- A 2025 survey (Derouiche, Brahmi & Mazeni, "Agentic AI Frameworks: Architectures, Protocols, and Design Challenges") finds agent protocols converge on JSON/JSON-LD dynamic discovery, but **"fragmentation persists"** — HTTP dominates as transport, yet semantic heterogeneity (custom performatives vs. goal-oriented messages vs. free-text PDs) blocks seamless cross-protocol integration; no standardized "service contract" comparable to WSDL exists yet for agents.

---

## Trade-offs and pitfalls

- **Standard fragmentation** — at least five actively-promoted protocols (MCP, A2A, two things called ACP, ANP, Agora) address overlapping problems with incompatible semantics; an agent built for one may need a translation layer for another.
- **Interoperability is declared, not guaranteed** — A2A's Linux Foundation announcement lists "trustworthy agent identity, delegated agent authority, governance policy, agent security and reputation" as *unsolved* follow-on work; the protocol standardizes wire format, not cross-org trust/authorization.
- **Security surface of opaque agents** — A2A treats the remote agent as a black box (you see its Agent Card and outputs, not internals/memory/tool use); good for encapsulation, but you're trusting a system you can't inspect. Auth is bolted on via Agent Card + HTTP headers, not baked into protocol semantics.
- **Latency and statefulness overhead** — Tasks, SSE streaming, and push-notification webhooks exist because agent-to-agent work can be long-running, but that means real infrastructure (open connections, webhook endpoints, task-state storage) vs. a single stateless tool call.
- **Message-passing needs discovery; blackboard needs a shared schema** — neither model is free: point-to-point needs an addressing/discovery step (Agent Cards, registries); blackboard needs everyone to agree on a common representation and creates contention at the shared store.

---

## NVIDIA-native angle

- NVIDIA does **not** ship a competing wire protocol. The **NeMo Agent Toolkit (NAT)** ships an **A2A integration** (`nvidia-nat-a2a` package, built on the official A2A Python SDK) letting a NAT workflow act as:
  - an **A2A client** — connects to and delegates tasks to remote A2A agents (each user gets an isolated client instance with its own auth/session state), or
  - an **A2A server** — publishes a NAT workflow as a discoverable A2A agent (via `nat a2a serve`), with OAuth2/JWT auth.
- NAT's docs echo A2A's own split: "You typically use A2A to delegate complex tasks to other agents and MCP to access tools and context. You can use both protocols together for maximum flexibility." Worked example: `examples/A2A/math_assistant_a2a` mixes an A2A-delegated calculator, an MCP time tool, and local logic.
- **Read critically:** the "NVIDIA-native" contribution is **protocol conformance and tooling** (SDK wrapper, auth, framework-agnostic client/server glue) on top of an externally-governed, Linux Foundation standard — not a proprietary protocol. NVIDIA's glossary entry on multi-agent systems lists "interoperability and agent communication" as a top tip and notes **decentralized** orchestration "requires sophisticated communication protocols" — but points outward to standards like A2A rather than defining one.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| MCP vs. A2A | MCP = agent-to-tool (stateless, function-shaped); A2A = agent-to-agent (stateful, task-shaped, multi-turn). Don't call MCP a multi-agent protocol. |
| Capability discovery | FIPA/KQML had no standard discovery layer; A2A solves it with the **Agent Card**; ANP uses DIDs + JSON-LD; older systems used manual config or a facilitator/registry. |
| Message structure lineage | FIPA ACL's mandatory `performative` + optional sender/receiver/content/conversation-id maps onto A2A's Message (role + Parts) + Task (ID, lifecycle) + contextId. |
| Coordination primitive | CNP = announce (call-for-proposals) → bid (propose/refuse) → award (accept/reject) → report (inform/cancel). Ancestor of A2A task delegation. |
| Message passing vs. shared memory | Point-to-point (FIPA ACL, A2A) addresses a specific receiver, needs discovery; blackboard coordinates indirectly through shared state, needs a common schema. |
| Transport reality | Nearly all converge on HTTP(S) + JSON (JSON-RPC for MCP/A2A, JSON-LD for ANP) — the fight is over message *semantics*, not transport. |
| Governance signal | A2A and AGNTCY's ACP are both under the **Linux Foundation** (vendor-neutral); MCP is Anthropic-originated, open-sourced but not transferred to a neutral foundation. |
| NVIDIA's role | NAT ships an A2A client/server (conformant SDK wrapper), not a new protocol — "instrumentation, not paradigm." |

---

## Quick self-check

1. A question describes an LLM agent calling a weather API through a structured, stateless function call. Is that MCP or A2A, and why?
   **A:** MCP — it's an agent-to-tool call (structured input/output, stateless), which is exactly MCP's scope; A2A is for stateful, multi-turn agent-to-agent tasks.
2. Name the three fields/objects A2A uses to represent, respectively: a single conversational turn, a long-running stateful unit of work, and a concrete deliverable output.
   **A:** **Message** (one turn, role + Parts), **Task** (stateful unit of work with ID and lifecycle), **Artifact** (tangible output like a document or image).
3. Walk through the four message types of a basic Contract Net Protocol exchange, in order.
   **A:** Call-for-proposals (manager announces) → proposal or refuse (contractor bids) → accept-proposal or reject-proposal (manager awards) → inform or cancel (contractor reports completion or failure).
4. What's the difference between message-passing coordination and blackboard coordination, in terms of whether the sender needs to know the receiver?
   **A:** Message-passing requires the sender to address a specific receiver (needs discovery); blackboard coordination is indirect — agents read/write shared state without knowing who else exists.
5. Why is it risky to assume "ACP" refers to a single protocol on this exam?
   **A:** Because two distinct specs share the initialism — Cisco Outshift/AGNTCY's Agent Connect Protocol (REST/OpenAPI) and IBM's separate Agent Communication Protocol (RESTful, JSON goals/actions/intents).

---

## Sources used

- [A2A and MCP: Detailed Comparison — A2A Protocol docs (Linux Foundation)](https://a2a-protocol.org/latest/topics/a2a-and-mcp/)
- [Core Concepts and Components in A2A — A2A Protocol docs (Linux Foundation)](https://a2a-protocol.org/latest/topics/key-concepts/)
- [Google Cloud donates A2A to Linux Foundation — Google Developers Blog](https://developers.googleblog.com/en/google-cloud-donates-a2a-to-linux-foundation/)
- [Model Context Protocol — Specification (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25)
- [Introducing the Model Context Protocol — Anthropic](https://www.anthropic.com/news/model-context-protocol)
- [FIPA ACL Message Structure Specification (SC00061G) — Foundation for Intelligent Physical Agents](https://www.fipa.org/specs/fipa00061/SC00061G.html)
- [Knowledge Query and Manipulation Language — Wikipedia](https://en.wikipedia.org/wiki/Knowledge_Query_and_Manipulation_Language)
- [Contract Net Protocol — Wikipedia](https://en.wikipedia.org/wiki/Contract_Net_Protocol)
- [Blackboard system — Wikipedia](https://en.wikipedia.org/wiki/Blackboard_system)
- [Agentic AI Frameworks: Architectures, Protocols, and Design Challenges — arXiv:2508.10146 (Derouiche, Brahmi & Mazeni, 2025)](https://arxiv.org/abs/2508.10146)
- [Agent Connect Protocol Specification — AGNTCY (GitHub README)](https://github.com/agntcy/acp-spec)
- [Agent-to-Agent Protocol (A2A) — NVIDIA NeMo Agent Toolkit docs (GitHub source)](https://github.com/NVIDIA/NeMo-Agent-Toolkit/blob/develop/docs/source/components/integrations/a2a.md)
- [What are Multi-Agent Systems? — NVIDIA Glossary](https://www.nvidia.com/en-us/glossary/multi-agent-systems/)
- [Introduction to AI Agents — Prompt Engineering Guide (DAIR.AI)](https://www.promptingguide.ai/agents/introduction)
