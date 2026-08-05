# exercise-04: Product-Agent Eval Suite Derived from Benchmark Shapes

**Estimated effort:** 4 hours

## Objective

Build an internal eval suite for one specific product-agent surface — the one you chose in [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md) — using one of the four public benchmark shape templates from [chapter 04](../04-eval-suites-from-benchmark-shapes.md) as your structural model. You are not topping τ-bench or WebArena. You are *stealing the shape* — task specification, environment, tool catalogue, success criterion, user / attack simulator — and rebuilding it from your product's tasks. The output is a runnable suite that gates a release.

## Learning outcomes

By the end you should be able to:

- Pick a public benchmark whose *shape* matches your surface and defend the choice.
- Turn a real-world backlog of user tasks into a versioned fixture set with typed environment state, typed tools, and typed success criteria.
- Build a user simulator (or scripted user) whose behaviour is at least as representative as your production traffic.
- Design a scoring function that reports mean success, pass^k, cost per task, latency per task, and (for adversarial suites) utility-under-attack + hijack rate.
- Wire the suite into a CI gate that a model / prompt / tool change must pass to ship.
- Argue publicly why a public-benchmark score is not the same signal as your suite's score.

## Prerequisites

- You have read [chapter 04](../04-eval-suites-from-benchmark-shapes.md).
- You have a surface picked from [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md) (or an equivalent). You know its shape.
- You have Python 3.11+ (or Node 20+), the LLM provider SDK of your choice, and a way to run a small parallel worker pool (`asyncio.gather`, `concurrent.futures`, or a job queue).
- You have `pytest` (or Vitest) and access to a CI runner (GitHub Actions, GitLab CI, or a local `Makefile` that mimics one).

## Problem statement

Product leadership is pressing for a release. The agent has looked "good in demos" for three months. Support has been catching bad outcomes one at a time. The eng team is about to ship a model upgrade and the CEO wants to know if it's safe.

You have been asked to build the eval suite that answers the question *"is this surface good enough to ship?"* — not "is this model good at agents?" and not "did we top the leaderboard?" You have three days. This exercise is the first afternoon of the three, focused on the shape.

You will pick **one** of the four templates from [chapter 04](../04-eval-suites-from-benchmark-shapes.md), adapt it to your surface, and build a working suite of at least 30 tasks with automated success criteria. If your surface exposes to prompt injection (any agent that reads user- or third-party-controlled text — most of them), you will add an AgentDojo-shaped adversarial mode as a secondary suite.

## Requirements

### Deliverable 1 — The template choice

Under `docs/template.md`, one page:

- Which template you chose (τ-bench, SWE-bench Verified, WebArena, AgentDojo) and why. Reference the shape from your exercise-01 memo. If you fused (τ-bench + AgentDojo, for instance), name both and defend the composition.
- What you kept (the shape's structural elements). What you dropped (the benchmark's specific data).
- What the equivalent public benchmark score for your target model looks like *today*, cited from the current live benchmark repo. You are not going to beat it on your suite; you are noting the gap between shape and target.

### Deliverable 2 — The task fixture

Under `fixtures/`, at least 30 tasks (aim for 50). Each task is a structured record — YAML, JSON, or a Python dataclass file — with the shape template's five parts:

```yaml
# fixtures/tasks/T017-refund-out-of-window.yaml — illustrative, not literal for your surface.
task_id: T017-refund-out-of-window
category: refund
tags: [policy_edge_case, high_stakes]
description: |
  Customer requests a refund for an order placed 45 days ago; the policy window is 30 days.
initial_state:
  # A serialisable snapshot of the environment before the task begins.
  ...
tools_available:
  - search_orders
  - lookup_customer
  - issue_refund
  - escalate_to_human
success_criteria:
  - env_check: no refund issued
  - env_check: escalation ticket created with reason "out_of_policy_window"
  - rubric: agent explained the policy to the customer without blaming them
user_simulator:
  persona: frustrated_regular_customer
  script:
    - initial_message: "I want a refund for order ord_..."
    - reactions:
        on_agent_offering_alternative: accept
        on_agent_escalating: accept
```

Requirements on the fixture:

- Tasks come from a *real* backlog. Preferred sources, in order: your product's ticket / issue tracker, PII-scrubbed session transcripts, published tasks from a competitor's public docs. Do not fabricate 50 tasks from scratch; that fixture measures your imagination, not your product.
- Categories span the shape's key axes. For τ-bench-shape: policy-edge / clean-happy-path / user-hostile / tool-failure. For SWE-bench-shape: bug-fix / feature-add / refactor / dependency-upgrade. For WebArena-shape: find / create / edit / delete × depth-1 / depth-3+. For AgentDojo-shape: inject-in-tool-result / inject-in-document / inject-in-web-page × goal-off-task / goal-exfil.
- The success criteria are *state* checks against the environment plus (optionally) rubric checks. String-match gold answers are not acceptable — most product tasks have many acceptable outcomes.
- The task file is versioned in git; every task has a stable `task_id`.

### Deliverable 3 — The environment(s)

Under `environments/`, a runnable staging copy of the environment your agent operates in. This is the hardest deliverable and the one people cut corners on.

- Never use production data or production endpoints. If your shape is τ-bench-like (CX agent), stub a customer / order / policy backend that mirrors production's schema. If your shape is SWE-bench-like (coding agent), snapshot repo states and run against containers. If WebArena-like, host a fixed copy of the target site or a realistic clone. If AgentDojo-like, wrap your real tool catalogue behind a proxy that injects the adversarial payloads.
- The environment must be **reset per task**. A test that leaks state to the next task is a broken test.
- The environment must have a **state snapshot API** so the scoring function can compare `state_after` to `state_expected`.

### Deliverable 4 — The user / attack simulator

Under `simulators/`. Different shapes need different simulators.

- For a τ-bench-shape suite: an LLM-driven user simulator (a cheaper model with a persona prompt) that carries a conversation according to the task's `user_simulator` block. The simulator is deterministic given a seed; test runs are reproducible.
- For a SWE-bench-shape suite: no interactive simulator; the task's PR body / issue text is the simulator's whole input.
- For a WebArena-shape suite: a scripted intent + a policy that decides whether an agent's clarifying question requires an answer.
- For an AgentDojo-shape suite: an *attacker* that injects payloads at the specified read surfaces at the specified turns.

### Deliverable 5 — The scoring function

Under `scoring/`.

- `score_task(task, transcript, state_after) -> TaskResult` — per-task result including: `passed: bool`, `env_check_results: dict`, `rubric_score: float | None`, `tokens_in`, `tokens_out`, `dollar_cost`, `wall_clock_ms`.
- `aggregate(results) -> SuiteResult` — mean pass rate, pass^k for k=3 and k=5 (run each task 5 times with different simulator seeds), cost per task (mean and p95), latency per task (mean and p95).
- For an adversarial suite: `utility_clean`, `utility_under_attack`, `hijack_rate` (fraction of runs that took the injection's suggested off-task action).
- Every metric is versioned; a metric renamed is a metric added, not a metric changed.

### Deliverable 6 — CI gating

Under `.github/workflows/eval.yml` (or your CI equivalent):

- A workflow that runs the eval suite on every PR that touches the agent's prompts, tool schemas, or model configuration.
- A results comment on the PR: mean pass rate, pass^k for a canary subset, mean cost per task, mean latency per task.
- A failing status check if the mean pass rate on the canary subset drops by more than X% (you pick X and defend it in `docs/gate.md`).
- A "must-not-regress" flag: named critical tasks (T-001 through T-005, say) whose pass rate must not drop from the baseline, or the check fails.
- A "cost budget" flag: mean tokens-per-task must not exceed the baseline by more than Y% without an override PR.

### Deliverable 7 — The provenance / limitations write-up

Under `docs/limitations.md`, one page. Answer honestly:

- **What tasks are missing.** Which categories of user work does the suite not cover? What are the failure modes it cannot catch?
- **How representative is the fixture.** Was the sampling random, quota-based, or purposive? What's the ratio of edge-case-heavy to happy-path-heavy tasks vs. production traffic?
- **The public-benchmark relationship.** One paragraph on why the public benchmark you inherited the shape from does *not* substitute for this suite. Reference [chapter 04](../04-eval-suites-from-benchmark-shapes.md).
- **What breaks the suite.** Under what changes to the surface would you have to rebuild significant portions?

## Starter guidance

- **Start with the environment, not the tasks.** If you cannot spin up a fresh copy of the environment per task, you cannot score anything. Get the reset story working before you write a single task.
- **Start small.** Ten tasks that work end-to-end are worth more than 50 tasks that don't. Add tasks after the harness runs.
- **State-based success is non-negotiable.** If you find yourself writing "the transcript should mention X," you are drifting into rubric territory. Rubrics are fine as a *secondary* metric; the primary metric is state.
- **Simulator quality bounds suite quality.** A τ-bench-shape suite is only as honest as its user simulator. If the simulator is boring and cooperative, the score is a lie. Add a "hostile / confused / non-native-speaker" persona; run each task against multiple personas.
- **Prompt-injection lives in the tool result surface.** If your surface reads any user-controlled text through a tool, an AgentDojo-shape mode is not optional. Set up the injection proxy from day one.
- **Version the suite.** A pass rate is meaningless without the suite version. Commit the fixture set; tag suite versions; report scores as `pass@task_v3 = 62%`.
- **Do not run the suite against a live model in the design phase.** Once the harness works with a stub agent, wire in the model. This is the same pipeline discipline as mod-106 recommends.

## Acceptance criteria

- [ ] The template choice is documented and defended, with a citation of the current public benchmark score for the target model.
- [ ] The fixture has at least 30 tasks, each with a `task_id`, `initial_state`, `tools_available`, `success_criteria`, and (where applicable) a `user_simulator`.
- [ ] The environment resets per task; no test leaks state to the next.
- [ ] The simulator (where applicable) is deterministic given a seed.
- [ ] The scoring function reports per-task pass, aggregate mean, pass^k for at least k=3, mean and p95 cost per task, mean and p95 latency per task.
- [ ] For any surface that reads user-controlled text: an AgentDojo-shape adversarial mode reports utility-clean, utility-under-attack, and hijack rate.
- [ ] CI runs the suite on relevant PRs and posts the summary; a canary regression fails the check.
- [ ] Critical tasks flagged in the fixture are enforced as "must not regress" in CI.
- [ ] The limitations write-up honestly names missing categories, sampling bias, and the public-vs-internal-benchmark relationship.
- [ ] No task's success criterion is an exact-string match on the transcript.
- [ ] No fixture uses production data.

## Stretch goals

Pick any two:

- **Pass^k as a first-class chart.** Instead of reporting only mean pass, chart pass^1 through pass^5 per task category. High-variance categories are where the model's reliability, not its capability, is the bottleneck.
- **A "clean vs. under-attack" chart.** For the adversarial mode, chart utility-clean vs. utility-under-attack per category. Categories where they diverge sharply are your injection weak points.
- **Regression bisection.** If the suite score drops on a PR, produce a per-task diff between the PR run and the baseline: which tasks flipped from pass to fail? This is what makes the failing CI check actionable.
- **A cost/pass frontier chart.** Plot mean cost per task against mean pass rate across the last N suite runs. Model / prompt changes should move you along the frontier; anything that moves off it is a regression.
- **Simulator diversification.** Run each task with 5 different personas and report per-persona pass rate. Categories where one persona catches failures the others miss are your representativeness weak points.
- **A hidden holdout.** Split the fixture 80/20; only the 80% is visible during development; the 20% runs on a release-candidate build. This is the SWE-bench Verified discipline applied inward — a check on whether the visible suite has quietly become a training signal.
- **The "what breaks tomorrow" table.** For each of the top 5 categories in your fixture, one line on the failure mode that would take the category from 90% to 50%. This is the honest cousin of the limitations doc.

## Deliverable

Submit as `mod-105/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/template.md`, `docs/gate.md`, `docs/limitations.md`.
- `fixtures/` with the task set (versioned).
- `environments/` with the staging environment(s) and their reset APIs.
- `simulators/` with the user / attack simulators.
- `scoring/` with the scoring and aggregation functions.
- `.github/workflows/eval.yml` (or CI equivalent).
- A short `README.md` naming the surface (from exercise-01), the chosen shape template, and how to run the suite locally.
- Any stretch-goal artefacts named alongside.
