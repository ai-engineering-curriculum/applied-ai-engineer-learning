# Cross-Provider Contract Testing and Per-Provider Golden Traces

## The failure a contract test catches

Chapters 02 and 04 both stopped at the same warning: "the fallback must produce equivalent shape, or the router's swap is an unconsented A/B." This chapter formalises "equivalent shape" into an artefact — the **cross-provider contract test** — and pairs it with **per-provider golden traces** so a provider swap does not silently regress quality, cost, or format.

The failure mode this discipline exists to catch is boring and expensive. A team ships route `chat.mid` with OpenAI as the primary and Anthropic as the fallback. On Tuesday, OpenAI has a soft outage. The router fails over to Anthropic. Anthropic returns tool-calls in a different shape than OpenAI: `content: [{ type: "tool_use", id, name, input }]` instead of the OpenAI-shaped `tool_calls: [{ id, type, function: { name, arguments } }]`.<!-- needs-research: confirm current Anthropic Messages API tool-use content-block shape. --> The router's shape-normaliser was written for a request path that only ever saw the OpenAI shape. Downstream tool dispatcher crashes on every Anthropic-routed call. No test caught it — no test *ran* the request against Anthropic.

The discipline: **every route on the route table is exercised, on every provider on its fallback list, against a fixed set of golden requests, before it is trusted to serve traffic.**

Two kinds of test do the work. **Contract tests** check the *shape* of the response — same fields present, same types, same enums, same schema conformance. **Golden traces** capture the *behaviour* of the response — a snapshot of what "good" looks like, per provider, per fixture — and diff a new run against it to catch quality or cost regressions.

## What is under contract

Before writing any tests, name what the contract covers. A minimal shape for a chat / completion route:

- **Response envelope.** The top-level fields the caller reads: `id`, `model`, `created`, `choices` / `content`, `usage`, `finish_reason` / `stop_reason`. The router either normalises to a single shape (typical) or the caller handles both shapes explicitly. Whichever, the contract test asserts the normalised shape.
- **Message content shape.** For text, a single string. For structured output, valid JSON conforming to the caller's schema. For tool-calling, the normalised tool-call array (name, arguments-as-JSON, id).
- **Tool call semantics.** If the model chose to call a tool, the call is present in the response with the right arguments. If the model chose *not* to call a tool when the fixture expected one, that is a behaviour delta a golden trace catches, not a shape delta a contract test catches.
- **Usage / token accounting.** Per-provider token counts differ (different tokenisers, different image-token accounting, cache-hit reporting), but the *field* has to be present and non-null so the accounting layer (mod-109 chapter 04) works.
- **Finish / stop reason.** `stop`, `length`, `tool_calls`, `content_filter` — normalised to the caller's vocabulary. A provider that emits a fifth stop reason your caller does not know is a failure.
- **Safety / refusal shape.** A refusal produced by provider A and provider B should be *legible as a refusal* the same way. If one refuses by returning empty content with a `content_filter` stop reason and the other refuses by returning a policy-text message with a normal stop reason, downstream refusal-handling code will diverge.
- **Streaming shape.** If the route streams, the chunk shape and delta semantics are under contract. This is where a lot of gateway leakage happens — a fallback path that streams "one message, done" instead of token-by-token is technically valid and product-visibly wrong.
- **Latency and cost envelope.** Not a hard equality, but bounded — a fallback that costs 3x the primary or blows past the p95 SLO by 2x is a fail even if the shape is correct.

Naming what is under contract *and what is not* keeps the tests honest. Word-for-word output equality is not under contract (models are non-deterministic, even at temperature 0, especially across providers). Deep semantic equivalence is not under contract (that is the eval suite in mod-106, not the contract test).

## The contract-test skeleton

The test is a fixture set + a runner + assertions per provider. A minimal shape:

```yaml
# contract/fixtures/chat-mid/factual-qa.yaml
id: factual-qa-001
route: chat.mid
messages:
  - role: system
    content: "You are a concise assistant. Answer in one sentence."
  - role: user
    content: "What is the capital of Peru?"
expect:
  response_shape:
    choices: { type: array, min: 1 }
    choices[0].message.content: { type: string, min_length: 1, max_length: 200 }
    choices[0].finish_reason: { in: [stop, length] }
    usage.total_tokens: { type: int, min: 1 }
  content_semantics:
    must_contain: ["Lima"]
  cost:
    per_call_usd_p95_max: 0.01
  latency:
    ttft_ms_p95_max: 1500
```

```yaml
# contract/fixtures/chat-mid/tool-call.yaml
id: tool-call-001
route: chat.mid
messages:
  - role: user
    content: "What's the weather in Paris tomorrow?"
tools:
  - name: get_weather
    parameters:
      type: object
      properties:
        city: { type: string }
        date: { type: string, format: date }
      required: [city, date]
expect:
  response_shape:
    tool_calls: { type: array, min: 1 }
    tool_calls[0].name: { equals: get_weather }
    tool_calls[0].arguments: { valid_against: tools[0].parameters }
  content_semantics:
    tool_calls[0].arguments.city: { equals_ci: "Paris" }
```

Two fixture shapes cover most needs — a *response-shape* fixture (does the provider return the field, the type, the enum) and a *content-semantics* fixture (does the content match a rubric your feature depends on). Contract tests deliberately keep the semantic bar low — `must_contain: ["Lima"]` is not a benchmark, it is a *the model actually answered the question* smoke test.

The runner walks every provider on the route's fallback list (including the primary) and runs every fixture:

```python
# contract/run.py — sketch, not a library.
async def run_contract(route_name: str) -> ContractReport:
    route = load_route(route_name)
    fixtures = load_fixtures(route_name)
    report = ContractReport(route=route_name, at=timestamp())
    for provider in [route.primary, *route.fallbacks]:
        for fx in fixtures:
            result = await provider.call(fx)
            report.record(
                provider=provider.name,
                fixture=fx.id,
                shape_ok=assert_shape(result, fx.expect.response_shape),
                semantics_ok=assert_semantics(result, fx.expect.content_semantics),
                cost=cost_of(result),
                latency=latency_of(result),
                trace=capture_trace(result),
            )
    return report
```

Two things the runner does that a naive one does not. First, it runs **against every provider on the fallback list**, not just the primary — the fallback path is the whole point. Second, it **captures a trace** per provider per fixture so the golden-trace machinery below has raw material to diff against.

## Golden traces per provider

A **golden trace** is a captured, normalised, sensitive-data-scrubbed record of a specific request/response pair on a specific provider — the response body, the token counts, the tool-calls, the finish reason, the latency, the cost. Named, versioned, checked into the repo (or into an artefact store the CI reads from).

For each route × provider × fixture combination, the runner captures a trace on the first successful run and files it as the golden:

```
contract/goldens/
  chat-mid/
    factual-qa-001/
      openai.golden.yaml           # captured against openai/gpt-4o-2024-08-06
      anthropic.golden.yaml        # captured against anthropic/claude-3-5-sonnet-20241022
      google.golden.yaml
    tool-call-001/
      openai.golden.yaml
      anthropic.golden.yaml
```

Each golden pins the model ID (dated, not `latest`) and the runner's version. Subsequent runs *diff* the new trace against the golden. The diff is not "assert equal" — it is a structured comparison:

- **Response envelope fields present** — hard fail if missing.
- **Message content length within band** — soft warn if outside ±30% of golden length; hard fail if empty when golden was not.
- **Tool call name and argument keys equal** — hard fail if different.
- **Semantic must-contain checks pass** — hard fail if not.
- **Token counts within band** — soft warn if outside ±20% of golden; the goal is to catch a provider-side tokeniser change, not to enforce equality.
- **Latency within band** — soft warn if p95 outside ±50% of golden.
- **Cost within band** — hard fail if outside ±30% (price changes need explicit human sign-off, not silent absorption).

Two important disciplines. First, **hard vs soft** is a policy decision, not a runtime one — write it down in the contract config so a reviewer can argue about it in code, not in a Slack thread. Second, the golden's shelf life is not infinite — providers legitimately update model behaviour. Re-capture goldens on a schedule (typically after a model rev, on the pinned dated ID) and *diff the diff* — reviewers approve the new golden explicitly, so a "silently absorbed drift" is impossible.

## The CI story

Contract tests and golden diffs run in CI. Two triggers matter:

- **On every change to the route table, gateway config, or feature code that calls the router.** Catch the "I renamed a field in the router" kind of bug at PR time.
- **On a nightly / weekly schedule against the current pinned provider models.** Catch the "provider silently rolled a model behind the same dated ID" kind of drift.

The CI job's output is the same **contract report** the runner produced. A PR that makes a hard-fail diff worse against any provider on any route is blocked. A soft warn is surfaced in the PR comment; a reviewer chooses to accept or reject. A golden re-capture is a PR of its own — the reviewer sees `+ new golden` next to a link to the old one and decides.

Mod-106 chapter 03 develops the eval-gated CI substrate in full — the contract-test job is one job on that pipeline, adjacent to the offline-eval suites and the trace-snapshot regressions. Keep contract tests *separate from* offline evals conceptually: contract tests check the provider swap does not break the pipe; offline evals check the feature has not regressed on quality. Same substrate, different failure modes.

## Handling the necessary noise

Contract tests will produce false positives. The applied engineer's job is not to eliminate them (impossible) but to make the noise legible.

Sources of legitimate variance:

- **Non-deterministic decoding.** Even at temperature 0, KV-cache non-determinism, batching effects, and provider-side re-tokenisation produce different outputs across runs. The soft-warn bands are wide on purpose.
- **Model updates behind dated IDs.** Providers occasionally patch models without bumping the ID (bug fixes, safety updates). The nightly job catches these. Re-capturing the golden is the correct response, not "raising the tolerance."
- **Regional variance.** The same model in `us-east-1` and `eu-west-1` may produce subtly different behaviour due to independent deployments. If the route pins a region, so does the golden.
- **Real regressions on the provider side.** The provider ships a change that breaks your workload. Now you have evidence — a diff — to file a support ticket with, and a fallback to route around it while you wait.

Legitimate variance is captured by the *tolerance bands*, not by "ignore the fixture." A fixture the tests routinely ignore is not under contract; delete it or fix it.

## The three artefacts

The chapter's discipline produces three artefacts that, together, make provider swaps safe:

1. **The contract-test fixture set.** Small, versioned, pointed at each response property the router or feature code reads. Not an eval — this is a *pipe* test.
2. **The per-provider golden traces.** Captured once per (route, provider, fixture), version-pinned to the dated model ID, refreshed via explicit PR.
3. **The contract report.** Emitted every run, machine- and human-readable, with per-provider passes / soft-warns / hard-fails. Blocks PR merge on any hard-fail; surfaces soft warns for reviewer sign-off.

Without these three, "we support Anthropic as a fallback" is a promise, not a fact.

## Anti-patterns

- **The one-provider contract.** Tests exist, but only run against the primary. The fallback path is never exercised until it fires in prod. Fix: the runner enumerates the fallback list; CI is unhappy if any is missing.
- **The equal-string assertion.** Fixture insists the response equals the golden token-for-token. Passes on the first run, fails on the second. Team raises the tolerance to "contains a period," and now the fixture asserts nothing. Use *shape* + *must-contain* + *bounded numerics*, not equality.
- **The golden never refreshed.** Model was silently updated six months ago; the golden is stale; every run soft-warns; the alert channel is muted; a real regression hides in the noise. Refresh goldens on a cadence, and every refresh is a reviewed PR.
- **The `latest` alias in the fixture.** The fixture calls `gpt-4o-latest`. Two months later `gpt-4o-latest` points at a different snapshot. Diff comes back nonsense. Fixtures pin dated IDs; the fixture record includes the ID it was captured against.
- **Contract tests as a substitute for eval.** The router swap is "safe" because the contract tests pass, so no offline eval is run. The offline eval would have caught a 4-point quality drop on the target metric. Contract tests check the pipe; eval suites (mod-106) check the quality. Both, always.
- **Semantic checks that leak PII into the fixture.** A real user's message becomes the fixture. Six months later the fixture is in a public repo and the user's message is on GitHub. Fixtures are synthetic or explicitly-consented; the trace store's scrubbing (mod-107 chapter 03) applies here too.
- **Ignoring the streaming contract.** Contract tests only run on the non-streaming call. The streaming path's chunk semantics differ across providers and no one notices until the frontend shows a broken stream. Include a streaming fixture per streaming-capable route.

## Peer-role handoffs

- **`model-evaluation-engineer`** owns the deeper quality eval (public benchmarks, LLM-as-judge suites, calibration studies). The contract-test's semantic checks are *thin* on purpose; the deep quality bar is theirs.
- **`sre` / `platform`** consume the contract report as a signal in the release-gating pipeline. Chapter 05 of mod-111 develops the rollout / rollback story that the contract report feeds.
- **`ai-security-engineer`** owns the deeper security posture for third-party API dependencies (SBOM equivalents for provider APIs, data-flow reviews, DPA management). The contract-test discipline gives them concrete evidence of what data leaves your boundary and to whom.

## Summary

- A provider swap that silently changes response shape, tool-call format, refusal shape, cost, or latency is the specific incident this chapter's discipline exists to catch.
- **Contract tests** check the *shape* of the response against a fixture set. Run every fixture against **every provider on the fallback list**, not just the primary.
- **Golden traces** capture the *behaviour* per (route, provider, fixture), pinned to a dated model ID, refreshed by explicit reviewed PR.
- The **contract report** grades new runs against goldens with hard-fail / soft-warn bands. It runs in CI on every PR that touches the router and on a nightly schedule against pinned models.
- Contract tests are separate from offline eval. Contract tests check the pipe; eval suites check the quality. Ship both.
- The three artefacts — fixture set, per-provider goldens, contract report — are what turn "we support X as a fallback" from a promise into a fact.

This is the last chapter in the module. The routing discipline the module has built — the gateway seam (chapter 01), the router state machine (chapter 02), the self-host decision (chapter 03), the modality-aware routes (chapter 04), and the contract-tested fallbacks (this chapter) — is what a serious applied surface actually ships with. The rest of the track (mod-109 for the cost / latency SLOs the router enforces; mod-110 for the observability the router emits; mod-111 for the rollouts and rollbacks the contract report gates) folds this discipline into the wider shipping loop.
