# mod-109-cost-latency-slos-for-llm-features: Cost and Latency SLOs for LLM Features

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 12 hours

## Learning objectives

- Measure app-altitude latency the way users experience it: TTFT, TPOT, streaming p50 / p95, retry latency, tool-call round-trip latency — with a spec for each surface
- Land provider prompt caching (OpenAI, Anthropic, Bedrock caches) as an explicit design constraint on the system prompt shape and versioning cadence
- Move offline workloads onto batch APIs (OpenAI Batch, provider batch) to cut cost 40-50% where SLA permits — with a written eligibility rubric
- Build a per-feature and per-tenant cost / token accounting layer so 'cost per successful outcome' becomes a first-class product metric
- Reason about retry / hedging strategies against provider tail latency; set an SLO budget error rate and burn-rate alert; consume MLPerf serving benchmarks as a reference frame owned by `ai-infra-performance` and `model-evaluation-engineer`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
