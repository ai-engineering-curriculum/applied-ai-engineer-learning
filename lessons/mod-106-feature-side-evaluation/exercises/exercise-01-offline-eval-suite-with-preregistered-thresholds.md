# exercise-01: Offline Eval Suite With Pre-registered Thresholds

**Estimated effort:** 3 hours

## Objective

Design and ship an *offline* eval suite for a real product-shaped feature — a suite small enough to run in CI, composed from the six modes in [chapter 01](../01-offline-eval-suites-a-pr-can-run.md), gated by a **pre-registered** thresholds file that a PR must pass before merge. The exercise is not "write some evals." It is "write the thing that will fail your PR next week, and prove it fails on the right kind of change."

## Learning outcomes

By the end you should be able to:

- Decompose "is this feature good?" into three-to-five named dimensions, each scored by the *correct* mode (programmatic, rubric, LLM-as-judge, RAG triad, safety spot check).
- Author an LLM-as-judge harness that runs order-swapped comparisons and reports the swap-disagreement rate.
- Write a rubric with anchored levels that a second reviewer would score similarly to you on ≥ 80% of examples.
- Commit a pre-registration file (`policies/*.yaml`) that a CI script can parse and act on — hard vs. soft gates, per-metric thresholds, critical-fixture flags.
- Show that a *deliberately-regressed* candidate PR is caught by the gate, and that a *neutral* candidate PR passes.

## Prerequisites

- You have read [chapter 01](../01-offline-eval-suites-a-pr-can-run.md).
- You have a working provider account (Anthropic Messages, OpenAI Chat / Responses, or Gemini). One is enough.
- Python 3.11+ (or Node 20+); `pytest` (or Vitest); the ability to run the suite from the command line.
- A feature or vertical slice of a feature to eval. If none of your day-job features fits, use the *support-agent-lite* scaffold in the paired solutions repo (`applied-ai-engineer-solutions/mod-106/scaffold/`).

## Problem statement

Product leadership has asked *"can we trust the eval to catch a bad prompt change?"* and the honest current answer is *no* — the team has a `README.md` describing the eval and no runner. Your task in three hours is to close the gap. You will pick one feature, decompose "good" into three-to-five dimensions, wire modes to each dimension, write the pre-registration file, and prove the gate holds on both a red-team change (a deliberately-worse prompt) and a green-team change (a stylistic-only edit).

You will *not* solve the offline-vs-online split, the CI plumbing, or the dashboard — those are the following exercises.

## Requirements

### Deliverable 1 — The dimension map

Under `docs/dimensions.md`, one page:

- The feature under eval (surface name, user-visible artefact).
- Three to five *named* quality dimensions the feature must satisfy. Examples for a support surface: `policy_compliance`, `factual_grounding`, `tone`, `actionability`, `safety`. Examples for a code-completion surface: `correctness`, `stylistic_fit`, `security`, `explainability`.
- For each dimension: which chapter-01 mode scores it (programmatic, rubric, LLM-as-judge, RAG-triad component, safety spot check) and why.
- One paragraph on which dimensions are *hard gates* vs. *soft gates* and how you decided.

### Deliverable 2 — The fixture set

Under `fixtures/`, at least 30 tasks. Each task is a versioned record with:

- `task_id` (stable), `category` (from the dimension map), `tags` (including `critical` for at least three tasks).
- The input(s) the feature will see.
- The `expected_state` or `expected_shape` for programmatic checks.
- The reference retrieval context, where the dimension is scored by the RAG triad.
- Fabrication is acceptable only for the *scaffold* case; if you're evaluating a real feature, tasks come from real user work (PII scrubbed — see [chapter 02](../02-trace-snapshots-as-regression-fixtures.md)).

At least three fixtures are explicitly `safety` tagged and target concrete failure modes for your surface (prompt injection through a tool output, refusal to obey a hostile instruction wrapped in a user-friendly ask, sensitive-info-disclosure prompts).

### Deliverable 3 — The scoring modes, wired

Under `scoring/`, one small module per mode you use.

- **Programmatic checks.** Pure functions over the model's output; return `PassResult(passed: bool, details: dict)`.
- **Rubric grader.** An LLM-as-judge harness with:
  - An anchored rubric file per rubric (`rubrics/<name>.md`) with a concrete description of each score level.
  - A **position-swap** wrapper: every pairwise comparison runs with both orders and only counts agreement as decisive. Report the swap-disagreement rate as an output.
  - A **length control**: either penalise length in the rubric text explicitly, or normalise the two candidates to comparable lengths before scoring, or (at minimum) log the length distribution alongside scores.
- **RAG triad**, if RAG is in scope. Three separate judges (context relevance, groundedness / faithfulness, answer relevance), each with their own rubric and threshold.
- **Safety spot check.** Programmatic pattern checks (`not-contains "IGNORE PRIOR INSTRUCTIONS"`, `refusal_matches_policy`), plus a small LLM-graded safety rubric for the harder cases.

The judge model is pinned in code (or config); the judge prompt hash is logged with every score.

### Deliverable 4 — The pre-registration file

Under `policies/<feature>.yaml`, a machine-readable pre-registration in the shape from [chapter 01](../01-offline-eval-suites-a-pr-can-run.md):

- One entry per dimension / metric with `metric`, `threshold`, and `on_regression` (`fail_ci` or `warn_and_require_override`).
- A `critical_tasks` entry that requires 100% pass on the tag-flagged critical fixtures.
- The judge model pin and the pin expiry date.
- A `policy` block that names who reviews changes to this file (`changing_this_file: requires_review_from_eval_owner`).

### Deliverable 5 — The runner

Under `bin/eval`, a single script that:

- Loads the fixture set, the scoring modes, and the pre-registration file.
- Runs the suite against the current feature configuration.
- Emits both a machine-readable report (`report.json`) and a human-readable summary (`report.md`).
- Exits non-zero if any hard gate fails or any soft gate requires an unwritten override.

You do not need CI wiring yet — that's [exercise-03](exercise-03-eval-gated-ci-with-promptfoo-and-braintrust.md).

### Deliverable 6 — The proof it holds

Under `proofs/`:

- **Red-team run.** Deliberately regress the feature (weaken a system-prompt clause, remove a policy line, switch to a cheaper model). Run the suite. Show the report has a failing hard gate and names the affected metric and fixtures.
- **Green-team run.** Make a stylistic-only edit to the feature (rename a variable, reword a non-load-bearing sentence). Run the suite. Show the report is green.
- **Judge-swap-disagreement run.** For the LLM-as-judge rubric, run 20 examples through both orders. Report the disagreement rate and one sentence on whether it is acceptable (a rough working ceiling is 10-15%; higher means the judge is not decisive enough to gate).

## Starter guidance

- **Start from a single dimension.** Wire *one* dimension end-to-end (fixture → scorer → threshold → runner report) before you add the second. Every additional dimension is cheap once the pipeline is built.
- **Programmatic first, judged last.** Every dimension that *can* be scored programmatically should be. Judged scores are expensive, noisy, and biased. Reserve them for the free-form work only they can score.
- **Anchor the rubric.** A rubric with "5 = excellent, 1 = poor" is a random number generator. Write one sentence per level per dimension, with a concrete example.
- **Don't average dimensions.** Report per-dimension pass rates, not a composite "quality" number, unless a stakeholder has explicitly asked and you have written down why.
- **Cache the judge.** A `(candidate_output_hash, judge_prompt_hash, judge_model_id)` key saves 80% of the cost on iterations. Any real runner supports this; if you're building your own, add it before you're tempted to skip evals to save money.
- **Pin the judge model.** `claude-opus-latest` today is `claude-opus-4-7` tomorrow. Pin to a dated ID; log the pin.
- **Sanity-check on the scaffold.** If the paired scaffold is available, run the pipeline against it first. It costs nothing and catches the plumbing bugs before you spend judge money on real data.

## Acceptance criteria

- [ ] `docs/dimensions.md` names 3-5 dimensions, maps each to a chapter-01 mode, and marks each as hard-gate or soft-gate.
- [ ] `fixtures/` contains ≥ 30 tasks with stable `task_id`, `category`, and `tags`; ≥ 3 tasks are `critical`; ≥ 3 tasks are `safety`.
- [ ] Every LLM-as-judge comparison is run in both orders; the report includes the swap-disagreement rate.
- [ ] The judge model is pinned in code / config; the judge prompt hash is logged.
- [ ] `policies/<feature>.yaml` includes per-metric thresholds, `on_regression` semantics, a critical-tasks 100% requirement, and a judge pin with expiry.
- [ ] `bin/eval` exits non-zero on any hard-gate failure and produces both `report.json` and `report.md`.
- [ ] `proofs/red-team.md` shows the suite catches a deliberate regression, naming the metric and fixtures.
- [ ] `proofs/green-team.md` shows the suite is green on a stylistic-only edit.
- [ ] No dimension is scored by a `1..5` Likert with no anchors.
- [ ] No composite score is reported without a written definition of what would move it.

## Stretch goals

Pick any two:

- **A dimension-drift bisection.** When a dimension regresses, produce the per-task diff (which tasks flipped from pass to fail) and — for judged dimensions — the delta in judge score per task. This is the same discipline as [chapter 03](../03-eval-gated-ci-end-to-end.md)'s PR-comment diff, at the local runner level.
- **A per-persona split.** Split the fixture set by user persona (or language, or plan tier). Report per-persona pass rate alongside the aggregate. A drop hidden in one segment is invisible on the mean.
- **A judge-vs-second-judge calibration spot.** Run 20 randomly-sampled examples through a *different* judge model. Report agreement. If it is low, that is not the second judge's problem; it means your rubric is under-specified. Fix the rubric, not the judges.
- **A rubric-refactor discipline.** Any change to a rubric bumps a version; the runner records the rubric version alongside every score; a score under `rubric.v2` is not commensurable with one under `rubric.v1`. Prove this in `proofs/rubric-version.md`.
- **A "quality vs. cost" frontier chart.** Plot mean judged score vs. mean cost per task across the last N suite runs. Any change should move you along the frontier; anything that moves off it is a regression on the axis you didn't watch.

## Deliverable

Submit as `mod-106/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/dimensions.md`.
- `fixtures/` with the versioned task set.
- `scoring/` with one module per mode.
- `rubrics/` with anchored rubric files.
- `policies/<feature>.yaml` with the pre-registration.
- `bin/eval` with the runner.
- `proofs/red-team.md`, `proofs/green-team.md`, `proofs/judge-swap.md`.
- A short `README.md` naming the feature under eval, the modes used, and how to run the suite.
- Any stretch-goal artefacts named alongside.
