# Per-Request Budgets and Graceful Degradation

## Why a budget, not a limit

An agent that can call tools can also spend money, tokens, and wall-clock time proportional to how confused it becomes. A loop that "just one more search should do it" is the difference between a $0.02 turn and a $12 turn. A model that decides the right response is nine parallel Notion lookups is the difference between a 900 ms response and a browser tab that spins for two minutes. The runtime does not stop this on its own; the framework does not stop this on its own. The applied-AI engineer stops it — with a **per-request budget** wired into the tool loop from the first line of code, not bolted on when the first alert fires.

The word matters. A **limit** is a hard stop that fails the request when hit. A **budget** is a hard stop paired with a *plan for what the surface does when the budget is exhausted*: return partial output, fall back to a cheaper strategy, hand off to a human, or explain to the user that the request was too big. Silent truncation — the tool loop hits a cap and the model's next call is discarded without the surface noticing — is the single easiest way to ship a product agent that gives wrong answers on the requests users cared about most.

This chapter is the last of the four artefacts an applied-AI engineer owns on the tool-calling surface: after the schema (chapter 01), the error taxonomy (chapter 02), the MCP boundary (chapter 03), and the graph shape (chapter 04), the budget is what keeps the whole surface economically honest.

## The four budgets, and why you enforce all of them

There is not one budget. There are four, and they cap different failure modes.

### 1. Token budget

Both input and output tokens are metered by every commercial model provider — Anthropic Messages, OpenAI Chat Completions and Responses, Google Gemini, and every hosted open-weight endpoint. A token budget is the number of tokens the whole request may consume across all sampling calls in the turn.

Why cap tokens specifically:

- **Cost.** Tokens are the primary billing unit at every provider. Every retry, every tool result appended to context, every message the agent adds to its scratchpad — all count.
- **Latency floor.** Very large prompts take longer to prefill; very long generations take longer to sample. A token budget indirectly bounds wall clock.
- **Context-window boundary.** Every model has a hard input-token ceiling. A request that would overshoot it fails at the provider edge with an ugly error. The token budget catches it earlier, in your code, with a graceful message.

The budget is a **turn-level counter**, not a per-call counter. Every call to the model reports its input and output tokens in the response; the runtime accumulates them into the request context and checks before the next call.

### 2. Step budget

A step is a discrete transition in the agent loop: a model call, a tool invocation, an MCP round trip, a graph node execution. The step budget caps the total number.

Why the step budget matters even when the token budget seems tight enough:

- **Stall detection.** A model stuck in a `search → search → search` loop is not always burning tokens (the tool results may be small). Without a step cap, the loop runs to the token cap, which may be minutes away in wall clock. The step cap catches the pathology on the first repeat.
- **Human intuition.** "This surface takes at most 8 tool calls to answer" is a reviewable claim. "This surface spends at most 60,000 tokens per turn" is not, unless the reviewer sits down with a token counter.
- **Debugging.** When a step cap fires, the trace is bounded and easy to walk through. When a token cap fires 12 tool calls in, the trace is a wall.

Frameworks expose the step budget as `max_iterations` (LangGraph), `max_turns` (Claude Agent SDK), or `max_iterations` on the `Runner` (OpenAI Agents SDK). Set it explicitly on every surface. The framework default is not a decision you made.

### 3. Wall-clock budget

The wall-clock budget is the number of seconds from the user's request landing to the response being returned. This is a *user experience* budget — the number picked by the surface's SLO, not by the provider.

Why this can't be inferred from the other two:

- A backend call inside a tool (chapter 02) can block for 30 seconds even at 500 tokens.
- Streaming with a slow first-token generates no tokens the runtime can meter until the model starts producing them.
- Cascading retries within a single tool call (chapter 02) burn wall clock without adding steps.

Rule of thumb: a chat surface budgets 5–15 seconds for the first token and 30–60 seconds total; a batch pipeline budgets minutes. Whatever number you pick, it is enforced by a `deadline` propagated into every tool call and every model call in the turn, not by a top-level `asyncio.wait_for` that kills the request mid-flight.

### 4. Dollar budget

Tokens are a proxy for cost, but not a good enough one:

- Different models have wildly different token prices. A turn that spends 5k tokens on Opus and 5k tokens on Haiku is not the same cost.
- Cached prompt reads (Anthropic's prompt caching, OpenAI's cached-input pricing) discount input tokens; the runtime must know which prompts are cached to convert tokens to dollars correctly.
- MCP tools that wrap paid third-party APIs (a `search_web` tool that calls a paid search API, a `lookup_person` tool that calls a data broker) cost dollars per call independent of tokens.

The dollar budget is a first-class runtime counter, computed by mapping every token count and every metered tool call to a price. The map lives in one config module per provider and is updated when pricing changes.

## Where the budgets live in the code

Two shapes ship. Both work. Pick one and commit; do not have half the code path checking a global counter and half checking a passed context.

### Shape A — a `RequestBudget` on the request context

```python
@dataclass
class RequestBudget:
    tokens_in_max: int
    tokens_out_max: int
    steps_max: int
    wall_clock_max_s: float
    dollars_max: float

    # Live counters, mutated by the runtime.
    tokens_in_spent: int = 0
    tokens_out_spent: int = 0
    steps_spent: int = 0
    dollars_spent: float = 0.0
    started_at: float = field(default_factory=monotonic)

    def remaining_s(self) -> float:
        return self.wall_clock_max_s - (monotonic() - self.started_at)

    def can_afford_step(self) -> bool:
        return (self.steps_spent < self.steps_max
                and self.tokens_in_spent < self.tokens_in_max
                and self.tokens_out_spent < self.tokens_out_max
                and self.dollars_spent < self.dollars_max
                and self.remaining_s() > 0)

    def charge_model_call(self, usage: TokenUsage, price: DollarPrice) -> None:
        self.tokens_in_spent += usage.input_tokens
        self.tokens_out_spent += usage.output_tokens
        self.dollars_spent += price.for_usage(usage)
        self.steps_spent += 1

    def charge_tool_call(self, tool_name: str, dollars: float = 0.0) -> None:
        self.dollars_spent += dollars
        self.steps_spent += 1
```

Every model call and every tool call goes through the same charge method. Every gate — "should we make this next call?" — asks `can_afford_step()`. When the answer is no, the runtime doesn't call again; it enters the degradation path (below).

### Shape B — hooks on the framework

For the Claude Agent SDK, this is a `PreToolUse` / `PostToolUse` hook pair; for the OpenAI Agents SDK, a `Runner` configuration + a stop-condition callback; for LangGraph, a `pre_model_hook` and a per-node interceptor. The counter lives inside the hook implementation and closes over a `budget` object that the surface owns.

Both shapes work; the framework-hook shape is preferable when it fits because the framework knows about model calls the surface doesn't (e.g., internal reflection or planning steps some SDKs make). The context-object shape is preferable when the surface spans multiple framework calls or a hybrid inner-outer loop (chapter 04).

## The graceful-degradation path

When any budget is exhausted, the surface must return something coherent. The mistake to avoid: raising an exception that unwinds the whole turn and shows the user "Something went wrong."

Four degradation shapes, in decreasing order of user-friendliness. Match each surface to whichever it can produce.

### 1. Partial output with an explicit "I ran out of budget" acknowledgement

The best outcome. Everything the agent produced up to the cap is rendered, with an explicit trailing message ("I was mid-way through gathering evidence when I ran out of time. Here's what I found so far, and here's what I would have done next."). Preserving intermediate state is what makes this possible — the graph-shaped frameworks (chapter 04) win here because state is inspectable at every node.

```python
async def run_with_budget(user_message, budget):
    partial = PartialResponse()
    try:
        async for event in agent.stream(user_message):
            if not budget.can_afford_step():
                partial.set_early_stop(reason=budget.first_exhausted())
                break
            partial.append(event)
        else:
            partial.set_complete()
    except BudgetExhausted as e:
        partial.set_early_stop(reason=str(e))
    return partial
```

`partial.render()` is a method the surface calls to produce the user-visible response. On complete, it renders the full agent output. On early stop, it renders the intermediate state, a graceful sentence explaining the stop, and any suggestion the surface can make for how the user might reformulate.

### 2. Fallback to a cheaper strategy

Some surfaces have a "fast path" they can fall back to when the "slow path" runs out. A summarisation agent that would normally read six chunks and cross-reference them can, when the token budget is 30% remaining, drop to reading the top-ranked chunk and answering from it alone. The fallback is a separately-defined code path with its own prompt and its own eval fixtures — it is not "the same agent but with less" — because "the same agent but with less" is the pathology this chapter is preventing.

### 3. Hand off to a human

For surfaces where the agent is a co-pilot behind a human confirmation (a triage co-pilot, a code assistant), the natural degradation is to render whatever the agent has so far as a draft and hand the rest to the human. The graph pattern from chapter 04 already supports this — the "human confirms" node is a first-class stop.

### 4. Refuse cleanly

When none of the other three fit, refuse. The response is an explicit, non-scary "I couldn't complete this request within the current limits. Here's what to do next." Log the trace_id. This is the last-resort degradation, and it is still better than a broken UI.

## Cross-references: budgets during retries and repairs

Chapter 02's retry / repair loops eat budget. Wire them together explicitly.

- **Transient retries subtract from the wall-clock and dollar budgets.** A three-attempt exponential backoff on a slow tool is real seconds and real API calls. The retry policy takes the request's remaining budget as an argument and stops early if a further attempt won't fit.
- **Model-side repair attempts are a full model call.** They subtract from the token, step, and dollar budgets like any other model call.
- **MCP tool calls carry per-tool caps.** Chapter 03 flagged "this tool may be called at most 8 times per turn." Enforce that at the same layer as the step budget — a per-tool step counter inside the budget object.

The pattern: no code path spends any budget dimension without incrementing the counter. Any code path that increments a counter first checks whether the spend is affordable. There are exactly two functions in your runtime that mutate the budget object, and they are trivially auditable.

## Pre-flight vs. reactive enforcement

Two enforcement styles, both correct in different places.

**Pre-flight** — the runtime estimates the next call's cost and refuses to make it if the estimate exceeds the remaining budget. Requires accurate estimates (input-token count is straightforward; output-token count and tool wall-clock are harder). Use for the token budget: at every step, sum the accumulated context length and reject a call that would exceed the model's context window or the request's token cap. Use for the wall-clock budget: at every step, check `remaining_s()` and reject a step whose worst-case (backoff exhausted, provider slow) doesn't fit.

**Reactive** — the runtime accumulates as the call happens and stops the loop after the call that broke the budget. Use for the dollar budget when pricing depends on cached-input hits that can only be measured after the response. Use for the step budget as a backstop — a bugged pre-flight check that lets one extra step through is caught reactively at the next iteration.

The pattern: pre-flight where you can estimate cheaply, reactive where you can't. Never trust one alone.

### Estimating input tokens without a sampler

Every provider ships a tokenizer (Anthropic ships `count_tokens` on the client; OpenAI ships `tiktoken`; Google ships tokenizers per model). The estimator uses the same tokenizer as the sampler, applied to the messages that would be sent. Do not eyeball; do not use word count as a proxy. A pre-flight token check that under-counts is a token cap that lets the request through anyway.

### Estimating output tokens

Harder. The two options:

- **Cap the sampling call's `max_tokens` at whatever the remaining budget affords.** The model can still finish early. Downside: a plan-then-execute agent that decided to write a long summary and got a small cap produces truncated summaries — this is a form of silent truncation unless the surface catches it.
- **Reserve a fixed reserve for the final summary at the start of the turn** and dispatch tool-use calls with the remaining tokens as the budget. The final summary always fits.

The second pattern is the correct default for surfaces that end with a user-visible generation. Reserve enough tokens for the summary the user needs; spend the rest on the tool loop.

## Instrumentation, per turn

Every completed turn writes one budget row. The row has:

- `trace_id`
- `surface_id`, `tenant_id`
- Budget caps that applied (`tokens_max`, `steps_max`, `wall_clock_max_s`, `dollars_max`)
- Actuals spent for each dimension
- `outcome` in `{complete, early_stop_token, early_stop_step, early_stop_wall_clock, early_stop_dollars, hard_error}`
- Which stop, if any, was the first-triggered budget
- Per-tool step counts
- Per-tool dollar spend

Two aggregate metrics land on the mod-110 dashboard.

- **Distribution of spent-vs-cap per dimension.** If p95 of `steps_spent / steps_max` is 0.4, the cap is loose (fine — headroom is safety). If it is 0.98, the cap is a chokepoint (users are hitting it often; either the cap is too tight or the surface is looping).
- **`early_stop_*` rate per surface.** Any non-zero rate is a signal. The specific distribution (are stops mostly token or mostly wall clock?) tells you which budget dimension the surface is bumping into.

An alert to write: **early-stop rate above baseline by more than the noise floor**. That is the "the surface started looping this week" alarm. Correlate with model version, prompt version, and tool-catalog version to find the cause.

## Picking the numbers

The budget numbers are surface-specific. There is no universal "right" cap.

A rough starting point for a chat-shaped consumer surface with a modern frontier model and a small tool set:

- Token budget: 50k–120k tokens per turn (input + output). Enough for a healthy tool loop with a decent context but not enough to allow six wasted searches.
- Step budget: 8–16 steps. Most surfaces resolve in 3–5.
- Wall-clock budget: 15–45 seconds. Chat UX degrades sharply above 30 s without a streaming spinner.
- Dollar budget: 5×–10× the p50 turn cost, so a bad turn is capped at an order of magnitude over a normal one.

A batch-shaped or async surface (nightly summariser, ticket-triage worker) tolerates 10× the wall clock and step count. Ship those on separate infrastructure whose budget shape doesn't share code with the interactive surface.

Whichever numbers you pick, the discipline is:

1. **Start conservative.** Tighter caps expose surface bugs early. Loosen once the eval loop (mod-106) says the surface converges within the cap on live traffic.
2. **Measure spend-vs-cap distribution on real traffic** before tuning.
3. **Move the cap in one dimension at a time.** If token cap and step cap both change in the same deploy, you can't tell which one changed the early-stop rate.

## Budget interactions with LLM providers

A few provider-specific things to know when you wire the counters.

- **Anthropic Messages** — response includes `usage.input_tokens` and `usage.output_tokens`. Prompt caching populates `cache_creation_input_tokens` and `cache_read_input_tokens`; the dollar-price map must treat cached reads at the discounted rate.
- **OpenAI Chat / Responses** — response includes `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens`. Cached input tokens surface separately as `usage.prompt_tokens_details.cached_tokens`. The Responses API's built-in tools (web search, file search) have their own metered pricing; check the current pricing docs. <!-- needs-research: pin the current field name for cached-input tokens in the OpenAI SDK — the shape has moved between versions. -->
- **Google Gemini** — response includes `usage_metadata.prompt_token_count`, `usage_metadata.candidates_token_count`, `usage_metadata.total_token_count`.
- **Streamed responses** — every provider streams usage in the final event, not in intermediate deltas. The reactive counter updates on stream completion, not per-chunk.

Anthropic and OpenAI both publish token-per-second rate limits (RPM, TPM, RPD) at the account and organisation level. The per-request budget is not a substitute for account-level throttling — the two work together. If your turn's budget lets 200 tool calls happen but your TPM cap allows only 50, the runtime queues; queueing eats wall clock, which trips the wall-clock budget, which triggers degradation. Wire the two together in the same trace so incidents don't have to correlate two separate alerts.

## The "silent truncation" anti-pattern in detail

The failure this chapter exists to prevent is subtle enough to warrant a section on its own.

Silent truncation happens when the runtime, a framework, or a provider stops producing output partway through and the surface renders the partial output as if it were complete. The user sees an answer; the answer is missing the second half.

The channels through which this leaks in a tool-calling surface:

1. **`max_tokens` on a sampling call** — the model wanted to write 4k output tokens; you capped at 2k; the response's `stop_reason` is `max_tokens`; the surface didn't check `stop_reason` and rendered the truncated string.
2. **Framework `max_iterations` reached silently** — the agent loop hit its step cap; the framework returned whatever the last state was; the surface didn't check the "did this end cleanly?" flag.
3. **Streaming reader stopped on backpressure** — the network layer between the model and the runtime terminated a stream early; the runtime treated it as end-of-stream; the surface rendered a partial answer.
4. **Token counter overshoot at the pre-flight gate** — the runtime blocked a tool call because "the token budget is 200 tokens short," and the model's next generation was going to explain the answer. The surface rendered no explanation and returned the last tool call.

The invariant to enforce in code: **every rendered response has a `completion_status` field.** It is one of `{complete, early_stop_<dimension>, hard_error}`. The UI renders differently for each. `complete` is the only status that renders without a caveat. Any other status renders with an explicit acknowledgement of the stop, at minimum: "I stopped early because …".

`stop_reason == "max_tokens"` on a model call is not a natural stop; it's an early stop, and the surface must treat it that way. Same for `max_iterations`. Same for a token cap hit inside a hook.

## A worked example: the refund co-pilot's budget

From chapters 01 and 02, the refund co-pilot. A concrete budget for the interactive surface:

- Tokens: 80k input + 8k output per turn. Enough context for the order history, prior refunds, and evidence chunks; capped output so the summary is one screen, not five.
- Steps: 10. The typical path is 4 (classify → search order → search prior refunds → propose amount → confirm → execute → respond), and 10 leaves headroom for one repair loop.
- Wall clock: 30 s hard, first-token target 3 s. If the refund service is slow, the surface degrades to "I couldn't reach the refund system — try again from the order page."
- Dollars: $0.25 per turn. If a turn exceeds $0.25, the counter trips; the p50 is about $0.03.
- Per-tool caps: `search_orders` ≤ 3 per turn (a fourth is looping), `issue_refund` ≤ 1 per turn (a second is a duplicate-write bug in the runtime).

Degradation shapes:

- **Token cap hit before the execute step**: partial output rendering the proposed refund and asking the user to confirm via the order page. The refund is *not* issued.
- **Step cap hit**: same. The runtime hasn't reached the execute step, so nothing has committed.
- **Wall-clock cap hit during `issue_refund`**: ambiguous outcome per chapter 02. The message is "I tried to issue the refund but couldn't confirm it went through. Please check the order before I retry." The trace_id is logged; the runbook checks the backend's idempotency ledger.
- **Dollar cap hit**: this shouldn't happen in normal traffic; it's a backstop. When it does, the surface refuses cleanly and the on-call engineer walks the trace to find the pathology.

Every one of these is a testable outcome. The mod-106 eval harness includes fixtures for each — synthetic traces designed to trip each cap in turn, with the acceptance criterion being "the correct degradation path fired and the user-visible message is coherent."

## Anti-patterns

Six failures at the budget layer, all with real incident write-ups behind them.

1. **`max_tokens` alone as the budget.** The output-cap is set, nothing else. The token cap protects one dimension; the surface still loops indefinitely, still stalls on a slow tool, still burns dollars on repeat calls.
2. **Global counters instead of per-request.** Two concurrent requests share a counter and race. The first user's turn eats the second user's budget.
3. **Enforcement in the UI only.** The frontend times out after 30 seconds. The backend loop continues, spending, until it finishes. The user sees an error; the bill grows.
4. **Silent `stop_reason` handling.** The response has `stop_reason == "max_tokens"`; the code renders it verbatim; the user gets a mid-sentence answer.
5. **No degradation path defined.** The budget triggers `raise BudgetExhausted`. The exception surfaces as HTTP 500. The user sees "Something went wrong." The engineer sees no signal to correlate. This is the failure mode that makes people call budgets "too aggressive"; the real problem is that the budget didn't have a graceful path.
6. **Budgets set once and never revisited.** Traffic patterns change. A cap that fit last quarter is a chokepoint this quarter. The spent-vs-cap distribution is the signal; if it isn't on the dashboard, the cap is drifting silently.

## Summary

- Every product agent enforces four per-request budgets: **tokens** (input + output), **steps**, **wall-clock**, **dollars**. Each caps a different failure mode; none substitutes for another.
- Budgets are enforced by the runtime, not by the frontend and not by the provider. A `RequestBudget` object (or an equivalent hook-managed counter) is threaded through every model call, every tool call, every retry.
- Enforcement is pre-flight where the cost can be estimated cheaply (tokens, wall clock) and reactive where it can't (dollars for cached input, backstop for the step cap). Never trust one alone.
- Every budget dimension has a **graceful-degradation path**: partial output with acknowledgement, fallback to a cheaper strategy, hand off to a human, or refuse cleanly. Silent truncation — a mid-sentence response rendered as if complete — is the anti-pattern this chapter exists to prevent.
- Every response carries a `completion_status`; the UI renders different chrome for `complete` vs each `early_stop_*` state. `stop_reason == "max_tokens"` from a provider is treated as an early stop, not a natural one.
- Instrument every turn with actuals-vs-caps per dimension and an `early_stop_*` reason. The distribution across live traffic drives cap tuning; the rate of early stops per surface is a first-class alert.
- Six anti-patterns to memorise: `max_tokens`-only, global counters, UI-only enforcement, silent `stop_reason`, missing degradation path, and budgets set once.

This chapter closes the tool-calling module. The next module ([mod-105 — Shipping Product Agents](../mod-105-shipping-product-agents/README.md)) takes the surface built across chapters 01–05 and turns it into a shippable feature — packaging, deployment, versioning, and the first day of production traffic.
