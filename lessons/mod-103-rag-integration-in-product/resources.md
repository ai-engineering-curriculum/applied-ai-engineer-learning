# Resources for mod-103 — RAG Integration at the Product Surface

Curated primary sources for the topics covered in the chapters. Standards, provider documentation, and canonical open-source projects first; secondary references are named as such. Retrieval APIs, ranking benchmarks, and evaluation tools move on a monthly cadence — always consult the current live docs before pinning behaviour.

## Vector stores and hybrid retrieval (context for [chapter 01](01-consuming-a-retriever-you-do-not-own.md))

You do not implement these on the applied side, but you need enough vocabulary to talk to the retriever team. Start with the primary docs, not the marketing pages.

- **pgvector** — Postgres extension for vector similarity search. `https://github.com/pgvector/pgvector`.
- **Qdrant docs** — open-source vector database with filtering, hybrid search, and RBAC-style access control. `https://qdrant.tech/documentation/`.
- **Weaviate docs** — vector database with hybrid search (BM25 + dense) and modular ranking. `https://weaviate.io/developers/weaviate`.
- **Elasticsearch — Vector search / kNN** — dense retrieval in Elasticsearch with hybrid and filtering. `https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html`.
- **OpenSearch — k-NN plugin and neural search** — analogous surface on OpenSearch. `https://opensearch.org/docs/latest/vector-search/`.
- **Vespa docs** — search + ranking + vector platform used at scale. `https://docs.vespa.ai/`.
- **Pinecone docs** — hosted vector store; useful for tenant/namespace patterns. `https://docs.pinecone.io/`.
- **Milvus docs** — open-source vector database. `https://milvus.io/docs`.
- **LanceDB docs** — embedded vector database. `https://lancedb.github.io/lancedb/`.
- **Google Vertex AI Vector Search** — `https://cloud.google.com/vertex-ai/docs/vector-search/overview`.
- **AWS Bedrock Knowledge Bases** — hosted RAG surface on Bedrock. `https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html`. <!-- needs-research: confirm the canonical URL — Bedrock renames sections regularly. -->
- **Azure AI Search** — hybrid retrieval (BM25 + vector) on Azure. `https://learn.microsoft.com/azure/search/search-what-is-azure-search`.

## ANN algorithms and the retrieval-side references

- **HNSW — the original paper** — Yu. A. Malkov and D. A. Yashunin, *"Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs."* `https://arxiv.org/abs/1603.09320`.
- **DiskANN / FreshDiskANN** — Microsoft Research work on billion-scale ANN with SSD-backed indices. `https://www.microsoft.com/en-us/research/publication/diskann-fast-accurate-billion-point-nearest-neighbor-search-on-a-single-node/`.
- **FAISS** — Facebook AI Similarity Search, the reference ANN library. `https://github.com/facebookresearch/faiss`.
- **ANN Benchmarks** — comparative benchmarks across ANN libraries. `https://ann-benchmarks.com/`.

## Retrievers, rerankers, and reciprocal rank fusion ([chapter 05](05-feature-side-levers.md))

- **BM25 — Robertson & Zaragoza, *The Probabilistic Relevance Framework: BM25 and Beyond*.** `https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf`.
- **Reciprocal Rank Fusion — Cormack, Clarke, Buettcher, *"Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods."*** `https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf`.
- **BGE / FlagEmbedding** — open-weight embedding + reranker models from BAAI. `https://github.com/FlagOpen/FlagEmbedding`.
- **Cohere Rerank** — hosted cross-encoder reranker. `https://docs.cohere.com/docs/rerank`.
- **Voyage Rerank** — hosted reranker family. `https://docs.voyageai.com/docs/reranker`.
- **Jina Reranker** — hosted reranker. `https://jina.ai/reranker/`.
- **Sentence-Transformers docs** — cross-encoder training reference. `https://www.sbert.net/`.

## Embedding models cited in code / examples

- **OpenAI Embeddings** — `text-embedding-3-*` family. `https://platform.openai.com/docs/guides/embeddings`.
- **Cohere Embed v3** — multilingual embedding family. `https://docs.cohere.com/docs/cohere-embed`.
- **Voyage Embeddings** — `voyage-*` family. `https://docs.voyageai.com/docs/embeddings`.
- **Google Vertex AI text embeddings** — `text-embedding-004` and later. `https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-text-embeddings`.
- **MTEB — Massive Text Embedding Benchmark** — the standard embedder leaderboard; useful for the retriever team, not a substitute for a product-altitude gold set. `https://huggingface.co/spaces/mteb/leaderboard`.

## Query-side levers ([chapter 05](05-feature-side-levers.md))

- **HyDE — Gao et al., *"Precise Zero-Shot Dense Retrieval without Relevance Labels."*** `https://arxiv.org/abs/2212.10496`.
- **Step-Back Prompting — Zheng et al., *"Take a Step Back: Evoking Reasoning via Abstraction in Large Language Models."*** `https://arxiv.org/abs/2310.06117`.
- **RAG-Fusion — Raudaschl, *"Forget RAG, the future is RAG-Fusion."*** Practitioner writeup that introduced the multi-query + RRF pattern by that name. <!-- needs-research: pin a stable URL for the RAG-Fusion writeup — the original post has moved. -->
- **Multi-Query Retriever — LangChain docs** — one implementation of the multi-query pattern. `https://python.langchain.com/docs/how_to/MultiQueryRetriever/`.
- **Self-Query Retriever — LangChain docs** — structured-filter extraction from natural-language queries. `https://python.langchain.com/docs/how_to/self_query/`.
- **Contextual Compression — LangChain docs** — post-retrieval compression pattern; useful as a mental model. `https://python.langchain.com/docs/how_to/contextual_compression/`.
- **LlamaIndex — Query Transformations** — `https://docs.llamaindex.ai/en/stable/optimizing/advanced_retrieval/query_transformations/`.

## Evaluation frameworks — the RAG triad and beyond ([chapter 04](04-retrieval-quality-instrumentation.md))

- **RAGAS docs** — Faithfulness, Answer-Relevancy, Context-Relevancy, Context-Recall metric implementations with configurable judge models. `https://docs.ragas.io/`.
- **RAGAS — original paper**, Es et al., *"Ragas: Automated Evaluation of Retrieval Augmented Generation."* `https://arxiv.org/abs/2309.15217`.
- **TruLens docs** — feedback functions for the RAG triad + per-call trace. `https://www.trulens.org/`.
- **TruLens — RAG triad feedback function reference** — `https://www.trulens.org/getting_started/core_concepts/rag_triad/`.
- **DeepEval docs** — LLM-eval test framework composable with pytest. `https://docs.confident-ai.com/`.
- **promptfoo docs** — command-line eval harness for LLM apps. `https://www.promptfoo.dev/docs/intro/`.
- **Arize Phoenix docs** — open-source LLM observability + evaluation. `https://docs.arize.com/phoenix`.
- **LangSmith — Evaluations** — hosted eval + trace store. `https://docs.smith.langchain.com/evaluation`.
- **Langfuse — Evaluations** — open-source observability + eval. `https://langfuse.com/docs/scores/overview`.
- **Weights & Biases Weave** — trace + eval surface. `https://weave-docs.wandb.ai/`.

## Retrieval benchmarks — the retriever-side numbers you consume

- **BEIR benchmark** — Thakur et al., *"BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models."* `https://arxiv.org/abs/2104.08663` and `https://github.com/beir-cellar/beir`.
- **MS MARCO** — passage ranking benchmark. `https://microsoft.github.io/msmarco/`.
- **Natural Questions** — Google's open-domain QA benchmark. `https://ai.google.com/research/NaturalQuestions/`.
- **HotpotQA** — multi-hop QA benchmark. `https://hotpotqa.github.io/`.

These matter for the retriever team's technical choices. They are **not** a substitute for a product-altitude gold set at the applied-AI altitude — see [chapter 04](04-retrieval-quality-instrumentation.md).

## Attribution and citation research ([chapter 03](03-citations-and-evidence-ux.md))

- **Attributed QA — Bohnet et al., *"Attributed Question Answering: Evaluation and Modeling for Attributed Large Language Models."*** `https://arxiv.org/abs/2212.08037`.
- **Perplexity's approach to citations (product writeup)** — practitioner reference for citation UX. <!-- needs-research: find a canonical stable link — Perplexity's citation-mechanics posts have moved between the blog and documentation. -->
- **Google — *AI-generated results and how they cite sources*.** <!-- needs-research: pin a stable canonical URL for the current AI Overviews / SGE citation-behaviour docs. -->

## ACL, permissions, and multi-tenant retrieval ([chapter 02](02-freshness-coverage-permission-contract.md))

- **OWASP Top 10 for Large Language Model Applications** — the `LLM06: Sensitive Information Disclosure` and `LLM02: Insecure Output Handling` entries frame the surface-side risks. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **NIST SP 800-207 — Zero Trust Architecture** — the framework that motivates "never grant access based on network position" for retrieval boundaries. `https://csrc.nist.gov/publications/detail/sp/800-207/final`.
- **NIST AI Risk Management Framework (AI RMF 1.0)** — Measure and Manage functions map to the eval + gate loop across chapters 02 and 04. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **Google Cloud — Row-level security patterns for retrieval** — reference implementation writeups on per-row / per-passage ACL enforcement. <!-- needs-research: pin an authoritative reference for row-level security applied to vector search on Vertex or BigQuery. -->
- **Postgres row-level security docs** — the primary reference for the retriever-enforced pattern when the underlying store is pgvector on Postgres. `https://www.postgresql.org/docs/current/ddl-rowsecurity.html`.

## Freshness pipelines (context for [chapter 02](02-freshness-coverage-permission-contract.md))

- **Debezium — Change Data Capture (CDC) reference** — the standard CDC surface. `https://debezium.io/documentation/`.
- **Kafka Connect — sink connectors** — plumbing for source-to-index pipelines. `https://kafka.apache.org/documentation/#connect`.
- **Apache Airflow — data pipelines primer** — batch refresh orchestration. `https://airflow.apache.org/docs/apache-airflow/stable/`.
- **AWS DMS / Kinesis Data Streams** — hosted CDC alternatives. `https://docs.aws.amazon.com/dms/` and `https://docs.aws.amazon.com/streams/`.

## RAG-application frameworks (used at the surface layer)

- **LangChain — Retrieval documentation** — retriever abstractions, vector-store integrations. `https://python.langchain.com/docs/how_to/#retrieval`.
- **LlamaIndex — RAG documentation** — indexes, query engines, evaluation. `https://docs.llamaindex.ai/en/stable/`.
- **Haystack docs** — retrieval + generation pipelines. `https://docs.haystack.deepset.ai/docs/`.
- **DSPy** — programmatic pipelines with automatic prompt / weight tuning. `https://dspy.ai/`.

## Secondary and background reading

- **Lewis et al., *"Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks."*** The paper that named RAG. `https://arxiv.org/abs/2005.11401`.
- **Simon Willison — RAG posts** — practitioner writeups on RAG failure modes and citation UX. `https://simonwillison.net/tags/rag/`.
- **Anthropic — *Contextual Retrieval* engineering blog** — a practitioner writeup on chunk-level context injection. `https://www.anthropic.com/news/contextual-retrieval`.
- **Google Research — *Retrieval-augmented generation for large language models: A survey*** — survey framing the field. <!-- needs-research: confirm canonical URL — multiple surveys share the name; pick the one most cited. -->
- **Chip Huyen — *Building LLM applications for production* / *Designing Machine Learning Systems*** — general engineering framing that applies to RAG production surfaces. `https://huyenchip.com/`.

## Standards previewed here, owned in other modules

- **NIST AI RMF 1.0** — the `Measure` and `Manage` functions map to the eval-and-rollback loop in chapters 04 and 05; owned in mod-106 and mod-110.
- **OWASP Top 10 for LLM Applications** — surface-side taxonomy for the ACL / injection surfaces in chapter 02 and the query-rewriter in chapter 05; owned in mod-107.
- **Prompt caching and structured outputs** — the mechanics behind the pack-shaped prompt and the `GroundedAnswer` schema in chapter 03 live in mod-102.
