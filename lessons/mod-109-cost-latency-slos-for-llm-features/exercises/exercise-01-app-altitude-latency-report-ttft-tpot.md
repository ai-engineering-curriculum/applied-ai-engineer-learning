# exercise-01: App-Altitude Latency Report — TTFT / TPOT

**Estimated effort:** 2 hours

## Objective

Instrument a real (or scaffold) LLM surface with the five-number latency vocabulary from [chapter 01](../01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md), run a short traffic sample, and produce a **per-surface latency spec** with green / yellow / red budgets that a peer can sign off on. The exercise is not "make a chart" — it is "produce the artefact that ends every future debate about whether the surface is fast enough."

## Learning outcomes

By the end you should be able to:

- Time and emit TTFT, TPOT, retry latency, and tool-call round-trip separately on every LLM call, not as a single "response time" scalar.
- Compute streaming p50 and p95 over a rolling window, per component and composite.
- Distinguish first-attempt latency from retry-inflated latency in your telemetry.
- Write a per-surface latency spec with observable budgets, excluded populations, a dashboard link, and a scheduled revisit.

## Prerequisites

- You have read [chapter 01](../01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md).
- Access to a real LLM feature on your day-job, OR the scaffold surface in the paired `applied-ai-engineer-solutions/mod-109/scaffold/` (a small streaming-chat harness that already calls an LLM provider).
- A provider API key with permission to make a few hundred low-cost calls (the exercise uses cheap models on purpose).
- Basic familiarity with your team's observability stack (OpenTelemetry + your backend of choice), OR willingness to write results to a local JSONL for the exercise.

## Problem statement

Your team runs a streaming chat surface backed by a single provider. Nobody knows what the p95 TTFT actually is because the current dashboard aggregates "response time" — total wall-clock from send to close — into one bar. Last week's incident (a provider region got slow, the surface stayed nominally green on the dashboard, users complained) was invisible because retries hid inside the average.

Your job over two hours: instrument the surface with the five-number vocabulary, run a traffic sample large enough to compute stable p50 / p95, and produce the **per-surface latency spec** that names what green / yellow / red look like on this surface.

## Requirements

### Deliverable 1 — Instrumentation

Wire the surface (or the scaffold) so every LLM call emits *one span* with, at minimum:

- `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`
- `app.latency.ttft_ms`, `app.latency.tpot_ms`, `app.latency.total_ms`
- `app.latency.retry_ms`, `app.retry.attempt_number`
- `app.tool.round_trip_ms` and `app.tool.name` if any tool call fired

TTFT is measured from "handed to transport" to "first non-empty content delta." TPOT is `(total_body_ms - ttft_ms) / n_output_tokens`. Retry latency is zero on first-attempt success. Do not collapse any of these into a single scalar.

Spans can go to your real OTel backend or to a local JSONL — the analysis step just needs a queryable log.

### Deliverable 2 — Traffic sample

Drive the surface with 200–500 requests representative of production shape:

- Mixed prompt lengths (short chat turns and longer context-heavy turns).
- At least one tool-invoking flow if the surface has tools.
- If possible, run the sample across two windows separated by a few hours to catch tail variance.

Do NOT run 10k requests against a paid API to "get better percentiles" — a few hundred is enough for the exercise and keeps cost trivial. Save the raw span log with the run.

### Deliverable 3 — The latency spec

Under `docs/latency-spec.md`, produce the artefact from [chapter 01](../01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md):

```markdown
# Latency spec — <surface> — <date>

## Surface shape
<one paragraph on what "the user is waiting" means here>

## Budgets (rolling 5-minute p95, prod)
| Metric                       | Green | Yellow | Red |
| ---------------------------- | ----- | ------ | --- |
| TTFT p95                     |       |        |     |
| TPOT p95                     |       |        |     |
| Total latency p95            |       |        |     |
| Retry latency p95            |       |        |     |
| Tool-call round-trip p95     |       |        |     |
| Tool-call round-trip p99     |       |        |     |

## Excluded from these budgets
<e.g. warm-up traffic, requests > 100k tokens>

## Where the numbers come from
<link to dashboard or JSONL; span names; retention>

## Owner and revisit
Owner: <team>. Reviewed: <date>. Next review: <quarter>.
```

Every cell in the budget table must be a real number derived from your sample plus a defensible ceiling. Justify the yellow / red thresholds in one sentence each — usually a reference to what breaks the user experience on this surface.

### Deliverable 4 — Retry-and-tool separation report

Under `docs/retry-and-tool-report.md`, a short (½-page) analysis:

- What fraction of your sample retried at least once?
- What is the p95 TTFT for first-attempt-only requests versus the p95 TTFT of the *whole* population? Show that separating them is not academic.
- For any tool-invoking flow, what fraction of the total per-turn latency was tool-call round-trip? On the p95 turn.

The purpose is to prove — in your own data — that retries and tool calls are the two biggest hidden sources of user-perceived latency.

## Starter guidance

- **Do not over-instrument.** One span per call, fixed attribute set. Resist the temptation to add sub-spans per SSE chunk.
- **Time from the client side of your gateway.** The provider's own timing is not what the user experiences.
- **Use `time.perf_counter()` (or equivalent) not `time.time()`.** Wall-clock skew across a request boundary is exactly the kind of noise that ruins p95s.
- **Compute p95 with a real percentile function**, not `sorted(xs)[int(0.95 * len(xs))]` which underestimates. `numpy.percentile(xs, 95, interpolation="linear")` or your query engine's equivalent.
- **Ground the yellow / red thresholds in the user experience of the surface**, not in "what looks nice." For a streaming chat, TTFT > 3 s is red because users start clicking away; for a background summariser it might not matter at all.

## Acceptance criteria

- [ ] Every LLM call emits one span with the fields from Deliverable 1; a sample span is included in the deliverable.
- [ ] A traffic sample of at least 200 requests is captured with the raw span log checked in (or linked from the deliverable).
- [ ] `docs/latency-spec.md` exists, contains all six rows of the budget table with real numbers and defensible thresholds, and names the excluded populations.
- [ ] `docs/retry-and-tool-report.md` shows the first-attempt vs whole-population TTFT p95 delta and the tool-call round-trip fraction of the p95 turn.
- [ ] The latency spec has an owner and a next-review date.
- [ ] The commit message names the surface and the version of the spec, not just "add latency spec."

## Stretch goals

Pick any one:

- **Two-provider comparison.** Run the same sample against a second provider (via the mod-108 gateway) and add a per-provider row to the budgets table. Which provider wins on which component?
- **Prompt-length curve.** Bucket your sample by input-token length (short / medium / long) and plot TTFT p95 per bucket. Prove or disprove "long prompts wreck TTFT" on your provider today.
- **Cache-hit vs cache-miss delta.** If the surface already uses prompt caching (chapter 02), split TTFT by cache-hit vs cache-miss. The delta is the argument for prioritising cache-shape work.
- **Tail-decomposition sketch.** For the ten slowest requests in your sample, break down the total latency into (client-side send, TTFT, decode, retry, tool round-trip). Where is the tail actually coming from?

## Deliverable

Submit as `mod-109/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/latency-spec.md` with all sections filled.
- `docs/retry-and-tool-report.md`.
- The raw span log (JSONL or link to the dashboard query).
- The instrumentation code (`instrumentation.py` or equivalent).
- A short `README.md` naming the surface and pointing at the spec.
