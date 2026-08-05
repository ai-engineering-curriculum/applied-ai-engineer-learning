# A/B Experiments: The Interface to the Experimentation Platform

## Where the applied engineer's job starts and stops

An experimentation platform — Statsig, Eppo, LaunchDarkly Experimentation, GrowthBook, Optimizely, Split, or an in-house one built on top of a warehouse like Snowflake or BigQuery — is a substantial piece of software. It handles assignment, exposure logging, guardrail-metric monitoring, sample-ratio-mismatch detection, sequential-testing corrections, variance reduction (usually CUPED), and the statistical inference that turns raw metric deltas into "was this actually better." The methodology that underlies all of that — the variance-reduction technique, the sequential-testing correction, the choice of test statistic, the multiple-comparisons discipline — is *deep* work, and the AICG track hands it to `senior-ml-engineer` and `model-evaluation-engineer` on purpose. Applied engineers do not need to derive the CUPED estimator or debate a group-sequential test against an always-valid inference sequence.

What the applied engineer *does* own is **the seam between the LLM feature and the experimentation platform** — the code and the metric definitions that let the platform do its job. That seam is what this chapter is about. It's four things:

1. The **assignment call** — asking the platform which variant this user should get, deterministically and stickily.
2. The **exposure log** — telling the platform this user actually experienced the variant, when, and in what context.
3. The **metric emission** — writing the numeric events the platform will slice by variant to compute the result.
4. The **guardrail-metric wiring** — the specific LLM-shape safety / cost / latency metrics that gate an experiment independent of its "did it move the primary metric" question.

If the applied engineer does those four things faithfully, the peer roles can run every LLM experiment the org needs. If any of the four is wrong (stale assignment, missing exposure log, metric aggregated in the wrong unit, no guardrails), no amount of statistical sophistication downstream can rescue the result.

This chapter names the interface. It also names — with intent — the places where deeper methodology starts, so the applied engineer knows *when to bring in the peer role* rather than trying to solve it themselves.

## Assignment: the deterministic hash-then-bucket call

Every experimentation platform's SDK exposes a call of roughly this shape:

```python
# GrowthBook example — every serious platform has an isomorphic call.
from growthbook import GrowthBook

gb = GrowthBook(
    api_host="https://cdn.growthbook.io",
    client_key=os.environ["GB_KEY"],
    attributes={"id": user.id, "tenant_id": tenant.id, "locale": user.locale, "internal": user.internal},
)
gb.load_features()

variant = gb.get_feature_value("support_agent.prompt_experiment", "control")
```

The platform hashes the assignment key (usually `user.id`, occasionally `tenant.id` for B2B experiments, occasionally a stable `session.id` for anonymous surfaces) with the experiment's salt into `[0, 1)` and buckets the result according to the experiment's traffic-allocation weights. Two invariants make this call correct:

- **Deterministic and sticky.** For a given `(experiment, assignment_key)` pair, the assignment is the same on every call — across turns, across sessions, and across the client and the server. If your call flips assignment mid-experiment, the treatment effect estimate is meaningless.
- **Named at the *right* unit of randomisation.** If the metric you care about ("did the user come back and click X the next day") is measured at the user level, the assignment key is `user.id`. If the metric is measured at the tenant level, the key is `tenant.id`. Mismatching the unit of assignment and the unit of analysis is a bug the platform will surface as a *sample-ratio mismatch* alert.

This maps 1:1 onto the flag-layer `targeting_key` from [chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md). In fact, LaunchDarkly, GrowthBook, Split, and Statsig all use the same feature-flag SDK for both flag delivery and experiment assignment — the experiment is a targeting rule on the flag. That's why the flag substrate has to be right before you can run an experiment; the experiment platform is *reading* the flag layer.

## Exposure: log that the user actually saw the variant

An "assignment" is a call to the SDK. An "exposure" is the observation that the user *experienced the variant that assignment produced*. They are not the same thing. The gap matters in two failure modes:

- **The user is assigned but never exposed.** The code called `get_feature_value` on page load, then never actually rendered the feature (the user closed the tab). Counting this user as "in the treatment" biases the estimate toward "the treatment did nothing," because the treatment didn't happen.
- **The user is exposed but never logged.** The code did render the feature but forgot to fire the exposure event. The user is under-counted in the treatment cohort, which shows up as a sample-ratio mismatch and invalidates the whole experiment window.

Every serious experimentation platform expects the app to emit an **exposure event** at the moment the user actually experiences the variant. Some SDKs (GrowthBook, Statsig) do this implicitly by intercepting the `get_feature_value` call; others (LaunchDarkly Experimentation) require an explicit `track` call. For an LLM feature, "the user actually experienced the variant" usually means "the LLM call was made and a response was returned to the user" — not "the flag was evaluated."

```python
variant = gb.get_feature_value("support_agent.prompt_experiment", "control")

with tracer.start_as_current_span("support_agent.turn") as span:
    template = prompt_registry.get(variant)
    span.set_attribute("app.experiment.id", "support_agent.prompt_experiment")
    span.set_attribute("app.experiment.arm", variant)

    response = call_llm(template, user_input, ctx)

    # Exposure fires *after* the user got the response — not on flag lookup.
    experiment_platform.track_exposure(
        experiment_id="support_agent.prompt_experiment",
        variant=variant,
        user_id=user.id,
        properties={"tenant_id": tenant.id, "surface": "chat_panel"},
        timestamp=now_ms(),
    )
    return response
```

The exposure event is what the platform joins metrics to. If it fires too eagerly (before the response was actually served) or fails to fire (async task swallowed the error), the metric aggregation is wrong.

The trace attribute `app.experiment.arm` is a separate emission — it lives on the OpenInference / OTel span and is what the observability side ([mod-110](../mod-110-observability-feedback-and-incidents/README.md)) slices by. Emit both. The exposure event goes to the experimentation platform's metric store; the span attribute goes to the trace backend. Different consumers, both needed.

## Metric emission: the four shapes that matter for LLM experiments

The primary metric of an LLM experiment is almost always a *product-level* metric — did the user click the "helpful" thumbs, did they open a support ticket in the next 24 hours, did they retry the query, did the conversion event fire downstream. That metric is often something the product team already tracks; you emit it once, from the product-events pipeline, and the experimentation platform slices it by exposure.

Alongside the primary metric, an LLM experiment needs four LLM-specific metric shapes emitted from the feature itself:

- **Turn-level quality signal.** Thumbs-up / thumbs-down / regenerate ([mod-110 chapter 03](../mod-110-observability-feedback-and-incidents/03-user-feedback-triage-pipeline.md)), or the LLM-judge score if the feature has one running online. Unit: per turn.
- **Latency and cost.** TTFT ms, total ms, `app.cost.usd` from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md). Unit: per turn. The experimentation platform lets you slice these by variant and see, for example, that variant B increased "helpful" rate by 1.2% but also cost 18% more per outcome — the CPSO delta from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md), computed on the experiment cohort.
- **Guardrail decisions.** Refusal rate, moderation-block rate, tool-error rate. Unit: per turn. These are the guardrail metrics that gate the experiment independent of the primary metric.
- **Session-level roll-ups.** For agentic surfaces, the metrics that actually match user perception are session-level, not turn-level: "did the user's session end in a resolved state," "how many turns did it take," "did the user retry mid-session." Emit these at session-close, keyed by the *initial* assignment (a mid-session variant flip is a bug — assignment is sticky).

The unit of analysis matters. If your assignment unit is `user.id` but your metric is emitted per turn, the platform has to aggregate "turns per user" into a user-level metric before computing the treatment effect. Every serious platform does this correctly (that's what CUPED and the standard delta method estimator are for), but only if the events carry both a `user_id` and a per-event timestamp. Emit both, always.

## Guardrail metrics that gate the experiment, independently

In experimentation vocabulary, a **guardrail metric** is a metric that must not regress, regardless of whether the primary metric moves. If the primary metric ("helpful" rate) went up 1.2% but the guardrail metric (refusal rate) went up 5%, the platform surfaces a *guardrail regression* and the experiment does not ship even if the primary is significant.

The applied engineer's job here is to name the LLM-specific guardrails the platform should watch, in the platform's UI, as guardrail metrics tied to the experiment:

- **Refusal rate delta** (safety guardrail).
- **Moderation-block delta** (safety guardrail).
- **CPSO delta** (cost guardrail).
- **TTFT p95 delta** (latency guardrail).
- **Retry-rate delta** (reliability guardrail).
- **Support-ticket-opened-within-24h delta** (product guardrail — the "if this goes up, we made things worse regardless of what the primary metric said").

The specific thresholds are the same numbers pre-registered in the canary plan from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md). The experimentation platform's guardrail is a *statistical* version of the canary gate — instead of a hard threshold, it computes whether the metric regressed by a statistically significant amount. Both mechanisms exist; the canary gate catches obvious regressions early, the experiment guardrail catches statistically significant regressions at the end.

## Sample-ratio mismatch — the alarm that says "your interface is broken"

Every experimentation platform runs a background check called **sample-ratio mismatch (SRM)**. If the experiment was configured as a 50/50 A/B and the actual observed exposure counts are 51% / 49%, the platform runs a chi-squared test on the split and fires an alert when the observed split deviates from the configured one by more than chance.

SRM is the platform's most important self-check. When it fires, the *first* thing to look at is not the experiment result — it is the interface. An SRM alarm almost always means one of:

- **Assignment is not deterministic** (the same user is getting different variants on different calls).
- **Exposure logging is broken on one side** (the treatment code path fails to fire exposures more often than the control does, so the treatment cohort looks smaller than it is).
- **The unit of randomisation and the unit of analysis don't match** (assignment by user, exposure count aggregated over sessions, and different variants have different session counts per user).
- **The flag layer is fanning out early to a variant** (a default fallback to "control" during an outage inflates control counts).

If SRM fires, the applied engineer's job is to debug the *interface* between the feature and the platform, not to reason about the experiment result. Any experiment where SRM fired is invalidated for that window and re-run once the interface is fixed. `senior-ml-engineer` should never have to say "your instrumentation is wrong" twice.

## The peer-role boundary — what applied does NOT own

There are four families of experimentation work that live one seniority-tier deeper than this module. When any of them surfaces, the right move is to bring in `senior-ml-engineer` or `model-evaluation-engineer` rather than to solve it in the feature code:

- **Variance reduction.** CUPED, stratified sampling, or synthetic-control constructions to tighten confidence intervals so the experiment reaches significance faster. Owned by the peer role because the pre-treatment covariate has to be chosen with care to avoid introducing bias.
- **Sequential testing / early stopping.** Alpha-spending functions (Pocock, O'Brien-Fleming), always-valid inference (mSPRT, e-values), or Bayesian stopping rules that let an experiment be checked repeatedly without inflating the false-positive rate. The naive "just check every day" pattern is *worse* than fixed-horizon testing and needs the correction from a peer role.
- **Heterogeneous treatment effects.** "The average effect is +1%, but SMB is +5% and enterprise is -3%" — segmenting the effect estimate. Statistically correctable but easy to overfit; the peer role owns the segment discipline and multiple-comparisons correction.
- **Ratio metrics and delta-method estimation.** Metrics like "clicks per session" are ratios of correlated random variables. The naive standard error is wrong; the delta-method estimator is standard practice for the peer role.

The applied engineer's stance: **name what you want to learn, hand the peer role a clean interface (assignment + exposure + metric + guardrail), and ask their read on which statistical treatment fits.** Do not derive the estimator; do not read only the p-value; do not skip the guardrail. Those are the three shapes of "applied engineer trying to do the peer role's job" that most often produce broken launches.

## Small concrete example — the primary metric decision for a summarisation feature

A concrete: the team wants to A/B a new prompt for a support-ticket summarisation feature. The primary product metric is "was the summary accepted by the agent without edits" (a click event fired by the agent's UI). Guardrails: refusal rate, CPSO, TTFT p95, and the LLM-judge score on a rolling sample.

- **Assignment key**: `agent.id` (not the ticket ID — one agent sees many tickets, and the metric is about the *agent's* trust in summaries).
- **Exposure**: fired when the summary is rendered in the agent's UI. Fires once per rendered summary, not once per prompt evaluation.
- **Primary metric**: `summary_accepted_without_edits`, boolean, per exposure. Aggregated to a per-agent rate for the treatment-effect estimate.
- **Guardrail metrics**: `refusal_rate` per exposure, `cost_usd` per exposure, `ttft_ms` per exposure, `judge_score` on a 5% sample.
- **Traffic split**: 50/50 for a two-week window, MDE (minimum detectable effect) of 2 percentage points on `summary_accepted_without_edits`, sample size ~5000 agents per variant. The MDE and sample size come from a conversation with `senior-ml-engineer` after the applied engineer lays out the metric and rate.
- **SRM check**: platform monitors observed split; on any deviation, the experiment is paused and the interface is debugged.

Two weeks in, the platform surfaces: primary metric +2.7 pp (significant at α=0.05), CPSO +14% (guardrail regression, statistically significant), TTFT p95 +80 ms (not significant). The product decision is now real: is the +2.7% acceptance worth +14% cost? That decision is a product decision. The applied engineer's job was to make it *legible*.

## Anti-patterns

- **Assignment key that changes between calls.** Users flip variants mid-session and the experiment is invalidated. Use the flag substrate from [chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md); the platform SDK guarantees stickiness if you give it a stable key.
- **Exposure fired on flag lookup instead of on user experience.** Users who never actually saw the treatment are counted in the treatment cohort. The primary metric is diluted; the effect estimate is biased toward null.
- **No SRM alerting or ignored SRM alerts.** The experiment reads as significant but the split is 51/49; the platform's alarm is telling you the interface is broken, not that you should read the p-value harder.
- **Primary metric is "did the LLM output look good in eyeballing."** A/B experiments are for causal inference on shipped features. If you cannot pre-register the metric and its aggregation, you are not running an experiment — you are running a demo.
- **Peeking without correction.** Checking the p-value every day for two weeks and declaring victory when it first crosses 0.05. Sequential-testing correction is not optional; bring in `senior-ml-engineer` to pick the right one.
- **No guardrail metrics.** The primary went up 1%, ship it, spend went up 30%, whoops. Guardrail metrics are the ergonomic version of the pre-registered canary gates from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md).
- **The applied engineer computing the treatment effect by hand.** SQL query, `AVG` in the treatment vs `AVG` in the control, ship a screenshot. Every experimentation platform runs the correct estimator (delta method for ratios, CUPED for variance reduction) — use the platform's number.
- **Running two experiments on the same surface simultaneously with no interaction plan.** The variants interact; the peer role has patterns (factorial designs, layered experiments) for this; do not overlap two unrelated tests on the same LLM turn without one.

## Summary

- The applied engineer owns the **seam** between the LLM feature and the experimentation platform — assignment, exposure, metric emission, guardrail wiring. Everything statistical happens downstream.
- **Assignment** is a deterministic hash of a stable key (user, tenant, or session) into a bucket. The unit of randomisation must match the unit of analysis; the assignment key is the same as the flag `targeting_key` from [chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md).
- **Exposure** logs that the user experienced the variant, fired at the moment the user actually sees the feature — not on flag lookup. Getting exposure wrong is the source of most sample-ratio mismatches.
- **Metric emission** includes the product primary metric plus four LLM-shape metrics per turn (quality signal, latency, cost, guardrail decisions) plus session-level roll-ups for agentic surfaces. Every event carries `user_id` and timestamp.
- **Guardrail metrics** — refusal, moderation FP, CPSO, TTFT, retry rate — are configured in the platform as guardrails on every experiment, using the same thresholds as the canary plan.
- **Sample-ratio mismatch** is the platform's canary-in-the-coalmine for a broken interface. When it fires, debug the interface first; do not read the result.
- The **deep methodology** — variance reduction (CUPED), sequential testing, heterogeneous treatment effects, ratio-metric estimation — is owned by `senior-ml-engineer` / `model-evaluation-engineer`. The applied engineer names what they want to learn and hands over a clean interface.

[Chapter 04](04-hot-rollback-of-a-bad-prompt.md) turns from "how do I decide the new variant is better" to "how do I get the bad variant off traffic in five minutes when it isn't" — prompt registry semantics, rollback SLA, and blast-radius containment.
