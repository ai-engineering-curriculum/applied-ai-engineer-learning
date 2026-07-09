# exercise-04: Query Rewriting and Classification-Gated Retrieval

**Estimated effort:** 3 hours

## Objective

Ship two applied-side retrieval levers — query rewriting and classification-gated retrieval — through the discipline of [chapter 05](../05-feature-side-levers.md): a written hypothesis before you build, an offline eval on the gold set from exercise-03, an online eval, and a kill-if-it-didn't-work rule. The goal is not to *use* the levers; it is to prove they are worth the latency and cost they add.

## Learning outcomes

By the end you should be able to:

- Turn a triad readout into a specific lever hypothesis with a measurable target.
- Implement a query rewriter that is instrumented against silent drift and prompt injection.
- Implement a query classifier that gates retrieval (and, when applicable, routes it to a specific source) with an audit trail.
- Cost each lever against a latency budget and reject one that lifts recall but breaks the SLO.
- Reject a lever that didn't move its metric, even if the trend looks favourable.

## Prerequisites

- You have read [chapter 05](../05-feature-side-levers.md).
- You have completed exercise-03 (gold set + eval pipeline) or have equivalent infrastructure.
- You have a small mock retriever (from exercise-01) with two or more sources.
- You have a model provider for the rewriter and the classifier — they can share a model or run different ones.

## Problem statement

You inherit the **helpdesk-reply-copilot** surface after two months of live traffic. The [chapter 04](../04-retrieval-quality-instrumentation.md)-style readout looks like:

| Stratum | recall@5 | context-relevancy (mean / p10) | faithfulness (mean / p10) | notes |
|---|---|---|---|---|
| head | 0.91 | 4.5 / 3.9 | 4.4 / 3.8 | Fine. |
| torso | 0.86 | 4.4 / 3.8 | 4.3 / 3.6 | Fine. |
| long_tail | 0.63 | 3.6 / 2.4 | 4.1 / 3.5 | Poor context-rel; faithfulness is OK — model correctly refuses more. |
| multi_hop | 0.58 | 3.5 / 2.2 | 3.9 / 2.8 | Bad. |
| chit_chat | n/a | n/a | n/a | ~12 % of live queries are conversational filler that shouldn't retrieve at all. Currently they do — and the model sometimes cites irrelevant passages. |
| adversarial | 0.71 | 3.9 / 3.1 | 4.0 / 3.2 | Not this exercise's job. |

Your SLO from mod-101 has ~150 ms of retrieval-side p95 headroom. The retriever team is not shipping a reranker upgrade this sprint.

You have two candidate levers to consider:

- **Query rewriting** on multi-turn or verbose queries (aimed at the long-tail and multi-hop strata).
- **Classification-gated retrieval** to skip retrieval on chit-chat queries and to route to the correct corpus on the rest.

Your job is to prove — or disprove — each with the discipline from [chapter 05](../05-feature-side-levers.md).

## Requirements

### Deliverable 1 — Two hypotheses

A file `hypotheses.md` containing two hypotheses, one per lever, following the shape from chapter 05:

- Named target metric.
- Named stratum (or strata) it applies to.
- Named delta target ("recall@5 on long_tail from 0.63 to ≥ 0.75").
- Named SLO cost target ("without moving p95 retriever latency by more than +80 ms").
- Named kill condition ("if the delta is below +0.05 or the p95 grows by more than +100 ms, don't ship").

A hypothesis you can't state in numbers is a hypothesis you can't reject. This deliverable is small; it is not optional.

### Deliverable 2 — The query rewriter

Implement `rewrite(query, conversation, tenant_id) -> RewriteResult` where `RewriteResult` includes `original`, `rewritten`, `rationale`, and `changed: bool`. Requirements:

- The rewriter is a bounded LLM call with a **frozen prompt version** (mod-102 chapter 05's versioning discipline). Log the prompt version alongside every rewrite.
- The rewriter refuses to expand facts. If the rewrite adds a claim not present in the original + conversation (an invented entity, an assumed policy detail), the rewriter's parser rejects it and the surface falls back to the raw query. Include a small test that exercises the reject path.
- Prompt-injection defence: the user's query is passed as *data*, not as *instructions*, and the rewriter's system prompt names an ignore-instructions rule. Include one adversarial test case (a query containing `Ignore all previous instructions and…`) and prove the rewriter does not comply.
- The rewritten query is shown in the "why this answer" trace ([chapter 03](../03-citations-and-evidence-ux.md)) whenever `changed=True`.
- If the rewrite doesn't materially change the query (edit distance below a small threshold), skip it and pass the original through.

### Deliverable 3 — The classifier

Implement `classify(query, conversation) -> Classification` where `Classification` includes `intent` in `{policy, billing, product, chit_chat, other}`, `retrieve: bool`, `sources: list[str]`, and `confidence: float`. Requirements:

- The classifier is a small LLM call (or a distilled model, or a rule-based classifier — your choice; document the trade-off).
- When `retrieve=False`, the downstream generator uses a prompt that **cannot** emit substantive claims — only acknowledgements, clarifications, or "I don't have information on that." This is the guard for the chapter-05 "missed-retrieval hallucination" failure mode.
- The classifier logs `(query, classification, model_version)` for every call to a store you can hand-review.
- On classifier failure (timeout, parse error), the surface fails **into retrieval** (fail-open on the retrieval decision, not on the ACL — the classifier's job is to save cost, not to enforce security).
- Every classifier decision is versioned so a policy change is auditable.

### Deliverable 4 — Offline eval on the gold set

Run the offline eval from exercise-03 with four arms:

- `A`: baseline — no rewriter, no classifier.
- `B`: rewriter only.
- `C`: classifier only.
- `D`: both.

Report, per arm, per stratum:

- recall@5, MRR.
- Context-relevancy, answer-relevancy, faithfulness (means and p10) from the online judge, if you can afford the eval budget offline.
- The p95 rewriter + retriever latency delta vs. baseline (measure the rewriter's contribution and the classifier's contribution separately).
- The rate at which the rewriter's "no material change" branch fires.
- The classifier's confusion matrix on a labelled subset of 50 queries you hand-label.

### Deliverable 5 — The decision memo

A file `decision.md` that:

- States, per lever, whether the hypothesis was met.
- Names the arm you are proposing to ship (A/B/C/D) and why.
- Names any lever you are proposing to *kill* because the hypothesis wasn't met, and resists any temptation to move the goalpost.
- Updates the retriever contract from exercise-01 with a note about the classifier's routing behaviour (the retriever team needs to know when a source becomes contingent on a classifier decision).
- Names three things the ongoing daily readout should monitor to catch drift on the shipped lever: e.g., "rate of chit_chat misclassification," "rate of rewriter refuse-to-expand-facts fires."

## Starter guidance

- **The rewriter is a rewrite tool, not a query-generator.** Its input is the query and the last few turns of conversation. Its output is a query. It does not do decomposition, expansion, HyDE, or step-back — those are separate levers with separate hypotheses.
- **Instrumentation is the point.** A rewriter that lifts recall by 8 % but silently mutates 3 % of the queries into off-intent rewrites is a lever that shipped a bug. The `rationale` and `changed` flags exist so you can catch the drift.
- **Classification-gated retrieval is a two-decision lever.** Decision A: retrieve or not. Decision B: which source. Both are wrong sometimes. Design counters for both. The wrong-source failure is subtle — the model has passages, they're plausible, and the answer is wrong. Catch it with the wrong-source case in the classifier's audit log and by comparing the classifier's `sources` against the answer's citations.
- **Don't stack levers if only one works.** If the classifier lifts multi-hop recall by 10 % and the rewriter is within noise, ship the classifier alone. Stacking a losing lever "just in case" adds latency, cost, and failure surface.
- **The classifier's misclassification rate matters more than its accuracy.** A classifier that is 90 % accurate on head queries and 60 % on long-tail is misleading in the aggregate; the long-tail is often where the surface fails. Report the confusion matrix per stratum.
- **You may find that neither hypothesis is met.** That is a valid outcome. Write the memo that says so and file the readout as-is. This exercise passes on the discipline, not on shipping a lever.

## Acceptance criteria

- [ ] `hypotheses.md` states each lever with a named target metric, delta target, SLO cost bound, and kill condition — in numbers.
- [ ] The rewriter is version-stamped, refuses to expand facts, has an adversarial-injection test that passes, and shows itself in the "why this answer" trace when `changed=True`.
- [ ] The classifier logs every decision with the model version, its "no-retrieval" branch uses a substantively-constrained prompt, and its failure mode is fail-open on retrieval.
- [ ] The offline eval reports arms A / B / C / D per stratum with recall, MRR, and the p95 latency delta split by lever.
- [ ] The classifier's confusion matrix is reported per stratum on a hand-labelled subset of 50.
- [ ] `decision.md` states the ship / kill decision per lever against the pre-registered hypothesis. If a lever is killed, the memo explains why the trend didn't count.
- [ ] The retriever contract is updated with any classifier-routing implications.
- [ ] Three drift-monitoring metrics are named for the shipped lever(s) and wired into the exercise-03 daily readout.

## Stretch goals

Pick any two:

- **HyDE, as a rejection.** Add a fifth arm `E` that layers HyDE on top of the best of A/B/C/D. Report the delta. If HyDE is within noise (as [chapter 05](../05-feature-side-levers.md) warns), reject it in the memo — and describe what would have to be true for you to reconsider it later.
- **Step-back on multi_hop only.** Add a sixth arm `F` that fires a step-back retrieval only when the classifier flags a query as multi_hop. Report the delta on the multi_hop stratum specifically and the p95 latency on non-multi-hop (which should be unchanged).
- **Rewriter cache.** Cache rewrites keyed by `(query, conversation_hash)`. Report the cache hit rate and the p95 latency delta on cache hits vs. misses. Do not cache across tenants.
- **Adversarial rewriter eval.** Curate 20 injection-shaped queries — direct injections, indirect injections carried into the query by conversation history, unicode-obfuscated instructions — and prove the rewriter refuses each. Any failure joins the [chapter 04](../04-retrieval-quality-instrumentation.md) adversarial stratum for the next eval round.
- **Automatic hypothesis re-check.** A weekly job that reruns the offline eval, notices the shipped-lever delta, and — if the delta collapses below the hypothesis target — files a ticket for review. This is the "kill unused features" discipline, applied to levers.

## Deliverable

Submit as `mod-103/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `hypotheses.md`
- `rewriter.py` + its tests.
- `classifier.py` + its tests.
- `eval_arms.py` — the four-arm A/B/C/D eval.
- `results.md`, `results.json`
- `decision.md`
- The updated retriever contract diff (link to exercise-01's file).
- Any stretch-goal artefacts named alongside.
