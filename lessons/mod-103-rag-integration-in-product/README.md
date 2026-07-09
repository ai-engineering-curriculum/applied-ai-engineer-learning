# mod-103-rag-integration-in-product: RAG Integration at the Product Surface

**Estimated effort:** 12 hours

The applied-AI engineer does not own the retriever. The `rag-engineer` peer track does. This module is about consuming that retriever from a product surface — negotiating a contract, wiring citations and evidence UX, instrumenting retrieval quality at the product altitude, and picking the feature-side levers that actually earn their latency budget.

## Learning objectives

- Consume a retriever + vector store designed by the `rag-engineer` peer track and wire it into a product surface without owning the internals.
- Define the freshness / coverage / permission contract (per-tenant scoping, ACL enforcement, index recency SLOs) between the RAG layer and the feature.
- Design the citations, evidence panels, and "why this answer" UX that ships alongside the generated response — including how to handle "no good evidence" gracefully.
- Instrument retrieval quality at the product altitude: recall@k on a labelled gold set, hit rate on live traffic, faithfulness / answer-relevancy sampled online, RAG-triad breakdown per surface.
- Add applied levers that belong on the feature side, not the retriever side: query rewriting, query-classification-gated retrieval, HyDE or step-back only when the eval justifies them, reranker consumption from the peer track.

## Chapters

1. [Consuming a Retriever You Do Not Own](01-consuming-a-retriever-you-do-not-own.md) — the applied-AI / rag-engineer boundary, what crosses it, and the thin retriever client that lives on it.
2. [The Freshness / Coverage / Permission Contract](02-freshness-coverage-permission-contract.md) — per-source freshness SLOs, coverage manifests, and the three defensible ACL enforcement patterns.
3. [Citations and Evidence UX](03-citations-and-evidence-ux.md) — sentence-level grounding, evidence panels, "why this answer", and the "no good evidence" branch.
4. [Retrieval Quality Instrumentation](04-retrieval-quality-instrumentation.md) — the gold set, recall@k, live hit-rate flavours, and the RAG-triad readout per surface.
5. [Feature-Side Levers](05-feature-side-levers.md) — query rewriting, gated retrieval, HyDE, step-back, and reranker consumption, with the discipline to reject a lever that didn't earn its budget.

## Exercises

- [exercise-01 — retriever contract with the rag engineer](exercises/exercise-01-retriever-contract-with-rag-engineer.md).
- [exercise-02 — citations and evidence UX](exercises/exercise-02-citations-and-evidence-ux.md).
- [exercise-03 — retrieval quality instrumentation](exercises/exercise-03-retrieval-quality-instrumentation.md).
- [exercise-04 — query rewriting and classification-gated retrieval](exercises/exercise-04-query-rewriting-and-classification-gated-retrieval.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- Prompt caching and structured outputs used by the `GroundedAnswer` schema live in **mod-102**.
- Retriever-side implementation details (chunking, embeddings, ANN, rerankers) live in the `rag-engineer` peer track.
- Product-KPI evaluation that consumes this module's RAG triad lives in **mod-106**.
- Prompt-injection defence for query rewriters lives in **mod-107**.
- The daily readout in [chapter 04](04-retrieval-quality-instrumentation.md) is a feed into **mod-110**'s observability + incident story.
