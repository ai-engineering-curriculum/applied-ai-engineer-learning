# OpenTelemetry Gen-AI + OpenInference: The Span Vocabulary a Debugger Needs

## Why a standard vocabulary, not a bespoke one

Every LLM feature team invents its own span shape on day one. `input`, `prompt`, `messages`, `request_text`, `user_query` — pick your organisation and you will find at least three of those in the same repo. Six months in, the observability backend has spans from four services, three SDKs, and two internal wrappers, and the on-call cannot answer "show me every call to `gpt-4o-2024-08-06` that used more than 2000 input tokens" without a query that hard-codes every alias.

The applied-AI engineer's job here is not to invent yet another attribute scheme. It is to **adopt the emerging standard so tooling can consume traces without translation**. Two vocabularies matter:

- **OpenTelemetry Semantic Conventions for Generative AI** (`gen_ai.*`) — the OTel-native, provider-agnostic attribute set for LLM operations. Vendor-neutral; consumed by any OTel-compatible backend (Grafana Tempo, Honeycomb, Datadog, Elastic, Jaeger, Signoz). Status: **experimental**, evolving faster than a release cycle; some attributes have shipped as stable while events and metrics move in ~monthly increments.<!-- needs-research: pin the exact stability status per attribute at the point of authoring an internal doc — the OTel GenAI semconv is still moving. -->
- **OpenInference** — Arize's open-source semconv layered on top of OTel, more prescriptive about LLM app shapes (LLM / RETRIEVER / TOOL / EMBEDDING / CHAIN span kinds) and about the payload attributes a debugger actually opens the trace to see (the full messages array, the retrieved documents with their scores, the tool arguments). Consumed by Phoenix, Arize, LangSmith, and any backend that understands its attribute set.

The two are complementary and coexist on the same span. **Use OTel GenAI for the fields any generic backend needs to route, filter, and aggregate. Layer OpenInference on top for the payload fields a human on-call opens the trace to read.** The two together are the observability equivalent of "structured logs plus a message field" — the semconv is your indexed columns, the OpenInference payload is the row body.

This chapter names the attribute vocabulary, the span decomposition (one span per unit of work — not one giant span per user turn), and the code shape that emits it. Sampling and PII scrubbing — how you keep this from being ruinously expensive or a data-leak — are [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md).

## The unit-of-work rule: one span per thing that can fail on its own

The first design decision is not "which attributes to emit" — it is "what gets a span at all." The wrong extreme in each direction:

- **Too coarse**: one span per user turn, with the LLM call, retrieval, tool calls, and re-prompt all attributes on that single span. The trace is unreadable. When retrieval returns junk, the LLM call is nominally successful but the answer is bad, and there is no span that says "retrieval-side latency was 800 ms and the top-1 hit had a score of 0.12."
- **Too fine**: one span per SSE chunk, one span per JSON parse. The trace is 4000 spans per turn and the storage backend charges by the span. The signal is drowned in framing.

The rule: **a span per unit of work that can fail, be slow, or be interesting on its own.** For a typical RAG-flavoured feature turn, that's:

- One span for the outer turn (the surface's request handler). Span kind: `SERVER` or `INTERNAL`.
- One span per **retrieval** call (embedding + vector store + optional rerank — split further only if any sub-step actually varies independently). Span kind: OTel `CLIENT`, OpenInference `RETRIEVER`.
- One span per **LLM call** (a "generation" in OTel's language — one prompt / response pair, whether streamed or not). Span kind: OTel `CLIENT`, OpenInference `LLM`.
- One span per **tool call** (each function the model invoked; MCP calls count). Span kind: OTel `CLIENT`, OpenInference `TOOL`.
- One span per **guardrail** invocation if you run a classifier before the response paints. Span kind: OpenInference `GUARDRAIL`.

Every child span links to its parent via OTel context propagation — that is built into the SDK; you do not roll your own trace IDs. On an agentic loop with three tool calls, the resulting trace is one turn span, one retrieval span, one LLM span for the first generation, three tool spans, and then further LLM spans for each follow-up. About ten spans for a rich turn. That is the right order of magnitude.

## The OTel GenAI attribute set

Below is the working attribute set the mod-110 observability substrate assumes on every LLM-adjacent span. Names track the OTel GenAI semantic conventions where those conventions are stable; where the semconv is still shifting, the `app.*` prefix marks the internally-owned attribute that will move to `gen_ai.*` when the spec catches up.

### On every LLM span

```python
# Provider / model identity — the backbone of every filter and rollup.
span.set_attribute("gen_ai.system", "openai")               # or "anthropic", "bedrock", "vertex_ai", "azure_openai", ...
span.set_attribute("gen_ai.request.model", "gpt-4o-2024-08-06")
span.set_attribute("gen_ai.response.model", resp.model)     # what was actually served — providers alias-swap silently
span.set_attribute("gen_ai.operation.name", "chat")         # "chat", "completion", "embedding", ...

# Invocation shape.
span.set_attribute("gen_ai.request.temperature", req.temperature)
span.set_attribute("gen_ai.request.max_tokens", req.max_tokens)
span.set_attribute("gen_ai.request.top_p", req.top_p)

# Usage — the numbers finance and SREs both want.
span.set_attribute("gen_ai.usage.input_tokens", usage.input_tokens)
span.set_attribute("gen_ai.usage.output_tokens", usage.output_tokens)

# Response identity.
span.set_attribute("gen_ai.response.id", resp.id)                        # provider request ID for support tickets
span.set_attribute("gen_ai.response.finish_reasons", [fr for fr in ...]) # "stop", "length", "tool_calls", ...
```

Two attributes carry the most product-facing weight and are worth calling out.

**`gen_ai.response.model` — what the provider actually served, not what you asked for.** OpenAI and Anthropic silently roll aliases (`gpt-4o` → `gpt-4o-2024-08-06` → `gpt-4o-2024-11-20`); Bedrock and Vertex tag responses with model IDs that differ from what you passed on the request. When the response text changes shape overnight and no one deployed anything, the answer usually lives in this field. If your telemetry only records `gen_ai.request.model`, the provider alias-swap is invisible.

**`gen_ai.response.id` — the provider's request ID.** Support tickets to OpenAI, Anthropic, Bedrock, and Vertex all start with "please send us the request ID." If you never captured it, the ticket is unactionable. Every LLM span emits it.

### On every LLM span — the app-owned additions

The GenAI semconv does not (yet, at the time of writing) cover cost, hashed user identity, feature attribution, prompt version, or cached-token accounting. These are load-bearing for a product-facing observability layer and are worth owning under an `app.*` prefix until the semconv catches up.

```python
# Cost accounting (mod-109 chapter 04's ledger — same attribute names).
span.set_attribute("app.cost.usd", cost_estimate_usd)
span.set_attribute("app.pricing_version", "2026-01-15")
span.set_attribute("gen_ai.usage.cached_input_tokens", usage.cached_tokens)  # if the provider surfaces one

# Product attribution.
span.set_attribute("app.feature", "support_agent_v3")
span.set_attribute("app.surface", "chat_panel")               # "chat_panel" / "inline_completion" / "batch" / ...
span.set_attribute("app.tenant_id", tenant_id)                # or `app.tenant_id.hash` if the raw id is sensitive

# Prompt version — see [mod-109 chapter 02] and [mod-106] for the versioning discipline.
span.set_attribute("app.prompt.id", "support_answer_v14")
span.set_attribute("app.prompt.template_hash", "sha256:...")  # so you can diff two revisions in a query

# User identity, always hashed.
span.set_attribute("app.user.id_hash", sha256_hex(salted(user_id)))
span.set_attribute("app.session.id_hash", sha256_hex(salted(session_id)))

# Latency components from [mod-109 chapter 01] belong on the same span.
span.set_attribute("app.latency.ttft_ms", ttft_ms)
span.set_attribute("app.latency.tpot_ms", tpot_ms)
span.set_attribute("app.latency.total_ms", total_ms)
span.set_attribute("app.retry.attempt_number", attempt)
```

**Never send a raw user ID or email into the trace backend.** The vendor's system is a data-processor with a DPA of its own; anything you send flows through it. Hash and salt at the SDK boundary — see [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) for the scrubbing discipline and [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md) for the deep taxonomy of what gets scrubbed and where.

### The prompt / response payload — event or attribute?

The single largest chunk of any LLM span, in bytes, is the prompt and the response. Two ways to carry them:

- **As OTel span events** (the direction the GenAI semconv is heading). Prompt content becomes a `gen_ai.content.prompt` event on the span; response content becomes a `gen_ai.content.completion` event. Events are carried in the same trace but can be sampled or scrubbed independently of the span itself — critical when the span is cheap to store but the payload is a data-privacy problem.<!-- needs-research: exact event names for prompt / response content vary across recent OTel GenAI drafts; verify against the current semconv version at authoring time. -->
- **As OpenInference message attributes**. `llm.input_messages.{i}.message.role`, `llm.input_messages.{i}.message.content`, `llm.output_messages.{i}.message.role`, `llm.output_messages.{i}.message.content`. The trace backend indexes them as first-class fields and the UI (Phoenix, Arize) renders the conversation directly.

Pick one carrier for prompts and one for responses per surface and stick with it. The failure mode is emitting the same content twice (once as event, once as attribute), which doubles the storage cost and creates two sources of truth to keep in sync.

### On every retrieval span

```python
# OpenInference retrieval semantics.
span.set_attribute("openinference.span.kind", "RETRIEVER")
span.set_attribute("retrieval.query", query_text)             # scrub PII first — see chapter 02
span.set_attribute("retrieval.k", k)
span.set_attribute("retrieval.index", "support-articles-v3")
span.set_attribute("retrieval.rerank.model", "cohere-rerank-3")  # if used

# Per-document attributes — indexed per-hit so the debugger can filter on `top-1 score < 0.4`.
for i, hit in enumerate(hits):
    span.set_attribute(f"retrieval.documents.{i}.document.id", hit.id)
    span.set_attribute(f"retrieval.documents.{i}.document.score", hit.score)
    span.set_attribute(f"retrieval.documents.{i}.document.content", truncate(hit.content, 512))
    span.set_attribute(f"retrieval.documents.{i}.document.metadata", json.dumps(hit.metadata))
```

The **per-hit score** attribute is what makes retrieval traceable. When the answer is bad, the on-call opens the trace and sees "top-1 was 0.11" and knows the retrieval step failed, not the LLM. Without the per-hit score attribute, the trace collapses retrieval-fail and generation-fail into "the answer is wrong."

Truncate document content to a bounded length (usually 512–2048 characters per hit) before writing it to the span, or the payload will blow past sane span sizes on long documents. The truncated content is enough for a debugger to recognise "these are the wrong articles"; the full content is derivable from the document ID.

### On every tool-call span

```python
span.set_attribute("openinference.span.kind", "TOOL")
span.set_attribute("tool.name", tool_call.name)
span.set_attribute("tool.description", tool_registry[tool_call.name].description)  # short — one sentence
span.set_attribute("tool.parameters", json.dumps(tool_call.arguments))              # scrub secrets first

# Outcome.
span.set_attribute("app.tool.status", "ok" | "error" | "timeout" | "rejected_by_guard")
span.set_attribute("app.tool.round_trip_ms", round_trip_ms)   # mod-109 chapter 01 vocabulary
span.set_attribute("app.tool.error.class", err.__class__.__name__)   # only on error
```

For agentic loops with many tool calls per turn, the tool spans are what make the trace comprehensible — the human on-call scrolls the waterfall and reads "the model retrieved, called `list_open_tickets`, called `get_ticket_body(id=42)`, then generated the summary." A trace without tool spans reads as one opaque long span and you spend twenty minutes reconstructing what happened from the LLM prompt payload.

### On every guardrail span

Any classifier or filter that gates the response is a span. This makes over-blocking and false-positive spikes debuggable in the trace, not just in the moderation dashboard.

```python
span.set_attribute("openinference.span.kind", "GUARDRAIL")
span.set_attribute("guardrail.name", "output_moderation_v2")
span.set_attribute("guardrail.decision", "allow" | "block" | "retract")
span.set_attribute("guardrail.categories", ["self_harm", "hate"])   # only categories that fired
span.set_attribute("guardrail.threshold", 0.6)
span.set_attribute("guardrail.model_version", "llama-guard-3-8b")
```

When the false-positive alert in [chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md) fires, this span is what the on-call opens to see whether the classifier or the provider layer over-blocked. Without the span, both layers are invisible from the trace UI.

## The whole thing in code

The following pattern, written once as a wrapper, gives every downstream feature the full attribute set with one call:

```python
from contextlib import contextmanager
from opentelemetry import trace

tracer = trace.get_tracer("llm.feature")

@contextmanager
def llm_span(*, feature, surface, tenant_id, user_id, model, operation="chat", prompt_id=None):
    with tracer.start_as_current_span(f"llm.{operation}") as span:
        # OTel GenAI identity.
        span.set_attribute("gen_ai.system", provider_slug(model))
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.operation.name", operation)
        # App-owned attribution.
        span.set_attribute("app.feature", feature)
        span.set_attribute("app.surface", surface)
        span.set_attribute("app.tenant_id", tenant_id)
        span.set_attribute("app.user.id_hash", sha256_hex(salt(user_id)))
        if prompt_id:
            span.set_attribute("app.prompt.id", prompt_id)
        try:
            yield span
        except Exception as exc:
            span.set_status(trace.StatusCode.ERROR, str(exc))
            span.set_attribute("app.error.class", exc.__class__.__name__)
            raise

# Usage:
with llm_span(feature="support_agent_v3", surface="chat_panel",
              tenant_id=t, user_id=u, model="gpt-4o-2024-08-06",
              prompt_id="support_answer_v14") as span:
    resp = await client.chat.completions.create(...)
    span.set_attribute("gen_ai.response.model", resp.model)
    span.set_attribute("gen_ai.response.id", resp.id)
    span.set_attribute("gen_ai.usage.input_tokens", resp.usage.prompt_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", resp.usage.completion_tokens)
    span.set_attribute("app.cost.usd", price_for(resp))
```

Two invariants make this pattern hold up under production drift:

1. **Every feature calls the wrapper, not the SDK directly.** The wrapper is the single point where new attributes get added when the semconv or the accounting requirements move. Direct SDK calls become instrumentation blind spots the moment they multiply.
2. **The wrapper never swallows an exception.** Errors are recorded on the span and re-raised. Silent trace success on user-visible failure is worse than no trace at all — the on-call opens the trace, sees green, and closes the ticket.

## Auto-instrumentation and where it stops

Both OpenTelemetry and OpenInference ship auto-instrumentation packages for the mainstream SDKs (OpenAI, Anthropic, LangChain, LlamaIndex, Bedrock, Vertex, `httpx`, `requests`). Turn them on. They give you the provider-side attributes and the base spans for free, without a per-feature integration burden.

What auto-instrumentation does **not** give you:

- `app.feature`, `app.surface`, `app.tenant_id`, `app.prompt.id`, `app.cost.usd` — these are product-side attribution the SDKs cannot know.
- The **retrieval span** for a bespoke vector store or a custom two-stage retrieval — auto-instrumentation covers popular libraries, not your in-house pipeline.
- The **tool spans** for tools you dispatch outside the LLM SDK — MCP, in-process handlers, external HTTP calls to your own services.
- The **guardrail span** for a classifier you run yourself.
- Correct **PII scrubbing** — the SDK captures the full messages array by default; that is a data-leak in most organisations unless you configure a span processor that scrubs before export (see [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md)).

The right posture is: **auto-instrumentation for the provider SDK, hand-written spans for retrieval / tool / guardrail / product-attribution**, and a span processor that decorates every auto-generated LLM span with the `app.*` attributes at export time. This is a small amount of shared infrastructure — one library, one initialiser — and pays for itself in the first month.

## Anti-patterns

- **One giant span per turn.** Retrieval, LLM call, tool calls all folded into a single "process_turn" span. When the answer is wrong, you cannot tell which sub-step failed. Every unit of work gets its own span.
- **Different attribute names in every service.** `prompt` in the chat service, `input` in the summariser, `messages` in the reranker. Rollups across services are impossible. Adopt OTel GenAI / OpenInference names *once* and enforce them via a linter or a wrapper library.
- **Raw user IDs and emails on spans.** A DSAR request three months from now becomes a trace-backend migration. Hash at the SDK boundary; store the salt separately.
- **Auto-instrumentation only.** Provider spans are green, product attribution is missing, and you cannot answer "what is p95 TTFT for the support_agent_v3 feature on tenant XYZ." Layer product attributes on every span.
- **No `gen_ai.response.model`.** Provider alias-swaps silently roll under you. The "why did the tone of the answers change overnight" incident becomes a two-day investigation. Emit both request and response model on every span.
- **Prompt / response content emitted twice** — once as an OpenInference attribute, once as an OTel event. Doubles the payload cost. Pick one carrier per surface.
- **No retrieval scores on the span.** Retrieval failures look like LLM failures. Every hit gets `document.score` at minimum.
- **A "trace everything" default.** No sampling policy, no budget, no scrubbing — the bill from the trace backend arrives at end of month, someone kills traces entirely, and you are back to no observability. Sampling and budgets are [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md); do not skip.

## Summary

- Adopt the **OTel Gen-AI semantic conventions** (`gen_ai.*`) for provider-agnostic fields any backend can filter on. Layer **OpenInference** (`openinference.span.kind`, `llm.*`, `retrieval.*`, `tool.*`) for the payload attributes a human debugger opens the trace to read.
- **One span per unit of work that can fail on its own** — outer turn, retrieval, each LLM call, each tool call, each guardrail. About ten spans per rich turn. Not one; not four thousand.
- On every LLM span: provider identity (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.response.id`), usage (`gen_ai.usage.input_tokens`, `output_tokens`, `cached_input_tokens`), app attribution (`app.feature`, `app.surface`, `app.tenant_id`, `app.user.id_hash`, `app.prompt.id`), cost (`app.cost.usd`, `app.pricing_version`), and the latency components from [mod-109 chapter 01](../mod-109-cost-latency-slos-for-llm-features/01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md).
- On every retrieval span: per-hit `document.score`, `document.id`, and truncated `document.content`. This is what makes retrieval-fail vs generation-fail distinguishable in the trace UI.
- On every tool span: `tool.name`, `tool.parameters` (secrets scrubbed), `app.tool.status`, `app.tool.round_trip_ms`. On every guardrail span: `guardrail.decision`, `guardrail.categories`, `guardrail.model_version`.
- Wrap every LLM call with a **span helper** that emits the full attribute set. Auto-instrumentation is the base layer; app-owned attribution is added on top by a span processor at export time.
- **Never emit raw user IDs, emails, secrets, or unscrubbed prompt content.** Hash and scrub at the SDK boundary — [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md).

[Chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) turns from "what to emit" to "how to keep the volume affordable and safe" — head sampling, tail sampling on error and quality signal, PII scrubbing at the export boundary, and the trace budget as a real line item.
