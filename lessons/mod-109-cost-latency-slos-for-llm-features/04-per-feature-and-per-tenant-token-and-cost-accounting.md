# Per-Feature and Per-Tenant Token and Cost Accounting

## Why cost has to become a product metric

The first sign that a team has outgrown "check the provider bill on the first of the month" is a conversation like this: finance asks *which feature drove the 40% spend spike last week*, and no one knows. There is a spend number per API key, sometimes per team, sometimes per model — never per feature, never per tenant, and never per *successful outcome*. So the answer is a spreadsheet exercise that ends in "probably the summariser, we launched a new prompt on Tuesday."

That is the failure mode this chapter fixes. On a mature applied-AI surface, **cost is a first-class product metric with the same instrumentation discipline as latency and error rate**. Every request is attributed to a feature, a tenant, and (where meaningful) an outcome; every dashboard has a per-feature spend chart next to the per-feature p95 chart; every retro on a spend anomaly can be answered in minutes, not days.

The applied engineer owns this accounting layer because they own the seam where the request is issued. Finance owns the invoices and the reconciliation with the provider (an invoice-vs-metered variance check, quarterly at least); `ai-security-engineer` owns the DPA / data-flow story per provider; but the code that stamps a request with its `feature` and `tenant_id` and computes its cost sits with the applied engineer.

## The unit of accounting: one span per LLM call

The vocabulary from chapter 01 already carried the pieces — `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, model name, provider — and this chapter adds the attribution and cost fields that turn those spans into a queryable ledger.

Every LLM call, sync or batch, emits one span. Non-negotiable attributes:

```python
span.set_attribute("gen_ai.system", "openai")               # provider
span.set_attribute("gen_ai.request.model", "gpt-4o")        # requested
span.set_attribute("gen_ai.response.model", "gpt-4o-2024-08-06")  # served (drift-visible)
span.set_attribute("gen_ai.usage.input_tokens", n_in)
span.set_attribute("gen_ai.usage.output_tokens", n_out)
span.set_attribute("gen_ai.usage.cached_input_tokens", n_cached)   # chapter 02

# Attribution — the load-bearing keys for finance-legible dashboards.
span.set_attribute("app.feature", "chat_summary")           # short slug per feature
span.set_attribute("app.tenant_id", tenant_id)              # opaque, hashed if PII-sensitive
span.set_attribute("app.env", "prod")                       # separate dev / staging / prod
span.set_attribute("app.request_id", request_id)            # for trace joins

# Cost — computed at request time, not left to a nightly job.
span.set_attribute("app.cost.usd", cost_usd)
span.set_attribute("app.cost.pricing_version", "2026-08-01") # so historical cost is reproducible

# Optional but strongly recommended when the surface has one.
span.set_attribute("app.outcome", "success")                # success | user_regenerate | user_reject | error
span.set_attribute("app.outcome_confirmed_at", ts)          # when the outcome resolved
```

Six properties make this a workable ledger:

1. **`app.feature` is a short slug, not a free-form string.** Enumerate the features in a registry (one file, checked in). If a feature is not in the registry, the request is rejected in dev and tagged `unknown` in prod. `unknown` is what fires the alert.
2. **`app.tenant_id` is stable and opaque.** Hash it upstream if you cannot legally put the raw ID in observability. What matters is that "cost per tenant" is queryable without joining against the user database at query time.
3. **Cost is computed at the request layer**, using a pricing table checked into the repo. Do **not** rely on the provider's usage endpoint to be your accounting; it lags, it lacks tenant attribution, and it is the wrong control loop for a product metric.
4. **`pricing_version` is a date-stamped identifier of the pricing table you used.** When a provider bumps prices, historical cost queries need the old table to remain valid.
5. **`app.env` splits your ledger.** Rolling dev + staging + prod into one dashboard is the first thing that hides a runaway job.
6. **`app.outcome` is the field that makes "cost per successful outcome" possible.** Chapter 04's headline metric depends on it.

## Computing cost at request time

Cost is a pure function of `(model, input_tokens, output_tokens, cached_tokens, cache_write_tokens, pricing_version)`. Write it as a function, check it in, and cover it with a unit test.

```python
# pricing.py — one file, sorted by pricing_version.
# Prices are USD per 1M tokens; verify against provider pricing pages.
PRICING = {
    "2026-08-01": {
        # openai
        "gpt-4o":              {"in": 2.50, "cached_in": 1.25, "out": 10.00},
        "gpt-4o-mini":         {"in": 0.15, "cached_in": 0.075, "out": 0.60},
        # anthropic — cache_write and cache_read priced differently from base input
        "claude-3-5-sonnet":   {"in": 3.00, "cached_in": 0.30, "cache_write_in": 3.75, "out": 15.00},
        "claude-3-5-haiku":    {"in": 0.80, "cached_in": 0.08, "cache_write_in": 1.00, "out": 4.00},
    },
    # older tables retained for historical replay
}
# needs-research: verify these numbers against the current OpenAI and Anthropic pricing pages
# at the time the module is used; the shape of the table is what matters, not the exact cents.

def cost_usd(pricing_version, model, in_tok, cached_in_tok, cache_write_in_tok, out_tok):
    p = PRICING[pricing_version][model]
    normal_in = max(in_tok - cached_in_tok - cache_write_in_tok, 0)
    c = normal_in * p["in"] / 1_000_000
    c += cached_in_tok * p.get("cached_in", p["in"]) / 1_000_000
    c += cache_write_in_tok * p.get("cache_write_in", p["in"]) / 1_000_000
    c += out_tok * p["out"] / 1_000_000
    return round(c, 6)
```

Three subtleties this shape handles that shortcuts miss:

- **Cache-read and cache-write are separate rates.** Anthropic reads at ~10% of base input; writes at ~125%. Rolling them into "input tokens" gives you a number that neither matches the invoice nor answers "is caching earning its keep." See chapter 02.
- **Discounts (batch, provisioned throughput) are their own price table.** Add a `discount_ratio` field or a distinct pricing entry per surface — `gpt-4o-batch` at half the base row is legitimate. What is not legitimate is quietly multiplying cost by 0.5 downstream and losing the audit trail.
- **Pricing table is version-pinned.** When you replay historical spans through the cost function to backfill, you use the pricing table that was in effect on the day of the request, not today's table.

## The attribution graph: feature × tenant × outcome × time

Once every span has the attribution fields, three queries become boringly easy:

- **Spend by feature over time.** `sum(app.cost.usd) group by app.feature, day`. Any launch-day spike is immediately localised.
- **Spend by tenant over time.** `sum(app.cost.usd) group by app.tenant_id, day`. This is the query that lets you spot a runaway integration or a tenant whose usage justifies a different pricing tier.
- **Spend by feature × tenant.** `sum(app.cost.usd) group by app.feature, app.tenant_id, day`. The two-dimensional cut that finance actually wants when they ask "who is expensive on which feature."

Add the outcome axis and the metric that matters most emerges: **cost per successful outcome**.

## Cost per successful outcome (CPSO)

The best single product metric for an LLM feature's health. Compute it as:

```
CPSO_{feature, day} = sum(app.cost.usd where feature = F and day = D)
                    / count(distinct request_id where feature = F and day = D and outcome = "success")
```

Two properties make CPSO more honest than raw spend or raw request count on their own:

- **Retries and regenerations are baked in.** If your users regenerate a bad answer three times before accepting one, CPSO shows you the true unit cost — 4× the naive per-request cost. Raw spend tells you "we're expensive"; CPSO tells you "we're expensive *because we're bad*."
- **Cross-feature comparison is meaningful.** Two features can look equally expensive by raw spend, but one has a 95% first-shot success rate and the other has 40%. CPSO makes the second feature stand out immediately as the improvement target.

The outcome field is where the product engineering meets the finance engineering. On a chat surface, "success" might mean "user did not regenerate within 30 seconds and did not thumbs-down." On a code inlay, "success" is "user accepted the completion (Tab)." On an agent, "success" is "the goal condition was met." **The definition of `success` per surface is a written product decision, colocated with the latency spec (chapter 01) and the eval suite (mod-106).**

CPSO drives real decisions. When a prompt release drops CPSO from $0.09 to $0.04, that is the release you scale. When a model swap raises unit cost per call by 2× but improves first-shot success rate by 3×, CPSO drops and the swap pays for itself. Neither conclusion is legible from a pricing spreadsheet.

## Per-tenant accounting and the quota story

The same attribution graph, sliced by tenant, is the substrate for four adjacent product features that finance and product owners will keep asking for:

- **Fair-use quotas.** Cap a tenant's monthly cost or monthly token usage; alert when they cross 80%. Because the accounting is per request, the quota check is cheap and can be enforced at the gateway.
- **Usage-based pricing.** If the product charges tenants by usage, the accounting layer *is* the meter. Do not build a second, parallel meter; the one that drives cost dashboards is the one that drives invoices.
- **Cost-based rate limiting.** Any single tenant driving > N% of daily spend for a period is a candidate for a targeted rate limit (or a sales conversation). The signal must be legible before any of that is possible.
- **Anomaly alerts.** Any tenant whose daily spend jumps > 5× their trailing 7-day median without a matching increase in outcomes is a candidate for an integration bug (an infinite loop, a runaway retry, a mis-configured cron). Alert on it.

The tenant field must be stamped in-band, at request time, by the gateway or the feature code — never inferred downstream from IPs or auth cookies (which lag, get wrong, or leak). If a request enters the accounting layer without a `tenant_id`, that is a defect, not a feature.

## Batch and cached tokens: showing the discount landing

Chapters 02 and 03 exist for cost reasons. The accounting layer is where you prove the savings.

Add two derived fields to the per-feature dashboard:

- **Effective per-request cost** — total spend / total requests for the feature. If you migrated a workload to batch or turned on prompt caching, this number moves down. If it does not, the migration did not land the savings you expected.
- **Cache utilisation** — `sum(cached_input_tokens) / sum(input_tokens)`. Higher is better, up to the surface's structural ceiling. Chapter 02 sets the target; the accounting layer confirms it.

The pattern in code:

```sql
-- effective per-request cost, per feature, per day
SELECT
  day,
  feature,
  SUM(cost_usd)                                                 AS total_cost,
  COUNT(*)                                                      AS n_calls,
  SUM(cost_usd) / NULLIF(COUNT(*), 0)                           AS cost_per_call,
  SUM(CASE WHEN outcome = 'success' THEN 1 ELSE 0 END)          AS n_success,
  SUM(cost_usd) / NULLIF(SUM(CASE WHEN outcome = 'success' THEN 1 ELSE 0 END), 0) AS cpso
FROM llm_spans
WHERE env = 'prod'
GROUP BY day, feature
ORDER BY day DESC, total_cost DESC;
```

That query, in the query language of the observability stack you standardised on, is the substrate for every finance conversation you will have this quarter.

## Reconciliation with the invoice

The accounting layer is a *meter*, not an *invoice*. Once a month, reconcile against the provider bill. Two questions to answer:

1. **Total spend variance.** Sum of `app.cost.usd` for the month vs the provider invoice, per provider. A well-shaped ledger lands within a small percent (a few) — the residual is time-boundary effects (requests that started before midnight UTC and were billed on the next day's invoice), rate updates you had not picked up, and honest rounding.
2. **Model-level attribution variance.** Sum by model vs provider invoice by model. Where the invoice shows a line for a model your ledger does not, that model was called by code path that did not go through the gateway. Fix the gateway seam.

If variance ever exceeds ~5%, treat it as an incident. Either the ledger is silently under-counting (someone is calling a provider without going through the gateway) or the pricing table is wrong (a rate changed and no one updated the constant). Both are real problems and neither becomes visible without the reconciliation ritual.

## Dashboards worth building once and never touching again

Three dashboards, one page each, tied to the ledger:

- **Spend by feature.** Bars per feature over the last 90 days, stacked by provider. One glance answers "which feature drives spend."
- **CPSO by feature.** Line per feature. Any release that materially improves or regresses a feature shows here first. Annotate the chart with release timestamps.
- **Top tenants by spend.** Table of top-N tenants by spend for the current month with delta vs previous month. Any anomaly is instantly visible.

Keep the dashboard set small. The failure mode of cost dashboards is *proliferation* — one per team, one per launch, one per exec — until the shared culture is that "no one knows which chart to trust." One canonical set, owned by the applied-AI engineer, is the substrate for all downstream cost conversations.

## Anti-patterns

- **Provider bill as accounting.** The bill arrives after the month closes, has no attribution to features or tenants, and does not answer any product question. It is a reconciliation input; it is not the ledger.
- **Cost as a monthly retro.** Spend spikes visible only in the invoice-review meeting are already three weeks old. Real-time per-feature dashboards catch the spike on the day of release.
- **Missing tenant IDs.** Half the requests carry `tenant_id`, half do not; the "spend by tenant" chart is meaningless. If a request enters the ledger without a `tenant_id`, that is a defect — add the assertion in dev.
- **Cost per call as the headline number.** Cost per call ignores how many calls the same successful outcome required. CPSO is the honest headline; cost per call is a supporting metric.
- **Cost table hard-coded in twelve places.** Every feature computes cost with its own copy of the price table. When prices change, five features update and seven don't. One `pricing.py` file, one function, unit tests.
- **Batch and cache discounts lost downstream.** The gateway computes cost correctly but the dashboard rolls up under a "average $/token" that averages sync and batch together; the batch savings are invisible. Show *effective per-request cost* and *cache utilisation* as first-class charts.
- **No reconciliation.** Ledger and invoice drift 20% over a quarter and nobody notices. Set up the monthly reconciliation as a recurring task on day one.

## Summary

- Cost is a first-class product metric with the same instrumentation discipline as latency. Every LLM call emits one span with `app.feature`, `app.tenant_id`, `app.env`, `app.cost.usd`, and `app.pricing_version`.
- Cost is computed at request time from a single, version-pinned pricing table checked into the repo. Cache-write, cache-read, and batch rates are separate columns — not collapsed into "input tokens."
- The headline product metric is **cost per successful outcome (CPSO)** — total spend / number of successful outcomes on that feature. It bakes in retries, regenerations, and quality; raw spend does not.
- Per-tenant accounting is the substrate for fair-use quotas, usage-based pricing, cost-based rate limits, and anomaly alerts. If `tenant_id` is not on the span, none of those work.
- The dashboards worth owning: spend by feature (stacked by provider), CPSO by feature with release annotations, top tenants by spend with month-over-month delta. Keep the set small.
- Monthly reconciliation between the ledger and the provider invoice is a ritual, not an option. Variance > ~5% is an incident.
- Chapter 02's caching wins and chapter 03's batch wins are only real if the ledger *shows* them. **Effective per-request cost** and **cache utilisation** are the charts that prove it.

[Chapter 05](05-retries-hedging-slos-and-burn-rate-alerts.md) closes the module by turning the latency, cost, and cache substrate into an SRE-shape SLO: how retries and hedging trade tail latency against cost, how to set an SLO budget and a burn-rate alert on a specific feature, and how to consume the MLPerf serving benchmarks owned by `ai-infra-performance` and `model-evaluation-engineer`.
