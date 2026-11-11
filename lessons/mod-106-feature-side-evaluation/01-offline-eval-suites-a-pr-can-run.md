# Offline Eval Suites a PR Can Run

## What "feature-side evaluation" actually means

The peer roles on this org chart (the `ai-eval-engineer` and the `model-evaluation-engineer`) own evaluation as a discipline: dataset curation methodology, judge-vs-human calibration, bootstrap confidence intervals, sequential-testing math, benchmark construction from first principles. Their job is to make eval numbers *trustworthy in the aggregate*, across models and across time. It is a research discipline with its own literature.

The applied-AI engineer's job is different, and this module is the honest boundary between the two. The applied engineer owns the eval suite for **one product surface**, from the perspective of "should this PR merge?" The suite is small enough to run in CI, opinionated enough to gate a release, and coupled tightly enough to real user tasks that a passing score means the surface still works. When a PM asks *"is the new prompt safe to ship?"* the applied engineer runs the suite; when a reviewer asks *"is the LLM-as-judge itself calibrated correctly?"* the applied engineer hands the question to the peer role. This chapter — and this module — is about the first question.

The through-line is that eval, for the applied engineer, is a **CI concern first and a research concern second.** A judge that takes forty minutes to grade a suite will not run on every PR, so it will not gate anything. A suite whose thresholds were negotiated after the results came in will bless whatever change is in front of it. A "success" that is scored by reading transcripts by hand will be skipped the week before a launch, when it matters most. The offline suite is only useful if it is fast enough, honest enough, and pre-committed enough to *fail a PR* — including your own.

## The six evaluation modes an applied surface needs

Not every surface needs every mode; every surface needs to have made the decision. The six modes below cover the shape of nearly every product-side eval suite. Each one has a canonical use, a canonical failure, and a canonical output.

### Mode 1 — LLM-as-judge, with position and length controls

**Use it for.** Free-form generations where an exact-string check is meaningless: assistant answers, summaries, chat replies, generated documentation, generated code review comments. The judge is another LLM (often a stronger model than the one under test) prompted with a rubric and asked to score the candidate.

**The mandatory controls.** Two biases are documented enough that ignoring them is malpractice:

- **Position bias.** In pairwise comparisons ("is A or B the better answer?") LLM judges systematically prefer the response shown first. Zheng et al. (*Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*, NeurIPS 2023) documented this early alongside verbosity and self-enhancement biases, and the effect has been replicated across judge models in subsequent work (see `resources.md`). Mitigation: run every comparison twice with the order swapped and only count agreements as decisive.
- **Length bias / verbosity bias.** Judges consistently rate longer answers higher, independent of quality. Mitigation options: penalise length in the rubric ("prefer the shorter answer when quality is comparable"), truncate all candidates to the same length before judging, or report the length distribution alongside the score so a reviewer can see when a "quality win" is really a "length win."

Other controls worth naming without dwelling on: self-preference (a judge prefers its own model family's outputs), reasoning-chain bias (judges over-weight verbose reasoning), and refusal bias (judges reward refusals over hard answers). Peer-role material — see `resources.md` for the primary citations — but the applied engineer should at least *know these exist* before signing off on a judge configuration.

**The output.** A per-example score (numeric or categorical), an aggregate for the suite, and a *disagreement rate* between the two order-swapped runs. A judge whose swap disagreement rate is high is a judge you cannot trust for gating.

### Mode 2 — Rubric grading

**Use it for.** Structured generations where the "right answer" has multiple named dimensions: a customer-support reply that must be *polite*, *policy-compliant*, and *actionable*; a generated SQL query that must be *correct*, *efficient*, and *safe*. Rubric grading forces the judge to score each dimension separately and to explain the score.

**Design discipline.** A good rubric has three to seven named dimensions, each with a *concrete anchor description* per score level (not "1 to 5" with no definitions; a Likert scale with no anchors is a random number generator). The rubric is versioned in the repo. When you edit the rubric, you have edited the suite — bump the suite version and re-baseline.

**The output.** A per-dimension score per example, a per-dimension aggregate, and (crucially) a *per-dimension pass rate* against a pre-registered threshold ("polite ≥ 4/5 on 95% of examples"). Aggregating multi-dimensional scores into one number loses the signal that rubric grading was supposed to give you; do not average dimensions into a single "quality" score unless a stakeholder has explicitly asked for it and you have written down why.

### Mode 3 — Code / task correctness (programmatic checks)

**Use it for.** Any output where the correctness question can be resolved by a machine: generated code (does the test suite pass?), generated SQL (does the query return the expected rows?), generated JSON (does it validate against the schema?), a tool-calling agent's end state (did the customer record end up in the expected state?).

**Why it is the highest-signal mode you have.** Programmatic checks do not drift. A test that passed yesterday and fails today measured a real change, not a judge's mood. Every mode that can be reduced to a programmatic check should be — even at the cost of narrower coverage.

**Design discipline.** The tests are *hidden* from the prompt (the model does not see them at generation time), and the test harness is versioned separately from the suite. When you edit a test to make a failing case pass, you have not fixed a bug; you have moved the goalposts. The paired-solution repo's git history is the audit trail.

### Mode 4 — RAG triad (context / answer / question)

**Use it for.** Any RAG-in-product surface. The triad — formulated in TruLens's "RAG triad" core-concept docs as *context relevance*, *groundedness*, and *answer relevance*, and reproduced with adjacent names (Faithfulness, Retrieval Relevance, Hallucination) in Arize Phoenix and Ragas — decomposes RAG quality into three independent judgements:

- **Context relevance.** Given the user's question, are the retrieved chunks actually relevant to it?
- **Groundedness (faithfulness).** Given the retrieved chunks, is the generated answer supported by them — or is it hallucinated / drifting?
- **Answer relevance.** Given the user's question, does the answer address the question — regardless of whether it is grounded?

**Why decomposing matters.** A single "was the answer good?" score cannot tell you whether the retrieval broke or the generation broke. A drop in context relevance is a retriever problem (mod-103); a drop in groundedness with intact context relevance is a generation problem (mod-102); a drop in answer relevance with intact groundedness and context relevance is usually a query-understanding problem. Diagnosis is what makes the eval loop actionable.

**Design discipline.** All three scores are LLM-judged and inherit the position / length controls above. The three thresholds are pre-registered independently. When a PR moves the numbers, the PR body must name which of the three moved and why.

### Mode 5 — Safety spot checks

**Use it for.** Any surface a stranger can prompt. The applied engineer's job here is *not* to build a red-team programme (that is `ai-safety-engineer` peer-role territory, developed in mod-107). The applied engineer's job is to keep a small, versioned suite of "the incident we already had" and "the incidents peer surfaces have already had," and to run it on every PR so that a regression on a fixed issue fails the build.

**What belongs in the offline safety suite.**

- The five to twenty prompts that surfaced a real past incident on your product. Every incident that produced a fix should mint a test.
- A rotating sample from the current OWASP LLM Top 10 categories (prompt injection, sensitive information disclosure, insecure output handling, etc.) — the applied engineer picks the categories that map to their surface, not the full taxonomy.
- A short adversarial prompt catalogue targeted at *your* tool catalogue and *your* system prompt — not generic jailbreaks copied from a public list. Refresh quarterly.

**What does NOT belong here.** A comprehensive red-team programme; a jailbreak-of-the-week corpus; a benchmark of general model safety. Those are the peer role's suites, and mixing them into the PR gate makes the gate too slow and too noisy to actually block anything.

### Mode 6 — Regression fixtures from production traces

Covered in depth in [chapter 02](02-trace-snapshots-as-regression-fixtures.md). Named here so the picture is complete: for any surface with real traffic, the most valuable offline eval you have is a small, rotating set of production traces captured (PII-scrubbed) with full lineage — model, prompt, retriever config, decoding params, seed. Replaying them on a PR shows you what would have happened to real recent users.

## Pre-registration: writing thresholds down BEFORE looking at results

Pre-registration is the eval-engineering discipline the applied engineer most often skips, and most often regrets skipping. The rule is:

> The pass/fail threshold for a suite (or a suite dimension) is written down and reviewed **before** the eval is run against the change under test. Adjusting the threshold after seeing the result is called *goalpost drift*, and it is how eval suites lose their meaning.

Pre-registration does not mean the threshold is permanent. It means it is a policy decision, not a per-PR fudge. Thresholds get updated on their own PRs, with an explicit justification, ideally by someone other than the person whose feature-work benefits.

A practical pre-registration format that fits in a repo:

```yaml
# eval/policies/support-agent.yaml
suite: support-agent-v3
gates:
  overall_pass_rate:
    metric: mean(rubric.pass_rate)
    threshold: 0.90
    on_regression: fail_ci
  critical_tasks:
    metric: pass_rate(fixture.tags contains "critical")
    threshold: 1.00        # must not regress
    on_regression: fail_ci
  rag_groundedness:
    metric: mean(rag_triad.groundedness)
    threshold: 0.85
    on_regression: fail_ci
  safety_spotcheck:
    metric: pass_rate(fixture.tags contains "safety")
    threshold: 1.00
    on_regression: fail_ci
  cost_per_task:
    metric: mean(cost_usd_per_task)
    threshold: 0.008
    on_regression: warn_and_require_override
  latency_p95:
    metric: p95(latency_ms_per_task)
    threshold: 4500
    on_regression: warn_and_require_override
policy:
  changing_this_file: requires_review_from_eval_owner
  suite_version_bump: requires_re_baselining
```

Two properties matter. First, the file is *machine-readable*: the CI runner (chapter 03) can parse it and act on it without a human interpreting a Markdown doc. Second, it distinguishes hard gates (`fail_ci`) from soft ones (`warn_and_require_override`); cost and latency almost always start as soft gates and harden as the surface matures.

## Composing the suite: how the six modes fit together

For a real product surface, the suite is not "one mode wins." It is a small stack:

1. **A programmatic-check tier** (mode 3): the fastest, highest-signal checks. Runs on every PR, every branch, unconditionally. Failure blocks merge.
2. **A judge-and-rubric tier** (modes 1, 2, 4 as applicable): the LLM-scored core of the suite. Runs on every PR that touches prompts, tools, retrievers, model config, or decoding parameters. Failure blocks merge unless the pre-registration marks the metric as soft.
3. **A safety spot-check tier** (mode 5): a small fixed panel that always runs, plus a rotating sample from the incident library. Failure blocks merge.
4. **A regression-fixture tier** (mode 6, chapter 02): production-trace replays. Runs on the same triggers as tier 2. Failure blocks merge.

The suite is small on purpose. A support-agent PR gate might be 60 tasks in tier 1 (programmatic), 40 tasks in tier 2 (judge + rubric + RAG triad), 20 tasks in tier 3 (safety spot check), and 40 traces in tier 4 (replay). At an average LLM-judged cost of a couple of cents per task, that is a few dollars per PR — small enough to run on every push, large enough to catch anything but the subtlest regressions.

For the numbers where a small suite is not enough — a headline "model X vs. model Y" comparison, a launch-readiness review, a scientific-methodology write-up — the peer role's larger suites take over. This chapter's suite is for CI. The peer role's suite is for launches and audits.

## Where a well-run offline suite fails silently

Six failure modes worth naming before the exercises make you meet them personally.

1. **Judge-model drift.** The judge silently upgrades under you (see chapter 04's drift monitor). Yesterday's 90% score is not comparable to today's 92% because the judge is not the same judge. Pin the judge model in the pre-registration.
2. **Test set memorisation.** The team has been iterating against the same 50 fixtures for six months. The prompt is now over-fit to the suite; the score is a lie. Rotate a fraction of the fixtures every quarter and keep a hidden hold-out (chapter 02).
3. **Prompt-in-fixture leakage.** Someone copied the system prompt into an example expected answer. Now the suite scores highest for outputs that quote the system prompt back. Grep the fixture files for the system prompt string on every suite bump.
4. **Judge sees the label.** The judge prompt accidentally includes the reference answer near the candidate answer. The judge is scoring "does the candidate match the reference?" — a much easier task than "is the candidate good?" Audit the judge prompt on every rubric change.
5. **Aggregate blindness.** The mean score is fine; a specific category (out-of-policy refunds, long-form summaries, non-English prompts) has quietly collapsed. Always chart per-category pass rate alongside the aggregate.
6. **The unpinned decoding config.** Temperature was `0.7` yesterday and `0.2` today; the suite score changed and nobody knows why. Pin decoding parameters in the run config; make them part of the pre-registration and the lineage (chapter 02).

## Summary

- Feature-side eval is a *CI concern first*: a suite that cannot fail a PR does not gate a release.
- Six modes cover the space: LLM-as-judge (with position + length controls), rubric grading, programmatic correctness, RAG triad, safety spot checks, and regression fixtures from traces.
- Every mode has a canonical output shape and a canonical failure mode; every mode's biases are documented — the applied engineer's job is to know they exist and to mitigate them, not to re-derive the calibration literature.
- Thresholds are **pre-registered** in a machine-readable file. Changing a threshold is a reviewed policy change, not a per-PR fudge.
- The suite composes as tiers: programmatic checks first (unconditional), then judge-scored, then safety, then production-trace replay. Fail fast on the cheap tiers.
- The suite is bounded by six known failure modes — judge drift, test-set memorisation, prompt-in-fixture leakage, judge-sees-label bugs, aggregate blindness, and unpinned decoding — each of which the following chapters address directly.

Chapter 02 turns to the highest-signal fixture source available to a shipped surface: real production traces, captured as regression tests with their lineage pinned. Chapter 03 wires the suite into CI with off-the-shelf runners (Promptfoo, DeepEval, Braintrust, Langfuse, Weave, Phoenix, LangSmith). Chapters 04 and 05 extend the eval loop to live traffic and the humans who read the dashboards.
