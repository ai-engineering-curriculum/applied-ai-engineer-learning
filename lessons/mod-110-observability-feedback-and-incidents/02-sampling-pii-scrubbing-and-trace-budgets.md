# Sampling, PII Scrubbing, and the Trace Budget

## Why sampling and scrubbing land in the same chapter

Trace every LLM call, verbatim, forever, and two bills arrive. The first is from the observability vendor: gigabytes of prompt / response / retrieval payload per day at their per-GB-ingested rate. The second is quieter, and lands as a Slack from Legal: the prompt content stream includes user emails, session tokens, and — in the RAG flow — customer document snippets the vendor's DPA does not cover for this use.

Both bills come from the same choice — "keep everything, worry later" — and they have the same fix: the export pipeline is a **policy boundary**, not a pass-through. Sampling decides *which* spans leave your process. Scrubbing decides *what* fields go with them. Both run in the same span-processor stage; both are configured together; both are governed by a written policy the on-call can point at.

This chapter names three artefacts:

- The **sampling policy** — head sampling on the ingress, tail sampling on error / quality signal, per-feature overrides.
- The **PII scrub matrix** — which fields to redact, hash, truncate, or drop, keyed to the categories from [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md).
- The **trace budget** — a monthly dollar (or GB) ceiling per feature, tracked as a first-class line item alongside the model spend from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).

## Head vs tail sampling: pick both

The two OpenTelemetry sampling positions are complementary, not alternatives.

### Head sampling — decide at span-start

Head sampling makes the keep/drop decision when the root span is created, before any of the request's work has happened. The decision is cheap (one hash), it propagates down the trace via the OTel context (either all children are kept or all are dropped, no partial traces), and it can be varied per attribute — per feature, per tenant, per surface.

The canonical head-sampler shape is a **parent-based sampler with a per-feature rate**:

```python
# Pseudo-code — actual OTel Python API is a Sampler subclass, but the shape is this.
DEFAULT_RATE = 0.05             # 5% of general traffic
FEATURE_RATES = {
    "support_agent_v3":   0.10, # noisier feature, sample more
    "batch_summariser":   0.01, # high volume, low value per trace
    "voice_realtime":     0.20, # high user-impact, small traffic — sample generously
}
FORCE_KEEP_TENANTS = {"design_partner_1", "design_partner_2"}  # early-adopter tenants — 100%

def should_sample(feature, tenant_id, trace_id):
    if tenant_id in FORCE_KEEP_TENANTS:
        return True
    rate = FEATURE_RATES.get(feature, DEFAULT_RATE)
    return (trace_id.hash() % 10_000) < int(rate * 10_000)
```

Two things to notice. First, sampling is **hashed on the trace ID**, not random per span. That way a given user's session is either fully sampled or fully dropped — either you have the whole trace or none of it. Random per-span sampling gives you partial traces which are worse than no traces because they cannot be debugged. Second, the rate is per-feature. A batch summariser producing a million calls per day and a voice surface producing a thousand calls per day should not use the same rate; you would either drown in the summariser's spans or never sample the voice surface.

Head sampling is right for **the base rate on all traffic** — the 5% (or 10% or 20%) you keep because you want a statistically representative slice of production. It is *wrong* as the only mechanism, because the interesting spans — the errors, the ones with a thumbs-down, the ones where the latency exceeded the SLO — are exactly the ones head sampling drops 95% of the time.

### Tail sampling — decide after the trace completes

Tail sampling runs in a separate OTel collector process. Every span from every trace flows into the collector, is held in memory for a short window (typically 5–30 seconds — the time the trace needs to complete), and then the collector's policy engine decides whether to export or drop. Because the decision is made after all spans in the trace are visible, tail sampling can key on outcome: keep on error, on high latency, on a specific attribute value, on a downstream quality signal.

The recommended per-feature tail policy for an LLM feature:

```yaml
# Tail-sampling processor config (OTel collector)
tail_sampling:
  decision_wait: 30s
  num_traces: 100000
  policies:
    # 1. Errors — always keep.
    - name: keep_errors
      type: status_code
      status_code: { status_codes: [ERROR] }

    # 2. High latency — anything over the SLO threshold, always keep.
    - name: keep_slow
      type: latency
      latency: { threshold_ms: 5000 }

    # 3. Thumbs-down or explicit user complaint — always keep.
    - name: keep_thumbs_down
      type: string_attribute
      string_attribute: { key: app.feedback.signal, values: ["thumbs_down", "reported"] }

    # 4. Guardrail block — always keep (the FP/FN report from mod-107 depends on this).
    - name: keep_guardrail_block
      type: string_attribute
      string_attribute: { key: guardrail.decision, values: ["block", "retract"] }

    # 5. Retries — always keep the first retry, sample subsequent ones.
    - name: keep_retries
      type: numeric_attribute
      numeric_attribute: { key: app.retry.attempt_number, min_value: 2 }

    # 6. Long-tail baseline — keep 5% of everything else so trends stay measurable.
    - name: baseline_5pct
      type: probabilistic
      probabilistic: { sampling_percentage: 5 }
```

The policy is `OR`-composed — any policy that says "keep" wins. In practice this means every error, every slow request, every thumbs-down, and every guardrail block reaches the backend, plus a 5% sample of everything else. The bill drops by an order of magnitude; the ability to debug does not.

Two edges to name.

**`decision_wait` must be longer than the longest trace.** If your agentic loop can run for 45 seconds, and `decision_wait` is 30 seconds, half the trace has already been export-decided by the time the last span arrives. The collector will fall back to a default (usually "drop"), and you will lose the tail of every long trace — which is where the interesting stuff lives. Set `decision_wait` to `p99_trace_duration × 1.5`, at minimum.

**Tail sampling costs memory in the collector**, proportional to `traffic_rate × decision_wait`. A million spans per minute × 30 seconds ≈ 500k spans held in memory at any time. Size the collector fleet accordingly; head sampling has zero memory footprint by comparison.

### The composition: head sample the volume, tail sample the tail

The two together are a two-stage funnel. Head sampling drops obvious high-volume, low-value traffic before it hits the collector — the collector is not asked to hold 100% of the batch summariser's spans in memory just to drop 95% of them. Tail sampling then upgrades the surviving traffic on any interesting signal. A typical composition:

- Head: keep 100% of interactive-surface traffic; keep 5% of batch traffic; force-keep design-partner tenants at 100%.
- Tail (in the collector, on the survivors): keep all errors, all slow requests, all thumbs-down, all guardrail blocks, plus 5% of the rest as baseline.

The observable result: on a healthy day you keep ~5-10% of the spans; on a bad day (an incident, a spike in errors) you keep near 100% and the on-call has every trace they need. **Sampling is elastic to the incident**, not to the calendar.

## PII scrubbing at the export boundary

The right place to scrub PII is a **span processor** that runs immediately before the exporter — after the SDK has captured the attributes, before they leave the process. Scrubbing in the app code is unreliable (every new feature has to remember); scrubbing in the vendor's ingest is too late (the data has already flowed across the DPA boundary).

OTel gives you the exact hook: implement `SpanProcessor.on_end`, receive the span, mutate its attributes, hand it to the next processor. The scrub logic is a table lookup — attribute name in, action out.

### The scrub matrix

| Attribute pattern                                | Action                              | Why                                                                                                    |
| ------------------------------------------------ | ----------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `app.user.id`, `user.email`, `user.name`         | Drop (should never be set — hash upstream) | Raw user identifiers must be hashed at the SDK boundary. If one arrives, it is a bug — drop and alert. |
| `app.user.id_hash`, `app.session.id_hash`        | Keep                                | Already hashed — safe.                                                                                 |
| `llm.input_messages.*.content`, `retrieval.query`, `retrieval.documents.*.content` | Regex-scrub (email, phone, SSN, credit-card, API keys) then truncate | Payload fields are the largest PII surface. Scrub known patterns, truncate to a bounded size.          |
| `tool.parameters`                                | JSON walk + secret scrub            | Tool arguments frequently contain secrets (API tokens, DB URIs, S3 paths with signed URLs).            |
| `gen_ai.response.id`, `gen_ai.request.model`, `gen_ai.usage.*` | Keep                                | No PII; needed for support and rollups.                                                                |
| `app.tenant_id`                                  | Keep (or hash if tenant IDs are sensitive) | Tenant IDs are usually opaque; if yours are customer names, hash them.                                 |
| `http.request.header.authorization`, `http.request.header.cookie` | Drop                                | Never let auth headers reach the trace backend.                                                        |
| Anything not in the allow-list                   | Log a warning; drop or keep per default | Unknown attributes are new fields the scrub matrix has not seen yet. Fail-safe defaults matter.        |

The matrix is a policy file the observability team owns. Feature teams do not edit it ad-hoc; changes go through a review that ends with `docs/observability/scrub-policy.md` updated and the processor's version bumped.

### The processor shape

```python
class ScrubbingSpanProcessor(SpanProcessor):
    """Mutates span attributes according to the scrub matrix before export."""

    SECRETS = re.compile(r"(?i)(api[_-]?key|token|secret|password)")
    EMAIL = re.compile(r"[\w.\-+]+@[\w\-]+\.[\w\-.]+")
    PHONE = re.compile(r"\+?\d[\d\-\s()]{7,}\d")
    SSN = re.compile(r"\b\d{3}-\d{2}-\d{4}\b")
    CREDIT_CARD = re.compile(r"\b(?:\d[ -]*?){13,16}\b")
    RAW_ID_KEYS = {"user.id", "user.email", "user.name", "app.user.id"}

    def on_end(self, span):
        attrs = dict(span.attributes)
        for key in list(attrs):
            if key in self.RAW_ID_KEYS:
                del attrs[key]
                span.set_attribute("app.scrub.dropped_raw_id", True)  # tripwire
            elif key.endswith(".content") or key.endswith(".query") or key == "tool.parameters":
                attrs[key] = self._scrub_text(attrs[key])
            elif key.startswith("http.request.header."):
                if key.endswith("authorization") or key.endswith("cookie"):
                    del attrs[key]
        self._replace_attributes(span, attrs)

    def _scrub_text(self, text):
        if not isinstance(text, str):
            return text
        text = self.EMAIL.sub("[email]", text)
        text = self.PHONE.sub("[phone]", text)
        text = self.SSN.sub("[ssn]", text)
        text = self.CREDIT_CARD.sub("[cc]", text)
        text = self.SECRETS.sub(r"\1=[redacted]", text)
        return text[:2048]  # hard bound — matches the retrieval-doc truncation from chapter 01
```

Three things this processor gets right and are often gotten wrong:

1. **The tripwire attribute (`app.scrub.dropped_raw_id`).** When the processor sees a raw user ID it should not have, it drops it *and* leaves a breadcrumb. The observability team monitors the count of tripwire spans; when it goes above zero, a feature is emitting unsafe data and the team pages the owner.
2. **Bounded truncation.** After scrubbing, every text field is capped at a known length. The trace UI is not designed to render 50KB payloads and the storage cost is real.
3. **Never scrub what a debugger needs.** The `gen_ai.response.id`, model names, usage counts, and per-hit retrieval scores are non-PII and stay untouched. Over-scrubbing is not safety; it is a debug-time regression.

### What scrubbing does not do

- It does not remove PII from the model provider's own log stream. The provider has already seen the prompt. The scrub happens on *your* observability leg only; the model provider's data handling is governed by the DPA and the API's `X-Anthropic-Store` / no-log / "disable prompt storage" toggles at that provider.
- It does not turn free-text payloads into safe payloads for arbitrary use. Regex scrubbers catch known formats and miss adversarial or context-specific PII (a name written in unusual formatting, a customer's address given as prose). The scrub is defence-in-depth against accidental leakage, not a compliance boundary.
- It does not exempt the trace backend from your DPA. Even scrubbed payloads flow through the vendor; the vendor is still a data-processor. The scrub reduces the surface, it does not eliminate the relationship.

For depth on the PII taxonomy, detector matrix, and DSAR pipeline extension, see [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md). That chapter owns the substrate; this chapter owns the specific application to the trace-export leg.

## The trace budget — a first-class line item

The observability vendor's bill is a real number on the P&L. At applied-AI-product scale — millions of turns per day, large prompt / response payloads, retrieval documents on every span — trace ingest can easily be $50k–$500k per month per feature if left unbudgeted. It is the same conversation as the model bill from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md), with the same shape: **name the budget, attribute the spend per feature, alert on burn.**

### The budget artefact

Per feature, one row in `docs/observability/trace-budget.md`:

| Feature             | Sampled rate | Est. spans/day | Est. GB/day | Monthly $ budget | Owner        |
| ------------------- | ------------ | -------------- | ----------- | ---------------- | ------------ |
| `support_agent_v3`  | 10% + tail   | 500k           | 4.5         | $2,700           | @agent-team  |
| `batch_summariser`  | 1% + tail    | 200k           | 1.8         | $1,100           | @batch-team  |
| `voice_realtime`    | 20% + tail   | 40k            | 0.5         | $300             | @voice-team  |

The estimated GB/day comes from `spans_kept_per_day × avg_bytes_per_span`. The avg bytes per span is a real number you can pull from the backend (spans of the shape from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) typically run 4–20 KB depending on retrieval / tool payload). The monthly dollar budget comes from `GB/day × 30 × vendor_$/GB`.

### Attributing the cost per feature

The trace backend needs to be queryable per feature so the bill can be attributed. That means every span carries `app.feature` (from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md)) *and* the backend supports per-attribute cost aggregation (most do, some do not — check before adopting). If the backend cannot attribute by attribute, use per-feature exporters (one collector pipeline per feature) so the vendor's bill line-items are already split.

### Burn-rate alerts on the trace bill

The same burn-rate mechanics from [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md) apply here — a fast burn (1-hour window) and a slow burn (6-hour or 24-hour window) on `GB_ingested_today / (GB_budget_month / 30)`. When a feature ships a change that suddenly emits 5× the spans, the alert fires before month-end, not after.

The most common burn cause is not "more traffic" — it is **a feature added a large new attribute** (an unbounded retrieval-document field, an unfiltered tool-parameters payload) and the average span size jumped. The alert catches this because the GB rate diverges from the span-count rate; watch both.

### The per-feature override

Individual features occasionally need to trace at 100% for a short window — a soak test, an investigation, a design-partner pilot. The override mechanism should be:

- A named, dated toggle (`overrides/support_agent_v3_full_trace_2026-03-18_to_2026-03-25.yaml`), not an environment variable someone flipped and forgot.
- An automatic expiry — the toggle drops back to the default rate on the end date without a human step.
- An owner and a reason recorded on the toggle. When the trace bill spikes, the on-call can grep the overrides directory and know why.

Full-tracing forever without an expiry is how a trace budget silently doubles.

## Sampling and eval — the second use of the sampled stream

The sampled trace stream is not just a debugging tool. It is also the intake for the **sampled online eval loop** in [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md). The eval loop pulls a subset of the sampled spans, runs a judge (an LLM-as-judge or a rule-based checker), and stores the judgment as a feedback signal.

For the eval loop to work, the sampled stream must:

- Be **representative** of the traffic. Head-sampled traffic is representative by construction (uniform on trace ID); tail-sampled traffic is *not* (it is skewed toward errors and slow requests). Feed the eval loop from the head-sampled slice, not the composed stream, or the online quality numbers will be dominated by failures.
- Retain the payload (`llm.input_messages.*`, `llm.output_messages.*`, `retrieval.documents.*.content`). Aggressive scrubbing that truncates too hard makes the eval un-runnable — the judge needs to see the answer to judge it.
- Be **filterable per feature and per tenant**. The eval loop scopes per feature; if the sampled stream cannot be filtered per feature, the eval loop cannot either.

The relationship: **sampling in this chapter is the input to the eval loop in mod-106 chapter 04**, which is the input to the drift monitor in that same chapter. The whole stack collapses if the sampled stream is not representative.

## Anti-patterns

- **Sampling as an on/off switch.** "We trace everything" vs "we trace nothing" — both are wrong. The bill argues for the first, the on-call for the second, and the compromise is a written policy the on-call and the CFO can both live with.
- **Random per-span sampling.** Half of each trace is missing. The trace UI shows an incoherent scatter of spans. Sampling must be per-trace, hashed on the trace ID.
- **Only head sampling, no tail sampling.** 95% of the errors are dropped because they landed inside the 95% you sampled out. The on-call has no data on the incidents they most need to see.
- **Tail sampling without a wait window long enough for the trace to complete.** Long-running agentic traces get their heads exported and their tails dropped. The trace UI looks incomplete; the debugger cannot tell whether the trace ended or was truncated.
- **Scrubbing in the app code, not the export pipeline.** Every new feature has to remember. It won't. The next PII incident is one un-scrubbed feature away.
- **Over-scrubbing.** Every payload replaced with `[redacted]`. Traces are unreadable, debugging is impossible, the observability layer becomes decorative. Bound the scrub matrix to known-sensitive patterns; leave the rest.
- **No trace budget at all.** The vendor's bill arrives, someone kills traces for the month to catch up, and observability regresses to "grep the app logs." The budget is the discipline; the burn-rate alert is the enforcement.
- **`sampling_rate = 0.01` because someone read a blog post.** The rate is set per-feature, from `expected_spans_per_day × avg_span_bytes × vendor_$_per_GB` against the budget row. Not from folklore.

## Summary

- **Head sampling** decides at trace-start (hashed on trace ID, per-feature rate, per-tenant overrides). **Tail sampling** decides in a collector after the trace completes (always keep errors, slow requests, thumbs-down, guardrail blocks; plus a 5% baseline). Both together are the two-stage funnel: head drops the boring volume, tail upgrades the interesting tail.
- Tail sampling's `decision_wait` must exceed the p99 trace duration or long traces get truncated. Size the collector for `traffic_rate × decision_wait` spans in memory.
- **PII scrubbing runs as a span processor at export**, not in feature code. The scrub matrix is a policy file: hash raw user identity (or drop if it slipped through with a tripwire), regex-scrub email / phone / SSN / credit-card / secrets in payload attributes, drop auth headers, keep everything else needed for debugging.
- The **trace budget** is a per-feature line item with a monthly $ or GB ceiling, owned by the feature team, alerted on with the same burn-rate mechanics as the model spend from [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md). Full-tracing overrides are dated, named, and auto-expire.
- The head-sampled stream feeds the **online eval loop** in [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md). Sampling policy is not just a cost lever; it is the eval loop's data source.

[Chapter 03](03-user-feedback-triage-pipeline.md) turns from the *system's* signal (spans) to the *user's* signal — thumbs, structured forms, free text — and how to dedupe, triage, and route it into an eval dataset an expert can review.
