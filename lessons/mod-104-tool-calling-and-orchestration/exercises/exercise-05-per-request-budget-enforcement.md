# exercise-05: Per-Request Budget Enforcement

**Estimated effort:** 1 hour

## Objective

Instrument a small tool-calling agent with the four per-request budgets from [chapter 05](../05-per-request-budgets-and-graceful-degradation.md) — tokens, steps, wall-clock, and dollars — and prove that hitting each budget triggers a graceful degradation rather than a silent truncation. This is a focused exercise: one hour of work that makes every future turn observable and every cap defensible.

## Learning outcomes

By the end you should be able to:

- Thread a `RequestBudget` (or equivalent hook counter) through every model and tool call in the loop.
- Distinguish pre-flight from reactive enforcement and pick correctly per dimension.
- Wire a `completion_status` on every response and render early-stop states with an explicit user-visible acknowledgement.
- Instrument a "spent vs. cap" row per turn and answer, from live traffic, whether a cap is a chokepoint or a safety net.
- Reject the silent-truncation anti-pattern in code review — with the specific check to run.

## Prerequisites

- You have read [chapter 05](../05-per-request-budgets-and-graceful-degradation.md).
- You have completed exercise-04 (or have an equivalent working agent — SDK-shaped or LangGraph — running against real tools).
- You have `pytest` (or equivalent) and are comfortable stubbing model and tool calls.

## Problem statement

Return to the support-triage co-pilot from exercises 03–04. The alpha shipped without any budget instrumentation. Two turns in the last week ran a `search_articles → search_articles → search_articles` loop until `max_turns` was reached; both users saw a blank spinner that resolved into "Something went wrong." One turn burned $4.20 on a single request when the model chained six MCP calls; the incident-cost review is on the calendar.

Your job for the next hour: wire budgets, prove each one degrades gracefully, and put the instrumentation on a dashboard the on-call engineer can read.

## Requirements

### Deliverable 1 — The `RequestBudget`

Add a `RequestBudget` object (or a framework-hook equivalent — see [chapter 05](../05-per-request-budgets-and-graceful-degradation.md) shapes A and B). Requirements:

- Four caps: `tokens_in_max`, `tokens_out_max`, `steps_max`, `wall_clock_max_s`, `dollars_max`. Set them per surface; the values for the triage co-pilot: 80k input, 8k output, 10 steps, 30 s wall-clock, $0.25.
- Four counters, updated exactly by `charge_model_call` and `charge_tool_call`.
- A `can_afford_step()` gate that all step-adding paths (model calls, tool invocations, retry attempts) consult before running.
- Every call site that mutates a counter is grep-able. There are exactly two mutating functions in the runtime.

### Deliverable 2 — Pre-flight and reactive enforcement

- **Token budget** — enforced pre-flight. Before every model call, sum the accumulated input tokens (use the provider's tokenizer — `count_tokens` for Anthropic, `tiktoken` for OpenAI, the Gemini tokenizer for Gemini). If the next call would exceed the cap, do not make the call. Trigger degradation.
- **Wall-clock budget** — enforced pre-flight. Every step's worst-case duration (upstream timeout + backoff worst case) is checked against `remaining_s()` before it starts.
- **Step budget** — enforced pre-flight *and* reactive. Pre-flight so the loop doesn't start a step it can't afford; reactive as a backstop that catches a hook that let one through.
- **Dollar budget** — enforced reactive. The dollar total updates after each model response arrives (because prompt-caching hits are known only in the response) and after each metered tool call. When the total crosses the cap, the loop terminates cleanly on the next `can_afford_step`.

### Deliverable 3 — The reserved final-summary tokens

Reserve `tokens_final_summary = 800` at the start of the turn. Every intermediate model call is capped at `tokens_out_max - tokens_out_spent - tokens_final_summary` on its `max_tokens` parameter. The final summary call gets the reserved 800 back. Prove the final summary is never truncated because the tool loop ate the budget.

### Deliverable 4 — The graceful-degradation path

Every response includes a `completion_status` field with values in `{complete, early_stop_token, early_stop_step, early_stop_wall_clock, early_stop_dollars, hard_error}`. The surface renders differently for each:

- `complete` — the model's response, no chrome.
- `early_stop_*` — the partial output (if any), an explicit sentence naming the stop cause and the next-step suggestion, and the trace_id.
- `hard_error` — the generic-error fallback + a "this is a bug" callout.

The rendering logic lives in one function; the tests exercise every branch.

### Deliverable 5 — The instrumentation row

Every completed turn writes one JSONL row:

```json
{
  "trace_id": "...",
  "surface_id": "triage-copilot",
  "tenant_id": "...",
  "caps": {"tokens_in_max": 80000, "tokens_out_max": 8000, "steps_max": 10, "wall_clock_max_s": 30, "dollars_max": 0.25},
  "actuals": {"tokens_in_spent": 41230, "tokens_out_spent": 2140, "steps_spent": 6, "wall_clock_s": 12.4, "dollars_spent": 0.041},
  "outcome": "complete" | "early_stop_token" | "early_stop_step" | "early_stop_wall_clock" | "early_stop_dollars" | "hard_error",
  "first_exhausted": null | "tokens" | "steps" | "wall_clock" | "dollars",
  "per_tool_steps": {"search_articles": 2, "lookup_customer": 1, ...},
  "per_tool_dollars": {"search_articles": 0.010, ...}
}
```

Write a small SQL-ish query (or a pandas snippet) against a corpus of rows that answers: (a) p50/p95/p99 of `steps_spent / steps_max`, (b) rate of `early_stop_*` per surface, (c) which tool has the highest per-turn dollar cost.

## Starter guidance

- **Do the reactive dollar counter last.** It requires a pricing map, and pricing maps rot. Start with the three easier dimensions (tokens, steps, wall-clock), then add the pricing lookup.
- **Test with synthetic caps.** Set `steps_max=2` in a test and prove the loop stops cleanly on the third attempt. Don't chase realism at test time; you can validate real-cap behaviour with real fixtures separately.
- **Prove reserved-summary tokens actually protect the summary.** A test that consumes all `tokens_out_max - tokens_out_spent - 800` during the tool loop should still produce a full final summary. If it doesn't, the reservation isn't wired.
- **`stop_reason == "max_tokens"` is not `complete`.** In the model-call layer, treat that `stop_reason` as an early-stop token event even if the outer loop hasn't hit its cap. That is the silent-truncation channel from the chapter.
- **Do not raise from the budget layer.** A `BudgetExhausted` exception at the top of the loop is a bug. The invariant is that every code path returns a structured response, with `completion_status` set. Tests should assert no path leaks.
- **Wire the same instrumentation from your exercise-04 implementation.** Both SDK and LangGraph implementations should use the same `RequestBudget` (or equivalent hook counter). If one path can't accept it, that is a framework-shape observation worth noting in `comparison.md`.

## Acceptance criteria

- [ ] `RequestBudget` (or framework-hook equivalent) tracks four dimensions; every step-adding call site consults `can_afford_step()`.
- [ ] Token counting uses the provider's tokenizer; pre-flight blocks a call that would exceed the input cap.
- [ ] Wall-clock deadlines propagate to every tool call and retry; a call whose worst-case would not fit does not start.
- [ ] Step budget is enforced both pre-flight and reactive; a backstop test proves the reactive gate catches a bugged pre-flight.
- [ ] Dollar budget updates reactively from the response; cached-input tokens are priced at the discounted rate.
- [ ] `tokens_final_summary = 800` is reserved at turn start; a test demonstrates the final summary is not truncated after a tool-heavy loop.
- [ ] Every response carries a `completion_status`; every early-stop state renders an explicit acknowledgement to the user.
- [ ] `stop_reason == "max_tokens"` on a model call is treated as early-stop, not complete.
- [ ] Every turn writes one JSONL row with caps, actuals, outcome, and per-tool breakdowns.
- [ ] A query against a fixture corpus answers p95 of `steps_spent / steps_max` and the per-tool dollar leader.
- [ ] A test proves the runtime does not raise on any budget-exhausted path.

## Stretch goals

Pick any one:

- **A cap-tuning readout.** Run 50 synthetic turns through the runtime. Produce a small report ranking each cap by "how often was it the first-exhausted budget?" Recommend the specific cap to relax if any distribution is over the 30 % threshold.
- **A silent-truncation red-team.** Add adversarial fixtures where the provider is stubbed to return `stop_reason=max_tokens` mid-sentence, or the streaming reader is stubbed to close early. Prove every case renders with the "I stopped early because …" chrome, not as a natural completion.
- **A per-tool rate limit inside the budget.** Add a per-tool step cap (`search_articles ≤ 5 per turn`, `create_comment ≤ 3 per turn`) enforced through the same `RequestBudget` object. Show that the per-tool caps trip a distinct `early_stop_*` reason (`early_stop_tool_cap`) so on-call can tell tool-loop from budget-loop.

## Deliverable

Submit as `mod-104/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `budget/request_budget.py`
- `budget/tokenizers.py` (or a wrapper around your provider's tokenizer)
- `budget/pricing.py`
- `runtime/loop_with_budget.py` — the integration with either the SDK or LangGraph implementation from exercise-04.
- `tests/` — unit and integration tests exercising every budget dimension and every degradation path.
- `queries.md` — the "spent vs. cap" and "which tool is most expensive" queries against a fixture corpus.
- Any stretch-goal artefacts named alongside.
