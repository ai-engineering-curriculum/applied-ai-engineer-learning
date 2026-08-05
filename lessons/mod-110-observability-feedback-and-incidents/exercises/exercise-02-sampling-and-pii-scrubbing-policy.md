# exercise-02: Sampling and PII Scrubbing Policy

**Estimated effort:** 2 hours

## Objective

Author a **written sampling and scrubbing policy** for the feature you instrumented in [exercise-01](exercise-01-otel-genai-instrumentation-for-a-feature.md), implement the two components (a per-feature head sampler + a tail-sampling policy in the collector; a scrubbing span processor for PII), and prove the policy honours a **written trace budget**. The exercise's artefact is the policy document plus the working pipeline it describes — not just code.

## Learning outcomes

By the end you should be able to:

- Split sampling into head (per-feature rate, hashed on trace ID, per-tenant overrides) and tail (keep-errors / keep-slow / keep-thumbs-down / baseline-5%) and configure both.
- Size a tail-sampling collector's memory footprint from `traffic_rate × decision_wait`.
- Author a scrubbing matrix mapping attribute patterns to actions (drop / hash / regex-scrub / truncate / keep) and implement it as an OTel span processor.
- Compute a per-feature monthly trace budget in $ (or GB) from `spans_per_day × avg_bytes_per_span × vendor_$_per_GB`.
- Alert on trace-budget burn rate with the same discipline as the SLO burn-rate alerts from [mod-109 chapter 05](../../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md).

## Prerequisites

- You have read [chapter 02](../02-sampling-pii-scrubbing-and-trace-budgets.md), plus [mod-107 chapter 03](../../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md) for the PII taxonomy background.
- [exercise-01](exercise-01-otel-genai-instrumentation-for-a-feature.md) is complete — you have a feature emitting the full span vocabulary.
- Access to an OTel Collector (local Docker is fine) and the trace backend from exercise-01.
- Your feature is emitting enough traffic to compute a real span/day and byte/span estimate (or you can synthesise the numbers from load-test data).

## Problem statement

Your feature currently traces 100% of traffic to the backend, with raw prompt / response payloads and un-hashed user identifiers included. Two problems:

- The trace backend bill is projected to hit $XX / month on this feature alone — an order of magnitude above what the team has budget for.
- Legal has flagged that user email addresses and (in a few cases) session tokens are showing up in trace payloads, in violation of the org's DPA with the trace vendor.

In two hours: land the sampling policy that shrinks the volume to fit the budget without losing incident signal, and land the scrubbing policy that removes PII at the export boundary. Produce the written policy document that governs both.

## Requirements

### Deliverable 1 — The policy document

Under `docs/observability/sampling-and-scrubbing.md`, one page:

```markdown
# Sampling & scrubbing policy — <feature> — <date>

## Head sampling (per-feature rate)
| Environment | Base rate | Override rules                             |
| ----------- | --------- | ------------------------------------------ |
| prod        | X%        | Force-keep tenants: [...]; Force-keep surface: [...] |
| staging     | 100%      | —                                          |

Rationale: <one paragraph — why this rate, given the traffic volume and the budget>

## Tail sampling (collector-side)
Policies (OR-composed):
- keep_errors — always keep
- keep_slow (> X ms) — always keep
- keep_thumbs_down — always keep
- keep_guardrail_block — always keep
- baseline_pct — X%

Decision wait: <N seconds — must be > p99 trace duration; document the number>
Collector memory sizing: <span rate × decision wait = spans held in memory>

## Scrubbing matrix
<the full table from chapter 02, adapted to this feature's payloads>

## Trace budget
Monthly budget: $X. Sourced from: <who signed off>
Estimated span rate: X spans/day; avg span size Y KB; monthly ≈ Z GB @ vendor rate.
Burn-rate alerts: fast (1h, 14.4×), slow (6h, 6×) on GB_ingested_today / (GB_budget_month / 30).

## Overrides
Full-tracing overrides live under `overrides/`. Each override is dated and expires automatically.

## Owner and revisit
Owner: <team>. Reviewed: <date>. Next review: <quarter>.
```

Every cell in every table must be a real, defensible number. The rationale paragraph is what a reviewer signs off on.

### Deliverable 2 — The head sampler

Under `observability/sampling/head.py` (or equivalent):

- A custom OTel `Sampler` implementing per-feature rates and per-tenant force-keep. Reads its config from the policy file (single source of truth).
- The decision is **hashed on the trace ID**, not random. Traces are all-or-nothing, never partial.
- Unit tests: for a fixed seed, verify that the sampled fraction is within ±0.5% of the configured rate over 10k trace IDs.

### Deliverable 3 — The tail-sampling collector config

Under `observability/collector/config.yaml`:

- An OTel Collector pipeline with a `tail_sampling` processor implementing all five policies (`keep_errors`, `keep_slow`, `keep_thumbs_down`, `keep_guardrail_block`, `baseline_pct`) OR-composed.
- `decision_wait` set to a specific number, justified in the policy document (must exceed p99 trace duration; if you do not know p99 trace duration for the feature, measure it first).
- An exporter to your trace backend.
- The config runs against a local Docker collector and processes at least 100 synthetic traces (a mix of errors, slow, thumbs-down, and normal); the observable outcome is that all interesting traces are exported and the baseline sample is roughly the configured percentage.

### Deliverable 4 — The scrubbing span processor

Under `observability/scrubbing/processor.py`:

- A `SpanProcessor` implementing the matrix from [chapter 02](../02-sampling-pii-scrubbing-and-trace-budgets.md), adapted to your feature's specific attribute set.
- Regex scrubbers for email, phone, SSN, credit card, and secrets (API keys, tokens, bearer credentials).
- **Tripwire attribute** (`app.scrub.dropped_raw_id`) set on any span where a raw user ID was found and dropped. Track this as a metric — the correct value is zero; any nonzero count is an upstream bug.
- Bounded truncation on payload attributes (`.content`, `.query`, `tool.parameters`).
- Unit tests: for each row of the scrub matrix, an input span and the expected post-scrub span. Include an intentional prompt with an email, phone number, and API key, and verify all three are scrubbed.

### Deliverable 5 — The trace budget & burn-rate alert

Under `docs/observability/trace-budget.md`:

- The feature's row in the budget table, with the arithmetic shown: `spans/day × avg_bytes/span × 30 = GB/month × vendor $/GB = monthly $`. Cite the vendor's public pricing.
- The two burn-rate alert rules (fast and slow) as YAML / DSL in your alerting substrate.
- A brief note (a paragraph) on the two most likely burn causes for this feature and how to detect the difference between them (usually: more traffic vs. bigger average span).

### Deliverable 6 — The proof

Under `docs/observability/proof.md`:

- Before / after storage estimate: 100% + no-scrub baseline, vs your policy applied.
- One example trace from the sampled stream (a keep_thumbs_down survivor) showing the full context intact.
- One example trace from the dropped stream (any baseline trace) showing why it was safely dropped.
- A "scrub audit" — 20 synthetic prompts containing email / phone / SSN / secrets, run through the scrubber, with the pre and post attribute values in a table.

## Starter guidance

- **Estimate before you configure.** Compute the current span-rate and average span-size from an hour of production traffic (from exercise-01's captured traces). Every sampling decision comes back to those two numbers.
- **`decision_wait` matters more than you think.** If your feature has an agentic loop that can run for 45 seconds, set `decision_wait ≥ 60s`. Otherwise the tail exporter drops the endings.
- **Head sampling is fast but blind. Tail sampling is smart but expensive.** The composition is what makes both economics and observability work.
- **The scrub matrix is a `dict` in a config file, not code branches scattered across the app.** One place to edit; every change goes through review.
- **Test the tripwire.** Force a raw user ID through the pipeline in a unit test and assert the tripwire attribute fires. If it never fires in tests, you cannot rely on it in prod.
- **Do not scrub what a debugger needs.** `gen_ai.response.id`, model names, usage counts, retrieval scores — all non-PII, all critical for debugging. Over-scrubbing is a form of blindness.

## Acceptance criteria

- [ ] `docs/observability/sampling-and-scrubbing.md` exists with every table filled and rationale paragraphs written.
- [ ] The head sampler is unit-tested and decides all-or-nothing per trace ID.
- [ ] The collector config runs locally and demonstrates all five tail policies work OR-composed against a synthetic trace set.
- [ ] The scrubbing span processor passes its unit tests, including the intentional-secret-injection test.
- [ ] `docs/observability/trace-budget.md` has the arithmetic shown and cites the vendor's pricing.
- [ ] Two burn-rate alerts (fast and slow) are configured as code in the alerting substrate.
- [ ] `docs/observability/proof.md` shows the before/after storage estimate, one kept and one dropped example trace, and the scrub audit table.
- [ ] The policy document has an owner and a next-review date.
- [ ] No sensitive fixture text (real user PII, real secrets) is committed to the repo — the audit uses synthetic data only.

## Stretch goals

Pick any one:

- **A per-tenant override.** Add a force-keep list for design-partner tenants (100% traced). Show that the head sampler honours the override and the trace backend has the tenant's full traffic while other tenants are down-sampled.
- **A quality-signal tail policy.** Add a tail-sampling policy that keeps traces where the online eval judge from [mod-106 chapter 04](../../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) marked the response as low-quality. This requires the eval score to be visible on the span at tail-sampling time.
- **A collector-side redaction proof.** Deliberately misconfigure the scrub processor (comment out the email regex) and run the traffic. The trace backend should show emails leak. Fix the config; re-run; show they no longer leak. Capture the two states as evidence — this is how you would demonstrate the scrub is load-bearing in a compliance review.
- **A DSAR extension.** Wire the trace backend into the org's DSAR pipeline (from [mod-107 chapter 03](../../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md)): a DSAR for user ID X purges all traces with `app.user.id_hash = sha256(salt + X)` within the DSAR SLA. Demonstrate the purge on a synthetic user.

## Deliverable

Submit as `mod-110/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/observability/sampling-and-scrubbing.md`, `docs/observability/trace-budget.md`, `docs/observability/proof.md`.
- `observability/sampling/`, `observability/scrubbing/`, `observability/collector/`.
- Alerting-substrate config (`alerting/`) for the two burn-rate rules.
- Unit tests for head sampler and scrubbing processor.
- A short `README.md` naming the feature, the trace budget, and how to run the pipeline locally.
