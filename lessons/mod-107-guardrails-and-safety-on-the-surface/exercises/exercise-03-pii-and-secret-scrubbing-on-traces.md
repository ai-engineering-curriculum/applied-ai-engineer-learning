# exercise-03: PII and Secret Scrubbing on Traces

**Estimated effort:** 2 hours

## Objective

Wire **PII and secret scrubbing** at every point where sensitive data can enter a store that outlasts the request — **inputs**, **prompts**, **outputs**, and **traces / telemetry** — and prove the pipeline holds with a labelled FP/FN eval and a "would-have-leaked" replay against real-shaped requests. The exercise is not "add Presidio to the tracer." It is "close the four leakage points as a *system*, with an eval that will catch when your scrubber's recall silently regresses."

## Learning outcomes

By the end you should be able to:

- Name the four scrubbing points and, per point, the detectors, replacement strategy, and audit obligations you have chosen.
- Compose a regex/heuristic layer (for secrets) with an ML/statistical layer (Presidio or equivalent) into a single scrubber, with sensible per-category defaults.
- Instrument a tracing SDK (OpenTelemetry span-processor, OpenLLMetry redaction hook, or a hosted-vendor's redaction API) so that scrubbing fires *before* exporters, once, for all downstream consumers.
- Produce a per-category precision / recall / false-drop report against a labelled fixture set drawn from public PII corpora plus your incident library plus synthetic augmentation.
- Extend the deletion (DSAR) pipeline to cover the trace store's DELETE path and note the model-provider-side retention windows.

## Prerequisites

- You have read [chapter 03](../03-pii-and-secret-scrubbing-across-the-stack.md).
- You have completed [exercise-02 in mod-106](../../mod-106-feature-side-evaluation/exercises/exercise-02-trace-snapshot-as-regression-fixture.md) or have equivalent experience with trace snapshotting.
- Python 3.11+ and one of the reference PII detectors installable: Microsoft Presidio, an equivalent open-source detector, or a cloud DLP API (Google Cloud SDP, AWS Comprehend PII, or Azure AI Content Safety PII).
- A tracing SDK you can instrument: OpenTelemetry with the OpenLLMetry instrumentation, or a hosted vendor (Langfuse, Arize Phoenix, LangSmith, Braintrust, Weave) whose redaction hook you can configure.
- A surface that produces traces you can point the pipeline at. Use the mod-106 scaffold if none from your day-job is appropriate.

## Problem statement

A recent incident: a customer's API key, pasted into a support conversation as "here's my key why isn't this working," is now stored in three places — the model provider's account (retained per their terms), the observability vendor's trace store (retained indefinitely per your plan), and the analytics warehouse's daily rollup (retained for 400 days by policy). None of the three has a scrubber. The incident-response cost is real.

In two hours, close the four scrubbing points for one surface, prove the pipeline works, and extend the DSAR pipeline so a future deletion request actually deletes.

## Requirements

### Deliverable 1 — The scrubbing policy

Under `docs/scrubbing-policy.md`, one page. For **each** of the four points (input, prompt, output, trace):

- The categories of data that must be scrubbed at that point (from the five in chapter 03: secrets, PII, special-category PII, business-confidential, contextually sensitive).
- The detector(s) you will use for that category at that point.
- The replacement strategy for that category at that point (redact / mask / tokenise / hash-HMAC).
- The audit posture (what is logged; who can read the logs).

The policy also names: the DSAR pipeline's coverage (trace store, provider caches, analytics warehouse), the fail-open / fail-closed policy on scrubber timeout at each point (fail-closed is the default for trace-side; other points may differ), and the review cadence.

### Deliverable 2 — The scrubber module

Under `scrub/`:

- `secrets.py` — regex-based detectors for the highest-value secret shapes (API keys with common prefixes, JWTs, private-key PEM headers, database URLs with embedded credentials, generic-high-entropy-string heuristics). Configurable via a YAML file so new prefixes can be added without a code change.
- `pii.py` — Presidio (or equivalent) integration, configured with the entity types your surface handles (`PERSON`, `EMAIL_ADDRESS`, `PHONE_NUMBER`, `CREDIT_CARD`, `US_SSN` or the equivalents for your jurisdiction, `LOCATION`, `IP_ADDRESS`).
- `combine.py` — a single `scrub(text: str, point: ScrubbingPoint) -> ScrubResult` API that composes the two layers with the policy from Deliverable 1. `ScrubResult` returns the scrubbed text plus a metadata dict `{categories_hit, spans_redacted, latency_ms, detector_versions}`. **It never returns the redacted values.**
- Per-point defaults: input-side is aggressive on secrets, contextual on PII; trace-side is aggressive on all categories.

### Deliverable 3 — The tracing SDK integration

Under `telemetry/`:

- An OpenTelemetry `SpanProcessor` (or equivalent) that runs `scrub()` on every span's LLM-relevant attributes (`gen_ai.prompt`, `gen_ai.completion`, tool arguments, tool results — whichever attribute names your SDK uses) *before* the span is exported.
- The processor sets `scrub.applied=true`, `scrub.categories_hit=...`, and `scrub.spans_redacted=N` as span attributes for observability rollups.
- Fail-closed on scrubber timeout: the span attribute is replaced with `[scrubbing_failed]` and an alert fires (in practice: increment a metric that alerting can watch).
- The processor is installed *once*, upstream of *all* exporters — the eval proof for this is that removing any exporter does not remove the scrubbing.

### Deliverable 4 — The labelled eval fixture set

Under `fixtures/scrub/`, at least **200 items**. Every item is a versioned record with:

- `id`, `text`, `expected_redactions` (list of `{start, end, category}` tuples), `provenance` (`incident-library`, `public-corpus:<name>`, `synthetic`, `benign-decoy`).

Composition:

- ≥ 20 items derived from incident-shaped situations (a secret pasted in a message, a PII-heavy support conversation).
- ≥ 80 items from at least one **public** PII corpus. Presidio ships fixtures; the CoNLL-2003 NER data is commonly used for name detection evaluation; synthesised data using the `Faker` library plugged into templated conversations counts as one source if you use another primary corpus.
- ≥ 40 **benign-decoy** items: text that *looks* like PII / secrets but should NOT be redacted. Examples: a code sample referencing a well-known dummy key like `pk_test_TYooMQauvdEDq54NiTphI7jx`, a fake credit-card number known to be invalid (`4111 1111 1111 1111`), a fake SSN pattern in a policy doc, an email address inside a URL that is clearly documentation-shaped. This is the over-redaction eval.
- ≥ 30 items in at least one non-English language, if any of your surface's traffic is non-English. Presidio's non-English recall degrades — this is exactly the sort of silent regression you want visible.

### Deliverable 5 — The FP/FN runner

Under `bin/scrub-report`:

- Loads the scrubber, the fixture set, and the current thresholds.
- Runs the scrubber against every fixture and computes per-category **precision** (of the spans redacted, how many were correct), **recall** (of the spans that should have been redacted, how many were), and **false-drop rate** (for traces flagged as too-sensitive-to-store, how many were false alarms).
- Emits `reports/YYYY-MM-DD.yaml` and `reports/YYYY-MM-DD.md`.
- Pins detector versions in every report.

### Deliverable 6 — The DSAR extension

Under `dsar/`:

- A `delete_user(user_id)` function that extends the memory-side deletion (mod-105 chapter 02) with:
  - Query for all traces attributed to `user_id` in the tracing vendor's store (via their API or via a materialised join in your own warehouse).
  - Delete the matching traces.
  - Emit a receipt including the count of deleted traces per store.
- A `docs/dsar-coverage.md` naming: which stores are covered (`trace_store_primary`, `analytics_warehouse`, `provider_cache_openai`, `provider_cache_anthropic`, etc.), which are *not* covered and why (with follow-up owner), and the published SLA.

### Deliverable 7 — The "would-have-leaked" replay

Under `proofs/would-have-leaked.md`:

- Pick 5 real-shaped requests that reached the surface (from the scaffold, or from a scrubbed incident). Show what the scrubber redacts at each of the four points. For each, name whether the redaction is correct against the policy (both under-redaction and over-redaction) and — where wrong — which detector missed / over-fired.

## Starter guidance

- **Do secrets before PII.** Secret detection is high-precision and high-value; it will catch real leaks in your first hour. PII is subtler; hold it for pass two.
- **Instrument the tracer *first*.** If the tracer captures raw text without scrubbing for even one deploy, you have shipped a leak. Start with the tracer; add the other points once the tracer's scrubber is proven.
- **Do not log redacted values, ever.** Every debug log the scrubber emits reveals what was redacted. That defeats the purpose. Log counts and categories; if a support case needs the actual value, it comes from the source system, not from the scrubber's log.
- **Watch the benign-decoy set.** The most common shipped bug is over-redaction. A code review of a scrubber that has never been tested against benign-decoy fixtures will approve a scrubber that eats every legitimate mention of the word "password" in your docs.
- **Presidio ships defaults; they are not tuned for your surface.** Tune the recogniser confidence thresholds on your own data, and document the tuning.
- **The DSAR pipeline is the honest test.** If you can point at a trace containing a user's data and delete it in under a minute, your pipeline works. If not, the pipeline exists on paper only.

## Acceptance criteria

- [ ] `docs/scrubbing-policy.md` names, per point (input, prompt, output, trace), the categories, detectors, replacement strategies, and audit posture.
- [ ] `scrub/` provides a single `scrub(text, point)` API that composes secrets and PII layers per the policy.
- [ ] `telemetry/` installs the scrubber into the tracing SDK as a span processor (or equivalent redaction hook) that fires before all exporters.
- [ ] Fail-closed on scrubber timeout at trace-side — proven by a test.
- [ ] `fixtures/scrub/` contains ≥ 200 items with the required composition, including ≥ 40 benign-decoy fixtures.
- [ ] `bin/scrub-report` produces per-category precision, recall, and false-drop rate; pins detector versions.
- [ ] `dsar/delete_user` extends the memory-side deletion to cover the trace store; the receipt names counts per store.
- [ ] `docs/dsar-coverage.md` names covered / uncovered stores with an owner for each uncovered one.
- [ ] `proofs/would-have-leaked.md` shows 5 real-shaped requests and the correct classification of the scrubber's decision on each.
- [ ] No log emitted by the scrubber contains a redacted value.

## Stretch goals

Pick any two:

- **Cross-lingual scrubbing.** Add a non-English language to the fixture set (or expand the existing coverage). Report per-language recall. Add a language-specific model or ruleset if the default degrades unacceptably.
- **Tokenised trace fixtures for offline eval.** Wire this scrubber into the trace-snapshot pipeline from [mod-106 chapter 02](../../mod-106-feature-side-evaluation/02-trace-snapshots-as-regression-fixtures.md) so trace snapshots are usable as eval fixtures without a human review pass. Prove the tokenisation is stable within a trace.
- **Provider-side retention audit.** For each model provider you use, document the current retention windows and DSAR-adjacent contract (Anthropic's zero-retention endpoints, OpenAI's Zero Data Retention posture, Gemini's data-usage terms). Note where your surface is or is not covered by each posture. This is a written artefact, not code — but it is the artefact incident responders will actually read.
- **Coverage as an SLO.** Emit a metric `scrub.coverage_ratio = scrubbed_spans / total_spans`. Alert when the ratio drops below a threshold — this is what catches "someone shipped a new instrumentation path that bypasses the scrubber."
- **Format-shift bug hunt.** Craft fixtures where scrubbed content is round-tripped through a downstream template (Deliverable 3 talks about this indirectly). E.g. a scrubbed prompt is fed to a model that reproduces the PII in a rephrased output; the output-side scrubber should catch it. Prove it does, or file the finding.

## Deliverable

Submit as `mod-107/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/scrubbing-policy.md`, `docs/dsar-coverage.md`.
- `scrub/` with the composed scrubber module.
- `telemetry/` with the SDK integration.
- `fixtures/scrub/` with the labelled set and a `PROVENANCE.md`.
- `bin/scrub-report` and `reports/` with at least one baseline report.
- `dsar/` with the extended deletion function and receipt shape.
- `proofs/would-have-leaked.md`.
- A short `README.md` naming the surface, the tracing vendor, and how to run the report.
- Any stretch-goal artefacts named alongside.
