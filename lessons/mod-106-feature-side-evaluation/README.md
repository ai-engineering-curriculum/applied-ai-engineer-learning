# mod-106-feature-side-evaluation: Feature-Side Evaluation: Offline, Online, and In CI

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Design offline eval suites that a PR can run in CI: LLM-as-judge with position / length controls, rubric grading, code / task correctness, RAG-triad, safety spot checks — with pre-registered pass/fail thresholds
- Snapshot production traces (with PII scrubbed) as regression fixtures with model / prompt / retriever / decoding / seed lineage pinned
- Wire eval-gated CI end-to-end on Promptfoo, DeepEval pytest hooks, Braintrust / Langfuse / Weave / Phoenix / LangSmith runners — so a prompt / chain / agent change gets an eval report before merge
- Add a sampled online eval loop that scores live traffic on quality / safety / cost / latency within a monthly eval budget, and detect judge drift when the judge model itself upgrades
- Build per-feature quality dashboards a PM, an on-call engineer, and a safety reviewer can each read — while deferring statistical rigour (bootstrap CIs, sequential tests, judge-vs-human calibration methodology) to the `ai-eval-engineer` and `model-evaluation-engineer` peers

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
