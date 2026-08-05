# App-Altitude Latency: TTFT, TPOT, Retries, Tool Calls, and the Per-Surface Spec

## Why "app-altitude" and not "provider-altitude"

Every provider ships a status page and a latency dashboard. Every serving framework (vLLM, TGI, SGLang) ships throughput and per-token latency numbers. None of them measure what the user actually experiences.

The user's clock starts when they hit `⏎` and ends when the surface stops feeling laggy — which is not the same instant on every surface. On a chat panel, the user is happy the moment the first token paints; on a code-completion inlay, they abandon the suggestion if the *whole* completion is not on screen inside a few hundred milliseconds; on a background summariser, they never see any of it and only care that the batch finished before the cron alert. Latency is not one number. It is a small vocabulary of numbers, each with a threshold set by the surface, not the model.

The applied-AI engineer's job is to measure at *app altitude* — the layer where the request leaves your code, the response is read from your code, and the user sees the surface update. Everything below that (queue wait on the provider's fleet, KV-cache eviction, GPU utilisation) is `ai-infra-performance`'s domain. What you own is the seam where retries, tool calls, streaming, and surface rendering compose into a real user-facing latency number. This chapter names the vocabulary, the instrumentation, and the artefact — a **per-surface latency spec** — that pins the numbers down for review.

## The vocabulary

Five numbers cover every LLM feature in production. Learn them once and use the same names everywhere.

### TTFT — time to first token

The elapsed wall-clock time from *your* request being handed to the transport (HTTP send, WebSocket write) until the *first* token of the model's response arrives back at your code. This is the number the user perceives as "how long before the screen starts moving." On a streaming chat surface, TTFT is the p95 people scream about; anything above ~1s on a conversational surface feels broken.

TTFT is dominated by three things: the prompt length (longer prefill = longer TTFT), how much of that prefill was cache-hit (see chapter 02), and provider tail behaviour (queue wait, model warm-up, region routing). It is not dominated by output length — output length lands in TPOT.

Two subtleties. First, TTFT must be measured from the client side of your gateway, not the provider side; you own the network hop. Second, TTFT includes any `role: "assistant"` framing tokens the provider streams before the model actually produces useful content — the OpenAI-shape `{"choices": [{"delta": {"content": ""}}]}` warmup chunks count, because they cost the user nothing psychologically but they *do* mean the pipeline is alive. Most teams measure TTFT as "time to first non-empty content delta"; a smaller number of teams measure "time to first SSE chunk of any kind." Pick one; document which.

### TPOT — time per output token

The average per-token latency of the *streamed body* after TTFT. Units are ms/token. TPOT is what determines whether a 400-token answer feels like it is being typed at a reasonable pace or feels like the model is stalling half-way through.

TPOT is dominated by decoding speed on the provider (the tokens/second of the serving stack) plus your own client-side buffering. It is almost independent of prompt length once decoding starts — a longer prompt makes TTFT worse but does not slow down TPOT.

On a chat surface, TPOT below ~40 ms/token (~25 tok/s) feels smooth to a reader; above ~80 ms/token feels visibly slow. On a code inlay where the user is not reading tokens one by one, TPOT matters less than the total wall-clock — a snappy inlay is one where **TTFT + (output_tokens × TPOT)** is below the surface's overall budget.

### Streaming p50 / p95

The two summary numbers that go on the dashboard. Compute p50 and p95 over the two components separately (TTFT_p50, TTFT_p95, TPOT_p50, TPOT_p95) *and* over the composite total (`total_latency_p95`). The composite is what SREs alert on; the components are what you debug with.

Do **not** average TTFT and TPOT — they measure different phases and the average is meaningless. Do **not** collapse to a single "response time" metric; when TTFT_p95 goes bad and TPOT_p95 stays flat, you are seeing a prefill / queue / cache-miss problem, and losing that signal costs you the incident.

Percentiles are computed over a rolling window — 5 minutes for the alerting number, 24 hours for the review number. Anything shorter than 5 minutes is too noisy for a rate-limited endpoint.

### Retry latency

The elapsed time added by any retry your gateway performs before the request succeeds (or exhausts the retry budget and fails). For a request that succeeded on the first attempt, retry latency is zero. For a request that succeeded on the second attempt after a 400 ms first-attempt timeout, retry latency is the first attempt's time + the backoff sleep + any second-attempt startup cost.

Retry latency is separate from TTFT / TPOT because a retry throws away everything the failed attempt did and starts over. When you say "our p99 TTFT is 900 ms," you usually mean "TTFT for successful first-attempt requests." When p99 gets scary, retry latency is almost always the reason — and if you have never separated it from TTFT, the dashboard cannot show you that.

The rule: emit `attempt_number` and `retry_latency_ms_total` on every span. Alert on retries as an independent series; do not let them hide in the TTFT number.

### Tool-call round-trip latency

For any request that invokes a tool (function call, MCP call, structured tool_use), measure the wall-clock from *the model finishing its tool-call decision* to *your gateway resuming the model with the tool result*. That is your gateway's tool-call round-trip. It is a number your team can move (by making the tool faster, running the tool in parallel with other work, or short-circuiting cheap tools inside your process) and a number the provider cannot see.

On agentic surfaces (mod-104, mod-105), the *end-to-end* latency of a turn is `TTFT + first_body + Σ (tool_call_round_trips) + Σ (post-tool decodes) + final_body`. A single 300 ms MCP call, invoked twice per turn, is 600 ms the user is waiting on that your provider-side dashboard does not even show. Measure it or it stays invisible.

## Instrumenting these numbers

The vocabulary above only pays off when your telemetry actually emits it. The pattern is one span per request with a fixed attribute set, aligned with the OpenTelemetry GenAI semantic conventions so the mod-110 observability substrate can consume it without translation.<!-- needs-research: confirm current OTel GenAI semconv attribute names for TTFT / TPOT / tool round-trip — the spec is evolving and the exact keys may not have stabilised. -->

### The minimum span

```python
# One span per LLM call, whether streamed or not.
# Attribute names mirror the OpenTelemetry GenAI semconv where they exist,
# and use `app.*` for the ones the semconv does not yet cover.

span.set_attribute("gen_ai.system", "openai")           # provider slug
span.set_attribute("gen_ai.request.model", model_id)
span.set_attribute("gen_ai.response.model", resp_model) # what the provider actually served
span.set_attribute("gen_ai.usage.input_tokens", n_in)
span.set_attribute("gen_ai.usage.output_tokens", n_out)

# The five app-altitude numbers.
span.set_attribute("app.latency.ttft_ms", ttft_ms)
span.set_attribute("app.latency.tpot_ms", tpot_ms)          # (total_body_ms - ttft_ms) / n_out
span.set_attribute("app.latency.total_ms", total_ms)
span.set_attribute("app.latency.retry_ms", retry_ms)        # 0 on first-attempt success
span.set_attribute("app.retry.attempt_number", attempt)

# Optional, present when a tool call fired.
span.set_attribute("app.tool.round_trip_ms", tool_rt_ms)
span.set_attribute("app.tool.name", tool_name)
```

### A minimal streaming timer

```python
import time
import asyncio

async def call_with_ttft_tpot(client, **req):
    """Return (result_text, ttft_ms, tpot_ms, total_ms, n_out)."""
    t_send = time.perf_counter()
    stream = await client.chat.completions.create(stream=True, **req)

    ttft_ms = None
    n_out = 0
    chunks = []
    async for chunk in stream:
        delta = chunk.choices[0].delta.content or ""
        if delta and ttft_ms is None:
            ttft_ms = (time.perf_counter() - t_send) * 1000
        if delta:
            chunks.append(delta)
            n_out += 1  # crude — real code counts tokens with the provider tokenizer
    t_end = time.perf_counter()

    total_ms = (t_end - t_send) * 1000
    body_ms = total_ms - (ttft_ms or 0)
    tpot_ms = (body_ms / n_out) if n_out else None
    return "".join(chunks), ttft_ms, tpot_ms, total_ms, n_out
```

Two things to notice. The timer starts at `create()` — before the first `await` in some SDKs actually opens the socket — so TTFT includes any client-side connection setup. This is deliberate: the user's clock includes that connection. Second, `n_out` here counts *delta chunks*, not tokens. For a real dashboard you want the provider's tokenizer (or the token count from the final usage message the provider emits after the stream closes). The chunk count is a reasonable approximation for TPOT trend, but do not compare it across providers — chunk sizes differ.

### Wrapping retries

Retry logic must accumulate `retry_ms` across attempts and emit `attempt_number` for the successful attempt. The common bug: instrument the *outer* wrapper so successful first attempts get `retry_ms = 0` and any subsequent attempt gets `retry_ms = prev_attempt_wall_ms + backoff_ms`.

```python
async def with_retry(client, max_attempts=3, base_backoff_ms=200, **req):
    retry_ms_total = 0
    for attempt in range(1, max_attempts + 1):
        t0 = time.perf_counter()
        try:
            result = await call_with_ttft_tpot(client, **req)
            return result, attempt, retry_ms_total
        except RetryableError:
            retry_ms_total += (time.perf_counter() - t0) * 1000
            if attempt == max_attempts:
                raise
            backoff_ms = base_backoff_ms * (2 ** (attempt - 1))
            retry_ms_total += backoff_ms
            await asyncio.sleep(backoff_ms / 1000)
```

Retry policy — how many attempts, what backoff, when to hedge — is chapter 05. What matters here is that the *measurement* separates retry cost from first-attempt cost, so you can see either signal on its own.

### Tool-call round-trip

```python
async def run_tool_and_resume(model_ctx, tool_call):
    t_dispatch = time.perf_counter()
    tool_result = await dispatch_tool(tool_call)
    t_resume = time.perf_counter()
    round_trip_ms = (t_resume - t_dispatch) * 1000
    model_ctx.record_tool_round_trip(tool_call.name, round_trip_ms)
    return tool_result
```

For agentic loops, sum the per-turn round-trips into a per-turn `tool_round_trip_ms_total` attribute; the p95 of that is often the largest debt on an agent's total-latency budget.

## The per-surface latency spec

The load-bearing artefact of this chapter is a short written document, one per surface. It answers: *for this surface, what latency numbers are green, yellow, red — and how do we measure them?*

The template:

```markdown
# Latency spec — <surface name> — <date>

## Surface shape
<one paragraph: is this streaming chat, code inlay, background batch, realtime voice, agentic multi-turn?
what does "the user is waiting" mean here?>

## Budgets (rolling 5-minute p95, per environment)
| Metric                       | Green | Yellow | Red    |
| ---------------------------- | ----- | ------ | ------ |
| TTFT p95                     | X ms  | Y ms   | Z ms   |
| TPOT p95                     | X ms  | Y ms   | Z ms   |
| Total latency p95            | X ms  | Y ms   | Z ms   |
| Retry latency p95            | X ms  | Y ms   | Z ms   |
| Tool-call round-trip p95     | X ms  | Y ms   | Z ms   |
| Tool-call round-trip p99     | X ms  | Y ms   | Z ms   |

## Excluded from these budgets
<e.g. "warm-up requests in the first minute after deploy",
"requests over 100k input tokens which are known to be batch-eligible">

## Where the numbers come from
<link to the dashboard; the span names; the retention window>

## Owner and revisit
Owner: <team>. Reviewed: <date>. Next review: <quarter>.
```

Four properties make the spec useful rather than decorative:

1. **Numbers are per surface, not per repo.** A chat panel and a batch summariser share code and share providers, but the green / yellow / red numbers are wildly different. Split the spec.
2. **Budgets are p95, not average.** Averages hide tail incidents. p99 optional if the surface is user-facing enough to care about long-tail; p95 non-negotiable.
3. **Retries and tool-call round-trip are named separately.** Rolling both into "total latency" hides the two levers you actually have.
4. **The revisit is scheduled.** Provider p95s drift as their fleet changes; your spec drifts as your prompts, tools, and users change. A yearly review is a fossil; a quarterly review is a living spec.

The spec is the artefact you point at when someone asks "is the surface slow?" — you either say "no, all numbers are green against the spec" or "yes, TPOT p95 has been yellow since Tuesday and here is the ticket."

## Per-surface examples of budgets

The exact numbers below are illustrative — your surface's users will tell you the real thresholds — but the *shape* of the budgets is consistent.

- **Conversational chat panel (streaming).** TTFT p95 ~1000 ms green / ~2000 ms yellow / >3000 ms red; TPOT p95 ~40 ms/token green / ~80 ms yellow / >120 ms red; total latency is loose because the user reads the tokens as they arrive.
- **Code-completion inlay (non-streaming, whole-completion-at-once).** Total latency p95 is the only number that matters; ~300–500 ms green / ~800 ms yellow / >1000 ms red. TTFT and TPOT are irrelevant to the user because they never see intermediate tokens.
- **Background summariser (async job).** No TTFT budget at all — the surface is a webhook or a database write. Budget is on total per-item latency (loose — minutes are fine) and per-batch throughput. This surface is the canonical candidate for chapter 03's batch API.
- **Realtime voice (duplex WebSocket).** TTFT p95 <300 ms is table stakes; anything longer is perceived as awkward silence. TPOT is measured differently (audio frames per second, not tokens). Tool-call round-trip is the highest-leverage number — a 500 ms MCP call kills the "conversation" feel.
- **Agentic multi-turn.** Total end-to-end latency of a turn is what the user perceives; the spec must include the per-turn cap on `tool_round_trip_ms_total` because that is where the budget silently goes.

## Anti-patterns

- **Provider-side dashboard as source of truth.** The provider dashboard shows their p95 for their fleet. It does not show your network, your retries, your gateway, or your tool calls. When it looks green and your surface feels slow, believe the surface.
- **One "response time" number for everything.** Collapses TTFT and TPOT into an average that neither describes prefill-bound problems nor decode-bound problems, and hides the retries entirely. Every meaningful debugging conversation starts by breaking it back out.
- **No retry attribute on spans.** The p99 latency creeps up. The team burns a week hunting a "slow prefill" that turns out to be a retry loop firing on transient 5xxs. Add `attempt_number` on day one.
- **Averaging over long windows.** Nightly average TTFT looks fine; the 4pm spike that ate 30 minutes of user experience is invisible. p95 over a 5-minute window is the alerting number.
- **Latency spec that lives in a PDF from last year.** No one reads it, no one updates it, arguments about "is this surface slow" restart from zero each time. The spec is a markdown file in the repo, next to the code it governs.
- **Tool-call round-trip untimed.** Agents look fast in isolation and slow in aggregate; the answer is almost always that two tool calls at 400 ms each are the whole budget. Instrument the tools.

## Summary

- App-altitude latency has five names, not one: **TTFT, TPOT, streaming p50/p95, retry latency, tool-call round-trip.** Learn the vocabulary and use it in every dashboard and every incident.
- TTFT is prefill + queue + cache. TPOT is decode. Never average them.
- Retries and tool-call round-trip are the two biggest hidden sources of user-perceived latency. Emit them as independent series on every span.
- Percentiles: p95 over a 5-minute rolling window for alerting, 24 hours for review. Averages are for people who have never had an incident.
- The load-bearing artefact is the **per-surface latency spec** — a short markdown file with green / yellow / red budgets, the excluded populations, the dashboard link, and a scheduled revisit. Split the spec per surface, because a chat panel and a batch summariser share nothing.

[Chapter 02](02-prompt-caching-as-a-design-constraint.md) turns from measuring latency to *shaping it*: how provider prompt caching (OpenAI, Anthropic, Bedrock) becomes an explicit design constraint on the shape of the system prompt and the cadence at which you version it.
