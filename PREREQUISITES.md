# Prerequisites for the Applied AI Engineer track

This curriculum starts at **level 30 (deep specialist, AI Engineering family — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer`; peer to `senior-ml-engineer` at the same level on the ML Engineering ladder)**. It does not re-teach first-time LLM app construction, and it does not re-teach RAG *engineering* depth, agent-framework internals, model-eval methodology depth, post-training depth, MLOps depth, or inference-serving depth. Before working through it, you should already be comfortable with the items below.

- The [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) track (level 20) is the canonical place to acquire the ML foundation.
- [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) (level 25) is the canonical on-ramp — this curriculum starts where that one ends.
- [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (level 25) is the canonical place for RAG engineering; this curriculum consumes its output.
- [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) covers the engineering craft below that.

## Required

- **Intermediate Python** — functions, classes, modules, virtual environments, packaging (`uv` / `poetry`), structured logging, unit / integration testing, and the ability to read a moderately complex open-source library. You will read LangChain / LangGraph / OpenAI Agents SDK / Anthropic Claude Agent SDK / LiteLLM internals.
- **LLM application development fluency at build altitude** — you have built and shipped at least one small LLM application: prompt design, tool / function calling, streaming, error handling, and at least one production-shape framework (LangChain, LangGraph, LlamaIndex, OpenAI Agents SDK, Anthropic Claude Agent SDK, raw provider SDK, or equivalent). This curriculum picks up at the *production-integration altitude*; if you have never shipped an LLM feature at all, do [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) first.
- **At least two LLM APIs used in anger** — OpenAI + Anthropic + Google / Meta open-source LLM + Bedrock, including function-calling and structured outputs. You will be routing across these in mod-108 and mod-109.
- **RAG working familiarity** — you have wired an embedding model, a vector store, and a generation model into a working retrieval pipeline once. RAG *engineering* depth is not required — mod-103 owns RAG *integration*, [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) owns the engineering.
- **Agent working familiarity** — you have implemented at least one ReAct-shaped tool-calling agent end-to-end. Multi-agent orchestration is not required — mod-104 and mod-105 own product-embedded agents; agent-framework methodology depth is owned by the peer [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning).
- **Classical ML evaluation fluency** — confusion-matrix-derived metrics (precision / recall / F1, accuracy), ROC / PR, train / dev / test discipline, at the level of `ml-engineer-learning`. Deep model-eval methodology is owned by the peer [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning).
- **Basic statistics** — distributions, expectations, the *meaning* of a confidence interval and a p-value. mod-106 assumes this and links to `ai-eval-engineer` / `model-evaluation-engineer` for depth.
- **Observability basics** — you have used OpenTelemetry, Datadog, Prometheus, or an equivalent tracing stack once. mod-110 extends this to Gen-AI conventions.
- **Linux command line, Git, and Docker** — assumed at the level of [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning). Docker is used for local eval-runner and guardrail-sandbox labs.
- **SQL and a Pandas / Polars equivalent** — cost reports, token accounting, trace warehouses, and eval-result stores live here.
- **CI/CD working familiarity** — you have wired GitHub Actions (or GitLab CI / CircleCI / Buildkite) for at least one project. mod-106 wires evals into that.
- **Web fundamentals** — HTTP APIs, at least one modern web framework (FastAPI / Next.js / SvelteKit), server-sent events or WebSockets. mod-102 needs streaming and cancellation end-to-end.

## Recommended

- **One production LLM incident felt or read** — a bad prompt push, a provider outage, a judge-drift regression, a price hike, or a silent quality regression. mod-110 and mod-112 land more concretely for engineers who have felt an incident cycle.
- **One eval integration shipped at any depth** — added a Promptfoo YAML, wired a DeepEval pytest, ran a RAGAS notebook, or added a Weave / Langfuse / Braintrust trace hook. mod-106 lands faster if you have.
- **One trace debugged in a real trace viewer** — spent an hour in a Phoenix / Langfuse / Weave / LangSmith trace tree finding a bad tool call. mod-104 and mod-110 feel intuitive if you have.
- **Feature-flag exposure** — used LaunchDarkly, Statsig, Unleash, or OpenFeature at any depth. mod-111 assumes this pattern.
- **API and CLI credits** — a budget of $60-$120 in frontier-API credits across at least two providers for the labs across mod-102 through mod-111, plus $30-$60 in mid-tier / open-source model credits for router bake-offs and judge tiers.
- **A 16-GB or larger GPU (or a Colab / Hugging Face Inference Endpoint budget)** — for the mod-107 guardrail model (Llama Guard 8B) and the optional mod-108 vLLM baseline. If you are API-only, the router bake-off can be run entirely on managed providers with a wider credit budget.
- **Exposure to A/B testing** — a college course, a previous job, or the Kohavi-Tang-Xu chapters. mod-111 covers the *interface* to the A/B platform; deep methodology is owned by `senior-ml-engineer` / `model-evaluation-engineer`.

## Out of scope as prerequisites

You do **not** need to know any of the following before starting; they are either taught in this curriculum or owned by a peer/upstream track:

- Provider prompt-caching semantics on OpenAI / Anthropic / Bedrock — taught here.
- JSON mode / function calling / constrained decoding (Instructor / Outlines) at production altitude — taught here.
- LangGraph / OpenAI Agents SDK / Anthropic Claude Agent SDK / MCP at product-integration altitude — taught here. Deep agent-framework internals are owned by `agentic-ai-engineer`.
- Product-agent memory system design, kill-switch UX, human-in-the-loop hand-offs — taught here.
- Eval-gated CI/CD for prompts, chains, and agents at feature altitude — taught here. Deep eval program methodology is owned by `ai-eval-engineer`.
- Surface-level guardrails and OWASP LLM / Agentic Top 10 mapping — taught here. Harm modelling and red-team data generation are owned by `ai-risk-engineer`.
- LLM gateway routing (LiteLLM / OpenRouter / Portkey) and cross-provider contract testing — taught here.
- App-altitude TTFT / TPOT SLO design, provider prompt caching, batch APIs — taught here. MLPerf-style serving depth is owned by `ai-infra-performance` / `model-evaluation-engineer`.
- OpenTelemetry Gen-AI / OpenInference instrumentation of LLM features — taught here.
- Feature-flag driven prompt rollout, canary / shadow launches, hot-rollback of a bad prompt — taught here.
- Applied hosted fine-tuning as a lever — taught here. SFT / PEFT / RLHF / DPO methodology depth is owned by [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning).
- **RAG engineering depth** (chunking, embeddings, vector DB internals, rerankers, hybrid search) — owned by [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (peer, level 25).
- **Agent-framework internals** (LangGraph / CrewAI / AutoGen internals, multi-agent orchestration) — owned by [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (peer, level 30).
- **AI-application eval program methodology** — owned by [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30).
- **Model-eval methodology depth** (validity theory, benchmark construction, statistical estimation, cross-modality methodology, MLPerf serving benchmarks) — owned by [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30).
- **Post-training depth** (SFT / PEFT / RLHF / DPO) — owned by [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, level 30).
- **Alignment-risk methodology / harm modelling / red-team data generation** — owned by [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30).
- **Release-assurance / regulator-facing shape** (audit trails, regulator-facing docs, third-party evaluator interface) — owned by [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family).
- **Governance / policy / model-card review depth** — owned by [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning).
- **Distributed-training platform engineering** — owned by [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning).
- **MLOps / model registry / deploy-pipeline internals** — owned by [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, level 25).
- **Self-service ML/AI platform engineering** — owned by [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) (peer, level 30).
- **Inference-serving depth** (KV cache, continuous batching, PagedAttention, quantisation, speculative decoding) — owned by [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) (peer, level 30).
- **Deep ML/AI security** (model extraction, eval-set exfiltration, prompt-injection defence at platform depth) — owned by [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) (level 35).

## Hardware and credits

- Modules 101 and 112 are document-shaped and need no compute.
- Module 102 (production prompting + structured outputs) is API-bound. Plan $10-$20 in frontier-API credits across at least two providers.
- Module 103 (RAG integration) is API-bound plus a small local vector store (Postgres/pgvector or a managed serverless index). Plan $15-$25 in API credits.
- Module 104 (tool calling + orchestration) is API-bound. Plan $20-$40 in API credits and an MCP-capable environment.
- Module 105 (shipping product agents) is API- and container-bound (sandboxed agent replay for browsing / coding surfaces). Plan $30-$50 in API credits and 8-16 GB RAM.
- Module 106 (feature-side evaluation) is API-bound plus GitHub Actions minutes; free GH tier is enough for the labs. Plan $20-$40 in judge-tier API credits.
- Module 107 (guardrails & safety) is API-bound with a small local guardrail model (Llama Guard 8B on a 16-GB GPU or via Hugging Face Inference Endpoint). **Do not collect or distribute attack content** — use referenced public benchmark suites read-only and delete after use.
- Module 108 (routing & provider selection) needs a spread of tiers across at least three providers plus an optional self-hosted vLLM baseline on a 24-GB GPU. Plan $30-$60 across three model tiers.
- Module 109 (cost & latency) is API-bound; plan $20-$40 in API credits.
- Module 110 (observability, feedback, incidents) needs a small container running Phoenix or Langfuse plus $10-$20 of API credits and a small object-storage account for trace samples.
- Module 111 (shipping, experimenting, rollbacks) needs a LaunchDarkly free tier or a self-hosted OpenFeature provider plus $20-$40 in API credits for the canary / shadow labs.

<!-- needs-research: confirm the hardware-and-credits budget against the live posting evidence on the next autonomous research cycle; if postings consistently demand multi-GPU self-hosted inference or larger guardrail models, raise the floor on the recommended-hardware list. -->
