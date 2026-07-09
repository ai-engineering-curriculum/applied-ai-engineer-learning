# exercise-02: SLO Mapping from Spec to Metrics

**Estimated effort:** 2 hours

## Objective

Take one of the specs from [exercise-01](exercise-01-feature-shape-decision-rubric.md) — or the worked warranty-Q&A spec from [chapter 02](../02-slos-from-spec-to-metrics.md) if you skipped exercise-01 — and produce an SLO table that a PM, an ML engineer, and an on-call engineer would each argue about *the numbers*, not *the columns*.

## Learning outcomes

By the end you should be able to:

- Turn an English-language product promise ("accurate," "trustworthy," "snappy," "affordable") into a well-formed SLO row with all six mandatory elements.
- Choose the right SLO family for each promise and reject the ones that don't apply.
- Distinguish task-success from groundedness and safety, and understand why they are *separate* SLOs.
- Compute cost per successful outcome from a candidate spend + successful-outcome pair.
- Specify the evaluator, the sample, and the calibration cadence for a quality SLO so it doesn't decay into a judge-drift disaster.

## Prerequisites

- You've read [chapter 02](../02-slos-from-spec-to-metrics.md).
- You've completed [exercise-01](exercise-01-feature-shape-decision-rubric.md), or you're using the worked warranty-Q&A spec.

## Problem statement

Pick one spec — A (sales research), B (contract extraction), or C (internal ops helper). Read it as a product owner would, extract every implicit or explicit promise about quality, safety, latency, cost, or freshness, and turn each into an SLO row. Then defend the numbers.

If you're inventing the numbers, that's fine — you must state the *basis* for each target (competitive benchmark, willingness-to-pay estimate, latency-perception literature, tenant contract, etc.). "It felt right" is not a basis.

## Requirements

Produce the following deliverables:

### Part 1 — Promise inventory (10 lines)

List every promise the spec makes, implicit or explicit. Example, from Spec C: "answers policy questions from the wiki" implies task-success on wiki-covered questions. "Explicitly says it doesn't know when it can't" implies a refusal-correctness SLO. "Slack bot" implies a chat-message latency target (a Slack message that takes 45 seconds to appear is broken).

Keep it terse; one line per promise.

### Part 2 — SLO table

One row per SLO. Every row has all six columns:

| Metric | Target | Window | Population | Evaluator | Owner |

- **Metric** — the name (e.g. `groundedness_faithfulness_rate`, `cost_per_answered_question`).
- **Target** — the number (or the range).
- **Window** — the reporting period (`trailing 7 d`, `trailing 30 d`, `trailing 1 d`).
- **Population** — the traffic slice (`production`, `production, non-trial`, `probe set: fixed`, `production, tenant tier != free`).
- **Evaluator** — how the number is computed (`judge:v3 rubric LLM-as-judge, 200/wk human-calibrated`, `client-side telemetry`, `billing pipeline`, `exact-match on labelled gold set`).
- **Owner** — who fixes it when it breaks (typically the applied-AI team, but freshness SLOs may be jointly owned with data-eng, and safety incidents may route to a governance owner).

Aim for **8–14 rows.** Fewer and you probably skipped a family; more and you're double-counting.

At minimum, one SLO from each of the six families where the family applies to your spec (task success, groundedness, safety, cost per successful outcome, latency, freshness). If a family does not apply, state which and why.

### Part 3 — Basis notes (one paragraph per row that isn't self-evident)

For each row whose target is not obviously self-evident, write one to three sentences on the basis. Examples:

- **TTFT p95 ≤ 800 ms** — "Nielsen's 1-second responsiveness threshold minus expected network jitter on Slack's endpoint; below this the surface feels instantaneous to a user typing in Slack."
- **Cost per answered question ≤ $0.02** — "PM's willingness-to-pay analysis puts the willing-to-fund upper bound at $0.05; targeting $0.02 leaves headroom for the guardrail and judge tiers and a 2× safety margin."
- **Groundedness ≥ 0.95** — "One in twenty responses with an unsupported claim is the tolerance level our legal reviewer accepted for the internal-only surface; would tighten to ≥ 0.99 before we open the surface to external users."

The basis is what turns the SLO from a wish list into a review artefact.

### Part 4 — Calibration plan

For every quality SLO whose evaluator is not deterministic (i.e. every SLO that uses a rubric-graded LLM-as-judge, or human review, or a hybrid), state the calibration plan in three lines:

- **Cadence** — how often is the judge / rubric calibrated against human review?
- **Sample size** — how many items per calibration round?
- **Trigger** — what disagreement rate between judge and human forces a rubric revision or a judge model swap?

If the plan is "we'll figure it out later," the SLO is not defensible.

## Starter guidance

- Start with the promise inventory. If you skip it, you'll write SLOs for capabilities the spec never asked for and miss ones it implied.
- Do **not** merge task success and groundedness into one row. They are separate promises to users; measure them separately. A model can score high on correctness *and* low on groundedness (correct-looking answer, not grounded in the retrieved evidence — the classic hallucination-that-happens-to-be-right).
- Latency percentiles matter. State p50 *and* p95. A p50 target with no p95 hides the tail; a p95 target with no p50 obscures the median experience.
- On cost per successful outcome, the denominator uses the *same* success signal as the task-success SLO. That is the whole point.
- The freshness SLO is often jointly owned with an upstream data / RAG team. Name them, and note the interface (e.g. "corpus lag SLO owned by data-eng; feature team consumes and alerts on it").

## Acceptance criteria

- [ ] The promise inventory covers every explicit and implicit promise in the spec.
- [ ] The SLO table has all six columns filled for every row. No blanks. No "TBD."
- [ ] Task success and groundedness are separate rows (if both apply to the shape).
- [ ] Safety is a *table*, not a single row — at minimum, one row for refusal correctness on unsafe requests, one row for over-refusal on safe requests, and one row for prompt-injection resistance if the surface uses tools or agent shapes.
- [ ] Cost per successful outcome is one first-class row; its denominator matches the task-success success signal.
- [ ] Latency rows include TTFT p95 (if streaming) and end-to-end p95, both measured client-side (state so).
- [ ] Every non-obvious target has a written basis in Part 3.
- [ ] Every rubric-graded quality SLO has a calibration plan in Part 4.

## Stretch goals

Pick any two:

- **SLO / SLI / SLA distinction.** For your table, promote two rows to a **service level agreement** an external customer might see, and rewrite them (only the two) as an SLA — softer target with a legal / financial consequence for breach.
- **Error budget policy.** Pick one SLO and write the error-budget policy: what happens when the burn rate exceeds the budget? A rollback? A pause on new prompt changes? A page? See Google SRE chapters 3 and 4 for the pattern.
- **Judge model choice.** For the quality SLOs whose evaluator is a rubric-graded LLM-as-judge, name the judge model and defend the "judge one tier above the model it judges" rule from mod-106.
- **Adversarial user path.** For Spec C, add a safety SLO row that targets prompt-injection attempts on the Slack bot (users pasting in wiki content that contains adversarial instructions). See mod-107 for the shape.

## Deliverable

A single markdown file, submitted to the paired solutions repo (`applied-ai-engineer-solutions`) under `mod-101/exercise-02/`.
