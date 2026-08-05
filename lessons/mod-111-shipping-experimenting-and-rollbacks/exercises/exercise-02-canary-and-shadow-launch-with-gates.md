# exercise-02: Canary and Shadow Launch with Pre-Registered Gates

**Estimated effort:** 3 hours

## Objective

Take the flag layer from exercise-01 and layer on a **pre-registered launch plan**: a shadow launch of one prompt version and a canary rollout of another, each governed by a written plan doc with the four gate families (quality / safety / cost / latency) frozen before the flag flip. The exercise is not "flip a flag then check dashboards." It is "produce the launch plan, the per-stage promotion artefact, and the auto-rollback rule that make the decision reproducible next quarter."

## Learning outcomes

By the end you should be able to:

- Distinguish shadow vs canary and pick the right one for a given change size.
- Draft a **launch plan doc** with per-gate thresholds, sample-size math, and an explicit auto-rollback rule — before the flag flips.
- Wire the shadow variant off the critical path, tagged with `app.shadow.*` attributes so downstream analytics do not double-count.
- Compute the sample size required to detect the smallest Δ any gate cares about, using the rate-metric heuristic `n ≈ 16 × p × (1 - p) / Δ²`.
- Produce a **promotion-decision artefact** per canary stage that a colleague can read six weeks later and reconstruct the launch.
- Wire an auto-rollback on hard-gate breach using the burn-rate alerting substrate from [mod-109 chapter 05](../../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md).

## Prerequisites

- You have read [chapter 02](../02-canary-and-shadow-launches-with-pre-registered-gates.md).
- You have completed [exercise-01](exercise-01-feature-flag-driven-prompt-rollout.md) or an equivalent flag substrate is in place.
- Access to a burn-rate-alerting stack (Prometheus + Alertmanager, Datadog, Honeycomb, Grafana, or equivalent). For this exercise, `alerts.yaml` for a Prometheus target is fine — the shape matters, not the vendor.
- The feature has offline-eval and judge substrates from [mod-106](../../mod-106-feature-side-evaluation/README.md), or a proxy for each — you cannot gate on quality without a quality signal.
- The feature emits CPSO and per-surface latency spec numbers from [mod-109 chapter 04](../../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) and [chapter 01](../../mod-109-cost-latency-slos-for-llm-features/01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md).

## Problem statement

Product wants to ship `support_answer_v15_concise` (the "concise" prompt from exercise-01) to general availability within two weeks. The last time the team shipped a prompt without a gated rollout, they promoted to 100% in an afternoon and later discovered the "concise" style dropped the answer-usefulness score for tenants on the enterprise tier by 6 percentage points — a regression that hit a segment that only represents 10% of traffic and therefore was invisible in the aggregate dashboards.

Over three hours: (1) shadow `v15_concise` on 10% of turns to catch the aggregate regression cheaply, (2) if shadow passes, canary from 1% → 5% → 25% → 100% with pre-registered gates and per-stage promotion artefacts, and (3) wire the auto-rollback on hard gates so a mid-canary regression trips the flag flip without a human.

## Requirements

### Deliverable 1 — The launch plan (frozen before flag flip)

Under `docs/launch-plan.md`, a written plan with:

- **Header**: variant IDs (`v14` = control, `v15_concise` = candidate), launch owner, launch tracking issue link, plan freeze timestamp.
- **Sequencing**: shadow at 10% for 3 days → canary 1% for 24h → 5% for 72h → 25% for 1 week → 100%. Adjust the timings to your feature's traffic; document your reasoning.
- **The four gate families** — one section each — with thresholds. At minimum:
  - **Quality**: offline eval composite ≥ v14 – 2; per-scenario floor ≥ v14 – 5; shadow judge win-rate ≥ 55%; user-feedback thumbs-down delta ≤ +0.5 pp.
  - **Safety**: refusal rate delta ≤ +1.0 pp; moderation FP delta ≤ +0.5 pp; injection-eval failures = 0 new.
  - **Cost**: CPSO delta ≤ +10%; absolute daily spend on canary cohort ≤ $X (a real number you can defend).
  - **Latency**: TTFT p95 delta ≤ +150 ms; total p95 delta ≤ +300 ms; retry rate delta ≤ +0.5 pp.
- **Sample-size math**: for each gate, name the smallest Δ that matters and compute the required n using `n ≈ 16 × p × (1 - p) / Δ²`. Confirm the observation windows in the sequencing section deliver that n at your feature's traffic rate.
- **Auto-rollback rule**: which gates trigger an automatic flag flip (hard gates), which trigger a page-only (soft gates), and the exact burn-rate threshold and window for each.
- **Rollback protocol**: which flag flips, to what value, by whom (the on-call), and by when (the TTE budget).

Freeze the doc — commit it, tag it, do not edit during the observation windows.

### Deliverable 2 — Shadow launch wiring

Modify the feature so that:

- When the flag `support_agent.shadow_variant` is set (a string) and `support_agent.shadow_enabled` is `True`, every N-th turn (where N is the inverse of the shadow rate — e.g., N=10 for 10% shadow) also fires the shadow variant *asynchronously off the request path*.
- The shadow's LLM call emits its own trace span with `app.shadow.of = <control variant>`, `app.shadow.variant = <shadow variant>`, and the same `app.feature`, `app.tenant_id`, and turn-correlation ID as the primary call.
- The shadow span is tagged `app.cost.usd` with the shadow cost, under a *distinct* `app.feature` value (`support_agent.shadow`) so the CPSO ledger does not conflate.
- The async executor has a **bounded queue** — on overflow, shadow calls are dropped, never queued indefinitely, and a counter increments.
- The user *never* waits for the shadow. If the shadow errors, the user response is unaffected.

Add a test that asserts the shadow does not block the primary path even when the shadow provider is stubbed with a 10-second sleep.

### Deliverable 3 — Shadow comparison analysis

After 3 days of shadow traffic (or however long fits your rate limits — reduce to 1 day for a low-traffic feature), produce `docs/shadow-report.md`:

- Sample size: N shadowed turns, M control turns paired to them.
- **Aggregate quality delta**: LLM-judge win-rate of shadow vs control (from a mod-106 judge on the paired turns).
- **Per-scenario quality delta**: same, broken down by the mod-106 scenario taxonomy — this is the deliverable that would have caught the enterprise-tier regression in the problem statement.
- **Cost delta**: shadow CPSO – control CPSO.
- **Latency delta**: not applicable for shadow (shadow is async), but note the shadow's own p95 for eventual canary sizing.
- **Decision**: does the shadow pass the pre-registered gates? Answer yes/no and cite the specific gate → observed value.

### Deliverable 4 — Canary rollout with per-stage promotion artefacts

Assuming shadow passed, proceed with the canary at the sequencing you documented. For each of the four stages (1%, 5%, 25%, 100%), produce a promotion-decision block in `docs/canary-log.md`:

```markdown
## Canary Promotion — v15_concise at <from>% → <to>%
Observation window: <start UTC> → <end UTC> (<duration>)
Samples: canary <N> turns; control <M> turns.

Gate                        Threshold           Observed    Status
Offline eval (composite)    ≥ v14 - 2 pts       ...         PASS/FAIL
Offline eval (worst scene)  ≥ v14 - 5 pts       ...
Thumbs-down rate delta      ≤ +0.5 pp           ...
Refusal rate delta          ≤ +1.0 pp           ...
Moderation FP delta         ≤ +0.5 pp           ...
CPSO delta                  ≤ +10%              ...
Daily spend on canary       ≤ $X                ...
TTFT p95 delta              ≤ +150 ms           ...
Retry rate delta            ≤ +0.5 pp           ...

Decision: PROMOTE / HOLD / ROLLBACK. Signed off by: <name>.
```

At least four blocks (one per stage). If any gate fails, either roll back or hold; do not silently move the number.

For this exercise, you may drive synthetic traffic to hit the sample-size targets — the point is the *artefact and the process*, not organic user traffic.

### Deliverable 5 — Auto-rollback rule

Under `alerts/canary-burn.yaml` (or your stack's equivalent):

- One **fast-burn alert** on each hard-gate metric (e.g., moderation FP delta > +2.0 pp over 5 min → auto-rollback). The alert action calls the flag-provider's API to flip `support_agent.prompt_id` back to `v14`.
- One **slow-burn alert** on each soft-gate metric (e.g., CPSO delta > +15% over 30 min → page the on-call).
- Comments in the YAML explaining the burn-rate thresholds and the choice of hard vs soft.

Then **exercise the rollback** at least once by synthetically forcing a metric over threshold (e.g., inject 1000 refusal responses into the canary cohort's log). Record in `docs/rollback-drill.md`:

- The time from injection to alert firing.
- The time from alert firing to flag flip landing (TTE).
- The observed rollback: how many spans post-flip carry `v14` vs `v15_concise`.

### Deliverable 6 — Retrospective

A one-page `docs/launch-retro.md` written *after* the 100% promotion, covering:

- What the plan doc predicted vs what actually happened.
- Which gates were close to failing.
- What the auto-rollback drill exposed.
- One thing to change in the next launch's plan.

## Starter guidance

- **Freeze the plan before the flag flip.** If you find yourself wanting to edit a threshold mid-window, that's the signal that the plan is doing its job — the threshold caught something. Either roll back or file a *plan revision* and note why, but do not silently move the number.
- **Shadow off the critical path is not negotiable.** A shadow that back-pressures onto the request path is worse than no shadow. Use `asyncio.create_task` (Python) / `queueMicrotask` or a bounded worker pool (Node), and drop on overflow.
- **The `n ≈ 16 × p × (1 - p) / Δ²` heuristic is intentionally rough** — it's the "avoid the obvious mistake" number. If your Δ is very small or your metric is a ratio of correlated variables, bring `senior-ml-engineer` in per [chapter 03](../03-ab-experiment-interface-to-the-experimentation-platform.md).
- **Per-scenario floors matter more than composite scores** for LLM launches. The composite-passes-but-a-scenario-collapsed failure is what the enterprise-tier regression in the problem statement was; the per-scenario gate is what catches it.
- **Rehearse the rollback before you need it.** The drill in Deliverable 5 is the whole point of Deliverable 5. Any auto-rollback that has never fired in rehearsal will fail in the incident.

## Acceptance criteria

- [ ] `docs/launch-plan.md` is committed and tagged before any live flag flip. All four gate families have specific thresholds; sample-size math is shown; auto-rollback rule is explicit.
- [ ] The shadow-launch wiring runs asynchronously, drops on queue overflow, emits `app.shadow.of` / `app.shadow.variant`, and tags cost under a distinct `app.feature` value.
- [ ] A test asserts the shadow does not block the primary path.
- [ ] `docs/shadow-report.md` includes aggregate and per-scenario quality delta, cost delta, and a pass/fail decision against the plan's gates.
- [ ] `docs/canary-log.md` contains at least four promotion-decision blocks matching the template.
- [ ] `alerts/canary-burn.yaml` defines fast-burn and slow-burn alerts on the gate metrics; the auto-rollback action flips the flag via provider API.
- [ ] `docs/rollback-drill.md` documents an executed rollback drill with measured time-to-alert, time-to-effect, and post-flip trace evidence.
- [ ] `docs/launch-retro.md` closes the loop: predicted vs actual, close-call gates, drill findings, one plan change for next time.

## Stretch goals

Pick any one:

- **Per-segment gate.** Add a per-tenant-tier gate (e.g., `enterprise` tier must not regress by more than 3 pp). Wire the shadow analysis to break down by segment; catch a segment-only regression that the aggregate would have missed.
- **Cost-aware canary sizing.** Compute the projected daily spend of the canary at each stage using the ledger from mod-109 exercise-04. If the projected spend exceeds the absolute cost gate at any stage, either extend the observation window (to amortise) or shrink the cohort.
- **Shadow judge drift protection.** Run the mod-106 judge drift monitor over the shadow window. If the judge's calibration on the frozen calibration set has drifted, invalidate the shadow signal and re-calibrate before proceeding.
- **Multi-variant canary.** Run `v15_concise` and `v15_thorough` as parallel canary cohorts (5% each), with the control at 90%. Extend the promotion-decision artefact to a 3-way comparison. Note the multiple-comparisons implication (a nod to [chapter 03](../03-ab-experiment-interface-to-the-experimentation-platform.md)).

## Deliverable

Submit as `mod-111/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/launch-plan.md`, `docs/shadow-report.md`, `docs/canary-log.md`, `docs/rollback-drill.md`, `docs/launch-retro.md`.
- `alerts/canary-burn.yaml` (or your stack's equivalent) with the fast-burn / slow-burn rules and the flag-flip action.
- The shadow-launch code diff plus the queue-overflow test.
- Sample trace exports showing shadow spans tagged correctly and post-rollback spans carrying `v14`.
- A short `README.md` naming the feature, the flag provider, the alerting stack, and the observation-window ratios you chose.
