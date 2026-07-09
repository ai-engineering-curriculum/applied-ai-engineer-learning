# exercise-03: Retrieval Quality Instrumentation

**Estimated effort:** 4 hours

## Objective

Stand up the product-altitude retrieval-eval stack for a specific surface: build the labelled gold set with a stratified query mix, compute recall@k / MRR in CI, wire the four live hit-rate flavours, run a sampled online judge for the RAG triad, and publish the resulting per-surface readout with the citation-quality metrics from [chapter 04](../04-retrieval-quality-instrumentation.md).

## Learning outcomes

By the end you should be able to:

- Curate a small (100–200) but stratified gold set with defensible labels.
- Compute and interpret recall@k, MRR, and hit rate at the product altitude — not the retriever's internal benchmark.
- Configure a RAG-triad judge (Faithfulness / Answer-Relevancy / Context-Relevancy) and calibrate it against a small hand-labelled subset.
- Read a triad readout and diagnose which side of the boundary a regression sits on.
- Wire the metrics into a daily readout other people can act on.

## Prerequisites

- You have read [chapter 04](../04-retrieval-quality-instrumentation.md).
- You have a mock or real retriever and a mock or real generator (any model provider).
- You are comfortable with a Python test runner and can plot or tabulate results.
- You have `ragas`, `deepeval`, `trulens-eval`, or a plain LLM-as-judge harness available. Any of them is fine; the point is the readout, not the tool.

## Problem statement

You are the applied-AI engineer on the **helpdesk-reply-copilot** surface from exercise-01. The retriever team has shipped an initial version. You now have to prove — with data — whether it meets the surface's needs, where it's underserving specific query strata, and whether a proposed reranker rollout next sprint will actually help.

You have two weeks of live traffic (~5,000 queries) available for sampling, and a corpus of 3 sources. You do not have infinite eval budget — you can afford ~2,000 judge calls per week on live traffic and one 10-minute CI run per PR.

## Requirements

### Deliverable 1 — The gold set

A file `gold_set.jsonl` (or `.yaml`) containing 100–200 queries with labels. Each entry has:

- `query`: the user string, as sent to the surface.
- `stratum`: one of `head`, `torso`, `long_tail`, `ambiguous`, `impossible`, `adversarial`.
- `must_contain_any`: list of `passage_id`s, at least one of which the retriever must return in the top `k`.
- `must_contain_all` (optional): list of `passage_id`s, all of which the retriever must return (for multi-hop).
- `must_not_contain` (optional): list of `passage_id`s that would be wrong to surface.
- `is_impossible`: `true` for the impossible stratum; the correct pack is empty.
- `notes`: one line on why this query is in the set.

Requirements:

- All six strata are represented; at minimum 10 queries per stratum.
- The head stratum's queries are sampled from real live traffic (you can synthesise 5,000 queries for the exercise; document how you would replace them with real ones in production).
- The adversarial stratum includes at least one prompt-injection-shaped query and one cross-tenant-ID query.
- Every label has a paper trail: the annotator (you), the date, and a passage-choice justification for the non-obvious queries.

### Deliverable 2 — Offline recall@k in CI

A CI script `eval_offline.py` that:

- Loads the gold set and calls the retriever client (mock or real) for every query.
- Computes `recall@1`, `recall@5`, `recall@k_max` (the surface's production `k`) both overall and per stratum.
- Computes `mrr` overall and per stratum.
- Emits a `results.json` and a `results.md` table.
- Fails the build if recall@5 on any stratum drops by more than a configurable threshold from the last committed baseline.

The build must run in under 10 minutes with a mock retriever; document how you'd budget it against a real retriever call.

### Deliverable 3 — Live hit-rate metrics

A live-traffic sampling script `eval_live.py` that, given a stream of production requests + responses:

- Computes per-day: `pack_returned_rate`, `usable_pack_rate`, `cited_pack_rate`, `no_good_evidence_rate`.
- Groups the numbers by surface, tenant, corpus, and prompt version.
- Emits a daily report to `daily_readout.md`.

You can synthesise the "stream" from a JSONL fixture.

### Deliverable 4 — Sampled online judges (the RAG triad)

A judge script `eval_triad.py` that:

- Samples 1–5 % of live traffic (target ~200 answers/day given the traffic).
- Runs an LLM-as-judge for Faithfulness, Answer-Relevancy, Context-Relevancy — using a **different model family** from the one that generated the answer.
- Publishes the rubric it uses in `judge_rubric.md`. The rubric is frozen for this eval version.
- Computes mean and **p10** for each metric per surface per day and joins them into the daily readout from deliverable 3.
- Includes an on-disk log of `(query, answer, judgment, judge_model_version)` so you can re-run against an updated rubric later.

### Deliverable 5 — Judge calibration

- Hand-label 30 sampled answers against the same rubric.
- Compute agreement between your labels and the judge's labels.
- Report the agreement rate in `calibration.md`.
- Set (and justify) a threshold below which the judge's scores are not trusted; if the calibration falls below it, block the judge from writing into the daily readout until the rubric or the judge model is re-tuned.

### Deliverable 6 — The daily readout

A single `readout.md` template (populate it with the values you've computed) that includes:

- The RAG-triad table from [chapter 04](../04-retrieval-quality-instrumentation.md), sliced by stratum and by tenant.
- The citation-quality metrics: `citation_hallucination_rate`, `cited_but_unsupported_rate` (from the online judge).
- The versions in play: retriever, embedder, reranker, prompt, gold-set version.
- The change log: which levers or contract changes shipped since the last readout.
- A "what to do" section — the three lowest cells and the current owner for each.

## Starter guidance

- **Write the "impossible" stratum first.** It clarifies your thinking. If you can't articulate what the retriever *should not* find, you don't yet understand what it should find.
- **Sample live queries by hand for the head stratum.** Pull a random 500 queries, cluster them (or eyeball them), and pick 30–50 that represent the main patterns. This is 45 minutes of work; a synthetic head stratum is worse than none at all.
- **Do not use the generator model as the judge.** [Chapter 04](../04-retrieval-quality-instrumentation.md) warns about this — the judge is 10–20 % more charitable to its own phrasings. Use a different family (Anthropic → OpenAI, or vice versa; if you have only one provider, at least use a different model tier and document the risk).
- **Freeze the rubric before you start scoring.** Every metric moves under a shifting rubric. Publish the rubric, version it, and only revise between eval runs — never mid-run.
- **Report p10, not just mean.** A mean of 4.5 hides answers scoring 1.0. The tail is where the failures live. Your readout is not finished if you can't quote the p10.
- **The recall@k threshold in CI is a business decision.** For this exercise, pick a threshold and justify it. In production the threshold is negotiated with the retriever team, ideally in the contract from exercise-01.
- **Do not use `ragas` or `trulens` without reading their current docs.** These tools' rubrics have moved; version-pin the tool and record the version alongside the judge.

## Acceptance criteria

- [ ] `gold_set.jsonl` has at least 100 queries across all six strata, with a minimum of 10 per stratum.
- [ ] Every labelled query has a `notes` field naming why it's in the set.
- [ ] `eval_offline.py` computes recall@1, recall@5, recall@k_max, and MRR overall and per stratum; results are in `results.md`.
- [ ] The CI script fails the build on a recall@5 drop beyond a configurable threshold; the threshold and its justification are documented.
- [ ] `eval_live.py` computes all four hit-rate flavours and produces `daily_readout.md`.
- [ ] `eval_triad.py` runs a judge in a different model family from the generator, uses a frozen rubric, and reports mean + p10 for each metric.
- [ ] `calibration.md` reports the judge/human agreement rate and names a threshold below which the judge is not trusted.
- [ ] `readout.md` is a single-page daily report showing the triad table sliced by stratum + tenant, citation-quality metrics, versions, and a "what to do" section.
- [ ] The pipeline exposes counts and rates via structured logging or metric points — no scraping HTML tables.

## Stretch goals

Pick any two:

- **Per-tenant regression detection.** A script that, per tenant, flags any triad column that dropped by more than a threshold day-over-day. Prove it doesn't fire on random noise (bootstrap-sample the same day's data 50 times and confirm the alert rate is low).
- **Rerankers, before and after.** Run the offline eval with `rerank=True` and `rerank=False`. Report the delta per stratum. Recommend whether to ship the reranker rollout based on the readout.
- **A/B lever eval.** Extend the pipeline to accept a `treatment` label per request and compute the triad delta between arms, with a naive confidence interval. Use this to answer: does query rewriting on multi-turn queries lift context-relevancy without dropping faithfulness?
- **Adversarial fixture rotation.** After each red-team pass, promote broken queries into the adversarial stratum. Write the promotion script and show the gold-set version bump.
- **Human-in-the-loop refresh.** A weekly script that samples 20 live-traffic queries, prompts a human to label them, and appends them to the gold set — including the version bump and change log.

## Deliverable

Submit as `mod-103/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `gold_set.jsonl`
- `eval_offline.py`, `results.md`, `results.json`
- `eval_live.py`, `daily_readout.md`
- `eval_triad.py`, `judge_rubric.md`
- `calibration.md`
- `readout.md`
- A short `README.md` explaining how to run the pipeline end-to-end.
- Any stretch-goal artefacts named alongside.
