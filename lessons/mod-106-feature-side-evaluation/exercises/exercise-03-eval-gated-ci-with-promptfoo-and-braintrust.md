# exercise-03: Eval-Gated CI With Promptfoo and Braintrust

**Estimated effort:** 3 hours

## Objective

Wire the offline suite from [exercise-01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md) and the trace snapshots from [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md) into a **two-track CI gate** on a real CI system (GitHub Actions, GitLab CI, or an equivalent), using **at least two** of the runners from [chapter 03](../03-eval-gated-ci-end-to-end.md). One track is a fast gate (programmatic + canary) that runs unconditionally; the other is a deep gate (judge + rubric + snapshot replay + safety) that runs on relevant paths and posts a diff comment to the PR.

## Learning outcomes

By the end you should be able to:

- Author the trigger set (path filters) that fires the eval on prompt / model / tool / retriever / agent / eval-suite changes and not on unrelated PRs.
- Build the two-track shape: fast gate (< 5 min, unconditional, required) and deep gate (< 30 min, path-triggered, required to merge).
- Configure at least two runners (Promptfoo + Braintrust is the canonical pair; substitute LangSmith, Langfuse, Phoenix, Weave, or DeepEval if your stack demands) and know why you'd pick each.
- Produce a PR comment that is *scannable* (pass/fail per pre-registered gate) and *drillable* (per-fixture diff against a pinned baseline).
- Enforce judge-cost discipline via `(output_hash, judge_prompt_hash, judge_model_id)` caching and a per-PR cost line on the comment.

## Prerequisites

- You have completed [exercise-01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md) (offline suite + pre-registration file) and [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md) (trace snapshots + replay).
- You have a GitHub repository (or equivalent) where you can add workflows and secrets.
- Accounts on at least two of the runners named in [chapter 03](../03-eval-gated-ci-end-to-end.md). Free tiers are typically enough for this exercise.
- Provider API keys for both the model under test and the judge model.

## Problem statement

Product leadership approved the eval work from exercises 01 and 02 but observed *"nobody's running this on PRs"*. Your task in three hours is to close the loop: fire the suite automatically on every PR that could plausibly move the score, block merge on a hard-gate regression, and give the reviewer a diff they can act on without leaving the PR page. You will demonstrate the gate holds on a red-team PR and passes on a green-team PR.

## Requirements

### Deliverable 1 — The trigger set

Under `.github/workflows/eval.yml` (or your CI equivalent):

- Path filters that fire on `prompts/**`, `tools/**`, `retriever/**`, `agent/**`, `eval/**`, `fixtures/**`, and the model-config file(s) — and nothing else.
- A separate `smoke-test` job that runs on every PR regardless of paths (the canary sub-suite from [chapter 03](../03-eval-gated-ci-end-to-end.md)).

Document the trigger set in `docs/triggers.md` — the *why* of each path, plus the anti-goal (which PRs *should not* fire the eval, and why).

### Deliverable 2 — The fast gate

A CI job `fast-gate` that:

- Runs the programmatic-check tier and the canary sub-suite (10-20 fixtures, no judge calls).
- Completes in under 5 minutes end-to-end (log it in the workflow output).
- Is a **required** check on the branch protection rule (screenshot or config export attached).
- On failure, blocks the deep gate from running.

### Deliverable 3 — The deep gate on two runners

A CI job `deep-gate` that:

- Runs on the same trigger set as the workflow.
- Uses **at least two runners** end-to-end. The canonical pair:
  - **Promptfoo** for the LLM-as-judge + rubric tier (the config-file discipline is the pre-registration file made mechanical).
  - **Braintrust** for the experiment / diff view (or substitute LangSmith / Langfuse / Weave / Phoenix / DeepEval per your team).
- Runs the trace-snapshot replay (from [exercise-02](exercise-02-trace-snapshot-as-regression-fixture.md)) in `frozen` mode.
- Runs the safety spot check.
- Applies the pre-registration policy from `policies/<feature>.yaml` via a script.
- Exits non-zero on any hard-gate failure; exits with a warning-but-pass on soft-gate failure that carries a written override comment on the PR.

Document your runner choice in `docs/runner-choice.md` using the five-question decision procedure from [chapter 03](../03-eval-gated-ci-end-to-end.md).

### Deliverable 4 — The scannable, drillable PR comment

A comment posted (or updated in place) on every PR with:

- The suite version and the baseline commit (`main` at the fork point).
- The judge model pin.
- A pass/fail line per pre-registered gate.
- A "Regressed fixtures" section naming every fixture that flipped `pass → fail` (with tags flagged; `[critical]` visible).
- A cost line: judge tokens spent and the dollar equivalent for this PR run.
- A link to the deep-runner's full experiment / dashboard view.

The comment is updated in place on subsequent pushes, not appended.

### Deliverable 5 — Judge-cost discipline

Show that the judge is **cached** across pushes:

- Cache key includes `(candidate_output_hash, judge_prompt_hash, judge_model_id)`.
- Cache is scoped to the PR (a fresh PR does not steal another PR's cache); durable across pushes within the PR.
- The PR comment reports the cache-hit rate.
- Push twice without changing any prompt / model config; the second push's judge cost is near zero.

### Deliverable 6 — The proof it holds

Under `proofs/`:

- **`red-team-pr.md`** — link to a PR that weakens the surface. Show the deep gate blocks merge, the comment names the failing fixtures, and the branch protection prevents merge until the regression is reverted or the pre-registration is intentionally updated (with the review the policy demands).
- **`green-team-pr.md`** — link to a PR that makes a stylistic-only edit. Show the fast gate + deep gate pass.
- **`cost-discipline.md`** — the cache-hit-rate line from two consecutive pushes on the same PR, showing near-zero incremental judge cost on the second push.

## Starter guidance

- **Start with the fast gate.** Get *one* required check firing on every PR in under 5 minutes before you touch the deep gate. Half the value of eval CI is the fast gate.
- **Pin secrets to the deep gate only.** The judge API key should not be exposed to fork PRs or to the fast gate. Use `secrets:` scoping.
- **Point the deep gate at a pinned baseline.** `main` at the fork point, not `main@HEAD`; a moving baseline makes the diff comment lie.
- **Use the runners' native CI integrations first.** Promptfoo ships a GitHub Action; Braintrust ships a GitHub Action; both handle the comment posting. Custom PR comments should be a *last* resort.
- **Cost-cap the deep gate.** A `budget_usd_per_run` config value that hard-stops the runner if it exceeds the cap. A runaway judge on one PR should not eat the month's budget.
- **Guard against fork abuse.** PRs from forks should not run the deep gate (or should run against a mocked judge). A malicious fork should not be able to spend your judge budget by opening a PR that pushes 200 times.
- **Version your runner configs.** The Promptfoo config, the Braintrust experiment spec, the pre-registration file — all versioned in the repo, all reviewed alongside the change. A change to any of them is a suite-behaviour change.

## Acceptance criteria

- [ ] `.github/workflows/eval.yml` (or CI equivalent) is committed and enabled.
- [ ] Path filters fire the deep gate on the six trigger paths and no others.
- [ ] `fast-gate` runs under 5 minutes and is a required check.
- [ ] `deep-gate` uses at least two runners (with `docs/runner-choice.md` defending the pair).
- [ ] The deep gate runs the trace-snapshot replay in `frozen` mode.
- [ ] The pre-registration policy is applied by a script; hard-gate failures exit non-zero.
- [ ] The PR comment is scannable (per-gate pass/fail), drillable (per-fixture diff), and includes the judge model pin and the run cost.
- [ ] The judge cache reduces incremental cost on a repeat push to near zero.
- [ ] `proofs/red-team-pr.md` shows the gate blocks a real regression.
- [ ] `proofs/green-team-pr.md` shows a neutral PR passes.
- [ ] Judge secrets are scoped to the deep gate; the fast gate cannot leak them.
- [ ] Fork PRs cannot spend the judge budget (documented behaviour + policy).

## Stretch goals

Pick any two:

- **A soft-gate override discipline.** When a soft gate warns (e.g. cost regressed by > 20%), the PR must include a `# eval-override:` block in the description naming the metric and justifying the change. The runner refuses to pass the PR without the block. This is the analog of the `# breaking-change:` block for schema changes.
- **A "must-not-regress" fixture tagging.** Beyond the `critical` tag, allow per-fixture overrides in the pre-registration file (`must_not_regress: [T-4713, T-4720]`). A regression on any listed fixture fails the gate irrespective of the aggregate pass rate.
- **A dependency-bump gate.** When `requirements.txt` / `package.json` bumps an LLM SDK, a provider client, or a runner library, run the deep gate anyway (the change can silently move behaviour). Add the paths to the trigger set with a comment explaining why.
- **A cross-provider CI matrix.** For a feature designed to run on more than one provider (mod-108), the deep gate runs the suite against each provider in a matrix and posts a per-provider report. The pre-registration file can name provider-specific thresholds.
- **A scheduled trace-replay nightly.** In addition to the PR-triggered deep gate, a scheduled job (nightly at 3am) replays the full snapshot set against `main` in `live` mode (against the current retriever + tool backends). Failures page the applied-eng-oncall.

## Deliverable

Submit as `mod-106/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `.github/workflows/eval.yml` (or CI equivalent).
- `docs/triggers.md`, `docs/runner-choice.md`.
- Runner configs (e.g. `promptfooconfig.yaml`, `braintrust/experiments/*.py`).
- The policy-application script (`bin/apply_policy.py` or equivalent).
- The PR-comment generator (or a config for the runner's native comment).
- The judge cache implementation (if not provided by the runner).
- `proofs/red-team-pr.md`, `proofs/green-team-pr.md`, `proofs/cost-discipline.md` with links to real PRs.
- A short `README.md` naming the runners, the trigger paths, and how to reproduce locally.
- Any stretch-goal artefacts named alongside.
