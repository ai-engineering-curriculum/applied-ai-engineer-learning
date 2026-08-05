# Retries, Hedging, SLOs, and Burn-Rate Alerts

## Why "SLO" and not "P95 dashboard"

The final piece of the cost / latency substrate is the SRE-shape SLO — the written promise of the shape *this feature delivers ≥ 99% of successful, ≤ 2500 ms responses per rolling 28 days* — plus the burn-rate alerts that tell you when you are on track to break it. A dashboard is a passive artefact; an SLO is a contract with a compliance number and an alerting posture. Every mature LLM feature past its first month of production traffic needs one.

Two things make LLM-feature SLOs different from regular web-service SLOs:

- **Provider tail latency is not yours to fix.** When Anthropic's fleet queues, your TTFT p99 goes with it. The lever you have is *routing behaviour* — retries, hedged requests, provider fallback — not "make the provider faster." Chapter's core skill is picking a lever that improves the tail without doubling the cost.
- **The MLPerf inference numbers are your reference frame, not your budget.** `ai-infra-performance` and `model-evaluation-engineer` publish serving-benchmark numbers per model / hardware / workload. Applied engineers consume these to sanity-check "is our TPOT plausible for this model on this provider" — not to author them, and not to set SLO numbers directly from them.

This chapter ties chapters 01–04 together: the latency vocabulary becomes an SLO, the retry / hedging strategy is the lever, the cost accounting bounds how far you can push the lever, and the burn-rate alert is what tells you to move.

## Retry strategy: exponential backoff, jitter, and a *retry budget*

The default provider SDK behaviour is often "retry on 5xx / 429 with exponential backoff up to N times." That default is a load-bearing choice; it is also the default that quietly turns a provider hiccup into a spend spike.

The retry axes an applied engineer picks deliberately:

- **What is retryable?** 5xx server errors and 429 rate-limit responses are retry-eligible; 4xx client errors (400, 401, 404) are not. Provider-specific: some overloaded-state errors (Anthropic's `overloaded_error`, OpenAI's `service_unavailable`) are retryable; content-policy refusals (`content_filter`, `prompt_blocked`) are not.
- **How many attempts?** Two or three is the safe range. Beyond that, you are almost always making the outage worse for everyone.
- **What backoff?** Exponential with jitter. `sleep = base * 2^attempt * (0.5 + random() * 0.5)`. Deterministic backoff synchronises retriers into thundering herds.
- **When to give up?** Not just after N attempts — also when a **retry budget** is exhausted. See below.

The retry budget is the piece most teams miss. Define it as *the maximum fraction of your feature's requests that can be retries in a rolling window* (typically 5-10%). When retries exceed the budget, further retries are refused and the request fails fast. This turns a provider outage from an amplification event (every request retries twice, you send 3× your normal load into a struggling provider) into a graceful degradation event (some requests fail immediately, load stays constant, upstream sees a legible signal instead of a slow bleed).

```python
class RetryBudget:
    def __init__(self, window_s=60, max_ratio=0.1):
        self.window_s = window_s
        self.max_ratio = max_ratio
        self.attempts = []  # list of (timestamp, was_retry) tuples

    def _prune(self, now):
        cutoff = now - self.window_s
        self.attempts = [(t, r) for (t, r) in self.attempts if t >= cutoff]

    def can_retry(self, now):
        self._prune(now)
        total = len(self.attempts)
        retries = sum(1 for (_, r) in self.attempts if r)
        # Allow retries only while (retries + 1) / total stays under the ratio.
        return total == 0 or (retries + 1) / total <= self.max_ratio

    def record(self, now, was_retry):
        self._prune(now)
        self.attempts.append((now, was_retry))
```

Every attempt to retry consults `can_retry` before sleeping. When the ratio is exhausted, the retry loop returns the original failure — the surface degrades, and the metric `retries_refused_total` is emitted for the alert (see burn-rate below).

The Google SRE book's chapter on load balancing formalises this pattern; the same shape shows up in gRPC's retry policy and every mature provider gateway.<!-- needs-research: cite Google SRE Book chapter on retry budgets ("Handling Overload" / gRPC "A6 Client Retries" design). -->

## Hedged requests: pay ~20% more, cut the p99 tail

When a single provider's tail latency is the constraint, **hedging** is the applied engineer's next lever. The pattern is elegantly summarised by Dean and Barroso: send a duplicate request after a short delay, take whichever returns first, cancel the loser.<!-- needs-research: cite Dean & Barroso, "The Tail at Scale," CACM 2013 — the canonical reference for hedging. -->

Concretely, for an LLM call with a target p95 TTFT of 1000 ms and a p99 that spikes past 3000 ms:

1. Fire the primary request at t=0.
2. If no first-token response by t=p95 (say 1000 ms), fire a **hedge** — an identical request, ideally to a different provider region or a different provider entirely (via the router seam of mod-108).
3. As soon as one succeeds, cancel the other.

The cost knob:

- **Never hedging.** Cheapest, but p99 is what the provider's slowest fleet region does.
- **Always hedging.** Doubles the request cost and hurts the provider's fleet globally. Never do this.
- **Hedging after p95.** In the well-behaved case (95% of the time), the primary returns before the hedge fires, and cost is unchanged. In the 5% tail, both requests are in flight for ~a second before the loser cancels; expected cost delta is roughly *5% × (marginal time in flight / total request time)*, typically **10–20% cost uplift** for a **substantial p99 reduction**.

```python
import asyncio

async def hedged_call(primary_client, hedge_client, hedge_after_ms, **req):
    async def one(client, delay_ms=0):
        if delay_ms:
            await asyncio.sleep(delay_ms / 1000)
        return await client.chat.completions.create(**req)

    primary_task = asyncio.create_task(one(primary_client))
    hedge_task = asyncio.create_task(one(hedge_client, delay_ms=hedge_after_ms))
    done, pending = await asyncio.wait(
        {primary_task, hedge_task}, return_when=asyncio.FIRST_COMPLETED
    )
    for p in pending:
        p.cancel()  # winner already known; loser cancelled
    winner = done.pop()
    return winner.result()
```

Two engineering subtleties:

- **Hedge to a *different failure domain*.** Hedging back to the same provider's same region does not help if the region is the source of the tail. Route the hedge to a second provider (or at least a second region) — the mod-108 gateway makes this a config edit, not a rewrite.
- **Cost-attribute the hedge as a distinct call.** The hedge is a real request that hit a real provider; it appears in the ledger (chapter 04). Losing hedges are visible as `hedge_role: "loser"`. If losers become the majority of your calls, the hedge threshold is too tight; if the p99 is unchanged, the threshold is too loose.

**Do not hedge in agent loops without care.** For a multi-turn agent, the hedged call and the winner may leave orphan tool invocations in the ecosystem. Hedge single-turn calls; be cautious about hedging turns inside a stateful chain.

## The SLI, the SLO, and the error budget

An **SLI** (service-level indicator) is a measurable quantity: `fraction_of_requests_faster_than_2500ms_and_returning_a_valid_response` over a rolling window. An **SLO** (service-level objective) is the target: `SLI >= 99% over the trailing 28 days`. The **error budget** is what is left: `1 - SLO = 1%`, i.e. 1% of requests per 28 days may be slow or invalid before the SLO is breached.

For an LLM feature, an SLI is usually a *composite* — a single request "counts as good" only if all of:

- TTFT < some threshold (from the per-surface latency spec, chapter 01).
- Response returned successfully (no 5xx, no unretried timeout, no content-filter refusal for surfaces where that would be a user-visible failure).
- Response passed the structural-validity gate (JSON parsed, tool call resolved, whatever mod-106 defined).

Write the SLI expression down before you write the SLO. If you cannot express it as a boolean over one span, you cannot alert on it.

```
SLI_good(span) := (
    span.app.latency.ttft_ms < 2500
    AND span.gen_ai.response.finish_reason IN ("stop", "tool_calls")
    AND span.app.structural_validity = "pass"
)

SLO: rolling_28d(SLI_good) >= 0.99   # 99% good over 28 days
Error budget: 1% of 28-day request volume
```

The error budget is a *quantity* — at 10M requests/28d, 1% budget = 100k allowed bad requests over the window. You spend it slowly (the normal state) or you spend it quickly (an incident). Alerting is on the *rate* of spend, not the absolute value.

Google's SRE workbook chapter on SLOs is the canonical reference here; the shape below is a direct instantiation of the "multi-window, multi-burn-rate" pattern.<!-- needs-research: cite Google SRE Workbook, chapter 5 "Alerting on SLOs" — burn-rate multi-window multi-burn pattern. -->

## Burn-rate alerts

A **burn rate** is how much of your error budget you are consuming per unit time, normalised such that a burn rate of 1.0 exactly exhausts the budget over the SLO window. A burn rate of 14.4 exhausts the *entire 28-day budget* in 2 days; a burn rate of 2 exhausts it in 14 days. The multi-window, multi-burn pattern configures two thresholds:

- **Fast burn** — 14.4× burn rate over a 1-hour window (would exhaust the budget in ~2 days). Page the on-call.
- **Slow burn** — 6× burn rate over a 6-hour window (would exhaust in ~4-5 days). Ticket the team.

In Prometheus-style pseudocode:

```promql
# error rate over 1h window
error_rate_1h = sum(rate(bad_requests[1h])) / sum(rate(all_requests[1h]))

# burn rate normalised against the SLO's error budget
burn_rate_1h = error_rate_1h / (1 - 0.99)   # = error_rate_1h / 0.01

# Fast-burn alert
ALERT SLOFastBurn IF burn_rate_1h > 14.4 FOR 5m

# Slow-burn alert
error_rate_6h = sum(rate(bad_requests[6h])) / sum(rate(all_requests[6h]))
burn_rate_6h = error_rate_6h / 0.01
ALERT SLOSlowBurn IF burn_rate_6h > 6 FOR 15m
```

Two properties make burn-rate alerts materially better than "alert on p95 > threshold":

- **Alert budget-aware.** A short blip that consumes 0.01% of a 1% budget is not worth waking anyone up; a sustained slow bleed *is*, even if no single window looks bad. Burn-rate captures both.
- **Alert reliability-shape-aware.** Two windows (1h fast, 6h slow) reduce false positives: a real incident trips both; a random spike trips only the fast window and self-clears before the slow one fires.

Wire the alert to the on-call notification you actually respond to, not to a channel that becomes background noise. The whole point of budget-aware alerting is that when the alert *does* fire, it means something.

## Putting the levers together

You have four levers when the SLO burns:

1. **Retry with a budget.** Fixes transient 5xxs; useless against sustained provider tail latency.
2. **Hedged requests.** Fixes tail latency at the cost of a small % of extra spend. Deploy against p99 spikes that come from provider-side variance.
3. **Provider / model fallback (mod-108).** Routes to a second provider when the primary sustains failures. The mod-109 SLO drives *when* the fallback fires; mod-108 chapter 02 owns *how*.
4. **Graceful degradation.** Return a shorter answer, disable a tool, skip a step, or surface a "having trouble, retry?" state to the user. Sometimes the right lever is not "make it fast" but "return something."

The SLO spec should name which lever fires at which burn rate. A sketch:

```markdown
## SLO spec — <feature> — <date>

SLI good = (TTFT < 2500 ms) AND (finish_reason IN {"stop","tool_calls"}) AND (structural_validity = "pass")
SLO = 99% good over 28d rolling
Error budget = 1% (≈ 100k requests / 28d at current volume of 10M)

## Levers
- Retry: max 2 attempts, exp-backoff w/ jitter, budget 5% of window traffic.
- Hedge: fire secondary to <fallback provider/region> at t = 1200 ms if no first token.
- Fallback: circuit-break to <fallback model> after 5 consecutive 5xx from primary.
- Degrade: on burn > 6× for 6h, disable the optional tool that adds ~800 ms to the tail.

## Burn-rate alerts
- Fast: burn_rate_1h > 14.4 for 5 min → page.
- Slow: burn_rate_6h > 6 for 15 min → ticket.

## Owner / revisit
Owner: <team>. Reviewed: <date>. Next review: <quarter>.
```

That artefact is the load-bearing deliverable of this chapter, and the pair to the per-surface latency spec (chapter 01) and the per-workload batch eligibility rubric (chapter 03).

## Consuming MLPerf serving benchmarks

MLPerf Inference is the industry-standard suite for serving benchmarks. MLCommons publishes results across scenarios (Server, Offline) with strict rules on model, dataset, and quality bar, so numbers across submitters are comparable.<!-- needs-research: confirm current MLPerf Inference version, scenarios (Server, Offline), and the language-model track (Llama-3-70B, GPT-J, Mixtral). -->

**Who owns it in your org.**

- `ai-infra-performance` runs MLPerf-style benchmarks against your own inference stack, or reads submissions to plan hardware and serving-framework choices.
- `model-evaluation-engineer` uses MLPerf and MLPerf-shape benchmarks to sanity-check per-model quality vs the published references and to keep the eval suite (mod-106) grounded.
- **The applied engineer *consumes* the outputs.** You do not run MLPerf; you read it as a reference frame that tells you whether your observed TTFT and TPOT are within an order of magnitude of what the model can plausibly do on the fleet you are using.

The three questions the reference frame answers for you:

1. **Is our p95 TPOT plausible for this model on this hardware?** If MLPerf-style benchmarks for the model on H100s report 90 tok/s server p95 and your production p95 is 15 tok/s, something on your path is broken (rate-limit throttling, batch size, prefix-cache misses, network) — the model itself is not the constraint.
2. **What does the ceiling look like if we self-host?** The self-host-vs-managed decision (mod-108 chapter 03) reads benchmark numbers as the *upper bound* on what a self-hosted deployment could deliver. Real production numbers land under that ceiling.
3. **When a new model launches, what should we expect?** MLPerf submissions typically appear within months of a model launch and give a per-hardware baseline of throughput and TTFT. Set your initial SLO expectations against these, then relax them if your surface is throughput-heavy vs latency-heavy differently from the reference.

**What MLPerf is not.**

- It is not your SLO. MLPerf is a controlled benchmark on specific hardware with a specific serving stack; your production SLO is what your users experience through your gateway, retries, and tool calls. The two numbers should be in the same universe; they should not be equal.
- It is not a quality benchmark. MLPerf enforces a quality bar (perplexity, ROUGE, etc.) to make throughput comparisons fair; it does not tell you whether your prompt is good, whether your retrieval is good, or whether your users are satisfied. The eval suite (mod-106) owns those questions.
- It is not comprehensive. Not every model, hardware, or provider is present in every round; when it is not, you fall back to provider-published serving numbers or your own baselining runs.

## Anti-patterns

- **Retry with no budget.** A single provider blip amplifies into 3× normal load, worsening the outage. Budget every retry loop.
- **Backoff without jitter.** Coordinated clients retry in lockstep. Add jitter on day one.
- **Retry on 4xxs.** A `content_filter` refusal or a `400 invalid schema` is retried three times. The prompt is invalid; retrying does not help; each attempt costs. Enumerate what is retryable.
- **Hedging always.** Doubles cost; hurts the provider's fleet globally; a source of "why is our bill 2× what we projected" incidents. Hedge only after the p95 threshold, and only for surfaces where p99 matters enough.
- **P95 threshold alerting instead of burn-rate.** Alerts fire on transient spikes; ops turns off the alert; the real bleed is invisible. Multi-window burn-rate is the industry pattern for a reason.
- **SLO without an SLI expression.** "We should be fast" is not an SLI. Write the boolean.
- **MLPerf as the SLO.** Setting a production TTFT SLO equal to the MLPerf reference is over-committing — production numbers include network, retries, tool calls, and provider-side variance that the benchmark does not.
- **No revisit cadence on the SLO.** Traffic grows, tools evolve, the model swaps; the SLO from 18 months ago is now aspirational or trivially met. Review quarterly.

## Summary

- Retries are a lever with a **budget** (typically 5–10% of window traffic); beyond the budget, requests fail fast to prevent amplification.
- Hedging (Dean & Barroso) fires a duplicate after a short delay to a different failure domain; expected cost uplift is ~10–20% for a substantial p99 improvement.
- The **SLI** is a boolean over one span (latency, success, structural validity); the **SLO** is the promise; the **error budget** is the quantity you spend.
- Alert on **burn rate**, not on absolute p95 thresholds. Two windows (1h fast, 6h slow) with two burn thresholds (14.4× fast, 6× slow) is the canonical pattern (Google SRE Workbook).
- The four levers when the SLO burns: retry, hedge, fallback (mod-108), degrade. Name in the SLO spec which lever fires at which burn.
- **MLPerf inference benchmarks** are a reference frame owned by `ai-infra-performance` and `model-evaluation-engineer`; the applied engineer consumes them to sanity-check plausibility, not to author the SLO.
- The load-bearing artefact is a per-feature **SLO spec** naming the SLI expression, the error budget, the levers, and the burn-rate alerts — pair it with the per-surface latency spec (chapter 01), the prompt-cache posture (chapter 02), the batch rubric (chapter 03), and the accounting ledger (chapter 04) to complete the mod-109 substrate.

The module ends here. What each mod-109 chapter emits — the latency spec, the cache posture, the batch rubric, the accounting ledger, the SLO spec — becomes the input for mod-110 (observability, feedback, incidents), mod-111 (shipping, experimentation, rollbacks), and the mod-112 capstone (end-to-end feature ownership).
