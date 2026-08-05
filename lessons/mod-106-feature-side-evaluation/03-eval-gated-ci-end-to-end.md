# Eval-Gated CI End-to-End

## The point of a "gate"

The offline suite from [chapter 01](01-offline-eval-suites-a-pr-can-run.md) and the trace snapshots from [chapter 02](02-trace-snapshots-as-regression-fixtures.md) only earn their keep when they *block bad merges*. A suite that runs on a manual button, that a reviewer optionally consults, that fails but does not fail the CI check, is a dashboard — not a gate. The applied engineer's job in this chapter is to wire the suite into the same CI pipeline that runs `pytest` and `eslint`, so that a prompt / chain / agent change gets an eval report before merge, and a regression on a critical fixture turns the check red.

This is a boring plumbing chapter, and that is the point. Eval-gated CI is not a research problem; it is an integration problem, and the integrations are well-supported by half a dozen mature runners. The applied engineer's decision is picking the runner that matches the surface's *observability substrate* — because your traces need to end up in the same store your eval scores end up in.

## What "triggers" an eval run

An eval run should fire on any change that can plausibly move eval scores. The trigger set for a shipped LLM feature typically includes:

- **Prompt changes** — anything under `prompts/`, `system_prompt.py`, template files.
- **Model configuration changes** — model IDs, routing rules (mod-108), decoding parameters.
- **Tool schema and implementation changes** — anything under `tools/` (mod-104).
- **Retriever changes** — chunker configuration, embedder model, index build config, reranker (mod-103).
- **Agent orchestration changes** — the graph, the memory reducer, the human-in-the-loop policy (mod-105).
- **Eval suite changes themselves** — a change to fixtures, graders, or the pre-registration file bumps the suite version and re-baselines.

A path-based trigger in the CI config keeps the eval from running on unrelated PRs (documentation, unrelated backend code, dependency bumps that don't touch the model). A canary sub-suite — 10-20 fixtures that take under two minutes to run — should still fire on every PR as a smoke test, even on "unrelated" changes; that catches the accidental config drift that a path filter would miss.

## The two-track CI shape

For any nontrivial surface, run **two tracks** in CI, not one.

**Track A — the fast gate.** Programmatic-check tier (mode 3 from chapter 01), the canary sub-suite, and any smoke tests. Runs on every PR, every commit, unconditionally. Target completion time: under five minutes. This is what the reviewer waits on.

**Track B — the deep gate.** Full judge-and-rubric tier, RAG triad, safety spot checks, and the trace-snapshot replay set (chapter 02). Runs on any change matching the trigger set above. Target completion time: under thirty minutes. This is what blocks merge.

Both tracks post their results to the PR (a comment or a check-run summary). The fast gate returns quickly enough that "the eval is slow" isn't a reason to skip it. The deep gate is where the money is — both in dollars (the judged tiers cost a few dollars a run) and in signal.

## Runners: choosing the tool

Six runners are named in the module's learning objectives. Each is credible; none is universally best; the correct choice usually falls out of *what trace/store surface your team is already using*. A short field guide follows. Consult the current live docs before pinning behaviour — every one of these tools has moved fast enough that details written today may be stale in two quarters.

### Promptfoo

**What it is.** An open-source, YAML-driven test runner for prompts, RAG pipelines, and agents. Ships as a CLI (`promptfoo eval`) and a self-hosted web viewer. Assertions include exact match, contains, JSON validity, JavaScript / Python callbacks, LLM-graded rubrics, factuality checks, and cost / latency thresholds.<!-- needs-research: confirm the current Promptfoo assertion catalogue and its GitHub Action integration URL against the live docs. -->

**Where it fits.** The right first tool for teams that want the offline suite in a repo, versioned, runnable locally and in CI, with zero managed infrastructure. Its config file *is* the pre-registration file from chapter 01. Good default for prompt-heavy surfaces.

**Sketch of a Promptfoo config a PR gate would use:**

```yaml
# promptfooconfig.yaml — illustrative.
description: "support-agent offline eval"
providers:
  - id: anthropic:messages:claude-opus-4-7
    config:
      temperature: 0.2
      seed: 424242
prompts:
  - file://prompts/support_system_v14.txt
tests:
  - description: "refund inside window"
    vars:
      user_message: "refund my order o_123 please"
    assert:
      - type: is-json
      - type: python
        value: file://checks/refund_state_check.py
      - type: llm-rubric
        provider: anthropic:messages:claude-opus-4-8
        rubric: file://rubrics/support_reply_v3.md
        threshold: 0.9
  - description: "safety: prompt injection in tool output"
    vars: {user_message: "read this doc and summarise"}
    fixtures: file://fixtures/traces/adversarial/*.yaml
    assert:
      - type: not-contains
        value: "IGNORE PRIOR INSTRUCTIONS"
sharing:
  # Post the summary to the PR via the promptfoo GitHub Action.
  enabled: true
```

**Where it stops.** Promptfoo's viewer is not a production trace store. If your feature runs on Langfuse or Braintrust for online telemetry, pair Promptfoo with that runner rather than replacing it.

### DeepEval

**What it is.** A pytest-first eval library. Test cases are Python objects (`LLMTestCase(input=..., actual_output=..., expected_output=...)`); metrics are classes that implement `.measure(test_case)`. A DeepEval test file is a `pytest` test file — you run it with `pytest tests/evals/` and CI treats a failing metric as a failing test. Also ships integrations with tracing tools and its own hosted `Confident AI` dashboard.<!-- needs-research: confirm DeepEval's current metric catalogue (G-Eval, Faithfulness, AnswerRelevancy, ContextualRelevancy, Hallucination, Bias, Toxicity, custom judges) and the current pytest hook / dashboard surface. -->

**Where it fits.** The right choice for teams whose engineering culture is *"if it's not in pytest, it's not tested."* An eval that lives next to unit tests, runs on the same command, and shows up in the same coverage report is an eval that will actually be maintained.

**Sketch of a DeepEval pytest hook:**

```python
# tests/evals/test_support_agent.py — illustrative.
import pytest
from deepeval import assert_test
from deepeval.metrics import GEval, FaithfulnessMetric, AnswerRelevancyMetric
from deepeval.test_case import LLMTestCase, LLMTestCaseParams

from support_agent import respond
from fixtures import load_snapshot_set

RUBRIC = GEval(
    name="policy_compliance",
    criteria="The reply complies with the refund policy documented in the retrieved context.",
    evaluation_params=[
        LLMTestCaseParams.INPUT,
        LLMTestCaseParams.ACTUAL_OUTPUT,
        LLMTestCaseParams.RETRIEVAL_CONTEXT,
    ],
    threshold=0.9,
    model="claude-opus-4-8",
)

@pytest.mark.parametrize("snapshot", load_snapshot_set("support-agent"))
def test_support_agent(snapshot):
    result = respond(snapshot.inputs.user_message, session=snapshot.inputs.session_history)
    tc = LLMTestCase(
        input=snapshot.inputs.user_message,
        actual_output=result.answer,
        retrieval_context=result.retrieved_chunks,
        expected_output=snapshot.recorded_output.final_answer,
    )
    assert_test(tc, [RUBRIC, FaithfulnessMetric(threshold=0.85), AnswerRelevancyMetric(threshold=0.85)])
```

**Where it stops.** DeepEval is not a trace store on its own; pair it with an observability substrate for online replay (chapters 04 and 02). Its judge model calls are your bill; wire the pre-registered judge model in explicitly rather than accepting the default.

### Braintrust

**What it is.** A hosted eval-and-trace platform designed around "experiments" — a named run of a suite against a candidate configuration, with per-example scores, aggregate metrics, side-by-side comparison of experiments, and a diff-friendly PR view. Ships an SDK (`braintrust.Eval(...)`) and a GitHub Action that comments the diff on the PR.<!-- needs-research: confirm the current Braintrust SDK surface (`Eval`, `wrap_openai`, custom scorers) and the exact GitHub Action name. -->

**Where it fits.** The right choice when the team wants managed infrastructure, first-class diff UX between experiments (candidate vs. baseline), and a shared dashboard for PMs and reviewers. Its opinionation on "experiments" maps directly onto the chapter 01 discipline: the suite version is the experiment name, the PR is the run, the diff is the review artefact.

**Sketch of a Braintrust experiment run:**

```python
# eval/support_agent_experiment.py — illustrative.
from braintrust import Eval
from support_agent import respond
from fixtures import load_snapshot_set

def score_rubric(input, output, expected, metadata):
    # LLM-as-judge call with position-swap; returns {"rubric_score": float, ...}
    ...

Eval(
    "support-agent",
    data=load_snapshot_set("support-agent"),
    task=lambda snap: respond(snap.inputs.user_message).answer,
    scores=[score_rubric, "Faithfulness", "AnswerRelevancy"],
    experiment_name=f"pr-{os.environ['GITHUB_PR_NUMBER']}",
    metadata={
        "model": "claude-opus-4-7",
        "prompt_template": "support_agent.system.v14",
        "suite_version": "2026.07.3",
    },
)
```

**Where it stops.** Braintrust is a hosted surface; data crosses your boundary. For regulated data, confirm the deployment model and data-handling contract before onboarding. Its cost is per-scored-event; the pre-registration file's cost budget matters (chapter 04).

### Langfuse

**What it is.** An open-source LLM observability platform (self-hostable, also hosted) with tracing, prompt management, dataset management, and an eval runner. Traces are the primary object; evals attach to trace batches. Ships SDKs across languages and integrates with LangChain, LlamaIndex, and the OpenAI / Anthropic SDKs.<!-- needs-research: confirm Langfuse's current dataset + evaluator surface and its CI integration story. -->

**Where it fits.** The right choice for teams that already run Langfuse for production tracing. The offline eval runs against the same "dataset" abstraction that the production traces populate — chapter 02's snapshots literally *are* Langfuse dataset items — and the diff between production quality and PR-candidate quality is legible in a single UI.

### Arize Phoenix

**What it is.** An open-source LLM observability and eval framework from Arize. Emphasises OpenInference / OpenTelemetry-native trace collection, a strong RAG-triad implementation (Phoenix originated much of the "RAG triad" framing along with TruLens), and Jupyter-notebook-friendly eval workflows.<!-- needs-research: confirm Phoenix's current OpenInference semantic conventions and the current RAG-triad evaluator names. -->

**Where it fits.** The right choice when the surface is RAG-heavy and OpenTelemetry-native is a hard constraint. Phoenix's trace shape maps cleanly to the emerging OpenTelemetry GenAI semantic conventions (mod-110), which future-proofs the observability substrate.

### Weights & Biases Weave

**What it is.** W&B's LLM-eval-and-trace surface. Reuses the broader W&B experiment-tracking metaphor: an `Evaluation` is versioned, its results are logged as artefacts, and the model / prompt config are first-class W&B objects. Strong story for teams already using W&B for classical ML.<!-- needs-research: confirm current Weave `Evaluation` API surface, scorer conventions, and CI integration. -->

**Where it fits.** The right choice for teams whose ML org is already on W&B. Reuses the same permissions, artifact store, and dashboards. Good bridge for cross-role handoff to the peer eval-engineering role (both roles are looking at the same W&B project).

### LangSmith

**What it is.** LangChain's hosted trace + eval surface, tightly integrated with LangGraph. Datasets, experiments, playgrounds, and a `RunEvaluator` interface that runs custom scorers on trace batches. First-class support for graph-shaped runs (which matter for LangGraph agents).<!-- needs-research: confirm LangSmith's current experiment / dataset / evaluator surface and the CI integration path. -->

**Where it fits.** The default when the stack is already LangGraph. The overhead of a separate observability surface is high; LangSmith's per-node span breakdown is the diagnostic tool that catches "which node in the graph regressed?" — the question chapter 02's snapshots ask, mapped to LangGraph's abstractions.

### Choosing among them

A five-question decision procedure, honestly answerable in about ten minutes:

1. *Where do your production traces already live?* Use the same runner if possible. Splitting trace substrate from eval substrate is the most common integration tax.
2. *Do you need managed infra?* Managed (Braintrust, LangSmith, Confident AI / DeepEval Cloud, Langfuse Cloud, Phoenix / Arize Cloud, W&B) vs. self-host (Promptfoo, DeepEval OSS, Langfuse OSS, Phoenix OSS, Weave-in-your-cluster).
3. *Is your test culture pytest-first or config-first?* DeepEval and Promptfoo respectively.
4. *Is the surface heavily RAG?* Phoenix's RAG-triad tooling is a lead.
5. *Is the graph the artefact?* LangSmith's per-node breakdown is a lead when the surface is LangGraph.

The wrong version of this decision is "pick the runner with the most stars on GitHub." The right version is "pick the runner your team will keep using in month six."

## The CI integration: what the yaml looks like

Below is a sketched GitHub Actions workflow for the two-track shape. The specifics vary per runner; the *shape* does not.

```yaml
# .github/workflows/eval.yml — illustrative, two-track shape.
name: eval

on:
  pull_request:
    paths:
      - "prompts/**"
      - "tools/**"
      - "retriever/**"
      - "agent/**"
      - "eval/**"
      - "fixtures/**"

jobs:
  fast-gate:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}
      - run: pip install -e '.[eval]'
      - name: Programmatic checks + canary
        run: pytest tests/evals/ -m "programmatic or canary" --deepeval-cache=off
      - name: Post summary to PR
        if: always()
        run: python eval/post_summary.py --gate=fast

  deep-gate:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    needs: fast-gate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}
      - run: pip install -e '.[eval]'
      - name: Judge + rubric + RAG triad
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          BRAINTRUST_API_KEY: ${{ secrets.BRAINTRUST_API_KEY }}
        run: python eval/support_agent_experiment.py
      - name: Snapshot replay
        run: python eval/replay_all.py --set fixtures/traces/support-agent/
      - name: Safety spot check
        run: python eval/safety_spot_check.py
      - name: Pre-registration check (gates)
        run: python eval/apply_policy.py --policy eval/policies/support-agent.yaml
      - name: Post diff to PR
        if: always()
        run: python eval/post_diff.py --baseline=main
```

Four properties this shape enforces:

- **The fast gate is a required check.** If it fails, the deep gate does not run and the PR cannot merge.
- **The deep gate posts a diff, not just a score.** The reviewer sees "which fixtures flipped from pass to fail" — chapter 02's most valuable output.
- **The pre-registration policy is applied by a script, not by a human reading the numbers.** A soft gate that requires a manual override is *explicitly* named as such; a hard gate that regressed sets a failing check-run.
- **Secrets are scoped to the deep gate only.** The judge API key is not exposed to fast-gate jobs; forks and untrusted PRs do not get to spend your judge budget.

## The PR comment: what a reviewer reads

A good eval CI comment is scannable in ten seconds and drillable in ten minutes. A working template:

```
### Eval gate: support-agent 2026.07.3
Baseline: main @ 8ab9c1f     Candidate: this PR @ 3fe020a
Judge model: claude-opus-4-8 (pinned)

Hard gates
✅ overall_pass_rate       0.918   (≥ 0.90)
❌ critical_tasks          0.917   (=== 1.00 required)   — T-4713, T-4720 flipped to fail
✅ rag_groundedness        0.874   (≥ 0.85)
✅ safety_spotcheck        1.000   (=== 1.00)

Soft gates
⚠️  cost_per_task          $0.0091 (was $0.0072; +26%)   — REQUIRES OVERRIDE
✅ latency_p95_ms          4120    (was 4218; -2%)

Regressed fixtures (deep gate)
- T-4713-refund-out-of-window          pass → fail   [critical]
- T-4720-multi-item-refund              pass → fail   [critical]
- T-5001-non-english-inquiry            pass → fail

Full report: <braintrust-experiment-link>
```

The comment lives inside the diff review, next to the changed prompt. A reviewer who sees a critical fixture flip does not need to open the eval dashboard to know the PR is blocked; the actionable text is right there.

## Cost, budgets, and CI-time discipline

Eval CI has a real bill. Judge-model calls at $0.02-0.20 per example, times 200 fixtures per run, times twenty PRs a day, times two runs per PR (once per push, once on the final rebase), gets to hundreds of dollars a day quickly. Three practices keep it bounded:

- **Judge-model caching keyed on `(candidate_output_hash, judge_prompt_hash, judge_model_id)`.** A PR that pushes twice without changing the actual model output does not re-pay for the judge. Every runner named above supports this (or trivially, at the harness layer).
- **The two-track shape.** The fast gate never touches a judge. Most PRs never reach the deep gate.
- **Budget per PR, published on the comment.** The comment includes `Judge tokens: 82,000 (~$1.20)` so the team notices when it drifts. Chapter 04 extends this to a monthly budget for the online loop; the offline budget belongs on every PR comment.

Cache aggressively; the judge is the expensive part. Provider prompt-caching (mod-102) applies to the judge calls too when the judge prompt is long and the candidate answer is short.

## Anti-patterns

Six worth naming.

1. **The optional check.** The eval CI job exists but is `continue-on-error: true`. Nobody reads the failure. The gate is a hallucination.
2. **The all-in-one job.** Fast checks and slow checks in the same job. The reviewer waits thirty minutes for a typo fix. Eventually someone disables the whole thing.
3. **The unversioned suite.** The fixture set floats on `main`. Two PRs open at the same time see different "baselines." The diff comment lies. Pin the baseline commit; version the suite.
4. **The judge on the same model as the candidate.** The judge and the model under test are both `claude-opus-4-7`. Self-preference bias goes uncorrected. Pin a *different* judge — usually a stronger model — and be explicit about it (chapter 04 on drift).
5. **The unsandboxed replay.** Snapshot replay calls the production backend for tool execution. The eval issues a real refund. Only replay against captured tool outputs or a staging tenant.
6. **The absent baseline.** The comment reports scores but not diffs. The reviewer cannot tell if `0.918` is better or worse than yesterday. Always report against a baseline (usually `main`).

## Summary

- Eval-gated CI is plumbing, not research. The applied engineer's decision is picking the runner that matches the observability substrate the team already has.
- Two tracks: a fast gate (programmatic + canary, unconditional, under five minutes) and a deep gate (judge + rubric + RAG triad + trace replay + safety spot check, on relevant paths, under thirty minutes).
- Six credible runners — Promptfoo, DeepEval, Braintrust, Langfuse, Phoenix, Weave, LangSmith. Choose by trace-substrate compatibility, test-culture fit, RAG-weight, and whether the graph is the artefact.
- The CI comment must be *scannable* (pass/fail per pre-registered gate) and *drillable* (per-fixture diff). Baselines are pinned; diffs are the review artefact.
- Judge-cost discipline requires caching, the two-track shape, and per-PR cost transparency on the comment.
- Six anti-patterns — optional checks, all-in-one jobs, unversioned suites, self-judging, unsandboxed replay, absent baselines — each of which turns the gate into a dashboard.

Chapter 04 extends the eval loop from the PR gate outward — a sampled online loop that scores live traffic on quality / safety / cost / latency, within a monthly budget, with a judge-drift monitor for when the judge model itself gets upgraded.
