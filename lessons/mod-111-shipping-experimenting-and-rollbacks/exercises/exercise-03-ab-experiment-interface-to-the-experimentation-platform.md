# exercise-03: A/B Experiment Interface to the Experimentation Platform

**Estimated effort:** 2 hours

## Objective

Wire the feature into an experimentation platform (GrowthBook, Statsig, Eppo, LaunchDarkly Experimentation, or an equivalent) as an *interface exercise*: assignment call, exposure logging, metric emission, and guardrail-metric registration. The deliverable is the *seam*, not the statistical analysis — the peer roles (`senior-ml-engineer`, `model-evaluation-engineer`) run the analysis once the interface is clean. What you own is that the interface is clean enough for their analysis to be trustworthy.

## Learning outcomes

By the end you should be able to:

- Call an experimentation platform's assignment SDK with a stable, correctly-scoped `assignment_key` matching the metric's unit of analysis.
- Fire an **exposure event** at the moment the user actually experiences the variant (not on flag lookup, not on page load).
- Emit per-turn metric events for the four LLM-specific metric shapes (quality signal, cost, latency, guardrail decision) plus session-level roll-ups.
- Register **guardrail metrics** in the platform so an experiment is blocked from shipping when a guardrail regresses, independent of the primary metric.
- Handle a **sample-ratio mismatch** alert as an *interface bug*, not as a result to interpret.
- Name the boundary between what applied engineers own (the interface) and what `senior-ml-engineer` owns (CUPED, sequential testing, HTE, delta-method).

## Prerequisites

- You have read [chapter 03](../03-ab-experiment-interface-to-the-experimentation-platform.md).
- The flag layer from [exercise-01](exercise-01-feature-flag-driven-prompt-rollout.md) is in place.
- Access to an experimentation platform. Free options: **GrowthBook** (self-hostable, OpenFeature-compatible), **Statsig developer tier**, **Eppo trial**, **LaunchDarkly free tier with Experimentation add-on if available**. The exercise assumes GrowthBook for code samples; the shape carries over.
- The feature emits the mod-110 span attributes (`app.feature`, `app.tenant_id`, `app.experiment.arm`, `app.cost.usd`, latency fields).
- Existing per-turn feedback capture — thumbs-up/down or equivalent — from [mod-110 chapter 03](../../mod-110-observability-feedback-and-incidents/03-user-feedback-triage-pipeline.md).

## Problem statement

Your team has been running canary rollouts (exercise-02) and they answer "is the new variant not worse." Product now wants to know whether `v15_concise` is *actually better* than `v14` on a real product metric: the fraction of turns that convert to a positive user action (thumbs-up, or the surface's equivalent conversion event) within the same session. They want a two-week experiment with a defensible result and clean guardrails.

Over two hours: wire assignment, exposure, primary metric emission, four guardrail metrics, and end-to-end verification. Along the way, deliberately mis-wire the exposure event once, observe the sample-ratio mismatch alert, and fix it — so you know what the alarm feels like.

## Requirements

### Deliverable 1 — Assignment call

Under `app/experiment.py` (or equivalent):

- An experimentation-platform client initialised at process start.
- A `get_assignment(user, tenant, surface, locale) -> str` helper that returns the variant for the current user *from the experimentation platform's SDK*, not from the flag layer directly. (Every serious platform layers experimentation on top of feature flags; the assignment call is the platform's version of the flag lookup.)
- The **assignment key** is `user.id`. Document in `docs/experiment-spec.md` why `user.id` is the correct unit for this experiment (the metric is a per-user thing; a per-tenant metric would want `tenant.id`).
- A test that asserts assignment stickiness: 1000 assignments for the same `user.id` return the same variant every time.

### Deliverable 2 — Exposure logging

Modify the turn handler so:

- The **exposure event** fires after the LLM response is served to the user, not on flag lookup.
- The event carries: `experiment_id`, `variant`, `user_id`, `tenant_id`, `surface`, `timestamp_ms`, and any additional properties your platform's UI slices on.
- If the LLM call errors before the user sees a response, the exposure does NOT fire (the user didn't actually experience the variant).
- The exposure is emitted **once per rendered response**, not once per LLM call in a multi-call turn.

Add a test that:

- Confirms exposure fires when the LLM call succeeds.
- Confirms exposure does NOT fire when the LLM call raises an exception before response completion.
- Confirms exposure fires exactly once when the turn involves two LLM calls (e.g., a re-prompt after a schema-validation failure).

### Deliverable 3 — Primary and secondary metric emission

Under `app/metrics.py` (or equivalent):

- A `record_turn_metrics(user_id, exposure_id, ...)` helper that emits the following events to the experimentation platform's metric store:

Primary metric:
- `summary_positively_received` — boolean per turn, `true` if the user hit thumbs-up or the surface's equivalent conversion event within 30 minutes of the turn.

Secondary metrics:
- `llm_cost_usd` — float per turn, sourced from the span's `app.cost.usd`.
- `ttft_ms` — int per turn, sourced from the span's `app.latency.ttft_ms`.
- `total_latency_ms` — int per turn.
- `retry_count` — int per turn.
- `refused` — boolean per turn (guardrail block or model-side refusal).
- `moderation_blocked` — boolean per turn.
- `session_resolved` — boolean per session (fired at session close), keyed by the user's *initial* assignment.

Every event carries `user_id`, `experiment_id`, `variant`, and `timestamp_ms`. Events are emitted to the platform's metric-ingest endpoint or an intermediate queue; they are **not** derived after the fact from traces (that's a valid pattern but different — call it out if you go that route).

### Deliverable 4 — Guardrail metric registration

In the experimentation platform's UI (or its config-as-code equivalent, e.g., GrowthBook's `datasource.yml` / Statsig's configuration API), register the following as **guardrail metrics** on the experiment:

- `refused_rate` (from `refused`).
- `moderation_blocked_rate` (from `moderation_blocked`).
- `avg_llm_cost_usd` (from `llm_cost_usd`).
- `p95_total_latency_ms` (from `total_latency_ms`).
- `p95_retry_count` (from `retry_count`).
- `support_ticket_within_24h_rate` (a product-side metric — if you don't have it, register a placeholder and note it).

Screenshot or export the guardrail configuration and check it in as `docs/guardrails.png` or `docs/guardrails.yaml`.

### Deliverable 5 — Sample-ratio-mismatch drill

Deliberately mis-wire the exposure event and observe the platform's SRM alarm:

- **Mis-wire**: modify the exposure call to fire *only* when the LLM call did not retry. (This is a realistic bug — an engineer added a `if not retried:` guard for "clean" exposures.)
- Drive at least 2000 turns through the feature at a nominal 50/50 split.
- Wait for the platform's SRM alert to fire (SRM checks run periodically — GrowthBook, Statsig, and Eppo run them at least daily; some run on every metric refresh).
- Under `docs/srm-incident.md`, document:
  - The observed split percentages (should be visibly off from 50/50 because retries are correlated with variant).
  - The platform's alert screenshot.
  - The debugging steps to find the mis-wire.
  - The fix and the re-launch of the experiment window.

The exercise's point is that when SRM fires, the *first* look is at the interface, not at the result.

### Deliverable 6 — Experiment spec doc

Under `docs/experiment-spec.md`:

- **Hypothesis**: "Users on `v15_concise` are more likely than users on `v14` to positively receive the response within 30 min of the turn."
- **Primary metric**: `summary_positively_received`, aggregated per user.
- **Unit of assignment**: `user.id`.
- **Traffic split**: 50/50.
- **Duration**: two weeks (or the shortest window that reaches the MDE at your traffic rate).
- **Minimum detectable effect (MDE)** on the primary metric, and the sample size that gives it. If the MDE calculation is more than "napkin math," name the peer role you consulted.
- **Guardrail metrics** and their thresholds (the platform's default is a two-sided test; you may tighten).
- **Stopping rules**: fixed-horizon (14 days). If you want early stopping, name the sequential-testing method and the peer role who reviewed it.
- **Analysis owner**: the applied engineer runs the *interface* and reads the platform's output; the peer role runs the analysis when the delta is non-obvious.

### Deliverable 7 — Peer-role boundary note

Under `docs/peer-role-boundary.md`, a short (½-page) note naming the four families of experimentation work that live one seniority-tier deeper — variance reduction (CUPED), sequential testing, heterogeneous treatment effects, ratio-metric delta method — and *what specific question in this experiment would trigger each*. This isn't a research exercise; it's a note so that six months from now, the next person on the rotation knows when to reach for the peer role.

## Starter guidance

- **GrowthBook is the fast free path.** It's OpenFeature-compatible, self-hostable in a Docker container, and has all four artefacts (assignment SDK, exposure event, metric datasource, guardrail metric UI). The exercise's shape works on any serious platform; GrowthBook is the reference.
- **Do not derive metrics from spans for this exercise.** Emit dedicated events to the platform's ingest endpoint. Span-derived metrics are a valid pattern for eng-side observability but couple your experiment result to your tracing config; keep the two separate.
- **Fire exposure last.** Wrap the exposure call in a `finally` block after the response is returned to the user — but skip it if the response was never returned. The exposure is the "did the treatment happen" record; get it wrong and the result is biased.
- **The MDE conversation is short.** Talk to `senior-ml-engineer` (or a peer with statistical background) for 10 minutes: "primary metric rate is X, we want to detect Δ pp; how many exposures per variant?" They will give you a number and probably a nudge toward CUPED if it's small. Do not compute it from a textbook without a sanity check.
- **SRM debugging starts at the exposure event.** In > 90% of real SRM incidents, the cause is an asymmetric exposure log — one variant fires exposure less often for structural reasons. Look there first.

## Acceptance criteria

- [ ] `app/experiment.py` exposes `get_assignment(user, ...)` calling the platform SDK with `user.id` as the assignment key.
- [ ] The stickiness test passes (1000 calls, same variant).
- [ ] The exposure event fires after the response is served to the user and only if the response was actually served.
- [ ] The three exposure tests pass (success, error, multi-call).
- [ ] `app/metrics.py` emits the primary metric and the six secondary metrics per turn, plus the session-level roll-up.
- [ ] Every metric event carries `user_id`, `experiment_id`, `variant`, and `timestamp_ms`.
- [ ] The platform's UI (or config-as-code) has the six guardrail metrics registered on the experiment, evidence checked into `docs/`.
- [ ] `docs/srm-incident.md` documents the deliberate mis-wire, the platform's SRM alert, the debugging trail, and the fix.
- [ ] `docs/experiment-spec.md` names hypothesis, primary metric, unit of assignment, split, duration, MDE, guardrails, and stopping rules.
- [ ] `docs/peer-role-boundary.md` names the four peer-role families and the specific question in this experiment that would trigger each.

## Stretch goals

Pick any one:

- **Layered experiments.** Configure a second, orthogonal experiment (e.g., a `regenerate-button-copy` UI experiment) on the same surface and register it in a different layer so it cannot interact with your prompt experiment. Document the layering scheme.
- **Session-level primary metric.** Change the primary metric to `session_resolved` (a session-level boolean) aggregated per user. Update the exposure and metric events to accommodate the changed unit of analysis. Note what breaks and how you fixed it.
- **A CUPED trial run.** Compute a pre-treatment covariate (e.g., the user's `summary_positively_received` rate in the two weeks before the experiment started) and hand it to the platform as a CUPED column. Compare the pre/post confidence-interval width. This is a peer-role activity done under your driver's-seat for one round.
- **Multi-arm experiment.** Register `v15_concise` and `v15_thorough` as two treatment arms with `v14` as control (33/33/33 split). Document the multiple-comparisons implication and the peer-role conversation about correction.

## Deliverable

Submit as `mod-111/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `app/experiment.py`, `app/metrics.py` and tests.
- The feature diff wiring exposure and metric emission.
- Screenshots or exported config of the platform's experiment definition and guardrail metrics.
- `docs/experiment-spec.md`, `docs/srm-incident.md`, `docs/peer-role-boundary.md`, `docs/guardrails.md` (or the image/config).
- A short `README.md` naming the platform, the experiment name, and how to reproduce the assignment/exposure/metric flow.
