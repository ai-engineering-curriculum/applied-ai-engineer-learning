# exercise-04: LangGraph vs. Agents SDK Bakeoff

**Estimated effort:** 3 hours

## Objective

Implement the same product agent twice — once with a loop-shaped SDK (Anthropic Claude Agent SDK or OpenAI Agents SDK — pick one) and once with LangGraph — and compare them on the four questions from [chapter 04](../04-orchestrating-with-langgraph-and-agent-sdks.md): what steps happened, which can retry independently, which can run in parallel, and where a human enters. The exercise is not about picking a "winner." It is about developing the instinct for which framework fits which surface — and knowing the specific cost you pay when you pick wrong.

## Learning outcomes

By the end you should be able to:

- Rebuild the same feature as (a) a loop-shaped SDK program and (b) a LangGraph state machine, and articulate why one shape reads more clearly for this feature.
- Wire per-node retry, parallel branches, and human-in-the-loop as first-class constructs in the graph shape.
- Wire the same behaviours as hooks / handoffs / guardrails in the SDK shape.
- Attach observability (LangSmith, Langfuse, OpenAI traces, OpenTelemetry) such that a debug session on turn 34 does not require reading source code.
- Produce a written recommendation defensible to a peer engineer who prefers the other framework.

## Prerequisites

- You have read [chapter 04](../04-orchestrating-with-langgraph-and-agent-sdks.md).
- You have completed exercise-01 through exercise-03. The tools, error taxonomy, and MCP integrations are what you'll wire under both frameworks.
- You have API access to at least one frontier model provider (Anthropic Claude, OpenAI, or a routing surface that proxies both).
- You have `langgraph`, one of `claude-agent-sdk-python` / `openai-agents-python`, and — optionally — LangSmith or Langfuse for tracing.

## Problem statement

You are still on the **support-triage co-pilot** (exercise-03). The current alpha version was built as a Claude Agent SDK loop with one system prompt, the seven allowlisted MCP tools, and a `max_turns=8`. It works, until it doesn't.

Three failure modes surfaced in the last on-call rotation:

1. **Debug is opaque.** When a turn goes wrong ("the co-pilot commented on the wrong issue"), the on-call engineer has to read the full streamed transcript top to bottom. There is no "step 3 was the classification" waypoint.
2. **A slow tool blocks unrelated work.** A `list_recent_outages` call that takes 4 s currently sits on the critical path even when the model already has the ticket body. Parallelising the two would trim p95.
3. **The PM wants a human confirmation.** For any comment posted on an *external* customer's issue (as opposed to an internal team's), the engineer must approve the exact text before it lands. The current loop has no natural place for this.

Your job: rebuild the surface twice with the *same tool set and same eval fixtures*, once in each framework, and produce a recommendation.

## Requirements

### Deliverable 1 — The shared eval fixture

Before writing any orchestration code, produce `fixtures/turn_examples.jsonl` with ≥ 10 fixtures. Each fixture has:

- A user message (a realistic support ticket).
- Whether the ticket belongs to an internal team or an external customer.
- The expected tool-call sequence (loose — any of `search_articles`, `lookup_customer`, `list_issues`, `create_comment`; internal tickets skip `create_comment` confirmation; external tickets require it).
- The expected final response, or an "insufficient context" outcome.

Both implementations run against the same fixture set. Neither implementation may pass a fixture the other cannot.

### Deliverable 2 — Implementation A: loop-shaped SDK

Under `impl_sdk/`:

- Pick either the Claude Agent SDK or the OpenAI Agents SDK. Document the pick and why (usually: which provider your team's other surfaces run on).
- Wire the seven allowlisted MCP tools from exercise-03.
- Include a `PreToolUse` (Claude) or `input_guardrail` (OpenAI) hook that:
  - Enforces the per-tool cap from exercise-03.
  - Rejects a `create_comment` on an external-customer ticket unless a `user_confirmed=True` flag is set on the session (see the next bullet).
- Model the human-in-the-loop step as an early-exit from the loop: when the model calls `create_comment` on an external ticket and `user_confirmed` is not yet set, the hook cancels the tool call, the surface returns a proposed-comment draft to the user, and a subsequent turn resumes with `user_confirmed=True`.
- Parallelise `list_recent_outages` and `lookup_customer` — the two are independent given the customer_id. If the SDK does not expose parallel tool-use natively, implement the parallelism inside a single `wrap_customer_context` tool that fans out and returns the merged result.
- Wire the exercise-02 error runtime for retries and repairs.
- Wire tracing to LangSmith, Langfuse, OpenAI traces, or OpenTelemetry — whichever your SDK integrates with. Every model call and every tool call is a span.

### Deliverable 3 — Implementation B: LangGraph

Under `impl_langgraph/`:

- Typed `TypedDict` or Pydantic `State` with fields for the ticket, the classified intent, the customer context, the proposed comment, `user_confirmed`, and any partial output.
- Nodes: `classify_ticket_kind` (model), `gather_context` (a `ToolNode` with parallel `list_recent_outages` + `lookup_customer`), `search_docs` (model + tool node), `draft_comment` (model), `human_confirm` (checkpoint-and-interrupt), `post_comment` (tool node), `respond` (model).
- Conditional edges: `classify_ticket_kind` routes to `respond` for tickets that need no evidence, to `gather_context` otherwise; `draft_comment` routes to `human_confirm` on external tickets, straight to `post_comment` on internal.
- Per-node retry policies on the tool nodes reuse exercise-02's retry logic.
- Wire `PostgresSaver` (or `SqliteSaver` for local dev). Prove that after the `human_confirm` node interrupts, the graph resumes from that node on the next turn without re-running earlier nodes.
- Wire tracing to LangSmith or Langfuse. Every node and every tool call is a span.

### Deliverable 4 — The comparison table

`comparison.md`, a one-page table + narrative comparing the two implementations on:

| Question | SDK | LangGraph |
|---|---|---|
| Lines of code (rough) | | |
| Time-to-first-working-tool | | |
| Debug story on a broken turn | | |
| Where the parallel `gather_context` lives | | |
| Where the human-in-the-loop lives | | |
| Where a per-tool retry is expressed | | |
| Where the graph diagram lives | | |
| What breaks when the schema of `create_comment` changes | | |
| What the on-call engineer opens first when a fixture fails | | |

Each row is a sentence or two; the goal is a written record you can hand a peer engineer.

### Deliverable 5 — The recommendation

`recommendation.md`, one page:

- One paragraph: which shape you would ship for this surface, given the three failure modes in the problem statement. Reference the [chapter 04](../04-orchestrating-with-langgraph-and-agent-sdks.md) triage question.
- One paragraph: what the "hybrid" (LangGraph outer, SDK inner) shape would look like for this surface, and what would justify reaching for it later.
- One paragraph: the specific piece of the losing implementation you would take with you into the winner (e.g., the SDK's hook mechanism as a hand-written middleware, or the LangGraph state-schema pattern applied to a growing SDK context).
- One paragraph: what fresh eyes might disagree with about your recommendation, and the evidence that would change your mind.

## Starter guidance

- **Ship the fixtures first.** They are the yardstick for both implementations. If a fixture is passing under one framework and failing under the other, the difference is a bug in the implementation, not the framework.
- **Do not sneak evaluation into either implementation.** Fixtures live in `fixtures/`; both implementations consume them read-only.
- **Timebox each implementation.** Ninety minutes each is the target. If one is taking three hours, stop and note *why* in `comparison.md` — that is data, not a failure.
- **Don't measure line count as a virtue.** Fewer lines is not automatically better if debug is worse. The comparison table has both columns for a reason.
- **Actually open the tracing UI once.** A conclusion about "the debug story" that came from reading source is not the same conclusion you get from walking a live trace. Do the latter for one fixture in each implementation before writing `comparison.md`.
- **The hybrid is a real option.** If you finish early, spike a version where a LangGraph outer graph invokes the SDK-shaped inner loop as one node. Note whether it feels natural or forced; the answer is data for the recommendation.
- **Do not hand-wave "we'll use LangSmith" for the tracing.** Actually wire it (or Langfuse; the open-source path is fine). Fake tracing is worse than no tracing because it looks like it's working.

## Acceptance criteria

- [ ] Both implementations run every fixture in `fixtures/turn_examples.jsonl` and produce structurally-equivalent outputs (tool calls made, final response present or explicitly refused).
- [ ] Both implementations enforce the per-tool caps from exercise-03.
- [ ] Both implementations parallelise `list_recent_outages` and `lookup_customer` when both are needed; the wall-clock trace shows the two overlapping.
- [ ] Both implementations have a working human-in-the-loop step for external tickets: an external-ticket fixture completes only after a second turn with `user_confirmed=True`.
- [ ] Both implementations retry a simulated transient failure on `create_comment` without duplicating the comment (idempotency-key reused).
- [ ] Traces from both implementations are viewable in the chosen observability surface; a debugger can walk turn 3 of any fixture without reading source code.
- [ ] `comparison.md` fills every row of the table and cites a specific artefact (file, line number, or trace URL) for each entry.
- [ ] `recommendation.md` states a shape you would ship and defends it against the "peer engineer with the other preference" question.
- [ ] A LangGraph diagram is rendered and committed (`graph.png` or `graph.mmd` from `graph.get_graph().draw_mermaid()`). The diagram matches the code.

## Stretch goals

Pick any two:

- **Add a second specialist agent.** In the SDK implementation, model an `escalation_specialist` as a handoff target. In LangGraph, model it as a separate subgraph. Show the multi-agent shape working end-to-end on an escalation fixture.
- **A trajectory-replay test.** Record a real conversation from either implementation; feed the recorded transcript back through the *other* implementation as a stubbed model output. Show that both graphs reach the same final state — or narrate why they diverged.
- **Cost side-by-side.** Wire the exercise-05 budget instrumentation (if you've done it) or a stub. Report per-fixture cost for both implementations and identify where they diverge.
- **Framework-lock-in audit.** Enumerate every place in each implementation where you called a framework-specific API. Propose a thin adapter interface that would let you swap frameworks without rewriting nodes / tools.
- **Chaos test.** Introduce a random 3 s delay on one tool. Show what each implementation does about it: where in the trace it shows up, whether the wall-clock budget from exercise-05 short-circuits it, and what the user sees.

## Deliverable

Submit as `mod-104/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `fixtures/turn_examples.jsonl`
- `impl_sdk/` — the loop-shaped implementation with tests + trace links.
- `impl_langgraph/` — the LangGraph implementation with tests + trace links + `graph.png` (or Mermaid).
- `comparison.md`
- `recommendation.md`
- Any stretch-goal artefacts named alongside.
