# Sampled Online Eval Loop with a Judge-Drift Monitor

## Why offline eval is not enough

The offline suite from [chapters 01](01-offline-eval-suites-a-pr-can-run.md) and [02](02-trace-snapshots-as-regression-fixtures.md) gated by the CI from [chapter 03](03-eval-gated-ci-end-to-end.md) tells you *"this PR does not regress on the fixtures we froze."* It does not tell you *"the feature is still working in production this week."* Between the offline pass and the online reality sit at least six sources of drift the applied engineer must plan for:

- The **input distribution** shifts as users find new ways to use the surface.
- The **retrieval corpus** shifts as the knowledge base grows and gets edited.
- The **model** shifts when a provider rolls a "silent" behind-an-alias update (mod-108).
- The **surrounding product** shifts as adjacent features change what users bring to the agent.
- The **downstream systems** shift as tool APIs are rev'd and errors change shape (mod-104).
- The **judge model** itself shifts when you upgrade it or when its provider updates it.

None of these show up on the offline suite until you add them as fixtures, and by the time you know to add them as fixtures, they are already an incident. The online eval loop is the applied engineer's *early-warning system*: a small, continuous sampling of live traffic scored on quality, safety, cost, and latency, watched for drift.

This is not the same instrument as production traces / metrics (mod-110's territory). Trace metrics tell you *what happened* (latency percentiles, error rates, tool-call durations). Online eval tells you *whether what happened was good* — a judgement, applied at some sample rate, priced against a monthly budget.

## The scoring plane: four dimensions, one loop

For any shipped feature, the online loop scores every sampled trace on four dimensions:

1. **Quality.** The domain-appropriate judged score from chapter 01 (rubric, RAG triad, LLM-as-judge). One or more numeric metrics per trace.
2. **Safety.** A classifier or judged check for the failure modes the surface is exposed to — prompt injection, sensitive-information disclosure, jailbreak drift, hallucinated authority. See mod-107 for the full guardrails picture; this chapter is about *measuring* their outcome.
3. **Cost.** Actual per-trace token spend from the provider, aggregated per user, per feature, per model tier. See mod-109 for the SLO framing; this chapter uses cost as an eval-loop metric alongside quality.
4. **Latency.** Wall-clock time from user submission to final response, with per-tool-call breakdown for tool-using surfaces. Same relationship to mod-109 as cost.

All four are logged on every trace (that's the observability substrate). Quality and safety are *scored* on the sampled subset. Cost and latency are already numbers — no judge needed — and are dashboarded on the full population.

The single most common early mistake is to conflate the four. Cost cannot be judged; quality cannot be measured with a stopwatch. A "quality-cost-latency composite" number hides the trade-off the applied engineer is paid to notice. Four dimensions, four metrics, four separate thresholds.

## Sampling: how much traffic to score

The sample size question resolves against three constraints — budget, statistical power, and observability. The applied engineer is deferring statistical rigour (bootstrap CIs, sequential tests, power analysis) to the peer eval-engineering role; the working defaults below are enough for the "did quality drop this week?" question, which is the applied engineer's actual question.

**Base sampling rate: 0.5% to 5% of traffic**, stratified by:

- **Feature / surface.** A rarely-used surface is over-sampled so the weekly count reaches a floor (typically 200-500 scored traces per surface per week).
- **User cohort.** Free tier vs. paid vs. enterprise; new vs. returning; representative of every persona the product ships to.
- **Feedback signal.** *Every* thumbs-down is scored (it's cheap and it is the population you most need judgment on). Thumbs-up sampled at the base rate. Silent traces sampled at the base rate.
- **Model routing arm.** Every routing arm (mod-108) gets a floor sample; a routing rule that sends 0.1% of traffic to a cheaper model is invisible to a 1% uniform sample.

For a support surface handling 100,000 requests a week, a 1% sample stratified as above yields ~1,000 scored traces per week, plus every thumbs-down, plus routing-arm floors. At $0.02-0.20 per judged score, that's $20-$200/week per surface — an order of magnitude cheaper than the offline suite runs across a busy PR week, and the primary lever for staying inside the monthly budget.

**When to bump the rate temporarily:**

- The first two weeks after a launch (100% for the first day, tapering to base rate over the fortnight).
- During an experiment (mod-111) — sample every trace in both arms until the experiment reads out.
- After an incident — 100% for the affected cohort until the fix is verified live.

**When to cut the rate:** never quietly. If the budget forces a cut, publish it, and hold the cut against the surface's SLO with a decision to escalate if quality signals start to fall below what the smaller sample can measure.

## The monthly eval budget

Online eval spend needs to be a *budget*, not a variable cost. Without a cap, the loop's cost drifts up unnoticed until finance flags it. With a cap, the loop forces trade-offs the applied engineer should be making anyway.

A working budget document, per surface, per month:

```yaml
# eval/policies/support-agent-online.yaml
surface: support-agent
period: monthly
budget_usd: 800.00
allocation:
  base_sampling: 500.00       # 1% base rate against ~100k reqs/week
  thumbs_down_100pct: 150.00  # score every negative feedback
  routing_arm_floor: 50.00
  post_incident_reserve: 100.00
policy:
  when_over_80pct: alert_eval_owner_and_pm
  when_over_100pct: reduce_base_rate_to_0.5pct_and_notify
  when_over_120pct: freeze_new_experiments
judge:
  model: anthropic/claude-opus-4-8
  pin_expires: 2027-01-31         # forces a re-review at that date
```

Three properties matter. The budget is per surface (a shared pool invites the loudest team to eat everyone else's share). It has *decision rules* attached to overrun thresholds (the applied engineer does not want to invent a policy at 3am when the budget breaches). And the judge model has an explicit pin with an expiry — the next section is why.

## Judge drift: the failure mode nobody catches on time

The single most-overlooked failure mode in a maturing eval loop is: *the judge model changes and the numbers move for no other reason.*

This happens in three ways.

- **Provider rolls the alias.** The team pinned `judge_model = gpt-4-turbo` a year ago; the provider silently updated the alias to a newer snapshot; scores drift.
- **Team upgrades the judge deliberately.** A new judge model ships (`claude-opus-4-8` supersedes `4-7`); the team bumps the pin without re-baselining; historical scores are no longer comparable.
- **Rubric drifts under the judge.** Same judge model, but someone edited the rubric prompt; the judge is now scoring a different task.

Each of these looks *exactly* like a real quality change on the dashboard. The applied engineer's job is to distinguish "the feature got better" from "the judge got easier" — and it is not a hard job if the instrumentation is in place from day one.

### The drift-detection design

Three instruments, running continuously:

**Instrument 1 — the judge pin.** The judge model ID is versioned in the pre-registration file (chapter 01), with an expiry date. Every trace in the online store carries `judge_model_id` and `judge_prompt_hash` alongside the score. When either changes, the change is visible in the trace log and requires an explicit PR-reviewed update to the pre-registration.

**Instrument 2 — a fixed calibration panel.** A small, permanent set of 30-50 fixtures with *human-authored ground-truth scores* (from the peer eval-engineering role's calibration work, or from a one-time joint effort). Every time the judge model changes — deliberately or otherwise — the panel is re-scored under the new judge and the delta against the human-authored scores is published. A judge whose calibration panel disagreement moved by more than a small threshold is not a drop-in replacement; the surface's scores under the new judge are not comparable to prior scores until re-baselined.

**Instrument 3 — parallel-judge shadow scoring.** During a judge upgrade window (typically two weeks), both the old judge and the new judge score every sampled trace. The dashboards report both; the alerting fires on the old judge until the new one has demonstrated agreement on the calibration panel. Only then does the new judge become authoritative. This is the online analogue of a canary rollout (mod-111), applied to the judge.

### The re-baselining protocol

When a judge upgrade is accepted (calibration panel agrees), all downstream historical scores must be re-baselined. Concretely:

- The suite version is bumped in the pre-registration file.
- Historical scores under the old judge are archived, not deleted (they remain queryable, but they no longer roll up into the "current" dashboards).
- The chapter 02 trace snapshots are re-scored under the new judge on their `graders` block.
- The dashboards from chapter 05 add an annotation on the timeline for the judge change, so a reviewer looking at a step-change six months later can trace the cause.

The protocol is not fast, and that is on purpose. Judge changes are rare events that should be treated with the same rigour as a model rollout for the feature under evaluation.

## What to do with a scored trace

A scored online trace generates three signals the applied engineer acts on:

1. **A per-trace record** — stored alongside the trace itself in the observability substrate (Langfuse, Braintrust, LangSmith, Phoenix, Weave). Queryable, aggregatable, filterable. This is the raw material for the dashboards in chapter 05.
2. **A candidate for the offline fixture set** — traces below a quality threshold are surfaced to a curation queue. The applied engineer (or a triage rotation) decides whether to promote them to trace snapshots (chapter 02), scrub the PII, and add them to the offline suite. Complaints promote automatically; low-quality-without-feedback traces get triaged.
3. **An alert, when the aggregate crosses a threshold.** The alerting shape below.

### Alerting: when the loop pages a human

Alerting on live judged scores is subtler than it looks. Judge scores are noisy at the trace level; a single low score is not an incident. A 7-day rolling drop in the median rubric score, on a specific persona segment, is. Design the alerts around *sustained aggregate drops on segments that matter*, not per-trace thresholds.

A working alerting shape:

```yaml
# eval/policies/support-agent-alerts.yaml
alerts:
  - name: quality_drop_overall
    metric: median(rubric.score)
    window: 24h
    baseline: rolling_mean(7d, offset=7d)
    trigger: current < baseline - 0.05 for 3 consecutive windows
    page: applied_eng_oncall
  - name: safety_incident
    metric: rate(safety.hijacked == True)
    window: 1h
    trigger: current > 0.001    # any nontrivial hijack rate
    page: safety_reviewer + applied_eng_oncall
  - name: cost_burn
    metric: cumulative(cost_usd) / budget_usd
    window: rolling_month
    trigger: current > 0.80 with 10 days remaining
    page: applied_eng_owner
  - name: latency_p95_slo
    metric: p95(latency_ms)
    window: 1h
    trigger: current > slo.p95_ms for 3 consecutive windows
    page: applied_eng_oncall
  - name: judge_drift_warn
    metric: judge_calibration_panel.disagreement_rate
    window: on_change
    trigger: current > baseline + 0.10
    page: eval_owner
```

Each alert names the *segment* (overall, per persona, per routing arm) that it fires on. A "quality drop" is much more actionable when it says "on the non-English cohort" than when it says "on the population average."

## Feeding the online loop back into offline

The online loop's most valuable output is not the dashboard; it is the **fresh material for the offline suite**. Without this feedback path, the offline suite ages into a memorial of past incidents.

The pipeline:

1. Sampled traces (quality below threshold, or from a complaint cohort, or from a new persona pattern) land in a curation queue.
2. The applied engineer, on a triage rotation, reviews the queue weekly. Every accepted trace becomes a chapter 02 snapshot with lineage pinned, PII scrubbed, and graders specified.
3. The suite version is bumped when the batch lands (weekly is fine; more often is fine).
4. The next PR is gated against the updated suite. The regression the online loop caught is now blocked from recurring.

This is the applied engineer's *feedback loop*, and it is the whole point. Every online incident becomes an offline fixture. Every offline fixture stays green in the online loop or gets triaged. The suite grows against the problems the surface actually has, not against the problems the applied engineer imagined it might have.

## Anti-patterns

Five worth naming.

1. **The static judge, forever.** The judge is pinned to a model that has since been deprecated. The eval runs on a legacy provider endpoint that will be shut off with 30 days' notice. Judge pins have expiry dates.
2. **The unbudgeted loop.** Sampling rate crept from 1% to 10% during a firefight, no one turned it down, finance flagged the bill. Budget is a policy, not a suggestion.
3. **Judged-only, feedback-agnostic sampling.** The loop scores random traffic and ignores the thumbs-down button. The single strongest label the user gives you goes unread. Score every negative-feedback trace at the top of the queue.
4. **The single-shot alert.** A one-trace low score pages the on-call. The on-call disables the alert. Alert on sustained aggregate drops on named segments, with a floor sample size.
5. **The dashboard-only loop.** Scores land in a dashboard, no one wires them back to the offline suite. The suite ages; the next incident is a fresh surprise. The online → offline feedback path is the loop's whole reason to exist.

## Summary

- The online loop is the applied engineer's early-warning system: a sampled, continuous, judged read of live traffic on quality, safety, cost, and latency.
- Sample at 0.5-5% base rate stratified by feature, cohort, feedback signal, and routing arm. Score 100% of negative feedback. Temporarily bump for launches, experiments, and post-incident.
- Publish a per-surface monthly budget with decision rules attached to overrun thresholds. Do not silently cut the sampling rate to fit budget; escalate.
- Judge drift is the failure mode most likely to fool a mature loop. Instrument it with a judge pin (with expiry), a fixed calibration panel, and parallel-judge shadow scoring across upgrade windows.
- Re-baseline when the judge changes: bump the suite version, archive old scores, re-score the offline snapshots' graders, annotate the dashboards.
- Feed the online loop back into the offline suite. Every incident becomes a fixture; every fixture stays green in the online loop or is triaged out.

Chapter 05 closes the module by asking: who reads the numbers this loop produces? The PM, the on-call engineer, and the safety reviewer each need a different view — and the applied engineer's dashboard design decides how loudly the loop actually gets heard.
