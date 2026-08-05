# Classifier-Gated Routing, Quality Gates, and Provider-Outage Fallback

## Three axes hiding under one word

"Routing" in a production LLM stack does three genuinely different jobs, and confusing them is the source of most router bugs. The three axes:

1. **Difficulty routing** — cheaper (or smaller / faster / open-weights) model when the task fits; only escalate to the frontier when the task actually demands it. This is where the cost story lives.
2. **Quality gates** — after a completion arrives, check whether the answer is *good enough*. If not, retry with a stronger model or a different route. This is where the quality-cost trade lives.
3. **Availability fallback** — the primary provider is timing out, rate-limiting, or 500-ing; route to a secondary path so the user's request still succeeds. This is where the reliability story lives.

Every serious router does all three, but each axis has a different signal, a different failure mode, and a different metric. Bolt them together without keeping them distinct and you get a router that "just retries on every request" — which quietly triples cost and papers over a real quality regression as a load-balancing shrug.

## Axis 1 — Difficulty (classifier-gated) routing

The premise: your traffic is not one distribution. Some requests are `what's 2+2`, some are `summarise this in one line`, some are `debug this cross-repository race condition and propose a patch`. Sending them all to the same frontier model has one virtue (uniform behaviour) and one very expensive vice (paying frontier prices for `2+2`). Difficulty routing tries to spend the expensive tokens only on requests that need them.

The three shapes that show up in the wild, in order of complexity:

**Shape 1a — Static heuristic gates.** No classifier. A small ruleset picks a tier based on visible request properties: prompt length, whether tools are needed, whether the surface is `chat` vs `background-batch`, whether the tenant is on the paid tier. Fast, transparent, easy to defend to reviewers, and often 80% of what more sophisticated schemes give you. Start here.

**Shape 1b — Classifier-gated routing.** A small, cheap model (a distilled encoder, a small LLM in JSON-mode, or even a rules-plus-embedding classifier) scores each request against a category or difficulty schema and picks a tier. Examples of what a classifier decides: *is this a factual question or a creative task; does it need retrieval; does it need code; is it likely to need multi-step reasoning; is it in-scope for our product at all.* The router then maps the classification to a route.

**Shape 1c — Learned / bandit routing.** The router treats route selection as a bandit problem — offline logs of (features, chosen route, judged outcome, cost) train a policy that picks the cheapest route with acceptable quality. This is the RouteLLM / hybrid-LLM / cascade literature.<!-- needs-research: RouteLLM (Ong et al., 2024) is the canonical citation; confirm the current name and any successor benchmark. --> Powerful, worth it at large scale, but a lot of machinery for a first cut.

For most applied surfaces, Shape 1a wins the first quarter, Shape 1b wins the second. Skip Shape 1c until the volume and metrics support it.

### The classifier's schema

A classifier without a schema is a wish. Write the schema down before writing the classifier. A minimum shape:

```yaml
# routing/classifier-schema.yaml
categories:
  - key: chit_chat
    example: "hello", "how are you", "what can you do"
    route: cheap
  - key: factual_short
    example: "what is the capital of Peru", "convert 5kg to lb"
    route: cheap
  - key: reasoning_multi_step
    example: "debug this race condition", "compare these two contracts"
    route: frontier
  - key: creative_long_form
    example: "write a 2000-word chapter", "generate ad copy variants"
    route: mid
  - key: tool_heavy
    example: "book me a flight", "run this query and summarise"
    route: frontier
  - key: out_of_scope
    example: "how do I hotwire a car"
    route: refuse
```

Three properties. First, every category has at least one seed example — the classifier's fine-tune (or its few-shot prompt) uses these directly. Second, `route: refuse` is a legitimate outcome — the router is where you honour the surface's scope, not the model. Third, `route` names a route, not a model — the route-to-model mapping is a separate config so a model swap is one edit.

### The route table

```yaml
# routing/routes.yaml
routes:
  cheap:
    primary:    openai/gpt-4o-mini
    fallbacks:  [anthropic/claude-haiku, self-hosted/llama-3.1-8b]
    max_p95_ms: 1500
    max_cost_per_call_usd: 0.003
  mid:
    primary:    openai/gpt-4o
    fallbacks:  [anthropic/claude-sonnet, google/gemini-flash]
    max_p95_ms: 4000
    max_cost_per_call_usd: 0.02
  frontier:
    primary:    anthropic/claude-opus
    fallbacks:  [openai/gpt-4o]
    max_p95_ms: 15000
    max_cost_per_call_usd: 0.20
```

The route table is the *only* place model names appear. Feature code says "give me a `mid` completion for this request" and never mentions a provider. The routes are pinned to dated model IDs (not `latest` aliases), so a provider alias flip does not silently shift your distribution.<!-- needs-research: illustrative model IDs above; the pattern of `-mini`, `-flash`, `-sonnet`, `-haiku` tiers is durable, but specific names should be updated at author time. -->

### Metrics that make a router honest

Every routed request emits a span with at least: `route.classified_category`, `route.chosen_route`, `route.chosen_provider`, `route.escalated_from` (if the request was escalated), `route.cost_estimate_usd`, and `route.quality_gate_result`. From those, a few first-order metrics:

- **Escalation rate** per category. If `chit_chat` is being escalated to `frontier` 15% of the time, the classifier is confused (or the quality gate below is too aggressive).
- **Cost per successful outcome** per route, per category. The cheaper route only wins if it also converts. A route that costs 90% less but forces 40% escalations is not saving you money.
- **Category distribution drift**. Weekly, compare the current distribution against last month's. A sudden spike in `out_of_scope` is a product signal, not a router bug.

The story you should be able to tell from these: "we routed 62% of traffic to `cheap` this month, 4% of which escalated; that saved us $X against 100% frontier, and quality metric Y did not regress." Without those numbers the router is a leap of faith.

## Axis 2 — Quality gates

Classifier-gated routing gets the *first* attempt right most of the time. Quality gates catch the times it did not — and are useful even without difficulty routing, because a low-confidence answer from *any* model is a candidate for a second look.

Four common gate shapes:

- **Structural validity.** The completion is supposed to be JSON conforming to schema S. Parse and validate. If it does not validate, re-route (either same model with a repair prompt, or escalate). This is the cheapest and highest-value gate — it prevents downstream code from crashing on malformed structured outputs. Mod-102's structured-outputs discipline is where the validators live.
- **Self-report confidence.** Ask the model, in the same response, for a confidence score against a rubric. Cheap, noisy, and useful only against a *threshold you calibrated on your own eval set*. Do not trust confidence scores in the abstract; calibrate them against labelled outcomes and re-calibrate periodically.<!-- needs-research: model self-report calibration is well-documented as unreliable in general but per-task tuning can help; be explicit that raw confidences are noisy. -->
- **Judge-side check.** A second, small model (or a rules layer, or a retrieval-grounded checker) reviews the completion against a rubric. Judge on refusal-shape, on schema conformance, on factuality-against-retrieval, on tone. The judge is fast; the escalation is triggered only on judge-fail.
- **Self-consistency / N-sampling.** Draw K samples, choose the most common (or the one the judge scores highest). Doubles-to-Ks the cost of that call; use sparingly for high-stakes turns.

The gate policy is a spec:

```yaml
# routing/quality-gates.yaml
gates:
  structured_output:
    applies_to: [routes.mid, routes.frontier]
    check: schema_validate
    on_fail:
      action: escalate
      to: frontier
      max_escalations: 1
  judge_check:
    applies_to: [routes.cheap]
    check: judge_model(prompt="grade against rubric.md", model=judges.small)
    threshold: 0.75
    on_fail:
      action: escalate
      to: mid
      max_escalations: 1
```

Two spec properties matter. First, **max_escalations is finite** — a runaway "escalate on any doubt" loop is the classic cost fire. Second, the gate's *own* cost is in the accounting; a judge that costs 30% of the primary call had better save more than 30% of primary-cost failures.

### The retry-loop-that-ate-the-budget

The single most common router incident: a subtle regression makes the primary route's completions fail the quality gate 40% of the time. The router silently escalates 40% of traffic to the next tier. The next tier fails its gate 15% of the time and escalates to frontier. The bill doubles overnight. Nothing pages because nothing errored.

Two disciplines prevent this. First, an alert on **escalation-rate-by-route** (page when it moves > N% off baseline). Second, **cost SLOs** at the route table, not just at the call. `cheap` has a `max_cost_per_call_usd` for a reason — a `cheap` call that ends up costing frontier prices because of a retry loop is a hard failure of the route, and the router should flag it, not silently absorb it. Chapter 04 of mod-109 develops the cost SLO in full; the router surface is where they *fire*.

## Axis 3 — Provider-outage fallback

Every provider goes down. OpenAI's status page has been coloured yellow at every one of your recent product reviews; Anthropic has too; Bedrock has too; every hyperscaler regional endpoint has too. The router's third job is to route around the outage.

Failure modes to plan for, in decreasing severity:

- **Provider region 5xx / connection error.** Full stop.
- **Rate limit (429) or capacity error (503) with retry-after.** Not down, but not serving *you* right now.
- **Slow-loris tail latency**. Requests eventually succeed at P99 of 60 seconds. Your users perceive an outage.
- **Model deprecation / removal**. The model ID silently returns "unknown model." Rare but has happened.<!-- needs-research: this has happened at multiple providers during model version transitions; be careful not to name specific incidents unless verified. -->
- **Regional partition**. The provider works from us-east but not eu-west.

### The circuit-breaker skeleton

A router without a circuit breaker will re-hit the failing provider on every request, adding queue latency to the outage. The pattern:

```python
# routing/circuit_breaker.py — sketch, not a library.
@dataclass
class Breaker:
    provider: str
    failure_window_sec: int = 60
    threshold: int = 5             # failures within the window
    open_duration_sec: int = 30    # cool-off before probe
    state: Literal["closed", "open", "half_open"] = "closed"

async def call_with_breaker(route, request):
    for provider in [route.primary, *route.fallbacks]:
        breaker = breakers[provider]
        if breaker.state == "open":
            continue
        try:
            resp = await asyncio.wait_for(
                provider.call(request),
                timeout=route.max_p95_ms / 1000 * 2,   # hard timeout > SLA
            )
            breaker.record_success()
            return resp
        except (TimeoutError, ProviderError) as e:
            breaker.record_failure(e)
            continue
    raise NoProviderAvailable(route.name)
```

The properties: a hard timeout on every provider call (not "provider's default"), a circuit breaker per provider (not per model — a provider-level outage takes every model on that provider with it), and a half-open probe so a provider that recovers gets traffic back automatically.

### The routing-decision log

Debugging a fallback path six hours after it fired is impossible without a routing-decision log. Per request, log:

```json
{
  "request_id": "req_...",
  "route": "mid",
  "attempts": [
    {"provider": "openai", "model": "gpt-4o-2024-08-06", "outcome": "timeout", "latency_ms": 4001},
    {"provider": "anthropic", "model": "claude-3-5-sonnet-20241022", "outcome": "success", "latency_ms": 1240}
  ],
  "final_provider": "anthropic",
  "final_model": "claude-3-5-sonnet-20241022",
  "escalated": false,
  "quality_gate": {"gate": "structured_output", "result": "pass"}
}
```

Two properties. First, the *attempts* list is preserved — not just the final. When the primary starts recovering, you can see the transition happen in the logs. Second, `escalated` and `attempts` are separate — a call that succeeded on the primary is not "escalated," and rolling one into the other loses the reliability-vs-quality distinction.

### The commensurable-fallback discipline

Falling back to a provider that produces subtly different answers is not a fallback — it is an A/B test the user did not consent to. Two rules:

1. **The fallback must produce equivalent shape.** If the primary returns tool-calls, the fallback returns tool-calls of the same shape. If the primary returns JSON matching schema S, the fallback returns JSON matching schema S. This is where chapter 05's contract tests earn their keep — a fallback that has not been contract-tested is a footgun.
2. **The fallback's quality is measured, not assumed.** Every fallback path emits its own quality metrics; a sudden spike in fallback traffic with a quiet degradation of the quality metric is exactly the shape of a hidden regression. The pre-registered quality metric from mod-106 chapter 01 is the same metric the router watches.

## Composing the three axes: the routing state machine

A minimally sane router does axis 1 (classify → pick a route), then axis 2 (validate the result → maybe escalate), and *all* the way through calls axis 3 (circuit-break every provider hop). The state machine:

```
   ┌──────────────┐
   │  Classify    │  (axis 1: pick a route)
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │  Call route  │  (axis 3: primary → fallback per breaker)
   └──────┬───────┘
          ▼
   ┌──────────────┐         gate.fail && attempts < max
   │  Quality     │  ──────────────────────────────►  ┌──────────────┐
   │  gate        │                                    │  Escalate    │
   └──────┬───────┘  gate.pass                         │  route       │
          ▼                                            └──────┬───────┘
   ┌──────────────┐                                           │
   │  Return      │  ◄────────────────────────────────────────┘
   └──────────────┘
```

The state machine deliberately keeps escalation (axis 2) separate from fallback (axis 3). Axis 2 changes the *tier* (cheap → mid → frontier) because the answer was bad. Axis 3 changes the *provider within a tier* because the provider was unavailable. Confusing them is how a router turns a provider outage into a bill spike (fallback fires, quality gate fails on the fallback's output, escalation kicks in, cascades to frontier, load pushes frontier to rate-limit, fallback again…).

## Anti-patterns

- **The router that is one `try/except`.** All fallback logic is `try: primary. except: secondary.` No circuit breaker, no timeout, no telemetry, no per-route policy. Works until the primary tail latency blows past your P99 for an hour and every request queues waiting for the timeout you never set.
- **The classifier that is the same model as the frontier.** Using GPT-4-class calls to decide which GPT-4-class calls to make. The classifier is often ≥ 20% of the cost of the frontier call it was supposed to save. Distil, use a small hosted classifier, or run a genuine rules layer for the obvious buckets.
- **The escalation loop with no cap.** A misconfigured quality gate escalates every call. The bill triples overnight. Fix: `max_escalations` is a hard number, and an alert fires when *the population* escalation rate breaches baseline.
- **The fallback that returns different shape.** Primary returns `{"tool_calls":[...]}`; the fallback returns `{"function_call":{...}}` because the shim was written for an older provider version. Downstream JSON parser crashes are the visible symptom; the routing decision that caused them is invisible. Contract tests (chapter 05) are the specific fix.
- **The "temperature=0" reliability fantasy.** The router promises stable behaviour by pinning temperature=0. LLMs at temperature 0 are less variable, not deterministic — GPU non-determinism, batching effects, and provider re-tokenisation all leak in. Route on shape and cost, not on "the outputs will be identical."<!-- needs-research: temperature=0 non-determinism is well-documented; keep the framing as "less variable, not deterministic." -->
- **The router with no dashboard.** Escalation rate, per-route cost, fallback fire rate, provider-outage minutes — none of these are on any screen. When the CEO asks "why did the bill double last week," the answer is a git-log-and-shrug.

## Summary

- Three axes hide under "routing": **difficulty** (classifier picks a cheaper tier), **quality gates** (validate the completion, escalate if it fails), **provider-outage fallback** (circuit-break the failing provider, retry on the next in the group). Keep them distinct — same word, different signal, different metric.
- Classifier-gated routing works best when the classifier's **schema is written down first**, the route table names dated model IDs (never `latest`), and the router emits per-route metrics for escalation rate, cost per successful outcome, and category drift.
- Quality gates are cheap (schema validation) to medium-cost (judge model) to expensive (self-consistency). Every gate has a **max escalations** cap and its own cost in the accounting.
- The fallback path uses per-provider circuit breakers, hard timeouts (not provider defaults), a routing-decision log with the *attempts* preserved, and only routes to fallbacks that have been contract-tested for equivalent shape (chapter 05).
- Escalation and fallback are separate axes; conflating them turns provider outages into cost fires.

[Chapter 03](03-self-hosted-vs-managed-with-vllm-baseline.md) turns from routing between providers to the deeper question underneath the route table: for a given route, should the model behind it be self-hosted or managed?
