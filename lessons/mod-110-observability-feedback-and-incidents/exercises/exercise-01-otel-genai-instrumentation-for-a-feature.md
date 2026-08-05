# exercise-01: OTel Gen-AI + OpenInference Instrumentation for a Feature

**Estimated effort:** 3 hours

## Objective

Instrument one LLM feature — end-to-end — with the OpenTelemetry Gen-AI + OpenInference span vocabulary from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md), then prove the instrumentation by running five debugger-shaped queries against the resulting traces. The exercise is not "add a decorator." It is "produce the trace shape that lets the on-call answer the questions in [chapter 04](../04-incident-playbooks-for-six-llm-failure-modes.md) without pulling logs from three services."

## Learning outcomes

By the end you should be able to:

- Emit spans for the four unit-of-work classes (LLM call, retrieval, tool call, guardrail) with the correct `gen_ai.*`, `openinference.span.kind`, and `app.*` attributes.
- Compose auto-instrumentation (provider SDK) with hand-written spans (retrieval / tools / guardrail) without duplicated payloads.
- Thread `app.feature`, `app.surface`, `app.tenant_id`, `app.user.id_hash`, `app.prompt.id`, `app.pricing_version`, and `app.cost.usd` onto every LLM span via a shared wrapper.
- Answer real debugger-shaped questions (retrieval-fail vs generation-fail, prompt-version A/B, provider alias swap) directly from the trace UI, not by grep.

## Prerequisites

- You have read [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md) and [chapter 02](../02-sampling-pii-scrubbing-and-trace-budgets.md).
- A working LLM feature — from your day job or the `mod-110/scaffold/` in the paired `applied-ai-engineer-solutions` repo (a small RAG-style Q&A surface with one tool call and one classifier guardrail).
- A trace backend that speaks OTel — Grafana Tempo, Honeycomb, Datadog, Jaeger, Phoenix (Arize), or SigNoz. Local Phoenix or Jaeger is fine and free.
- Python 3.11+ (or Node 20+ with `@opentelemetry/*` and OpenInference libraries).
- Access to at least one LLM provider (OpenAI, Anthropic, or Bedrock) and a small retrieval store (Weaviate / Qdrant / pgvector / an in-process FAISS index — the scaffold uses in-process FAISS).

## Problem statement

Your feature's current observability is: the provider SDK's built-in tracing (via auto-instrumentation) and app logs from your own code. When an incident hit last week — the tone of answers shifted overnight — the on-call spent an hour reading logs and could not tell whether the retrieval hits were bad, the prompt template had changed, or the provider alias-swapped the model. The trace UI showed one span per LLM call with the model ID and the response text; it did not show retrieval scores, tool status, prompt hash, or which feature the traffic belonged to.

Your job over three hours: instrument the feature (or the scaffold) with the full span vocabulary from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md) and prove the instrumentation via a written "debugger walkthrough" of five real questions.

## Requirements

### Deliverable 1 — The wrapper library

Under `observability/`:

- `spans.py` (or `spans.ts`) exposing the `llm_span`, `retrieval_span`, `tool_span`, and `guardrail_span` context managers from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md).
- Each wrapper sets both the OTel Gen-AI attributes AND the OpenInference attributes for its span kind.
- Each wrapper accepts the `app.*` product-attribution fields (`feature`, `surface`, `tenant_id`, `user_id_hash`, `prompt_id`, `pricing_version`) as constructor args and sets them on the span.
- The wrapper never swallows exceptions — records `status = ERROR`, sets `app.error.class`, and re-raises.

### Deliverable 2 — The auto-instrumentation + span-processor combo

- Turn on the provider SDK's OTel auto-instrumentation (`opentelemetry-instrumentation-openai`, OpenInference's `openinference-instrumentation-*`, or the equivalent for your provider).
- Add a **span processor** that decorates every auto-generated LLM span with the `app.*` attributes at export time. The processor reads the current feature / tenant / user context from a request-scoped ContextVar (Python) or AsyncLocalStorage (Node), so feature code does not have to pass attributes explicitly on every SDK call.
- Ensure prompt / response content is carried by ONE attribute set only — either OpenInference `llm.input_messages.*` / `llm.output_messages.*` OR OTel `gen_ai.content.*` events, not both. Document the choice in `docs/span-shape.md`.

### Deliverable 3 — The instrumented feature

Instrument the feature end-to-end so that a single user turn produces the trace shape from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md):

- One outer turn span (`SERVER` kind).
- One retrieval span with per-hit `document.id`, `document.score`, truncated `document.content` (bound to 512–2048 chars).
- One LLM call span with the full attribute set (Gen-AI identity, usage, app attribution, latency components from [mod-109 chapter 01](../../mod-109-cost-latency-slos-for-llm-features/01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md)).
- One tool call span with `tool.name`, `tool.parameters` (secrets pre-scrubbed), `app.tool.status`, `app.tool.round_trip_ms`.
- One guardrail span with `guardrail.decision`, `guardrail.categories`, `guardrail.model_version`.

If the feature has more than one LLM call per turn (e.g., an agent loop), each LLM call and each tool call gets its own span.

### Deliverable 4 — Drive representative traffic

Send at least 50 turns through the feature. Include:

- A mix of prompt shapes (short question, long context, an ambiguous question that likely retrieves poorly).
- At least 5 turns that trigger a tool call.
- At least 2 turns that trigger the guardrail (a benign trip is fine — a prompt that looks like it might get flagged).
- At least 1 turn that fails (a bad tool argument, a forced retry). Verify the error propagates through the span attributes correctly.

### Deliverable 5 — The debugger walkthrough

Under `docs/debugger-walkthrough.md`, five questions and the trace-UI query that answers each. Include a screenshot (or textual query result) per question.

Required questions:

1. **"Show me every LLM call in the last hour on `<feature>` where the top-1 retrieval score was below 0.4."** This proves retrieval scores are on the span and filterable.
2. **"What is the p95 total latency of turns where the guardrail blocked the response?"** This proves guardrail decisions are on the span and joinable with latency components.
3. **"For turns that used prompt hash X vs prompt hash Y, what is the difference in output token count?"** This proves `app.prompt.template_hash` is on the span and comparable across versions.
4. **"Which tenants have the highest tool-call error rate in the last 24 hours?"** This proves `app.tenant_id` and `app.tool.status` compose.
5. **"Did the provider alias-swap the model this week?"** — a query grouped by `gen_ai.response.model` over time. Any change means an alias swap.

### Deliverable 6 — The `no-duplicate-payload` proof

Under `docs/payload-audit.md`, a one-page check that the prompt and response content appear on the trace in exactly one carrier — not both an OpenInference attribute AND an OTel event. Include the trace-UI screenshot showing the fields present and absent as expected.

## Starter guidance

- **Do the wrapper first, feature second.** Land the four span helpers with tests that assert the exact attribute set, then thread them through the feature. This is much faster than instrumenting inline and refactoring later.
- **Use a request-scoped context.** ContextVar (Python) or AsyncLocalStorage (Node) — set at the request boundary, read by the span processor and the wrappers. Do not thread `feature` / `tenant_id` / `prompt_id` through every function signature.
- **Truncate retrieval-document content before setting it as an attribute.** Long documents blow through sane span sizes. 512–2048 characters is the working range; the full document is derivable from `document.id`.
- **Pin model IDs, not aliases, in the request.** If your code uses `gpt-4o`, `gen_ai.request.model` is `gpt-4o` (an alias) but `gen_ai.response.model` will be the dated version. The debugger walkthrough question 5 depends on this asymmetry.
- **Hash user IDs at the wrapper boundary.** `sha256(salt + user_id)` — the salt lives outside the trace. Never let a raw user ID reach the span attributes even in dev.
- **Do not implement your own OTel exporter.** Use the OTLP exporter and point it at your backend. The whole exercise assumes vendor-agnostic OTel export; anything else re-invents the substrate.

## Acceptance criteria

- [ ] `observability/spans.py` exposes four wrapper context managers, each with a test asserting the attribute set on the emitted span.
- [ ] Auto-instrumentation is on for the provider SDK; the span processor decorates auto-generated spans with `app.*` attributes.
- [ ] The instrumented feature produces the four-span-kind shape from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md) on every turn; a sample trace is checked in as `samples/turn-trace.json`.
- [ ] At least 50 turns of representative traffic have been driven; the traces are visible in the trace backend.
- [ ] `docs/debugger-walkthrough.md` answers all five required questions with screenshots / query results.
- [ ] `docs/payload-audit.md` proves prompt / response content is carried by exactly one attribute set, not two.
- [ ] `docs/span-shape.md` documents the attribute set and the payload-carrier decision.
- [ ] No raw user IDs or unredacted secrets appear on any span in the sample — verified by grep against the trace export.
- [ ] `gen_ai.response.model` appears on every LLM span (not just `gen_ai.request.model`).

## Stretch goals

Pick any one:

- **A `noop`-and-`otlp` split.** Wire two exporters: OTLP to the real backend, and a no-op exporter that captures the same spans in memory for tests. Add a pytest fixture that asserts the exact attribute set on a synthetic turn. Every future regression on the shape gets caught by CI.
- **A "prompt-version A/B" trace attribute.** Add `app.prompt.experiment_arm` on every span and drive traffic with a 50/50 A/B. Show the difference in a trace-UI split view.
- **A retrieval-quality snapshot.** For each turn, emit a synthetic span (`RETRIEVAL_QUALITY`) with `retrieval.quality.mrr` and `retrieval.quality.top1_score` derived from the hits. This is the seed for the retrieval-side eval in [mod-106](../../mod-106-feature-side-evaluation/README.md).
- **Cost fidelity check.** Emit `app.cost.usd` from a pricing table pinned by `app.pricing_version`. Reconcile the sum against the provider's usage API for the same window; the delta should be under 5%.

## Deliverable

Submit as `mod-110/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `observability/` with the wrapper library.
- The instrumented feature code (or a diff against the scaffold).
- `samples/turn-trace.json`.
- `docs/span-shape.md`, `docs/debugger-walkthrough.md`, `docs/payload-audit.md`.
- A short `README.md` naming the feature, the trace backend, and how to run the traffic driver.
