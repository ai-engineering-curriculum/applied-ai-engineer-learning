# exercise-05: SLO and Burn-Rate Alert for a Feature

**Estimated effort:** 2 hours

## Objective

Author a per-feature SLO with a written SLI expression and multi-window multi-burn-rate alerts, wire it against the span log from exercises 01 and 04, and document which retry / hedge / fallback / degrade **lever** fires at which burn rate. The deliverable is the artefact and the alert rules, not a heroic incident response — the point is to have a legible, revisitable SLO in the repo before you need it.

## Learning outcomes

By the end you should be able to:

- Write an **SLI** as a boolean over one span (latency + finish reason + structural validity).
- Set an **SLO** (target + rolling window) with a real error budget derived from your feature's traffic volume.
- Configure **burn-rate alerts** using the multi-window multi-burn pattern (fast: 14.4× over 1h, slow: 6× over 6h).
- Name which lever (retry budget, hedge, provider fallback, graceful degrade) fires at which burn level, so the on-call has a runbook, not a scramble.
- Consume MLPerf inference benchmarks as a *reference frame*, not as your SLO ceiling.

## Prerequisites

- You have read [chapter 05](../05-retries-hedging-slos-and-burn-rate-alerts.md).
- The per-surface latency spec from exercise-01 (or an equivalent) and the accounting ledger from exercise-04 (or an equivalent).
- Ability to define alert rules in your team's monitoring stack (Prometheus / Grafana, Datadog, New Relic, Honeycomb, or equivalent). If none of these are set up, you can produce the artefact as `alerts.yaml` for a fictional Prometheus target — the exercise is about the shape, not the vendor.

## Problem statement

Your team's chat feature has a p95 dashboard. It's been "green" all quarter. But you can name three incidents this quarter where users saw degraded latency for hours before the on-call was paged, because a p95 threshold alert only fires on the p95 crossing — it does not fire on the *rate at which the error budget is being consumed*, which is the number that actually decides whether you break your SLO.

Your job over two hours: convert the dashboard into an SLO, write the SLI expression, size the error budget from real traffic, define fast-burn and slow-burn alerts, and name the levers that fire when the alerts do.

## Requirements

### Deliverable 1 — SLI expression

Under `docs/slo-spec.md`, write the SLI as a boolean over one span:

```
SLI_good(span) := (
    span.app.latency.ttft_ms < <threshold from latency spec>
    AND span.gen_ai.response.finish_reason IN ("stop", "tool_calls")
    AND span.app.structural_validity = "pass"
    AND span.app.retry.attempt_number <= <max acceptable, e.g. 2>
)
```

Justify each clause in one sentence. If your feature has a specific "bad" state that the above does not cover (e.g. a moderation refusal that is legitimate and should NOT count against the SLO), name the exclusion.

### Deliverable 2 — SLO and error budget

In the same doc, name:

- The **SLO target** — usually 99% or 99.5% over 28 days. Do not pick 99.9% for an LLM feature without a strong reason; provider tail variance makes it aspirational.
- The **rolling window** — 28 days is standard; some teams use 7 days for early-lifecycle features that iterate fast.
- The **error budget** as a *quantity*: `(1 - SLO) × 28d request volume`. This is the number you actually spend.

Show your math from a real 28-day query against the span log.

### Deliverable 3 — Burn-rate alerts

Under `alerts/slo-burn.yaml` (or your monitoring stack's equivalent), define two alerts on the multi-window multi-burn pattern:

- **Fast burn** — 14.4× burn rate over 1h, holding for 5 min → page.
- **Slow burn** — 6× burn rate over 6h, holding for 15 min → ticket.

Provide the exact query (PromQL, Datadog query language, or equivalent) for each alert. Explain in a comment why 14.4 and 6 correspond to 2-day and 5-day burn-through of the budget respectively.

### Deliverable 4 — Levers doc

In `docs/slo-spec.md`, name — for this feature specifically — which lever fires at which burn:

```markdown
## Levers
- Retry: max 2 attempts, exp-backoff w/ jitter, retry budget 5% of window.
  - Fires: transient 5xx / 429 on the primary provider.
- Hedge: fire secondary to <fallback provider or region> at t = <ms> if no first token.
  - Enabled: <always | only during elevated tail | never for this surface>.
  - Cost impact: expected uplift <X%>.
- Fallback: circuit-break to <fallback model / provider> after N consecutive 5xx.
  - Fires: mod-108 chapter 02 substrate.
- Degrade: on slow burn > 6× for 6h, disable the optional tool that adds ~<X> ms to the p95.
  - Product sign-off: <name>.
```

Every lever must name what it costs (either in cost per call or in feature capability) — the point of naming levers is that on-call knows what they are trading away.

### Deliverable 5 — Runbook stub

Under `docs/runbook.md`, a short (½-page) runbook the on-call reads when the alert fires. Minimum:

- What the alert means (which SLO, which feature, which burn window).
- Where the dashboards are.
- Which lever to pull first, and how (config edit, kill-switch, DNS, whatever your surface uses).
- Who to escalate to and when.

The runbook is not a novel. It is what a groggy on-call reads at 3 AM.

### Deliverable 6 — MLPerf sanity check

In one paragraph, contrast your feature's current TPOT p95 against the closest applicable MLPerf inference reference number for the model / hardware you are using (or the closest provider-published serving benchmark if MLPerf does not cover the exact model). The point is not to *set* the SLO from MLPerf — it is to sanity-check that your production p95 is within an order of magnitude of what the model is capable of. If it is dramatically slower, that is a signal that the constraint is somewhere in your gateway / tools / retries, not in the model.

Cite the MLPerf submission or provider serving benchmark you referenced.

## Starter guidance

- **Start from the latency spec you already wrote (exercise-01).** The TTFT threshold in your SLI should match what you already called "yellow" or "red" in the spec. Do not invent a new number.
- **Do not pick 99.9%.** For a first LLM-feature SLO, 99% is honest; 99.5% is a stretch. 99.9% is only appropriate for a very mature surface with a very disciplined provider posture.
- **Multi-window is what makes the alerts useful.** A single-window burn alert either fires too often (short window, false positives) or too late (long window, budget already gone). The 1h + 6h pair is the industry pattern for a reason.
- **Every lever should already be built** — this exercise names *which one to pull*, not *how to build a hedged request path*. The hedge implementation belongs in code from chapter 05; this is the runbook.
- **The runbook is for humans**, not for auto-remediation. Write it in the second person: "You are on-call. You just got paged. Do X."

## Acceptance criteria

- [ ] `docs/slo-spec.md` contains the SLI expression, the SLO target and window, the error budget as a quantity, and the levers table.
- [ ] `alerts/slo-burn.yaml` (or equivalent) contains both fast-burn and slow-burn alert rules with real queries and threshold comments.
- [ ] The burn thresholds (14.4× / 6×) are explained in a comment tying them to 2-day / 5-day budget-through.
- [ ] `docs/runbook.md` covers alert meaning, dashboards, first lever, escalation.
- [ ] The MLPerf / provider-serving sanity check paragraph cites a specific benchmark or provider number.
- [ ] The doc names the owner and next-review date.

## Stretch goals

Pick any one:

- **Multiple SLIs, one composite SLO.** Add a second SLI for `total_latency_p95 < X` alongside the TTFT SLI and combine them into a single SLO. Reason about what that does to the error budget.
- **Alert firing rehearsal.** Synthetically inject bad requests (throttle a provider, invalidate outputs) into your dev environment enough to trigger the fast-burn alert. Confirm the paging path works before you need it.
- **Cost-aware degradation.** For the "degrade" lever, compute the expected CPSO delta (chapter 04) of turning off the optional tool. The product decision to degrade is more legible with the cost number in hand.
- **Cross-provider hedge cost model.** Compute the projected steady-state cost of enabling a hedge at your feature's current p95, given the ledger from exercise-04. Under what QPS does the hedge become uneconomical?

## Deliverable

Submit as `mod-109/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/slo-spec.md`, `docs/runbook.md`.
- `alerts/slo-burn.yaml` (or equivalent for your stack).
- The query that computed the error-budget size from a real 28-day window.
- A short `README.md` naming the feature and pointing at the SLO spec.
