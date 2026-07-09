# Consuming a Retriever You Do Not Own

## Why this is the first chapter

Retrieval-augmented generation is a joint delivery. There is a **retrieval side** — chunking, embeddings, vector index, hybrid ranking, reranker, refresh pipeline — and there is a **feature side** — the query that goes in, the prompt that consumes the passages, the citations that render, the "no good evidence" branch, the eval that decides whether the answer was good. In a mature org those two sides are owned by different tracks. The `rag-engineer` peer track owns the retriever and the vector store; the applied-AI engineer owns the surface that consumes them.

The whole module hinges on getting that split right. If the feature owner reaches into the retriever's internals — retunes chunk sizes to fix a citation bug, swaps the embedding model to chase a recall number, hard-codes a hybrid-weighting parameter into the surface — the split collapses. The retriever team can no longer change anything without breaking the surface, and the surface team ends up debugging a component they don't understand and don't operate on-call. Conversely, if the feature owner treats the retriever as a black box and doesn't articulate what they need from it, the retriever team ships something reasonable-on-paper that fails on live traffic in ways only the feature owner sees.

The applied-AI engineer's job is to draw the line, negotiate what crosses it, and instrument both sides of it. This chapter is that line.

## What the rag-engineer owns

Treat the following as **implementation details you deliberately do not depend on**. If you find your surface code referencing any of them by name or number, treat it as a boundary violation to be cleaned up.

- **Chunking strategy** — token counts, semantic vs. fixed windows, sliding vs. non-overlapping, per-doctype special cases. The retriever team may change these to move a recall number and you should not care beyond the SLO in [chapter 02](02-freshness-coverage-permission-contract.md).
- **Embedding model choice and dimensionality** — the retriever team may swap embedders (and re-embed the corpus) to change cost, latency, or quality. The surface never names the model.
- **Vector index and its parameters** — which store (pgvector, Qdrant, Weaviate, Vespa, Elasticsearch, OpenSearch, LanceDB, Milvus, Pinecone, Turbopuffer, Vertex Vector Search, Bedrock Knowledge Bases, etc.), which ANN algorithm (HNSW, IVF, ScaNN, DiskANN), and its tuning parameters (`ef`, `M`, `nlist`, `nprobe`).
- **Hybrid retrieval mechanics** — BM25/BM25F configuration, dense-sparse fusion (RRF, weighted sum), stopwords and stemmers.
- **Refresh pipeline** — how documents flow from source-of-truth into the index, how deletions propagate, how re-embedding runs. You consume the SLO, not the DAG.
- **Reranker choice** — whether they use a cross-encoder (BGE, MiniLM), a hosted reranker (Cohere Rerank, Voyage Rerank, Jina), or a fine-tuned in-house one. The surface asks for reranked results; the retriever provides them.

You *do* need a mental model of what these components are — [resources.md](resources.md) points at the primary references from the retriever side. But knowing how HNSW works so you can *talk* to the retriever team is different from having HNSW parameters in your surface config.

## What the applied-AI engineer owns

Everything that touches the product surface:

- **The query that goes to the retriever.** This includes any rewriting, expansion, decomposition, classification-gating, HyDE draft, or step-back generalisation you do on the way in ([chapter 05](05-feature-side-levers.md)).
- **The prompt that consumes the passages.** How they are formatted, how they are ordered, how many make it past the token budget, how citations are keyed, what happens if the pack is empty.
- **The citations and evidence UX.** What the user sees, how they verify, how you handle the "no good evidence" case ([chapter 03](03-citations-and-evidence-ux.md)).
- **The product-altitude metrics.** Recall@k on a labelled gold set, live hit rate, faithfulness, answer-relevancy, RAG-triad readout — measured on the surface, per surface ([chapter 04](04-retrieval-quality-instrumentation.md)).
- **The contract with the retriever team.** SLOs on freshness, coverage, ACL enforcement, retriever latency; the versioning discipline that governs breaking changes ([chapter 02](02-freshness-coverage-permission-contract.md)).

Two useful heuristics for deciding which side owns a question:

1. **If the fix requires editing the corpus, the index, or the ranker, it is retriever-side.** The applied-AI engineer files a bug against the retriever with a reproducer and a product-altitude metric that regressed.
2. **If the fix requires editing the prompt, the query, the UX, or the eval, it is feature-side.** The retriever team gets a heads-up if the fix changes the query shape they receive (see "what crosses the boundary" below).

## The consuming shape: a thin retriever client

The concrete artefact that lives on the applied-AI side of the boundary is a **retriever client** — a small, versioned interface the retriever team publishes and the feature calls. Everything on the surface goes through it. Nothing else in the surface talks to the vector store.

```python
# retriever_client.py — owned jointly, versioned, on the boundary.
from dataclasses import dataclass
from typing import Literal

@dataclass(frozen=True)
class RetrieveRequest:
    query: str
    tenant_id: str
    user_id: str            # for ACL — see chapter 02
    surface_id: str         # "helpdesk-copilot", "policy-qa", etc.
    k: int = 8              # cap; retriever may return fewer
    filter: dict | None = None   # opaque, retriever-defined filter schema
    rerank: bool = True
    trace_id: str = ""

@dataclass(frozen=True)
class Passage:
    passage_id: str         # stable across refreshes, retriever-owned
    doc_id: str
    doc_title: str
    doc_url: str            # canonical, for citations
    text: str
    score: float            # post-rerank if rerank=True; retriever-defined scale
    metadata: dict          # opaque; document-type, updated_at, section, etc.

@dataclass(frozen=True)
class RetrieveResponse:
    passages: list[Passage]
    debug: dict             # timings, index version, embedding model version
    index_freshness: dict   # per-source last-refresh timestamp — see chapter 02
```

Three things about this shape are load-bearing:

- **`tenant_id` and `user_id` are required, not optional.** ACL enforcement is a contract, not an opt-in ([chapter 02](02-freshness-coverage-permission-contract.md)).
- **`surface_id` is required.** Retrievers behave differently across surfaces (different reranker, different `k`, different filter shape). Rather than encode that in the caller, the retriever routes on `surface_id`. The applied-AI engineer negotiates its meaning with the retriever team.
- **The scoring scale is retriever-defined and opaque.** The surface may sort by it, may show a bucketed confidence in the UX, and may threshold on it — but only against a threshold the retriever team publishes for that `surface_id`. Never hard-code a numeric threshold that assumes the current embedder is still in place next quarter.

## What crosses the boundary — and how

The interesting engineering isn't in either side alone; it's in what the two sides publish to each other. Six artefacts cross the line in both directions.

### 1. The surface-side retrieval SLO (feature → retriever)

The applied-AI engineer writes down, per surface, what "good enough" retrieval means at the product altitude: `recall@k` on a labelled gold set, `hit rate` on live traffic, `p95 retriever latency`, and the freshness / coverage / ACL contract in [chapter 02](02-freshness-coverage-permission-contract.md). This is the retriever team's target — not their internal offline metric on the raw index.

### 2. The index version + embedding version + reranker version (retriever → feature)

The retriever returns, on every response, the current versions of the components it composed. The surface logs them next to every generation. When quality regresses, the two sides can trivially correlate "the answer-relevancy dropped at 14:07 UTC" with "the reranker rolled out at 14:05 UTC." Without this, every incident starts with an argument about whether "anything changed."

### 3. The filter schema (retriever → feature, versioned)

The retriever publishes a schema for the `filter` field ("you may filter by `doctype`, `updated_after`, `product_line`; you may not filter by `chunk_index` because chunking is our concern"). The surface validates its filter against that schema in CI. A breaking change to the filter schema goes through a deprecation window like any other API.

### 4. The corpus manifest (retriever → feature)

Which sources are indexed for this `surface_id`, when each was last refreshed, and what fraction of the source is in scope. The surface uses this to answer "why isn't X in the results" without a Slack thread, and to render coverage warnings when a known source is stale ([chapter 03](03-citations-and-evidence-ux.md)).

### 5. The joint gold set (feature ↔ retriever)

A labelled dataset of representative queries with the passages a good retriever should surface. The applied-AI engineer curates the queries (they know the product); the retriever engineer curates the labelled passages (they know the corpus). Both sides run `recall@k` on it in CI. Ownership is shared; the file lives in a repo both teams commit to.

### 6. The escalation channel (feature ↔ retriever)

A named channel — Slack, Linear, whatever — for surface-side regressions that look retriever-shaped and vice versa. This is the operational load-bearing artefact: without it, every degradation becomes a discovery process rather than a triage.

## Anti-patterns at the boundary

These have all bitten real teams.

1. **Reaching past the client.** The surface imports the vector-store client directly "just for this one debug query," and six months later half the surface is bypassing the retriever's ACL enforcement.
2. **Hard-coded score thresholds.** `if p.score < 0.72: skip` in the surface code. The retriever team swaps embedders, the score scale shifts, the threshold silently drops or admits garbage.
3. **The retriever team owning the prompt.** Because the retriever team built the demo and the prompt was easier to write than to hand off, the retriever team ends up owning a piece of surface-side code they don't run on-call. The prompt drifts because nobody with product context is editing it.
4. **The feature team owning the corpus.** Because "we needed just this one PDF indexed" the applied-AI engineer stands up a side ingestion. It has no refresh pipeline, no ACL, and no freshness SLO. It works for a demo and rots in production.
5. **Undeclared surface differences.** Three surfaces share one retriever, but each calls it with a different `k`, a different filter, and a different reranker flag, undocumented. When the retriever team wants to change a default, they can't tell whose surface will break.
6. **Reranker as a surface feature.** The surface team is convinced the reranker is the problem and swaps it out for a locally cached one. Now the surface and the retriever team disagree on what "the reranker" means, and RAG-triad numbers stop being comparable.

Each of these has the same root cause: an artefact that should have been on the boundary ended up on one side only. The fix is always the same — publish the artefact, version it, and put it in the retriever client's contract.

## A worked example: the helpdesk co-pilot

Return to the co-pilot reply drafter from mod-101. Where does the boundary fall?

- **Retriever side.** The knowledge-base corpus (public help docs), the previous-replies corpus (private per-tenant), the chunker for each, the embedder, the ANN index, the reranker, the refresh pipeline that watches the KB and the ticket store, the per-tenant ACL that ensures Acme's agents never see BigCo's tickets.
- **Feature side.** The query (the customer email + the ticket subject + a rewritten "user intent"), the prompt that consumes the passages, the citation panel in the reply sidebar, the "no good evidence — do not draft" branch, the eval that tracks whether sent replies were edited before send.
- **The boundary.** A `RetrieveRequest` with `surface_id="helpdesk-reply-copilot"`, `tenant_id`, `user_id` (the agent, for their ACL), a filter that scopes to this tenant's ticket history and their subscribed KB, and `k=6`. The retriever returns 6 reranked passages plus the current index-freshness manifest.

The applied-AI engineer's PRs edit the query rewriting, the prompt, the sidebar UX, and the eval. They *never* edit the chunker, the embedder, the index, or the reranker. When the reranker changes, the surface logs the new reranker version and watches the RAG-triad readout; if faithfulness drops, they file a bug with the reproducer, not a fix.

## Summary

- RAG is a joint delivery. Draw the line before you write the first `retrieve()` call.
- The `rag-engineer` peer track owns chunking, embeddings, the vector index, hybrid ranking, the reranker, and the refresh pipeline. The applied-AI engineer owns the query, the prompt, the citations UX, the product-altitude metrics, and the contract.
- The concrete artefact on the boundary is a versioned retriever client that requires `tenant_id`, `user_id`, and `surface_id`, returns opaque `passage_id`s and a retriever-defined score scale, and reports index / embedder / reranker versions on every response.
- Six things cross the boundary: the surface-side retrieval SLO, the versions readout, the filter schema, the corpus manifest, the joint gold set, and the escalation channel. Publish and version each of them.
- Six anti-patterns collapse the split: reaching past the client, hard-coded score thresholds, the retriever team owning the prompt, the feature team owning the corpus, undeclared surface differences, and reranker-as-surface-feature. Each is a missing boundary artefact.

Chapter 02 turns the informal "SLO" mentioned here into the concrete freshness / coverage / permission contract that governs how the surface trusts what the retriever returns.
