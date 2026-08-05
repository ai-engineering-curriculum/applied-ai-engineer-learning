# mod-106-feature-side-evaluation: Feature-Side Evaluation — Offline, Online, and In CI

**Estimated effort:** 14 hours

Evaluation for a shipped LLM feature is a *CI concern first and a research concern second*. The peer roles on this org chart — `ai-eval-engineer` and `model-evaluation-engineer` — own the measurement discipline: judge-vs-human calibration, bootstrap confidence intervals, sequential-testing math, benchmark construction. The applied-AI engineer owns the eval suite for **one product surface**, from the perspective of "should this PR merge?" This module is about that suite: small enough to run in CI, opinionated enough to gate a release, and coupled tightly enough to real user tasks that a passing score means the surface still works.

The module carries a single through-line: an eval that cannot **fail a PR — including your own** — is a dashboard, not a gate. Every chapter, every exercise, is pointed at that discipline.

## Learning objectives

- Design offline eval suites that a PR can run in CI: LLM-as-judge with position / length controls, rubric grading, code / task correctness, RAG-triad, safety spot checks — with pre-registered pass/fail thresholds.
- Snapshot production traces (with PII scrubbed) as regression fixtures with model / prompt / retriever / decoding / seed lineage pinned.
- Wire eval-gated CI end-to-end on Promptfoo, DeepEval pytest hooks, Braintrust / Langfuse / Weave / Phoenix / LangSmith runners — so a prompt / chain / agent change gets an eval report before merge.
- Add a sampled online eval loop that scores live traffic on quality / safety / cost / latency within a monthly eval budget, and detect judge drift when the judge model itself upgrades.
- Build per-feature quality dashboards a PM, an on-call engineer, and a safety reviewer can each read — while deferring statistical rigour (bootstrap CIs, sequential tests, judge-vs-human calibration methodology) to the `ai-eval-engineer` and `model-evaluation-engineer` peers.

## Chapters

1. [Offline Eval Suites a PR Can Run](01-offline-eval-suites-a-pr-can-run.md) — six evaluation modes (LLM-as-judge with bias controls, rubric grading, programmatic correctness, RAG triad, safety spot checks, trace regression), pre-registered thresholds, and the tiered composition that fits in a merge gate.
2. [Trace Snapshots as Regression Fixtures](02-trace-snapshots-as-regression-fixtures.md) — replayability and judgeability, the six lineage fields every snapshot pins, the three-pass PII-scrubbing pipeline, sampling from four buckets, and a worked snapshot schema.
3. [Eval-Gated CI End-to-End](03-eval-gated-ci-end-to-end.md) — the fast-gate / deep-gate two-track shape; Promptfoo, DeepEval, Braintrust, Langfuse, Phoenix, Weave, LangSmith as field-guide choices; the scannable PR comment; judge-cost discipline.
4. [Sampled Online Eval Loop with a Judge-Drift Monitor](04-sampled-online-eval-loop.md) — the four scoring dimensions, sampling rates and cohorts, the monthly eval budget, judge drift as a first-class failure mode, and the online→offline feedback path.
5. [Cross-Persona Feature Dashboards](05-cross-persona-feature-dashboards.md) — the PM dashboard (chart-first, five panels), the on-call dashboard (drillable, obeys the alert), the safety-reviewer dashboard (artefact-oriented), and the four peer-role dashboards the applied engineer explicitly does not build.

## Exercises

- [exercise-01 — offline eval suite with pre-registered thresholds](exercises/exercise-01-offline-eval-suite-with-preregistered-thresholds.md).
- [exercise-02 — trace snapshot as regression fixture](exercises/exercise-02-trace-snapshot-as-regression-fixture.md).
- [exercise-03 — eval-gated CI with Promptfoo and Braintrust](exercises/exercise-03-eval-gated-ci-with-promptfoo-and-braintrust.md).
- [exercise-04 — sampled online eval loop with judge-drift monitor](exercises/exercise-04-sampled-online-eval-loop-with-judge-drift-monitor.md).
- [exercise-05 — cross-persona feature dashboards](exercises/exercise-05-cross-persona-feature-dashboards.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- Prompt discipline, structured outputs, and rubric prompting patterns consumed by the LLM-as-judge and rubric-grading modes in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) live in **mod-102**.
- RAG retriever configuration and the retriever lineage pinned in [chapter 02](02-trace-snapshots-as-regression-fixtures.md) — plus the RAG-triad decomposition in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) — live in **mod-103**.
- Tool schemas, tool-call replay, and the tool-catalogue hash in [chapter 02](02-trace-snapshots-as-regression-fixtures.md) live in **mod-104**.
- Agent-shaped eval fixtures introduced in **mod-105 chapter 04** (τ-bench, SWE-bench Verified, WebArena, AgentDojo shape templates) feed the offline suite composition in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) and the trace snapshot design in [chapter 02](02-trace-snapshots-as-regression-fixtures.md).
- Safety spot checks in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) and the safety-reviewer dashboard in [chapter 05](05-cross-persona-feature-dashboards.md) narrow the scope from the full guardrails story in **mod-107**.
- Model routing decisions in the trace lineage ([chapter 02](02-trace-snapshots-as-regression-fixtures.md)) and per-arm sampling in the online loop ([chapter 04](04-sampled-online-eval-loop.md)) are governed by **mod-108**.
- Cost and latency SLOs referenced across [chapter 01](01-offline-eval-suites-a-pr-can-run.md)'s pre-registration file and [chapter 04](04-sampled-online-eval-loop.md)'s alerting shape are developed in **mod-109**.
- Trace collection, span attributes, and the observability substrate that the online loop in [chapter 04](04-sampled-online-eval-loop.md) samples from — and that the on-call dashboard in [chapter 05](05-cross-persona-feature-dashboards.md) queries — live in **mod-110**.
- Feature-flag, canary, and rollback discipline that the on-call dashboard's "rollback panel" ([chapter 05](05-cross-persona-feature-dashboards.md)) integrates with lives in **mod-111**.
- End-to-end ownership of a shipped feature with this module's eval loop wired in is the capstone in **mod-112**.

## Peer-role handoffs

Two peer roles carry the parts of evaluation this module *explicitly does not* claim:

- **`ai-eval-engineer`** owns judge-vs-human calibration methodology, bootstrap CIs on quality metrics, sequential-testing decisions on launch reads, and the "how do we know the numbers mean what we think they mean" surface. The applied engineer *reads* those outputs (chapter 05); the applied engineer does not maintain them.
- **`model-evaluation-engineer`** owns benchmark construction from first principles, cross-model comparability, and the deeper judge-model behavioural characterisation. The applied engineer's pre-registered judge pin (chapters 01, 04) is downstream of the peer role's judge-selection work.

Naming both boundaries out loud is deliberate. The applied engineer's suite is small on purpose; the peer roles' suites are large on purpose. The two together are what a serious release process reads.
