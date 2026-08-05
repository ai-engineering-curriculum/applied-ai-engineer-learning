# mod-109-cost-latency-slos-for-llm-features: Cost and Latency SLOs for LLM Features

**Estimated effort:** 12 hours

Every LLM feature has two silent debts the applied engineer inherits: **the latency the user actually feels** (not the number on the provider dashboard) and **the cost the feature accrues per successful outcome** (not the invoice total on the first of the month). Left unmanaged, both compound. TTFT p99 creeps up over a release cycle and nobody notices until Support hears about it. A single tenant's runaway integration doubles spend and finance asks which feature caused it and no one can answer. A "background" workload eats sync capacity for a year because "batch" was on nobody's list. The provider changes cache pricing and the estimated-vs-actual line diverges.

The module carries a single through-line: **cost and latency are product metrics with the same instrumentation discipline as error rate — measured at app altitude, attributed per feature and per tenant, and governed by written SLOs with burn-rate alerts.** Every chapter builds one piece of that substrate.

Peer-role boundaries are named up-front. `ai-infra-performance` owns the deep serving-side story (continuous batching, KV-cache paging, MLPerf-shape benchmarking). `model-evaluation-engineer` owns the quality-eval regime the SLO's `structural_validity` bit rides on. `sre` / `platform` runs the alerting substrate. The applied engineer owns the **product-facing seam**: the per-surface latency spec, the cache-shape design constraint, the batch eligibility rubric, the per-tenant accounting ledger, and the per-feature SLO with a real burn-rate alert.

## Learning objectives

- Measure app-altitude latency the way users experience it: **TTFT, TPOT, streaming p50 / p95, retry latency, tool-call round-trip latency** — with a spec for each surface.
- Land provider prompt caching (OpenAI, Anthropic, Bedrock, Gemini) as an explicit **design constraint** on the system prompt shape and versioning cadence.
- Move offline workloads onto **batch APIs** (OpenAI Batch, Anthropic Message Batches, Bedrock, Vertex) to cut cost ~50% where SLA permits — with a written eligibility rubric and a dual-run migration.
- Build a **per-feature and per-tenant cost / token accounting** layer so `cost per successful outcome` becomes a first-class product metric.
- Reason about **retry / hedging** strategies against provider tail latency; set an **SLO** with an error budget and multi-window **burn-rate alerts**; consume **MLPerf inference** benchmarks as a reference frame owned by `ai-infra-performance` and `model-evaluation-engineer`.

## Chapters

1. [App-Altitude Latency: TTFT, TPOT, Retries, Tool Calls, and the Per-Surface Spec](01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md) — the five-number vocabulary (TTFT, TPOT, streaming p50/p95, retry latency, tool-call round-trip), the instrumentation shape aligned with OTel GenAI semconv, and the per-surface latency spec as the load-bearing artefact.
2. [Prompt Caching as a Design Constraint on System Prompt Shape](02-prompt-caching-as-a-design-constraint.md) — how the four provider caches (OpenAI automatic, Anthropic `cache_control`, Bedrock `cachePoint`, Gemini `context_cache`) work; the stable-prefix / volatile-suffix rule; versioning cadence as a cache-hit-rate decision; the prompt-version doc with a cache-shape table.
3. [Batch APIs and the Offline Eligibility Rubric](03-batch-apis-and-the-offline-eligibility-rubric.md) — the four batch APIs, the three-question eligibility rubric (freshness SLA, structural fit, economics), the dual-run migration pattern, and the fall-through for late batches.
4. [Per-Feature and Per-Tenant Token and Cost Accounting](04-per-feature-and-per-tenant-token-and-cost-accounting.md) — the span attribution graph (`app.feature`, `app.tenant_id`, `app.cost.usd`, `app.pricing_version`), a version-pinned pricing table, **cost per successful outcome (CPSO)** as the headline metric, and the monthly reconciliation ritual with the provider invoice.
5. [Retries, Hedging, SLOs, and Burn-Rate Alerts](05-retries-hedging-slos-and-burn-rate-alerts.md) — retry budgets, hedged requests (Dean & Barroso), the SLI expression and error budget, multi-window multi-burn alerts (Google SRE Workbook), and MLPerf inference as a reference frame you *consume*, not author.

## Exercises

- [exercise-01 — app-altitude latency report: TTFT / TPOT](exercises/exercise-01-app-altitude-latency-report-ttft-tpot.md).
- [exercise-02 — prompt-cache-driven system-prompt design](exercises/exercise-02-prompt-cache-driven-system-prompt-design.md).
- [exercise-03 — batch eligibility rubric and migration](exercises/exercise-03-batch-eligibility-rubric-and-migration.md).
- [exercise-04 — per-tenant token and cost accounting](exercises/exercise-04-per-tenant-token-and-cost-accounting.md).
- [exercise-05 — SLO and burn-rate alert for a feature](exercises/exercise-05-slo-and-burn-rate-alert-for-a-feature.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references — provider caching docs, batch API docs, OTel GenAI semconv, SRE workbook, MLPerf.

## Cross-module context

- The **routing seam** the retries, hedges, and fallbacks in [chapter 05](05-retries-hedging-slos-and-burn-rate-alerts.md) fire against lives in **mod-108** — the SLO here drives *when* the fallback fires, mod-108 chapter 02 owns *how*.
- The **structured-output validators** the SLI's `structural_validity` bit reads live in **mod-102**.
- The **retrieved-context block** whose cache posture [chapter 02](02-prompt-caching-as-a-design-constraint.md) governs is developed in **mod-103**.
- The **tool-call round-trip** [chapter 01](01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md) times is defined by the tool-boundary work in **mod-104**.
- The **surface / agent shape** the per-surface latency spec ([chapter 01](01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md)) is written for is developed in **mod-105**.
- The **eval suite** that defines what "success" and "structural validity" mean per feature — the inputs to CPSO ([chapter 04](04-per-feature-and-per-tenant-token-and-cost-accounting.md)) and the SLI ([chapter 05](05-retries-hedging-slos-and-burn-rate-alerts.md)) — lives in **mod-106**.
- The **kill switch / degrade** paths the SLO spec names as a lever live in **mod-107 chapter 04**.
- The **observability substrate** consuming the spans this module emits — including the GenAI semconv attributes, cost fields, and burn-rate alerts — lives in **mod-110**.
- The **release / rollback discipline** that reads the SLO's burn state as a signal lives in **mod-111**.
- End-to-end ownership of a feature with a latency spec, a cache posture, a batch rubric where applicable, an accounting ledger, and a burn-rate-alerted SLO is the capstone in **mod-112**.

## Peer-role handoffs

- **`ai-infra-performance`** — owns the deep serving-side story (continuous batching, KV-cache paging, quantisation quality/throughput trade-offs) and runs the MLPerf-shape benchmarks the applied engineer *consumes* in [chapter 05](05-retries-hedging-slos-and-burn-rate-alerts.md).
- **`model-evaluation-engineer`** — owns the quality-eval regime the SLI's `structural_validity` and `outcome` fields depend on; the applied engineer's SLO measures what mod-106 defines as good.
- **`sre` / `platform`** — owns the alerting substrate the burn-rate alerts fire into and the on-call rotation that responds.
- **`privacy-engineer`** and **`data-security-engineer`** — own the residency posture that any provider swap (hedge to a second provider, fallback to a second region) must respect; the routing that respects them is in mod-108.
- **Finance / cost-owner (organisational)** — reads the CPSO and per-tenant dashboards; owns the monthly reconciliation between the ledger and the provider invoice.
- **`ai-security-engineer`** — reads the per-provider attribution to keep the data-flow / DPA story current across providers.

Naming the boundaries out loud is deliberate: cost and latency SLOs are a *shipping discipline* on purpose; the peer roles' work (serving performance, deep evaluation, SRE alerting) is the *depth* the discipline plugs into.
