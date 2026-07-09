# Orchestrating with LangGraph and the Agent SDKs

## The graph is a design artefact

A single tool call is a function invocation. A sequence of tool calls that plans, retrieves, decides, mutates, and reports back is a program. The applied-AI engineer's third artefact — after the tool schema and the error taxonomy — is the shape of that program.

The tempting first version is a `while` loop that calls the model, dispatches on any tool-use blocks, appends the results, and repeats until the model stops asking. That loop works for demos. It stops working when the surface has to answer four questions the loop doesn't:

- **What steps happened?** Debug and audit require replayable per-step state, not just a message list.
- **Which steps can retry independently?** A transient failure on step 4 should not re-run steps 1–3.
- **Which steps can run in parallel?** Reading two independent sources is not a serial job.
- **Where does a human enter the loop?** A confirmation, an approval, an intervention.

Each answer is a shape decision. The frameworks named in this chapter — LangGraph, the OpenAI Agents SDK, the Anthropic Claude Agent SDK — differ on how they encode those shapes. This chapter is a working knowledge of each, why you would pick one, and how to keep the graph shape maintainable as the surface grows.

Every framework named here changes on a monthly-to-quarterly cadence. API surface examples in this chapter show the *idea*; check current docs before pinning to a specific API.

## Two flavours of "agent framework"

The frameworks you'll compare fall into two rough camps.

**Loop-shaped SDKs** — the `while sample-tools-execute-repeat` inner loop is the framework's core. The developer writes tool implementations and a system prompt; the framework handles the loop, streaming, tool result formatting, and provider-specific quirks. **The Anthropic Claude Agent SDK** and **the OpenAI Agents SDK** live here. Both expose extension points (hooks, guardrails, handoffs) but you rarely write the outer loop yourself.

**Graph-shaped frameworks** — you define a state machine of typed nodes and edges. Nodes may or may not be LLM calls; edges route by inspecting the state. **LangGraph** is the reference case. The graph is a first-class object you version, checkpoint, and reason about.

Neither is strictly superior. Loop-shaped frameworks are faster to stand up and stay compact for surfaces with one clear loop. Graph frameworks pay their complexity budget when the surface has enough structure that "call the model, see what it says" isn't a fair summary of what happens between the user's request and the response.

A hybrid pattern shows up in production: a graph framework at the outer level (LangGraph orchestrating a plan → gather → decide → summarise structure) with an SDK-driven inner loop inside one of the nodes (a Claude Agent SDK loop that handles the tool-use conversation on one leg). This is often the right shape when the outer flow is deterministic but one leg is an open-ended agentic exchange.

## The Anthropic Claude Agent SDK

The Claude Agent SDK is Anthropic's harness for building agents against the Messages API. It handles the tool-use loop, streaming, and MCP wiring (chapter 03). The developer's contract:

- Register tools (native, in-process, or MCP-exposed).
- Provide a system prompt.
- Consume the streamed message events; each event tells you what the model said, called, and received.

A representative shape (the exact API evolves — check current SDK docs):

```python
from claude_agent_sdk import query, ClaudeAgentOptions, tool, create_sdk_mcp_server

@tool("search_orders", "Search orders by customer.", {"customer_id": str, "limit": int})
async def search_orders(args):
    …
    return {"content": [{"type": "text", "text": rendered_result}]}

async def run(user_message, ctx):
    options = ClaudeAgentOptions(
        system_prompt=SYSTEM_PROMPT,
        mcp_servers={"linear": {"type": "http", "url": "https://mcp.linear.app/sse", …}},
        allowed_tools=["search_orders", "mcp__linear__list_issues"],
        max_turns=8,
        hooks={"PreToolUse": [pre_tool_use_hook]},  # budget, audit, injection guard
        permission_mode="acceptEdits",  # or "plan", "bypass" per surface
    )
    async for msg in query(prompt=user_message, options=options):
        yield msg
```

The four things this shape gives you cheaply:

1. **Tool loop.** You don't hand-write "sample → detect `tool_use` → dispatch → append `tool_result` → sample again."
2. **MCP wiring.** Servers are configured, discovered, and integrated automatically.
3. **Hooks.** `PreToolUse`, `PostToolUse`, and stop-condition hooks let you plug budgets, guardrails, and audit trails into the loop without owning it.
4. **Streaming and cancellation.** The SDK yields incremental messages the surface can render.

What you give up:

- The loop is the graph. Complex control flow — "if the search returns no results, try three parallel rewrites; otherwise decide between two paths" — has to fit inside the model's tool-use decisions or into hook logic that inspects and steers the loop. Neither is unmaintainable, but neither is as legible as a graph diagram either.
- Multi-model orchestration (a cheap model for classification, a stronger model for reasoning — mod-108's territory) is possible but requires composing multiple SDK invocations rather than being modelled directly.

Where it fits: co-pilots and assistants whose central shape is one open-ended agentic exchange with the user, backed by a well-scoped tool set. Chat surfaces, code assistants, ticket triagers, most single-user product agents.

## The OpenAI Agents SDK

The OpenAI Agents SDK is OpenAI's harness for building agents against the Responses API. It ships with three primitives beyond tools: **handoffs** (agent-to-agent transfer), **guardrails** (parallel checks on input and output), and **sessions** (managed multi-turn state).

```python
from agents import Agent, Runner, function_tool

@function_tool
def search_orders(customer_id: str, limit: int = 20) -> list[Order]:
    …

triage_agent = Agent(
    name="triage",
    instructions="Route customer requests to the right specialist.",
    tools=[search_orders],
    handoffs=[refunds_agent, shipping_agent],  # transfer control to another agent
    input_guardrails=[pii_check],
    output_guardrails=[refund_amount_bound],
)

result = await Runner.run(triage_agent, user_message)
```

The distinctive move is **handoffs**: an agent explicitly transfers a turn to a specialist. Rather than one mega-prompt with fifteen tools, a triage agent hands off to a refunds specialist with its own three tools and its own system prompt. The framework represents the handoff as a first-class construct with its own tracing entry, not as an inline function call. Guardrails run in parallel with the model call and can short-circuit before a bad response is used.

MCP support is first-class in the SDK; the Responses API also accepts remote MCP servers directly as a tool type, so a surface can consume MCP either through the SDK's client abstraction or by declaring the server to the Responses API itself.

Where it fits: multi-agent product flows where routing to a specialist is a first-class step, and surfaces that already sit on OpenAI's Responses API and want the built-in tracing (`traces.openai.com`) to work out of the box. The handoff primitive is genuinely useful when the surface has more than one "role" the model plays.

What you give up: the framework's opinionated multi-agent model is where its value is. If your feature is really one agent with many tools, the handoff and guardrail machinery adds surface area you don't need.

## LangGraph

LangGraph is a state-graph library, not an agent SDK. The unit is a `StateGraph` of typed nodes with explicit edges. LLM calls, tool calls, retrievals, and human-in-the-loop pauses are all just nodes.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.postgres import PostgresSaver

class TriageState(TypedDict):
    messages: Annotated[list, add_messages]
    intent: str | None
    refund_context: RefundContext | None

def classify(state): …
def route(state): return "refund_agent" if state["intent"] == "refund" else "policy_qa"
def gather(state): …
def draft_reply(state): …

builder = StateGraph(TriageState)
builder.add_node("classify", classify)
builder.add_node("gather", gather)
builder.add_node("draft_reply", draft_reply)
builder.add_node("tools", ToolNode([search_orders, issue_refund]))
builder.add_conditional_edges("classify", route, {
    "refund_agent": "gather", "policy_qa": "draft_reply",
})
builder.add_edge("gather", "tools")
builder.add_edge("tools", "draft_reply")
builder.add_edge("draft_reply", END)
builder.set_entry_point("classify")

# Checkpointer persists state between steps — enables resume, human-in-the-loop, retries.
graph = builder.compile(checkpointer=PostgresSaver.from_conn_string(...))
```

What you get by adopting the state-graph frame:

- **Legibility.** The graph is a diagram; you can literally render it. Reviews compare graphs, not commit diffs.
- **Checkpointing and resume.** State can persist between nodes, keyed by a thread ID. A tool failure on step 4 doesn't lose steps 1–3; a human-in-the-loop pause is a first-class concept.
- **Parallelism.** Independent branches are independent nodes; the runtime executes them concurrently.
- **Streaming per node.** Every node's incremental output is emitted, letting the UI render intermediate steps rather than a spinner.

MCP-tool integration lives at the tool layer via `langchain-mcp-adapters` (chapter 03). LangGraph is model-agnostic; you can wire it to Claude, GPT, Gemini, or an open-weight model through LangChain's chat-model abstractions.

Where it fits: surfaces where the flow is more than a single-model tool loop — plan-then-execute, multi-step pipelines with heterogeneous nodes, multi-agent flows with explicit routing, agents that need to pause for a human. Anything where "what happened on step 4 given state 3" is a debug question you expect to ask.

What you give up: startup cost. The `StateGraph` is more setup than "sample the model and dispatch on tools." For surfaces that never need the state-graph properties, the setup pays no return.

## Picking one, and picking two

The one-question triage:

- **Is the surface fundamentally one open-ended chat with tools?** Claude Agent SDK or OpenAI Agents SDK. Pick by which provider stack you already ship on.
- **Is the surface a plan-then-execute pipeline, or a multi-agent flow with explicit routing, or has a human-in-the-loop step?** LangGraph.
- **Both, at different levels?** LangGraph outer, one of the SDKs inner. Common in mature surfaces.

Two follow-ups worth answering before you pick:

1. **What does debug look like on turn 34?** The framework whose logs, traces, and state-recovery story you can imagine cold is the one that will survive on-call.
2. **What does a Red Team run look like?** Which framework's hook surface lets you interpose the guardrails you need — input classification, output safety, spend cap — without rewriting?

## Graph shape principles

Whatever framework you pick, the graph shape is your artefact. Five principles keep it maintainable.

### 1. Nodes have a single responsibility

A node either calls the model, calls tools, transforms state, or waits for a human. Nodes that do two of those become debug traps. If a node "asks the model and then runs the tools it chose," split it — the LangGraph pattern is one node for the model call, a `ToolNode` for tool dispatch, and a conditional edge that routes based on whether the model asked for a tool.

### 2. State is typed and shared, not passed via messages

The temptation is to encode all cross-node state as messages in a growing chat history. It works and it hides regressions. A typed state object (`TypedDict`, Pydantic model, dataclass) is what the graph carries; messages are one field among many. `intent`, `tenant_id`, `budget_remaining`, `evidence_ids`, and any structured intermediate results live in the state and are inspectable at every step.

### 3. Parallel branches join deterministically

A `gather_ticket_context` node and a `gather_customer_context` node can run in parallel. They must both write to well-defined slots in the state; the join node reads both and proceeds. Racing to write the same slot is a bug the framework won't catch. Design the state schema so parallel branches don't overlap in what they set.

### 4. Retries are per-node

An orchestration where the "retry" happens by re-entering the whole graph is fragile. Frameworks that support per-node retry policies (LangGraph does at the node level, both SDKs do at the tool level) let you scope: "retry the tool call 3 times with backoff, but don't re-run classification." Chapter 02's retry-policy discipline is applied per node, not per turn.

### 5. Human-in-the-loop is a node type

Any point where a human confirms, approves, edits, or rejects is a node with a well-defined input (what you're asking approval for), a bounded wait, and a well-defined output (approved / edited / rejected). LangGraph exposes this directly with checkpoint-and-interrupt; the SDKs express it via a hook or a "reason" the loop stops. Either way, if the workflow can pause for a human, that pause is drawn on the graph, not left implicit in application code.

## State, memory, and multi-turn conversations

Chapter 01 (mod-102) already frames a prompt as a sequence of turns feeding a stateless model. In an agentic surface, that state grows: user messages, model messages, tool results, intermediate scratchpads, retrieved evidence. Where does it live?

- **In the framework's session / checkpoint store.** OpenAI Agents SDK ships session abstractions; LangGraph ships `SqliteSaver`, `PostgresSaver`, `RedisSaver`, and others. Both persist by a `thread_id`.
- **In your own store, keyed by request.** Suitable when the framework's session model doesn't fit (e.g., you need cross-user memory, or you're pairing a graph across two hosts).
- **Nowhere long-lived; state is a per-turn value.** Fine for surfaces where each user turn is independent. Do not overbuild persistence for a stateless UX.

Whichever you pick, three things are non-negotiable:

- **Per-tenant isolation.** No cross-tenant read of thread state. Enforce at the store, not the app.
- **State-schema versioning.** When you add a field, old checkpoints must still deserialise. LangGraph's Pydantic-model state and TypedDict state both let you evolve the schema; do it deliberately.
- **A retention policy.** Checkpoints and traces cost storage. Pick a retention window per surface, publish it, and enforce with a job. Include it in your privacy review.

## Observability at the graph level

The single biggest reason to work in one of these frameworks rather than a raw loop is that observability comes for free — if you turn it on and use it.

- **LangSmith** (LangChain-hosted) traces every LangGraph node and every LangChain LLM call. Free tier is enough for a single-team dev deployment; production tenancy is a paid tier.
- **Langfuse** is the open-source alternative; hosts the same trace surface with a self-host option.
- **OpenAI traces** (accessible via the Agents SDK) capture handoffs, tool calls, guardrails, and streamed outputs by default.
- **Anthropic Claude Agent SDK** integrates with your existing OpenTelemetry pipeline; the streamed messages are the primary observability signal, and hooks let you emit structured spans.

Whatever you use, mod-110 will consume this data. Make sure the trace has, per span:

- The tool name, arguments (with PII scrubbed), and result.
- The token counts, latency, and cost.
- The idempotency key (chapter 01) — so double-write debug is one filter query.
- The `trace_id` you propagate to every downstream service.

## Testing agent graphs

A stateful graph is testable in ways a raw loop isn't.

- **Node-level tests.** Each node is a function of state; test it in isolation with a hand-built state fixture. This is where the argument-fidelity tests from chapter 01 land — a "call the model with these tools and inspect what tool it chose" test is a node test.
- **Path tests.** Given a set of stubbed nodes, does the graph reach the expected terminal state? These verify routing logic without depending on live model output.
- **Trajectory replays.** A trace from a recorded conversation is replayed through the graph. Mismatches (a node that took a different branch than the recorded run) are regressions. Especially useful when the graph is stable but the model version bumped.
- **Eval-set integration.** Every fixture in the mod-106 eval harness runs against the graph, not the raw model, so eval measures the whole surface, not just prompt quality.

Two rules for the test suite:

1. **Don't test the framework.** LangGraph's `ToolNode` is tested; you don't need a test that "confirms tools run." Test your nodes, your state transitions, and your graph shape.
2. **Frozen fixtures for non-deterministic nodes.** Snapshot model outputs for the trajectory-replay tests; mock external services deterministically. A test that fails when the model rephrases is not a signal.

## A worked example: the refund co-pilot as a graph

Return to the refund surface referenced in chapters 01 and 02. As a Claude Agent SDK loop with one system prompt and a small tool set, it fits fine.

As a **LangGraph state graph**, it looks like:

- `classify_intent` — model call; sets `state.intent`.
- `authorise` — deterministic function; sets `state.principal_can_refund`.
- `gather_evidence` — tool node calling `get_order`, `list_prior_refunds`; sets `state.evidence`.
- `decide_refund_amount` — model call; sets `state.proposed_amount`.
- `confirm_with_user` — human-in-the-loop; sets `state.user_confirmed`.
- `execute_refund` — tool node calling `issue_refund` with `state.proposed_amount` and a canonical idempotency key; sets `state.refund_result`.
- `respond` — model call rendering the outcome; sets `state.response`.

Why the graph? Because the incident report from January said "the refund co-pilot double-charged a customer when the network flapped between step 5 and step 6." In the loop version, retry logic straddles many concerns. In the graph, step 6 has an idempotency key sourced from `state.canonical_key`, a per-node retry policy of 3 attempts with backoff, and a checkpoint before invocation — replays are safe. That is the graph paying for itself.

## Anti-patterns

Five that come up across all three frameworks.

1. **The mega-node.** One "agent" node that runs a full nested loop, calls tools, formats output, and does state transitions. The observability system sees one span for what was actually a mini-agent.
2. **Shared mutable dicts as state.** Multiple nodes writing to the same field with last-write-wins semantics. Race conditions in parallel branches. Symptoms: "tests pass but production is flaky."
3. **Retrying by re-entering the graph.** A caller loops "call graph, if it fails, call graph again," instead of the graph handling per-node retry. Loses the mid-flight state, wastes tokens, re-executes side effects.
4. **Framework lock-in bleed.** LangGraph node bodies call OpenAI Agents SDK directly. Now debugging requires knowing both frameworks. Prefer a small adapter layer over ad-hoc cross-calls.
5. **No graph diagram in the PR.** A change to the routing logic that only shows up as a diff is a change nobody reviewed. Render the graph in CI, commit the SVG, and require it in every PR that touches structure.

## Summary

- The graph is the surface's third artefact after schema and error handling. A raw sample-dispatch loop is a design shortcut that doesn't survive audit, retry-safety, parallelism, or human-in-the-loop requirements.
- Loop-shaped SDKs (Claude Agent SDK, OpenAI Agents SDK) win on ergonomics for single-loop surfaces. Graph frameworks (LangGraph) win on legibility and control for multi-step, multi-node, checkpointable flows. Hybrids are common.
- Nodes are single-responsibility; state is typed; parallel branches join deterministically; retries are per-node; human-in-the-loop is a node.
- Framework state stores are per-tenant, versioned, and retention-bounded. Observability (LangSmith, Langfuse, OpenAI traces, OpenTelemetry) is the reason to work in a framework at all — turn it on, feed it the right per-span fields, and it becomes the mod-110 substrate.
- Testing is node-tests + path-tests + trajectory replays + eval-set integration. Don't test the framework; do frozen-fixture the non-deterministic nodes.
- Five anti-patterns: mega-node, shared-mutable-state, retry-by-re-entering-graph, framework-lock-in bleed, no-diagram-in-PR.

[Chapter 05](05-per-request-budgets-and-graceful-degradation.md) turns to the last invariant every graph must respect: the per-request budget in tokens, steps, wall-clock, and dollars, and the graceful-degradation path that keeps the surface honest when a budget hits.
