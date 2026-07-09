# SLOs From Spec to Metrics

## Why SLOs come before prompts

A product spec ships in English: "answers should be accurate," "the assistant must be safe to leave running for our users," "it needs to feel snappy." These are promises to users. They are not, yet, engineering artefacts. An SLO — a **Service Level Objective** — is what those promises become when a specific, measurable target is attached to each. Without them, the eval is a vibe check, the on-call runbook has nothing to page against, and the model / prompt / retriever change that "feels better" cannot be shown to be better.

The pattern from SRE (Google's *Site Reliability Engineering*, chapter 4) transplants cleanly, with additions. You still have latency and availability, but you also have quality dimensions — task success, groundedness, safety — that are not binary and often not cheap to measure. And you have a cost dimension that classical SRE mostly ignored: LLM calls are metered.

This chapter names the six SLO families a shipped LLM feature needs and shows how to turn each into a concrete measurable target. The unifying principle: **every promise you make to a user must map to a number you can compute from telemetry.**

## The six families

### 1. Task success

The one that matters, and the one people skip because it is expensive to measure.

Task success is the probability that a user's request produces the outcome they wanted, judged by a source of truth that is *not* the model that generated the output. Depending on the shape:

- **Structured extraction**: field-level exact-match or fuzzy-match against a labelled gold set. Report per-field precision / recall / F1 and a per-document "all critical fields correct" rate.
- **Retrieval-with-citations**: (a) answer correctness (rubric or reference-based), (b) citation faithfulness (does each cited span actually support the sentence?), (c) refusal correctness on the "no good evidence" set. See mod-106 for the RAG-triad decomposition.
- **Co-pilot**: user acceptance rate, but paired with a *quality* signal — accepted-and-not-edited-within-N-minutes, accepted-and-sent, or an offline judge on a sampled trace set. Naked accept-rate is gameable by generating bland output.
- **Autonomous agent**: end-to-end task success on a τ-bench-shaped internal suite (mod-105 for the suite; mod-106 for the eval wiring). Report both success and *legal-trajectory* rate — did it succeed *and* stay inside its tool budget?
- **Chat**: hardest. Proxy with rubric-graded conversation-level judgments on a sampled slice, plus explicit thumbs-up / thumbs-down when the surface allows.

An SLO for task success looks like: *"90 % of extraction jobs on the production distribution have all 4 critical fields correct, evaluated weekly on a rolling 500-document sample judged against the gold set."* It has a target, a population, an evaluator, and a cadence.

### 2. Groundedness

Groundedness — the fraction of model claims that are supported by the evidence the model was given — is a separate SLO from task success, and it should be. A model can produce a correct-looking answer that is not grounded in the retrieved evidence (it drew on parametric knowledge, or hallucinated), and that answer will pass a naked correctness rubric but fail an audit.

Concretely:

- Per-sentence: for each generated sentence, was there a cited span that supported it? Metrics like the RAG-triad's *faithfulness* and *context relevance* land here.
- Aggregate: fraction of responses with **no unsupported claims**. This is a stricter, product-visible metric — one hallucinated sentence is enough to ruin trust in a regulatory answer.
- Refusal correctness: on questions where the retrieval turns up nothing sufficient, does the surface say "I don't have evidence for this" instead of guessing?

An SLO for groundedness looks like: *"On sampled retrieval-with-citations traces, ≥ 95 % of responses have every sentence supported by a cited span, judged by a rubric-graded LLM-as-judge with weekly human calibration on a 50-trace slice."* You must state the judge, the sample, and the calibration or you cannot defend the number.

Groundedness is unnecessary for shapes without retrieval (chat, one-shot generation). For those, "no hallucinated facts" is a rubric line in the task-success eval, not a separate SLO.

### 3. Safety

Safety is *many* SLOs, not one. At minimum, a shipped feature needs:

- **Refusal correctness on unsafe requests** — did the surface refuse the requests it should have refused, per your written policy? See mod-107 for the layered guardrail design; here the concern is the SLO shape.
- **Over-refusal rate on safe requests** — how often did the guardrails block a request that was fine? An overly cautious feature is a broken feature (Anthropic and OpenAI both publish over-refusal numbers because it is a real product problem).
- **Prompt-injection resistance on adversarial input** — measured against a fixed suite (AgentDojo shape for agents, InjecAgent for tool-calling features), reported as attack-success-rate. See mod-107.
- **PII / secret leakage** — outbound scans of model output; zero-tolerance target with a per-week incident count instead of a rate for genuinely rare events.

An SLO for safety looks like a **table**, not a single number. Every row has a category, a target, a test set, and an owner.

Safety SLOs are the ones a governance reviewer will look at first — see [chapter 04](04-one-page-feature-architecture-doc.md) on the sign-off flow.

### 4. Cost per successful outcome

The applied-AI-native metric. "Cost per token" and "cost per call" are engineering metrics; **"cost per successful outcome"** — dollars spent to get one user-facing success — is the product metric. It internalises everything: prompt caching, retry overhead, over-eager guardrails, wrong-tier routing, tool loops, judge tiers, and the hallucination-and-re-ask cycle.

Compute it as:

```
cost per successful outcome
  = (sum of all provider spend on the feature over window T)
    / (count of user-facing successes in window T)
```

The denominator uses the same success signal as your task-success SLO — thumbs-up, sent-message, accepted-suggestion, correct-extraction, whichever fits the shape. The numerator includes retries, guardrail calls, and every judge / router / rerank call attributable to the feature. See mod-109 for the per-feature and per-tenant accounting shape.

An SLO looks like: *"Cost per drafted-and-sent reply ≤ $0.04, measured weekly on the trailing 7-day window, reported per-tenant so a single power user's usage doesn't hide a regression on the median tenant."*

Target values are context-specific; write them down anyway so a router change or model-tier swap can be evaluated against them.

### 5. Latency — TTFT, TPOT, p95, retry latency

Users feel latency non-uniformly:

- **TTFT (time to first token)** — from when the user submits to when the *first* streamed token renders. Below ~200 ms it feels instant; between 200 ms and 1 s it feels responsive; above 2 s it feels broken unless there is a shown-and-honest progress signal. Streaming pushes TTFT down at the cost of adding a new failure mode (partial output rendered before an error).
- **TPOT (time per output token)**, sometimes called ITL (inter-token latency) — how fast the tokens keep coming after the first. A model with fast TTFT but slow TPOT is worse for long answers.
- **End-to-end p50 / p95 / p99** — the whole-request wall-clock the user feels. Report percentiles, not averages: an average hides the tail, and the tail is what users complain about.
- **Retry latency** — extra time spent because the first call failed a schema check, hit a rate limit, or produced a guardrail block that triggered a repair loop. If retries are budgeted (mod-102), the retry-latency budget must be part of the SLO.
- **Tool-call round-trip** — for agents, latency of individual tool calls, and the *number* of tool calls in a run; both feed the wall-clock.

An SLO for latency looks like: *"Streaming TTFT p95 ≤ 800 ms and end-to-end p95 ≤ 6 s for the co-pilot drafting surface, measured client-side, excluding runs cancelled by the user."*

Client-side measurement matters — server-side TTFT hides the network hop, and the network hop is what users feel.

Latency is deeply entangled with cost (a bigger model has better task success but worse TTFT and TPOT) and with routing (mod-108). Land the SLO first; use it to bound the router search space later.

### 6. Freshness

For anything with a data-backed answer — retrieval-with-citations, structured extraction over a live corpus, an assistant that reads a CRM — freshness is a real SLO. Users burn trust when they see a stale answer that a human would have caught.

Two dimensions:

- **Corpus freshness** — how recently was the underlying source ingested / indexed? Measured as a per-document lag distribution: p95 time between source update and appearance in the retriever.
- **Answer freshness** — for a user question with a time-sensitive answer, how often does the model use a stale document when a newer one exists? This is a retrieval-quality question with a freshness cut.

An SLO looks like: *"p95 ingestion lag ≤ 15 minutes for the policy-doc corpus; on a fixed set of 40 freshness-probe questions, 0 stale answers per week."*

The freshness SLO is often owned jointly with the RAG engineering team upstream. Land the contract in the architecture doc — see [chapter 04](04-one-page-feature-architecture-doc.md).

## Writing a target

A well-formed SLO row has all six of the following. If any are missing, you're going to argue about the number.

| Element | What it answers | Example |
|---|---|---|
| **Metric** | What is measured | `groundedness_faithfulness_rate` |
| **Target** | What "good" is | ≥ 0.95 |
| **Window** | Over what period | trailing 7 days |
| **Population** | On what traffic | production traffic, tenant tier ≠ trial |
| **Evaluator** | Who / what decides | rubric-graded LLM-as-judge, calibrated weekly |
| **Owner** | Who fixes it when it breaks | applied-AI team, on-call rotation |

If the metric requires human judgment (task success on a co-pilot surface, over-refusal on an ambiguous safety category), state the sampling rate and the calibration cadence explicitly. A quality SLO with no calibration decays into a judge-drift disaster inside a quarter (mod-106 covers the judge-drift monitor).

## From spec to SLO table — worked

**Spec fragment**: "The assistant should answer questions about our warranty policy accurately, cite the policy, feel snappy, and not cost us the earth."

Turn each promise into a row:

| SLO | Target | Window | Population | Evaluator |
|---|---|---|---|---|
| Task success (answer correctness) | ≥ 0.90 | trailing 7 d | production, non-trial | rubric LLM-as-judge, 200/wk human-calibrated |
| Groundedness (faithfulness) | ≥ 0.95 | trailing 7 d | responses with citations | rubric LLM-as-judge |
| Refusal correctness on no-evidence set | ≥ 0.98 | trailing 30 d | probe set (fixed) | exact-match |
| Over-refusal rate on safe set | ≤ 0.02 | trailing 30 d | probe set (fixed) | exact-match |
| Cost per answered question | ≤ $0.02 | trailing 7 d | production, non-trial | billing pipeline |
| Streaming TTFT p95 | ≤ 1000 ms | trailing 1 d | production, non-trial | client-side telemetry |
| End-to-end p95 | ≤ 5 s | trailing 1 d | production, non-trial | client-side telemetry |
| Corpus ingestion lag p95 | ≤ 30 min | trailing 1 d | policy corpus | ingestion pipeline |

Now the review has something to argue about — is 0.90 correctness the right bar? Should over-refusal be 0.05? — instead of arguing about whether "accurately" is the right word. Every row is a testable claim.

## Summary

- Six SLO families cover a shipped LLM feature: task success, groundedness, safety, cost per successful outcome, latency (TTFT / TPOT / p95 / retry / tool round-trip), freshness.
- Every SLO needs metric, target, window, population, evaluator, and owner. Anything missing turns into an unresolvable argument.
- Cost is not "cost per call" — it's **cost per successful outcome**, using the same success signal as task success.
- Latency is measured client-side and reported as percentiles; TTFT and TPOT are separate promises.
- Groundedness and safety are *many* metrics, not one; write them as tables.
- Freshness is often a joint SLO with an upstream data / RAG team; put the contract in the architecture doc.

Chapter 03 turns the shape (chapter 01) and the SLOs (this chapter) into the initial provider + tier choice, with a written trade-off note.
