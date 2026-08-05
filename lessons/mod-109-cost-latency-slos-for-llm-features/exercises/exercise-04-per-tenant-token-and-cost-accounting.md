# exercise-04: Per-Tenant Token and Cost Accounting

**Estimated effort:** 3 hours

## Objective

Ship the accounting layer from [chapter 04](../04-per-feature-and-per-tenant-token-and-cost-accounting.md): a per-call cost computation with a version-pinned pricing table, spans stamped with `app.feature` / `app.tenant_id` / `app.cost.usd`, three canonical dashboards (spend by feature, CPSO by feature, top tenants by spend), and a monthly reconciliation between your ledger and the provider invoice. The exercise turns "we have a big provider bill" into "we have a queryable per-feature, per-tenant, per-outcome ledger."

## Learning outcomes

By the end you should be able to:

- Compute per-request LLM cost from a version-pinned pricing table that handles cache-read / cache-write / batch discounts separately.
- Attribute every LLM call to a feature (from a registry) and a tenant (from an in-band ID).
- Compute **cost per successful outcome (CPSO)** for a feature over a time window from your span log.
- Reconcile your ledger's total against the provider invoice and diagnose any variance > ~5%.

## Prerequisites

- You have read [chapter 04](../04-per-feature-and-per-tenant-token-and-cost-accounting.md).
- Some LLM traffic in your dev / staging / prod environment (or the scaffold), spread across at least two features and at least a handful of tenants.
- Ability to query your span backend (SQL against a warehouse, PromQL, ClickHouse, DuckDB against a JSONL — any of these work).
- A provider invoice from at least one recent month (any provider) for the reconciliation step.

## Problem statement

Your team can answer "how much did we spend last month?" (the invoice says so) but cannot answer either of these:

1. **Which feature drove that spend?** Two features share the API key; leadership is asking to justify the summariser's launch cost; nobody has the number.
2. **Which tenants are expensive per successful outcome?** A single enterprise customer's integration is looping on retries; spend on their tenancy is 5× the trailing median; nobody knows because the field is not on the span.

Your job over three hours: ship the ledger, produce the three dashboards, and complete one monthly reconciliation.

## Requirements

### Deliverable 1 — Feature registry and enforcement

Under `code/features.py`, a checked-in registry of the features that may emit LLM cost:

```python
FEATURES = {
    "chat_summary",
    "code_completion",
    "background_enrichment",
    # ...
}
```

Add an assertion at the gateway seam: every request must set `app.feature` to a value in the registry, or the request is rejected in dev / tagged `unknown` in prod. Add a test.

### Deliverable 2 — Pricing table and cost function

Under `code/pricing.py`, a version-pinned pricing table and a pure `cost_usd(...)` function per [chapter 04](../04-per-feature-and-per-tenant-token-and-cost-accounting.md). Requirements:

- At least two providers and at least four models represented, with current prices (verify against the provider pricing page and cite the date on the table).
- Separate columns for base input, cached input read, cache-write, and output.
- Optional batch column (or a separate entry per batch model).
- Unit test that covers: no-cache case, high cache-hit case, cache-write-heavy case, batch case.

### Deliverable 3 — Span stamping

Wire the gateway (or the scaffold) so every LLM call span carries at least:

- `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cached_input_tokens`
- `app.feature`, `app.tenant_id`, `app.env`, `app.pricing_version`, `app.cost.usd`
- `app.outcome` (`success` / `user_regenerate` / `user_reject` / `error`) — even if the feature does not yet have an outcome-confirmation UX, define a placeholder logic (e.g. "success" if the response was non-empty and no error).

Include one sample span in `docs/sample-span.json` so a reviewer can see the shape.

### Deliverable 4 — Three canonical dashboards

Under `docs/dashboards.md`, wire (or sketch as SQL / PromQL and a screenshot):

- **Spend by feature** — bars per feature over the last 30 days, stacked by provider.
- **CPSO by feature** — one line per feature over 30 days, with release timestamps annotated.
- **Top tenants by spend** — table of top N tenants for the current month with delta vs previous month.

Each dashboard must have a query that runs against your real span log. Include the query.

### Deliverable 5 — CPSO for one feature

Compute **cost per successful outcome (CPSO)** for one feature over a real recent window (e.g. last 7 days). Report:

- Total spend for the window.
- Total requests.
- Total successful outcomes (`app.outcome = "success"`).
- CPSO = spend / successes.
- The "efficiency ratio" = successes / requests. What does the gap tell you?

If CPSO is materially higher than cost-per-call (say, > 1.5×), the surface is paying for retries and regenerations. Name the root cause you'd investigate next.

### Deliverable 6 — Monthly reconciliation

Reconcile last month's ledger total (sum of `app.cost.usd`, per provider) against the provider invoice. Under `docs/reconciliation.md`, report:

- Ledger total per provider.
- Invoice total per provider.
- Variance (absolute + percent).
- If variance > ~5%, at least one hypothesis for the root cause (a code path bypassing the gateway; a rate change the pricing table missed; a batch discount landing on the invoice but not the ledger; timezone effects at month boundary).

## Starter guidance

- **Register features on day one; assert on `unknown` in prod.** The first day you skip this, someone ships a new feature without registering it and the "unknown" bucket becomes half your spend.
- **Do not compute cost downstream from the provider bill.** Compute at request time from the pricing table. The provider bill is the reconciliation, not the source.
- **Hash `tenant_id` if it is PII-sensitive.** The ledger only needs an opaque stable identifier.
- **Do not average sync and batch together in the "cost per call" number** — that hides both discounts. Show effective per-request cost per surface, per pricing shape (sync / batch).
- **Keep the outcome definition simple to start.** Even a rough "success = non-empty response and no user regenerate within 30 seconds" is enough to make CPSO more honest than cost-per-call.
- **Reconciliation is a *ritual*.** Book time on the calendar; it will not happen otherwise.

## Acceptance criteria

- [ ] `code/features.py` registry exists and is enforced (unit test proves an unregistered feature is rejected in dev).
- [ ] `code/pricing.py` covers ≥ 2 providers, ≥ 4 models, with cache-read / cache-write / batch handled correctly. Unit tests pass.
- [ ] Every LLM call in the deliverable's traffic sample carries the span attributes from Deliverable 3.
- [ ] The three dashboards are wired (or sketched with runnable queries).
- [ ] `docs/reconciliation.md` shows ledger vs invoice for at least one month with variance computed and (if applicable) a root-cause hypothesis.
- [ ] CPSO for at least one feature is computed for a real window with the efficiency ratio and a named investigation lead.
- [ ] `docs/sample-span.json` shows a full span with all required fields.

## Stretch goals

Pick any one:

- **Per-tenant quota + alert.** Add a fair-use quota on daily tenant spend and an alert when any tenant crosses 80% of the quota. Prove it fires on a simulated runaway tenant.
- **Anomaly detector.** Compute the trailing 7-day median tenant spend and alert on tenants whose current-day spend is > 5× the median with no matching outcome increase.
- **CPSO regression check.** Wire a CI check that fails a PR if a change to a prompt (or model) is projected to move CPSO on a canary set by > X%. See mod-106 chapter 03 for the CI-gating substrate.
- **Provider-invoice download automation.** Automate the monthly invoice pull (where the provider exposes it via API) and produce the reconciliation report from a scheduled job.

## Deliverable

Submit as `mod-109/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `code/features.py`, `code/pricing.py`, and the tests.
- Instrumentation code stamping the span attributes.
- `docs/sample-span.json`, `docs/dashboards.md`, `docs/reconciliation.md`.
- A short `README.md` naming the surface and pointing at the ledger design.
