# Feature-Side Levers: Query Rewriting, Gated Retrieval, HyDE, Step-Back, Rerankers

## The temptation, and the discipline

Once you have the eval readout from [chapter 04](04-retrieval-quality-instrumentation.md), you will be tempted to try every RAG paper you've seen. Query rewriting, HyDE, step-back prompting, multi-query fusion, self-query, contextual compression, RAG-fusion, agentic retrieval — the surface is flooded with named techniques. Two things make this a bad instinct if you don't discipline it:

- **Every lever adds latency, cost, and failure modes.** A rewriter that takes 400 ms to save 3 % on recall@5 might still be a losing trade at the SLO in mod-101.
- **A lever that works on paper often fails on your traffic.** The techniques' original evaluations run on academic corpora with clean queries. Your live traffic has typos, product-specific jargon, missing context, and adversarial input. Recall on Natural Questions does not predict recall on your helpdesk tickets.

This chapter's job is to give you the catalogue *and* the discipline. Every lever gets four things: what it is, when it helps, when it hurts, and how the eval decides whether to keep it.

The organising rule: **you don't add a lever until the eval readout points at a specific failure mode, and you don't keep a lever unless the eval readout confirms it moved the metric it was supposed to move.**

## The catalogue

### 1. Query rewriting

**What**: an LLM call (or a deterministic normaliser) that turns the user's raw query into a form the retriever ranks better against. Fix typos, expand acronyms, resolve pronouns from the conversation, normalise jargon.

**When it helps**:

- Multi-turn conversations where the current query references prior turns ("what about the Enterprise tier?" → "how does the Enterprise tier compare to the Pro tier for annual billing").
- Domains with heavy jargon or acronyms that the corpus spells out ("MFA" → "multi-factor authentication").
- Verbose user queries where the retriever benefits from a distilled search intent.
- Cross-language queries where the corpus is in one language and the user query drifts.

**When it hurts**:

- Short, clean queries. Rewriting adds latency for no lift and occasionally makes the query *worse* (over-specification, drift).
- Corpora where the exact user phrasing is what matches. Legal / regulatory content often reads better on the user's own quoting.
- High-QPS surfaces where the rewrite's tail latency dominates the retrieval budget.

**Failure modes**:

- **Silent drift**: the rewrite reshapes the intent. Log the raw query and the rewrite; sample manually every week.
- **Hallucinated context**: the rewriter invents facts from thin air ("about our Enterprise plan" becomes "about our Enterprise plan that includes SSO and audit logs"). Constrain the rewriter's prompt; measure it.
- **Injection surface**: a rewriter that reads and re-emits user text opens a prompt-injection channel. See mod-107; sanitise, don't just concatenate.

**Instrumentation**: A/B the rewriter vs. no-rewriter on the gold set; track recall@k, faithfulness, and p95 retriever latency delta. Show the rewrite in the "why this answer" trace ([chapter 03](03-citations-and-evidence-ux.md)) so users can catch drift.

### 2. Multi-query / query expansion

**What**: instead of one query, send `n` variants (rephrasings, decompositions, sub-queries) to the retriever and fuse the results — most commonly with Reciprocal Rank Fusion (RRF).

**When it helps**:

- Complex or compound queries that don't retrieve well as a single string ("compare X and Y policies" splits into two queries).
- Ambiguous head queries where the "right" phrasing varies.
- Corpora where different sources use different vocabulary for the same concept.

**When it hurts**:

- Linear cost multiplier: `n` retrieval calls. On a surface with a tight `p95` budget this pushes you over.
- Overloads the reranker (more candidates to rerank) or the LLM (more passages in the pack, cache-key drift).
- The variants add nothing on clean queries and burn budget.

**Failure modes**:

- **Duplicate results across variants** dominate the fused list, crowding out unique matches. RRF handles this reasonably; naive union does not.
- **Variant quality collapse**: the LLM emits `n` variants that are near-identical paraphrases. Log the variants; measure lexical diversity; abort when variants are too similar.
- **Coverage explosion**: each variant may hit a different source, blowing past the pack size limit and pushing an on-topic passage out.

**Instrumentation**: measure the *marginal* recall lift of variants 2 through `n` on the gold set. If variant 2 adds nothing, `n=1` was correct.

### 3. Classification-gated retrieval

**What**: a lightweight classifier (small LLM, distilled model, or rule-based) inspects the query and decides *whether* to retrieve at all, and *which* index / source to retrieve from.

**When it helps**:

- Mixed-intent surfaces: chat that sometimes asks a product question (needs retrieval) and sometimes says "hi" (does not). Skipping the retrieval saves latency and prevents ungrounded citations on chit-chat.
- Multi-corpus surfaces: a query about billing goes to the billing sources; a query about product features goes to the KB. Cheaper and more accurate than always querying every source.
- Compliance branches: a legal-shaped query routes to the reviewed-legal-content source; a general query routes elsewhere.

**When it hurts**:

- The classifier is wrong. A misclassified query gets no retrieval and hallucinates from parametric knowledge — the exact failure mode you're paying eval to prevent.
- The classifier adds latency ahead of the retriever, which is the wrong place if the surface is already tight.
- Categories drift. What used to be a clean "billing" query starts including "cancellation" as billing changed hands.

**Failure modes**:

- **Silent misclassification** — the query goes to the wrong index and returns plausible-but-wrong passages. Guard: log classification decisions; sample and hand-label weekly; retrain when misclassification exceeds a threshold.
- **Missed-retrieval hallucination** — the classifier says "skip retrieval," the model answers from parametric knowledge, the surface renders it. Guard: on the "skip retrieval" branch, use a prompt that *cannot* emit substantive claims (limit to acknowledgements / clarifications).

**Instrumentation**: report the classification confusion matrix on a labelled subset every week. Report the "skipped retrieval" rate and audit a sample of those answers for grounding.

### 4. HyDE (Hypothetical Document Embeddings)

**What**: the LLM generates a hypothetical *answer* to the query, then the embedding of that hypothetical answer (not the query) is what the vector store searches with. The theory: user queries are short and stylistically different from documents; a hypothetical answer looks more like a document and matches better.

**When it helps**:

- Corpora where documents and queries have very different styles (documents are long, formal, structured; queries are short, colloquial).
- Zero-shot retrieval on niche corpora where fine-tuning the embedder isn't an option.
- Certain long-tail queries where a hypothetical answer disambiguates the intent.

**When it hurts**:

- Adds an LLM call and its tail latency into the retrieval hot path.
- The hypothetical answer can hallucinate specifics that mislead the retriever — you retrieve the wrong document *because* the hypothetical fabricated a plausible-sounding fact.
- On modern instruction-tuned embedders (E5, BGE, OpenAI `text-embedding-3-*`, Cohere embed v3, Voyage v3), which are trained on query→document pairs, HyDE's lift is often marginal or negative.

**Failure modes**:

- **Anchor drift**: a hypothetical answer that trends off-topic drags retrieval off-topic. Watch the top-hit stability across the raw-query and HyDE arms of the eval.
- **Retriever-specific behaviour**: HyDE lifts on one embedder and hurts on another. Bake the lift measurement into the retriever contract handoff.

**Instrumentation**: this is a lever you A/B on the gold set and *only* keep if the readout says so. Do not adopt HyDE by default. See "the decision framework" below.

### 5. Step-back prompting

**What**: before retrieving, the LLM generates a more *general* version of the query (a "step-back question"), retrieves against that too, and adds the results to the pack. The theory: for multi-hop or context-dependent questions, a more general query surfaces the enabling principle that the specific query would miss.

**When it helps**:

- Multi-hop questions where the answer needs a general principle plus a specific fact.
- Domains with a hierarchy (policies → procedures → forms; concepts → APIs → parameters) where the general layer often has better recall than the specific layer.

**When it hurts**:

- Simple factual queries: the step-back gets you noise; the specific query already retrieves the right document.
- The reranker has to earn the extra passages' relevance; if it can't, the step-back results crowd out the specific results.

**Failure modes**:

- **Over-generalisation**: the step-back becomes "what is our product?" and retrieves marketing homepage boilerplate.
- **Cost multiplier**: another LLM call + another retrieval call, both in the hot path.

**Instrumentation**: track recall@k on the gold set's multi-hop stratum specifically. Step-back that lifts multi-hop but degrades single-hop is fine if you can classification-gate it (only invoke for queries the classifier flags as multi-hop). Whole-surface adoption is rarely justified.

### 6. Contextual compression / summarised passages

**What**: after retrieval, each passage is passed through an LLM that extracts / summarises the parts relevant to the query. The prompt then consumes summaries instead of full passages.

**When it helps**:

- Long-passage corpora where a 2000-token chunk is mostly irrelevant to the specific query.
- Tight context budgets where you cannot afford to include full passages.

**When it hurts**:

- Every summariser call adds latency. `k` passages × summariser latency.
- The summary strips context the model needed. Faithfulness drops even as answer-relevancy rises.
- Citations lose fidelity — you're now citing a summary of the passage, not the passage itself. The evidence panel has to preserve the original span for the user to verify against.

**Failure modes**:

- **Silent evidence loss**: the summariser drops the sentence that would have supported the answer; the model then either hallucinates or refuses.
- **Consistency across summaries**: two similar passages summarised into contradictory blurbs. Rare; noticeable.

**Instrumentation**: measure faithfulness before and after; measure whether the evidence panel citations still land the user on the *quoted-in-the-summary* text of the original passage.

### 7. Reranker consumption

**What**: the retriever team ships a reranker (cross-encoder, hosted like Cohere Rerank, or a distilled in-house model). The surface consumes it, either via a flag on the retriever request (`rerank=True`) or as a separate reranker step in the surface's own control.

**What you own on the applied side**:

- **Choosing whether to rerank** for each surface. Reranking adds latency; on a surface where the retriever's top-k is already high-recall, reranking's lift can be within noise.
- **Choosing `k` in and `k` out.** The retriever returns a candidate set (`k_in`, e.g. 50); the reranker cuts it to `k_out` (e.g. 6). `k_in` too small starves the reranker; `k_in` too large adds cost and latency.
- **Consuming the reranker's scores.** Bucket, don't threshold on raw numbers. The retriever team publishes bucket boundaries; you consume the buckets.
- **Deciding when to skip.** For chit-chat queries the classifier flagged as non-retrieval, don't rerank.

**What you don't own**:

- The reranker model itself — that's the retriever team's decision.
- The reranker's training data or fine-tuning cadence.
- Its cost model (that's negotiated between the retriever team and finance).

**Failure modes on the applied side**:

- **Rerank-on-every-call by default** — burns cost / latency on surfaces where it's marginal.
- **Score threshold hard-coded** — reranker upgrade shifts the scale, threshold silently drops or admits garbage. See [chapter 01](01-consuming-a-retriever-you-do-not-own.md).
- **Reranker output ignored** — the surface asks for rerank=True and then re-sorts by the retriever's original score. Debug this before shipping.

**Instrumentation**: measure the rerank on/off delta on the gold set's context-relevancy and on live faithfulness. On some surfaces the delta is +8 pts on context-relevancy for +80 ms of p95; on others it's noise. Both are valid decisions.

## The decision framework

Given the catalogue, how do you actually decide? Two rules and a workflow.

### Rule 1: Point the lever at the failure mode

Read the [chapter 04](04-retrieval-quality-instrumentation.md) triad readout. Each column implies a lever:

| Failing column | Lever to consider first |
|---|---|
| Low context-relevancy on head queries | Query rewriting; reranker; check the retriever contract |
| Low context-relevancy on multi-hop | Step-back on the multi-hop stratum only |
| Low context-relevancy on jargon-heavy queries | Query rewriting (acronym / jargon expansion) |
| Low context-relevancy in a specific stratum | Classification-gated retrieval targeting that stratum |
| Low context-relevancy overall on a niche corpus | HyDE — but eval it before adopting |
| Low faithfulness with high context-relevancy | Not a retrieval lever — it's a prompt / model problem (mod-102, mod-108) |
| Low answer-relevancy with high context-relevancy | Prompt / query-rewriting problem |
| High no-good-evidence rate | The classifier is over-suppressing, coverage is short, or the retriever contract has a gap ([chapter 02](02-freshness-coverage-permission-contract.md)) |

Note the last row: low faithfulness at high context-relevancy is not solved by another retrieval lever. Diagnose before you pile on.

### Rule 2: Cost the lever against the SLO

Before adopting a lever, cost it against mod-101's SLO table:

- Added p95 latency (worst case, including cold-start on the extra LLM call).
- Added cost per successful outcome.
- Added failure-mode surface (each new stage is a new place to fail).

If a lever costs 200 ms of p95 and lifts recall@5 by 2 %, and your SLO already has 100 ms of headroom, the lever either doesn't ship or ships behind a classifier that invokes it only when it actually helps.

### The workflow

For any lever you're considering:

1. **Write the hypothesis.** "Query rewriting will lift recall@5 on the multi-turn stratum from 0.72 to ≥ 0.80 without dropping the p95 SLO." One sentence. If you can't articulate it, don't start.
2. **Run the offline eval.** On the gold set, in CI. Measure recall@k, MRR, and p95 latency delta.
3. **Run the online eval.** On a small slice of live traffic, measure faithfulness / answer-relevancy / hit rate with and without.
4. **Decide against the hypothesis, not the trend.** If the hypothesis said "≥ 0.80" and you got 0.76, the answer is "no" even if the trend went up. Don't rescue a losing lever with a moved goalpost.
5. **If you keep the lever, log it in the retriever contract.** Any lever added on the surface is a caller-side commitment the retriever team should know about. Update the contract doc.
6. **Add it to the failure-mode catalogue.** Every lever comes with its own failure modes. Wire them into the eval readout and the [chapter 03](03-citations-and-evidence-ux.md) trace.

### Named non-decisions

There are two "levers" that keep coming up and are almost always the wrong answer:

- **"Let's fine-tune the embedder."** This is retriever-side territory. If you're on the applied side and reaching for embedding fine-tuning, escalate to the retriever team with the eval readout, don't ship your own embedder.
- **"Let's build an agent that iteratively retrieves."** Agentic retrieval multiplies cost and latency by an order of magnitude for lifts that are often within noise. If you're tempted, first (a) confirm single-shot retrieval is genuinely failing on the gold set, and (b) look at what the multi-hop stratum needs — often step-back is enough. Agentic retrieval is a real technique; it belongs in mod-105 with the shipping-agents discussion, not as a default retrieval mode.

## A worked example: fixing a low-recall stratum

You inherit the helpdesk co-pilot from earlier chapters. The [chapter 04](04-retrieval-quality-instrumentation.md) readout looks like:

| Stratum | recall@5 | context-rel | faithfulness |
|---|---|---|---|
| Head | 0.91 | 4.5 | 4.4 |
| Torso | 0.86 | 4.4 | 4.3 |
| Long-tail | 0.63 | 3.6 | 4.1 |
| Multi-hop | 0.58 | 3.5 | 3.9 |
| Adversarial | 0.71 | 3.9 | 4.0 |

Faithfulness is fine everywhere — the model is not lying about what it's given. Recall on long-tail and multi-hop is bad; context-relevancy correlates.

Working the framework:

- **Long-tail: query rewriting.** Hypothesis: the long-tail queries are verbose and jargon-heavy; a rewriter will lift them. Offline eval: rewriter arm lifts long-tail recall@5 from 0.63 → 0.78 with +90 ms p95. SLO has 200 ms headroom. Ship, behind the "why this answer" trace so drift is visible.
- **Multi-hop: step-back, classification-gated.** Hypothesis: multi-hop queries need a step-back but head queries do not. Add a classifier that fires only on the multi-hop stratum. Offline eval on the multi-hop stratum: recall lifts from 0.58 → 0.72; on all other strata it's within noise (which is what you wanted — the classifier is doing its job). Ship.
- **Adversarial: not fixed here.** The adversarial stratum's issue is not retrieval — it's likely prompt / model / guardrail. Escalate to mod-107 and mod-102 rather than pile on a retrieval lever.
- **Reranker: no change.** The retriever team already includes a reranker on this surface. The eval says its rerank-on/off delta is +5 pts of context-relevancy on head + torso for +60 ms p95. Keep as-is.

Two levers ship, both grounded in the readout, both with an explicit hypothesis, both with instrumentation to catch drift. Zero levers ship because a paper described them.

## Summary

- The applied-side lever catalogue: query rewriting, multi-query fusion, classification-gated retrieval, HyDE, step-back, contextual compression, and reranker consumption.
- Each lever has a where-it-helps, where-it-hurts, failure-modes, and instrumentation profile. None is a default.
- The decision framework has two rules: point the lever at the failure mode the eval readout named, and cost the lever against the SLO. Add a written hypothesis before you build.
- Reranker choice is retriever-side; reranker *consumption* (when to invoke, how to consume its scores) is applied-side. The distinction survives the reranker changing under you.
- Two levers are almost always the wrong answer on the applied side: fine-tuning the embedder (escalate to the retriever team) and agentic retrieval (belongs with agents in mod-105).
- The worked example shows the discipline: two levers shipped, both traceable to the readout; the levers that weren't right for the failure mode did not ship.

Return to [`README.md`](README.md) for the module index. The exercises put this whole chain into practice: contract, UX, instrumentation, and levers, each in its own end-to-end problem.
