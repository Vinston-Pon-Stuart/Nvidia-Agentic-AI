## In one line

**Tool/function calling** is the concrete mechanism behind ReAct's "Action" step: the model emits a structured request (tool name + arguments) that your application code — never the model — executes, returning the result as an observation the model reads on the next turn.

---

## The core mechanism

- OpenAI's function-calling loop, five steps (same shape across every vendor/framework):
  1. Request the model with a list of available tools.
  2. Model returns a tool call — structured name + arguments.
  3. Application code executes it.
  4. Second request sends the tool's output back to the model.
  5. Model returns a final response (or another tool call, restarting the loop).
- **The fact most likely to be tested:** OpenAI's cookbook states it explicitly — *"the API will not actually execute any function calls. It is up to developers to execute function calls using model outputs."* Anthropic's docs describe the identical split: Claude returns `stop_reason: 'tool_use'` + `tool_use` blocks; **your code** executes and sends back a `tool_result`. **The model never executes anything itself.**
- Anthropic distinguishes **client tools** (your application executes them, user-defined or Anthropic-schema like `bash`) from **server tools** (Anthropic's infrastructure executes them, e.g., web search) — the round trip's shape is identical; only the executor changes.

---

## Anatomy of a tool definition

| Field | OpenAI (`tools`) | Anthropic (`tools`) | Purpose |
|---|---|---|---|
| Name | `name` | `name` (regex `^[a-zA-Z0-9_-]{1,64}$`) | Unique identifier the model emits to select the tool |
| Description | `description` | `description` | What it does, when to use it, when not to |
| Parameters | `parameters` (JSON Schema) | `input_schema` (JSON Schema) | Expected arguments, types, enums, required fields |

- Both use plain **JSON Schema** — same schema language MCP's `inputSchema` uses, so a well-formed function-calling tool and a well-formed MCP tool are structurally interchangeable.
- **The description is where reliability is won or lost**, not the schema:
  - **Anthropic:** "extremely detailed descriptions... by far the most important factor in tool performance" — what it does, when it should/shouldn't be used, each parameter's meaning, caveats; aim for 3–4+ sentences. Consolidate related operations into fewer tools with an `action` parameter; namespace tools (`github_list_prs`) as tool count grows.
  - **OpenAI:** clear, detailed names/parameter descriptions; apply the **"intern test"** — could a human use the function correctly given only what the model was given?; use enums/object structure so invalid states are unrepresentable; don't make the model guess arguments your code already knows (pass `order_id` from context).
- **`strict: true`** (both platforms) forces exact schema conformance instead of "best effort" — the direct fix for hallucinated/malformed arguments, at the cost of schema restrictions (e.g., `additionalProperties: false` required).

---

## Wiring real APIs — the handler's three jobs (the model never sees these)

- **Auth** — credentials live in the handler's HTTP layer, never in the tool schema or model context. MCP's Streamable HTTP transport carries auth via standard HTTP mechanisms (bearer tokens, API keys, custom headers), OAuth recommended.
- **Input validation** — JSON Schema validates *shape*; the handler still validates *legality* (does this resource exist, does this user have permission) before hitting a real API.
- **Output shaping** — Anthropic: tool responses should "return only high-signal information... semantic, stable identifiers rather than opaque internal references." A raw API dump wastes tokens and buries the field the model needs.
- **NVIDIA's NeMo Agent Toolkit (NAT)** formalizes this as a **Function**: a reusable component wrapping a Python callable with Pydantic-validated input/output schemas, async execution (`ainvoke`/`astream`), single or streaming output. Both a tool *and* an agent are Functions in NAT — which is why an agent can be handed to another agent as a tool.

---

## Parallel and multi-tool calling

- A single turn isn't limited to one call — Claude 4 calls multiple tools by default when useful; OpenAI controls this via `parallel_tool_calls` (`false` forces zero-or-one per turn).
- **Execution order is your decision, not the API's** — Anthropic: independent read-only operations are usually safe concurrently (lower latency); tools with side effects, shared state, or ordering dependencies should run sequentially.
- **Every `tool_use` needs a matching `tool_result`**, returned together in one message, matched by ID — even a skipped call needs a result (`is_error: true`). Splitting results across messages silently teaches the model to stop batching calls.
- NIM's OpenAI-compatible endpoint exposes the identical `tools` + `tool_choice` (`"auto"`/`"required"`/`"none"`/named-tool object) mechanism, docs deferring directly to OpenAI's reference.

---

## MCP vs. bespoke integration

- Bespoke = a schema + handler you hand-write per API, per framework, repeated every time.
- **Model Context Protocol (MCP)**, open-sourced by Anthropic, standardizes "connect once, reuse everywhere" via client-server architecture:
  - **Host** — the AI application (a NAT workflow, Claude Desktop, VS Code).
  - **Client** — one connector per server, maintaining a dedicated connection.
  - **Server** — exposes **tools** (executable functions), **resources** (contextual data), **prompts** (reusable templates) over JSON-RPC 2.0.
- Two layers: **data layer** (JSON-RPC 2.0 for lifecycle/capability negotiation + the three primitives) and **transport layer** — **stdio** for local servers, **Streamable HTTP** for remote (SSE for incremental streaming, standard HTTP auth, OAuth recommended).
- **Discovery** is what bespoke lacks: `tools/list` returns the same three fields as a hand-written definition (`name`, `description`, `inputSchema`) but generated dynamically by the server. Execution is `tools/call`. A `notifications/tools/list_changed` message tells connected clients to refresh if the server's tool surface changes — a failure mode bespoke integrations have no protocol-level defense against (schema drift, below).
- **NAT's `mcp_client`** function group: point at a server URL, NAT "handles the MCP server connection, tool discovery, and function registration," turning every discovered MCP tool into a regular NAT Function — no per-tool schema/handler to write. NAT can also publish its own workflows as an MCP server (`nat mcp serve` / FastMCP).
- **MCP ≠ A2A:** MCP standardizes **agent-to-tool** connection; **agent-to-agent** delegation between independently built agents is A2A (1.3) — different problems.

---

## Trade-offs and pitfalls

- **Wrong-tool selection** grows with tool count and vague descriptions. OpenAI: keep fewer than ~20 functions available at once; use tool search or MCP's `include`/`exclude` filtering (NAT exposes this) to defer rarely-used tools.
- **Hallucinated arguments** — the schema-conformance failure mode, mitigated (not eliminated) by strict/structured-output modes.
- **Excessive agency (security, not just accuracy)** — OWASP's LLM08 names three root causes: **excessive functionality** (a tool grants more capability than the task needs — a read-only summarizer wired to a plugin that can also send mail), **excessive permissions** (backend credential broader than the task requires), **excessive autonomy** (no human checkpoint before a high-impact action). OWASP's worked example: a malicious email tricks an agent into invoking a "send message" function it only had as an unused side effect of a "read mail" plugin. **Fix:** least-privilege credentials scoped per tool, narrow single-purpose tools (a file-write tool, not a raw shell-command tool), human-in-the-loop confirmation for irreversible actions.
- **Tool-call failure handling** is flagged here but covered fully in 2.4 — that both Anthropic's `is_error` flag and NAT's `reconnect_max_attempts`/`tool_call_timeout` exist shows the industry treats tool failure as routine, not edge case.
- **Latency and token cost** — every tool definition sent counts against context and is billed as input tokens (OpenAI); Anthropic's tool-use system prompt adds measurable overhead. More tools, longer descriptions, more round trips (each Thought→Action→Observation cycle) all cost real money and time.
- **Schema drift** — solved at the protocol level by MCP's `list_changed` notifications, but a silent risk in bespoke integrations: if an upstream API's contract changes and nobody updates the hardcoded schema, the model keeps emitting calls the backend no longer honors.

---

## NVIDIA angle, read critically

- **Conformance and tooling on external standards, not a proprietary mechanism**, throughout:
  - **NIM** exposes tool calling through an OpenAI-compatible `chat/completions` endpoint — same `tools`/`tool_choice` shape, docs point directly to OpenAI's reference. Code built against OpenAI's contract transfers to NIM with a base-URL swap.
  - **NeMo Agent Toolkit** contributes a genuinely useful abstraction: the typed, async, Pydantic-validated **Function**, unifying "tool" and "agent" into one interface — the plumbing that makes agent-as-tool composition and shared ReAct-loop execution possible.
  - **MCP support** in NAT (client and server) wraps Anthropic's protocol — connection handling, discovery, auto-registration, CLI/UI tooling — not a competing standard.
- Honest reading: NVIDIA's value-add is enterprise plumbing (observability, multi-framework tool wrappers for LangChain/CrewAI/Agno, auth, evaluation) layered over mechanisms — OpenAI-style function calling, Anthropic's MCP — it did not invent and does not control.

---

## How to think about it for the exam

| Idea | Mechanism / takeaway |
|---|---|
| Who executes a tool call | Never the model. The model emits a structured request (name + arguments); your application (or, for server tools, the vendor's infrastructure) executes it and returns an observation. |
| The tool-calling loop | Request with tools → model returns a tool call → your code executes it → you send the result back → model responds (possibly with another tool call). This *is* the mechanism behind ReAct's Action step (1.2). |
| Anatomy of a tool definition | `name` + `description` + JSON-Schema `parameters`/`input_schema`. Structurally identical across OpenAI, Anthropic, and MCP's `inputSchema`. |
| What drives correct selection | Description quality, not the schema. Detailed, unambiguous descriptions; fewer, consolidated, namespaced tools beat many overlapping ones. |
| Parallel tool calls | Model may emit several `tool_use`/`tool_call` blocks per turn; execution order (parallel vs. sequential) is the application's choice; every call needs a matched result. |
| MCP vs. bespoke integration | MCP = standardized client/server protocol (JSON-RPC 2.0) for discovering (`tools/list`) and calling (`tools/call`) tools at runtime, over stdio or Streamable HTTP; "connect once, reuse," with built-in change notifications. Bespoke = hand-written schema + handler per API, no discovery, no drift protection. |
| MCP vs. A2A | MCP = agent-to-tool. A2A (1.3) = agent-to-agent delegation. Different problems, don't conflate. |
| Top failure modes | Wrong tool chosen (too many/vague tools); hallucinated arguments (mitigated by strict/structured schemas); excessive agency / injected tool calls (OWASP LLM08 — fix with least privilege + human-in-the-loop); tool failure handling (→ 2.4); latency/token cost from tool definitions and round trips; schema drift. |
| NVIDIA's actual contribution | OpenAI-compatible tool calling in NIM; a typed Function abstraction unifying tools and agents, plus MCP client/server support, in NeMo Agent Toolkit — conformance and enterprise plumbing on external standards, not a new protocol. |

---

## Quick self-check

1. When Claude or GPT returns a tool call, what exactly has happened — has any code been executed yet? Who executes it?
   **A:** No code has run yet — the model has only emitted a structured request naming a tool and its arguments. Your application code (or the vendor's infrastructure, for server tools) executes it.
2. Name the three fields every tool definition needs, across OpenAI, Anthropic, and MCP. Which one most determines whether the model picks the right tool, and why?
   **A:** Name, description, and a JSON-Schema parameters definition (`parameters`/`input_schema`/`inputSchema`). The **description** matters most — both OpenAI and Anthropic call it the primary driver of correct tool selection, since it's the only place explaining what the tool does and when to use it.
3. You have 60 tools registered and the agent keeps picking the wrong one. Name two concrete fixes suggested by OpenAI's and Anthropic's own guidance.
   **A:** Keep fewer than ~20 tools available at once (defer rarely-used ones via tool search or MCP include/exclude filtering), and consolidate/namespace related tools with clearer, more detailed descriptions rather than many overlapping ones.
4. What does MCP add that a hand-written, bespoke tool integration does not — specifically around discovery and schema changes?
   **A:** MCP adds runtime **discovery** (`tools/list` generates schema dynamically from the server, no hardcoding) and a `notifications/tools/list_changed` message that tells clients to refresh when the server's tools change — bespoke integrations have no protocol-level defense against schema drift.
5. Describe a concrete "excessive agency" scenario (per OWASP LLM08) involving a tool that has more capability than the task needs, and name the fix.
   **A:** A "read mail" plugin also has an unused "send message" function; a malicious email prompt-injects the agent into invoking it. Fix: least-privilege credentials scoped per tool, narrow single-purpose tools, and human-in-the-loop confirmation for irreversible actions.

---

## Sources used

- [Function calling — OpenAI API docs](https://developers.openai.com/api/docs/guides/function-calling)
- [How to call functions with chat models — OpenAI Cookbook](https://developers.openai.com/cookbook/examples/how_to_call_functions_with_chat_models)
- [Tool use with Claude — Claude Platform Docs (Overview)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Define tools — Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Parallel tool use — Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)
- [What is the Model Context Protocol (MCP)? — modelcontextprotocol.io](https://modelcontextprotocol.io/introduction)
- [Architecture overview — Model Context Protocol docs](https://modelcontextprotocol.io/docs/learn/architecture)
- [LLM08: Excessive Agency — OWASP Gen AI Security Project](https://genai.owasp.org/llmrisk2023-24/llm08-excessive-agency/)
- [Call Functions (Tools) — NVIDIA NIM for Vision Language Models (VLMs)](https://docs.nvidia.com/nim/vision-language-models/latest/function-calling.html)
- [Functions — NVIDIA NeMo Agent Toolkit docs (GitHub source)](https://raw.githubusercontent.com/NVIDIA/NeMo-Agent-Toolkit/develop/docs/source/build-workflows/functions-and-function-groups/functions.md)
- [NVIDIA NeMo Agent Toolkit as an MCP Client — NVIDIA NeMo Agent Toolkit docs (GitHub source)](https://raw.githubusercontent.com/NVIDIA/NeMo-Agent-Toolkit/develop/docs/source/build-workflows/mcp-client.md)
- [NVIDIA NeMo Agent Toolkit — README (GitHub source)](https://raw.githubusercontent.com/NVIDIA/NeMo-Agent-Toolkit/develop/README.md)
