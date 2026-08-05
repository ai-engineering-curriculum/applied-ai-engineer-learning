# exercise-04: Sampled Online Eval Loop With a Judge-Drift Monitor

**Estimated effort:** 3 hours

## Objective

Build a **sampled online eval loop** on top of the observability substrate from mod-110 for a shipped (or shipped-shaped) feature: sampling policy, per-dimension scorers (quality / safety / cost / latency), monthly budget with decision rules, alerting, and — the load-bearing piece — a **judge-drift monitor** that catches when the judge model behind the scores has moved. The loop must feed back into the offline suite: incidents surfaced online promote to fixtures via the [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md) pipeline.

## Learning outcomes

By the end you should be able to:

- Author a stratified sampling policy across feature / cohort / feedback signal / routing arm that hits a weekly sample-count floor and a monthly cost cap.
- Score sampled traces on four separately-reported dimensions (quality, safety, cost, latency).
- Publish a per-surface monthly budget with concrete decision rules attached to overrun thresholds.
- Instrument the judge with a pin (dated + expiry), a fixed human-calibrated panel, and parallel-judge shadow scoring across upgrade windows.
- Wire the online → offline feedback path: low-quality online traces surface to a triage queue that promotes to snapshot fixtures via the exercise-02 pipeline.

## Prerequisites

- You have completed [exercise-01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md) (offline suite + judge model pinned) and [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md) (trace snapshots + scrubber + replay).
- You have an observability substrate you can query: Langfuse, LangSmith, Braintrust, Phoenix, Weave, or a simple JSONL trace store. If you have none, use the exercise-02 scaffold's local store.
- Access to a real feature with sample traffic *or* a traffic simulator that generates realistic requests at a controllable rate (a synthetic-traffic runner in the paired solutions repo will do).
- Access to two candidate judge models (typically the current pin and its likely successor).

## Problem statement

The eval CI gate from [exercise-03](exercise-03-eval-gated-ci-with-promptfoo-and-braintrust.md) blocks PRs but does not tell you what's happening in production between PRs. Two weeks after the last change, the surface has quietly regressed on a persona segment nobody added a fixture for; three months in, the provider silently updated the judge and the whole score chart drifted. Your task is to close both gaps: a sampled loop that watches live traffic and a drift monitor that keeps the numbers honest.

## Requirements

### Deliverable 1 — The sampling policy

Under `online/sampling/policy.yaml`:

- Base sampling rate (0.5-5%), justified against the feature's traffic and the monthly budget.
- Stratification axes: at minimum `cohort` (free / paid / enterprise), `feedback` (`thumbs_down` sampled at 100%), `routing_arm` (per-arm floor).
- Weekly floor per stratum (typically 200-500 scored traces per surface / stratum / week).
- Bump conditions: launch window, active experiment, post-incident window.

A `sampler.py` (or equivalent) that reads the policy, subscribes to the trace stream (or reads a JSONL batch), and emits the sampled subset to the scoring queue.

### Deliverable 2 — The four scorers

Under `online/scoring/`, one scorer per dimension. **Report each separately; do not aggregate.**

- **`quality.py`** — the same judge / rubric / RAG-triad harness from [exercise-01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md), applied to the sampled live trace. Emits per-dimension numeric scores.
- **`safety.py`** — the safety spot-check patterns plus a small LLM-graded check for injection success / hijack / sensitive-info disclosure. Emits categorical results (`{category: pass|fail|N/A}`).
- **`cost.py`** — cost per trace derived from provider token usage. Emits `usd`, `input_tokens`, `output_tokens`, `cached_tokens`. No LLM call needed.
- **`latency.py`** — wall-clock time and per-tool-call breakdown from the span attributes. No LLM call needed.

The four scorers write their outputs back to the observability store (as span attributes or a related-record link) so the dashboards in [exercise-05](exercise-05-cross-persona-feature-dashboards.md) can query them.

### Deliverable 3 — The monthly budget with decision rules

Under `online/budget/<feature>.yaml`, a per-surface budget in the shape from [chapter 04](../04-sampled-online-eval-loop.md):

- `budget_usd` for the month.
- `allocation` split (base sampling, thumbs-down 100%, per-arm floor, post-incident reserve).
- `policy` block naming the decision at 80%, 100%, and 120% of budget (alert, reduce base rate, freeze new experiments) — **written before you need them**.
- `judge.model` pin with an explicit `pin_expires` date.

A `bin/budget_burn` script that queries the store and reports current-month burn against the budget, plus days remaining and projected end-of-month spend. The alerting from Deliverable 6 fires against this script.

### Deliverable 4 — The judge-drift monitor

Under `online/drift/`, three instruments:

- **`pin_check.py`** — asserts that the judge model ID and prompt hash in the pre-registration file match what the scorer is actually calling. A silent drift (someone bumped the pin without a review) fails the daily CI check.
- **`calibration_panel/`** — a fixed set of 30-50 fixtures with **human-authored ground-truth scores** (author them once, commit them, treat them as version-controlled). A `bin/re_score_panel` command scores the panel under a given judge model and reports the delta against human ground-truth. Any judge whose delta exceeds a documented threshold is *not* a drop-in replacement.
- **`shadow.py`** — during a judge upgrade window, score every sampled trace under both the current judge and the candidate judge. Log both scores to the store. The dashboards read the *current* judge; the alerts fire off the *current* judge; the candidate judge's scores are the material for the calibration-panel decision.

A `docs/judge-upgrade-runbook.md` naming the flow: (1) shadow the candidate for at least two weeks, (2) re-score the calibration panel, (3) if the delta is acceptable, bump the pin in a reviewed PR and re-baseline the offline snapshot graders ([exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md)), (4) annotate the dashboards.

### Deliverable 5 — The online → offline feedback path

Under `online/triage/`:

- A **curation queue** that receives low-quality-score or `thumbs_down` traces from the online loop.
- A weekly `bin/triage_review` runner that opens each queued trace in a review UI (or, for this exercise, a CLI review) and asks the applied engineer: promote to fixture, discard as one-off, or defer.
- A promotion path that runs the trace through the [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md) scrubber and appends it to the fixture set with a `source: online_loop_2026wNN` tag.

Prove the path end-to-end by promoting **at least one** online-loop trace into the offline fixture set during this exercise.

### Deliverable 6 — Alerting

Under `online/alerts/`, alert rules in the shape from [chapter 04](../04-sampled-online-eval-loop.md):

- `quality_drop_overall` — sustained (3 consecutive 24h windows) drop of ≥ 5 points on the median rubric score.
- `safety_incident` — any nontrivial rate of `safety.hijacked == True` over 1h.
- `cost_burn` — > 80% of monthly budget with > 10 days remaining.
- `latency_p95_slo` — three consecutive 1h windows above the SLO.
- `judge_drift_warn` — calibration-panel disagreement > baseline + 0.10.

Wire the alerts into your on-call system (PagerDuty, Opsgenie, a chat webhook — anything routable). Include the segment (persona / cohort) in the alert body so the on-call knows *where* the problem is.

## Starter guidance

- **Start with `cost` and `latency`.** They don't need judges; the plumbing is pure. Get the sampling → store → dashboard pipeline running on them before you spend a dollar on judged scoring.
- **Score 100% of thumbs-down.** The user's negative vote is the single strongest label you get for free. Priority-scoring it costs almost nothing and catches almost everything.
- **Budget the loop with real numbers.** Multiply sampling rate × traffic × judge cost per trace. If the number is > budget, cut the rate before you launch, not after finance calls.
- **Pin the judge with an expiry.** `pin_expires: 2027-01-31` forces a scheduled re-review. Without it, the pin outlives the person who chose it.
- **Author the calibration panel once, carefully.** 30-50 examples the applied engineer *and* one non-applied-engineer reviewer agree on. Human ground truth is expensive to create; treat it as versioned data and change it only through a reviewed PR.
- **Shadow, don't switch.** A judge upgrade is a two-week shadow-score followed by a re-baseline, not a config-file edit. The most common way to lose the ability to interpret your own dashboards is to swap the judge silently.
- **Version the promoted fixtures.** Every promoted-from-online fixture carries the online source tag; this is how you tell six months later that the fixture came from a real incident, not from a synthetic backlog.

## Acceptance criteria

- [ ] `online/sampling/policy.yaml` defines base rate, stratification, and per-stratum weekly floors.
- [ ] `sampler.py` reads traces from the observability store and emits the sampled subset.
- [ ] Four scorers (`quality`, `safety`, `cost`, `latency`) each write results back to the store; **no aggregated composite** score is reported.
- [ ] `online/budget/<feature>.yaml` includes budget, allocation, decision rules at 80/100/120%, and the judge pin with expiry.
- [ ] `bin/budget_burn` reports current burn vs. budget and projected end-of-month spend.
- [ ] `online/drift/pin_check.py` fails when the judge pin and the scorer configuration diverge.
- [ ] `calibration_panel/` contains 30-50 fixtures with human-authored ground-truth scores.
- [ ] `bin/re_score_panel` reports judge-vs-human delta.
- [ ] `shadow.py` scores every sampled trace under both current and candidate judges during an upgrade window.
- [ ] `docs/judge-upgrade-runbook.md` names the four-step upgrade flow.
- [ ] `online/triage/` receives low-quality / thumbs-down traces and offers a promotion path via the exercise-02 scrubber.
- [ ] At least one online-loop trace is promoted to the offline fixture set during this exercise.
- [ ] All five alerts fire on the correct condition; the alert body includes the affected segment.

## Stretch goals

Pick any two:

- **A `budget-vs-actual` weekly report.** A generated Markdown report each week comparing planned spend (per allocation bucket) to actual, plus the projected end-of-month burn. Committed to the repo; visible in the PR review of any budget-change PR.
- **A `pin_expires` warning cascade.** Warns two weeks before a judge pin expires; hard-fails the daily CI check the day after expiry. Forces the applied engineer to make a decision (renew the pin, upgrade the judge, or accept the risk with a written note).
- **A "silent alias" detector.** A daily job that hits the provider with the pinned model ID and asserts the returned model metadata matches the pin. A provider that silently rolls the alias behind a "-latest" style ID is caught within a day.
- **A per-persona quality alert.** Split `quality_drop_overall` by persona and fire per-persona alerts on segments the aggregate would hide.
- **A budget-elastic sampling loop.** When the burn projection exceeds 100% with more than 10 days remaining, automatically reduce the base sampling rate by a factor of two and notify the on-call. When the burn projection drops back below 90%, restore. This is the closed-loop version of the decision rules.
- **A "judge disagreement over time" trend chart.** Beyond a single calibration-panel score, chart the judge-vs-human agreement rate across every re-score of the panel. Sudden drops correlate with judge-model version bumps at the provider even if you did not explicitly change the pin.

## Deliverable

Submit as `mod-106/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `online/sampling/policy.yaml` and `sampler.py`.
- `online/scoring/{quality,safety,cost,latency}.py`.
- `online/budget/<feature>.yaml` and `bin/budget_burn`.
- `online/drift/{pin_check.py, shadow.py}`, `calibration_panel/`, and `bin/re_score_panel`.
- `online/triage/` with the promotion path.
- `online/alerts/` with the five alert rules and the on-call integration.
- `docs/judge-upgrade-runbook.md`.
- Proof: `proofs/promoted-fixture.md` (the promoted trace + the resulting fixture), `proofs/shadow-run.md` (two weeks of shadow-score comparison, or a synthetic equivalent if the exercise is time-boxed), `proofs/budget-burn.md` (current-month burn output).
- A short `README.md` naming the observability substrate, the two judge models tested, and how to run the loop locally.
- Any stretch-goal artefacts named alongside.
