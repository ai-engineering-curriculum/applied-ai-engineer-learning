# Job Requirements — Applied AI Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25)
**Track:** `applied-ai-engineer-learning`
**Research window:** 2026-04-09 → 2026-07-09 (last 90 days)
**Today:** 2026-07-09

This file documents the requirements catalog used to seed the Applied AI Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json).

## Status — bootstrap session, postings deferred

<!-- needs-research: collect ≥25 distinct in-window postings titled "Applied AI Engineer" / "Applied AI Engineer, [product/team]" / "Applied AI Software Engineer" / "Applied AI / Applied ML Engineer" / "Applied LLM Engineer" / "Applied Generative AI Engineer" / "AI Engineer, Applied" / "Applied AI Research Engineer" (product-embedded only). Filter OUT "Senior / Staff / Principal / Lead" modifiers (those inherit this packet), generic "ML Engineer / Data Scientist / Research Scientist" titles (owned by ml-engineer / senior-ml-engineer peers), pure "LLM Application Developer" at level-25 build-altitude (owned by llm-application-developer), pure "RAG Engineer" (owned by rag-engineer), pure "AI Agent Engineer" without product-integration emphasis (owned by agentic-ai-engineer), pure "AI Evaluation Engineer" (owned by ai-eval-engineer), pretraining / foundation-research titles, and pure infra / platform / MLOps titles. Re-validate every requirement against the live evidence and demote any whose evidence stays empty. -->

This packet was authored in a bootstrap session **without an exercised WebSearch / WebFetch pass against live job boards** (both tools returned permission-not-granted errors, and the delegated research subagent hit the same block). Per the project rules (*"Do not invent facts, incidents, or salary figures. Cite sources."*), the `postings` array in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) is intentionally empty — the curriculum will not claim to have analysed 25 live postings when none were fetched. This mirrors the precedent set by the sibling `ai-eval-engineer-learning` packet (level 30, same family) which shipped with the same posture.

The autonomous research loop is expected to fill that gap on its next cycle. To keep the loop deterministic and to give the curriculum a defensible baseline in the meantime, this document grounds the requirements catalog in **authoritative public references** that publish what the role is hired against:

- **Provider APIs & SDKs** — [OpenAI Platform](https://platform.openai.com/docs), [Anthropic API](https://docs.anthropic.com/), [Google Gemini API](https://ai.google.dev/gemini-api/docs), [Google Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs), [AWS Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/), [Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/). Applied AI engineers wire these into products; postings hire against fluent use of these SDKs.
- **Agent SDKs & orchestration** — [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), [Anthropic Claude Agent SDK](https://docs.anthropic.com/en/api/agent-sdk/overview), [LangChain](https://python.langchain.com/docs/introduction/) / [LangGraph](https://langchain-ai.github.io/langgraph/), [LlamaIndex](https://docs.llamaindex.ai/), [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction).
- **Structured-output libraries** — [Instructor](https://python.useinstructor.com/), [Outlines](https://dottxt-ai.github.io/outlines/), [Vercel AI SDK](https://ai-sdk.dev/docs/introduction).
- **LLM gateways / routing** — [LiteLLM](https://docs.litellm.ai/), [OpenRouter](https://openrouter.ai/docs/quickstart), [Portkey](https://portkey.ai/docs/introduction). Multi-provider routing, retries, fallbacks, and cost tracking are near-universal in Applied AI postings.
- **Observability + eval** — [Arize Phoenix](https://docs.arize.com/phoenix), [Langfuse](https://langfuse.com/docs), [W&B Weave](https://weave-docs.wandb.ai/), [Braintrust](https://www.braintrust.dev/docs/start), [LangSmith](https://docs.smith.langchain.com/), [Promptfoo](https://www.promptfoo.dev/docs/intro/), [DeepEval](https://docs.confident-ai.com/), [RAGAS](https://docs.ragas.io/), [TruLens](https://www.trulens.org/). [OpenTelemetry Gen-AI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and [OpenInference](https://github.com/Arize-ai/openinference) provide the wire format.
- **Guardrails** — [NVIDIA NeMo Guardrails](https://docs.nvidia.com/nemo/guardrails/), [Guardrails AI](https://www.guardrailsai.com/docs), [Meta Llama Guard / Purple Llama](https://ai.meta.com/research/publications/llama-guard-llm-based-input-output-safeguard-for-human-ai-conversations/), [OpenAI Moderation](https://platform.openai.com/docs/guides/moderation).
- **Inference serving (consumed at gateway boundary)** — [vLLM](https://docs.vllm.ai/), [HF TGI](https://huggingface.co/docs/text-generation-inference/index), [SGLang](https://docs.sglang.ai/). Deep serving depth is owned by `ai-infra-performance`.
- **Applied AI product references (worked examples)** — [Cursor](https://docs.cursor.com/), [GitHub Copilot](https://docs.github.com/en/copilot), [Sierra research](https://sierra.ai/research), [Harvey research](https://www.harvey.ai/research), [Perplexity Sonar API](https://docs.perplexity.ai/), [OpenAI Realtime API](https://platform.openai.com/docs/guides/realtime), [Hume EVI](https://dev.hume.ai/docs/empathic-voice-interface-evi/overview), [ElevenLabs](https://elevenlabs.io/docs).
- **Cost / caching / batching** — [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching), [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching), [OpenAI Batch API](https://platform.openai.com/docs/guides/batch), [MLPerf Inference Benchmark](https://mlcommons.org/benchmarks/inference-datacenter/).
- **Feature-flag / rollout** — [LaunchDarkly](https://docs.launchdarkly.com/), [OpenFeature (CNCF)](https://openfeature.dev/docs/reference/intro).
- **App-side safety, prompt-injection, and agent risk** — [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [OWASP Agentic AI threats & mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/), [MITRE ATLAS](https://atlas.mitre.org/), [Google SAIF](https://safety.google/cybersecurity-advancements/saif/), [OpenAI Usage Policies](https://openai.com/policies/usage-policies/), [Anthropic Usage Policy](https://www.anthropic.com/legal/aup). Foundational papers: [AgentDojo](https://arxiv.org/abs/2406.13352), [InjecAgent](https://arxiv.org/abs/2403.02691).
- **Foundational eval / judge research (feature-side depth)** — [MT-Bench / Chatbot Arena](https://arxiv.org/abs/2306.05685), [G-Eval](https://arxiv.org/abs/2303.16634), [τ-bench](https://arxiv.org/abs/2406.12045), [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/), [RAGAS paper](https://docs.ragas.io/), [Model Cards](https://arxiv.org/abs/1810.03993).
- **Governance / standards (link-out interface only)** — [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework), [NIST GenAI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile), [EU AI Act (Regulation 2024/1689)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), [ISO/IEC 42001](https://www.iso.org/standard/81230.html), [ISO/IEC 25059](https://www.iso.org/standard/80655.html).
- **Recipe libraries** — [OpenAI Cookbook](https://cookbook.openai.com/), [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook).

Every requirement below cites at least one such reference, and every requirement is shaped so that posting-frequency evidence can be added underneath it without restructure.

## Methodology

1. Sourced the canonical task domains for a deep-specialist Applied AI Engineer from the public references catalogued in [`.aicg/job-requirements.json → authoritative_references`](.aicg/job-requirements.json).
2. Mapped each task domain to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that will cover it.
3. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. In particular, deferred:
   - *Down* to [`llm-application-developer`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) (level 25) for first-time LLM application construction, to [`rag-engineer`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (level 25) for RAG *engineering* depth, to [`ml-engineer`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20) for classical ML / PyTorch / sklearn / packaging fundamentals, and to [`ai-infra-junior-engineer`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) for engineering-craft prerequisites.
   - *Sideways* to [`agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (peer, level 30) for agent-framework internals and multi-agent depth; to [`ai-eval-engineer`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30) for AI-application evaluation program methodology; to [`model-evaluation-engineer`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family) for benchmark and statistical methodology depth; to [`fine-tuning-engineer`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, level 30) for post-training depth; to [`ai-risk-engineer`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30) for alignment-risk methodology; to [`ai-governance-analyst`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning) (peer, level 25, Governance family) for policy / compliance depth; to [`ai-infra-mlops`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) / [`ai-infra-ml-platform`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) / [`ai-infra-performance`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) for MLOps / platform / inference-serving depth.
   - *Up* to [`senior-agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/senior-agentic-ai-engineer-learning), [`staff-ml-engineer`](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning), [`principal-ml-engineer`](https://github.com/ml-engineering-curriculum/principal-ml-engineer-learning), [`agentic-systems-architect`](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning), [`head-of-ai-governance`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning), and [`ai-infra-security`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) for the leadership, architectural, and deep-security scopes that inherit this packet.
4. Flagged everything that has not yet been validated against in-window postings so the next cycle can demote any requirement whose evidence stays empty.

## Requirement themes → curriculum ownership

The table below lists each requirement theme, its planned owner per the level hierarchy, and the curriculum coverage path. **Freq** is intentionally blank for this cycle — it will be backfilled by the next research pass.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Applied AI foundations & feature design: spec → LLM-feature architecture, provider / tier / shape choice, SLO mapping | <!-- needs-research --> | `applied-ai-engineer` (this) | [`mod-101-applied-ai-foundations-and-feature-design`](lessons/mod-101-applied-ai-foundations-and-feature-design) |
| 2 | Production prompting & structured outputs: prompt caching, JSON mode / function calling, constrained decoding, streaming, prompt versioning | <!-- needs-research --> | `applied-ai-engineer` | [`mod-102-production-prompting-and-structured-outputs`](lessons/mod-102-production-prompting-and-structured-outputs) |
| 3 | RAG integration in product: consuming a retriever design, citations / freshness / evidence UX, retrieval quality instrumentation | <!-- needs-research --> | `applied-ai-engineer` | [`mod-103-rag-integration-in-product`](lessons/mod-103-rag-integration-in-product) |
| 4 | Tool calling & orchestration at product altitude: JSON schemas, MCP servers, LangGraph / Agents SDK graphs, token / step / cost budgets | <!-- needs-research --> | `applied-ai-engineer` | [`mod-104-tool-calling-and-orchestration`](lessons/mod-104-tool-calling-and-orchestration) |
| 5 | Shipping product agents: coding co-pilot / customer-experience / retrieval-with-actions / browsing / support shapes, memory, hand-offs, kill-switches | <!-- needs-research --> | `applied-ai-engineer` | [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 6 | Feature-side evaluation: offline suites, judge with bias controls, snapshot-replay, eval-gated CI, sampled online eval, per-feature dashboards | <!-- needs-research --> | `applied-ai-engineer` | [`mod-106-feature-side-evaluation`](lessons/mod-106-feature-side-evaluation) |
| 7 | Guardrails & safety on the surface: content moderation, prompt-injection defence, PII scrubbing, tool-abuse, OWASP LLM & Agentic Top 10 mapping | <!-- needs-research --> | `applied-ai-engineer` | [`mod-107-guardrails-and-safety-on-the-surface`](lessons/mod-107-guardrails-and-safety-on-the-surface) |
| 8 | Model routing & provider selection: gateway (LiteLLM / OpenRouter / Portkey), classifier-gated routing, fallbacks, self-host vs managed, multi-modal | <!-- needs-research --> | `applied-ai-engineer` | [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) |
| 9 | Cost & latency SLOs for LLM features: TTFT / TPOT / p50 / p95, prompt caching, batching, streaming UX budget, per-tenant cost | <!-- needs-research --> | `applied-ai-engineer` | [`mod-109-cost-latency-slos-for-llm-features`](lessons/mod-109-cost-latency-slos-for-llm-features) |
| 10 | Observability, feedback & incident response: OpenTelemetry GenAI / OpenInference, span-attribute schema, sampling, incident playbooks, user-feedback triage | <!-- needs-research --> | `applied-ai-engineer` | [`mod-110-observability-feedback-and-incidents`](lessons/mod-110-observability-feedback-and-incidents) |
| 11 | Shipping, experimenting & rollbacks: feature flags (LaunchDarkly / OpenFeature), canary / shadow, A/B, hot-rollback of a bad prompt, applied fine-tuning as a lever | <!-- needs-research --> | `applied-ai-engineer` | [`mod-111-shipping-experimenting-and-rollbacks`](lessons/mod-111-shipping-experimenting-and-rollbacks) |
| 12 | Owning an applied-AI feature end-to-end: build-vs-buy across gateway / eval / agent SDK, incident-to-fixture loop, feature-card slice for governance | <!-- needs-research --> | `applied-ai-engineer` | [`mod-112-owning-an-applied-ai-feature-end-to-end`](lessons/mod-112-owning-an-applied-ai-feature-end-to-end) |
| 13 | Engineering craft (Python, packaging, HTTP, testing, git, Docker) | n/a — prerequisite | `ai-infra-junior-engineer` (level 10) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 14 | Classical ML / PyTorch / sklearn / FastAPI packaging / MLflow | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 15 | First-time LLM app construction (basic prompting, first chain, first tool call) | n/a — prerequisite | `llm-application-developer` (level 25) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught — this curriculum picks up at production-integration altitude |
| 16 | RAG engineering depth (chunking, embeddings, vector DB internals, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-103 owns RAG *integration*; RAG engineering linked out |
| 17 | Agent-framework depth (LangGraph / CrewAI / AutoGen internals, multi-agent) | n/a — peer track | `agentic-ai-engineer` (level 30) | mod-104 & mod-105 own *product-integrated* agents; framework depth linked out |
| 18 | AI eval program methodology depth | n/a — peer specialist | `ai-eval-engineer` (level 30) | mod-106 wires feature-side evals; program shape linked out |
| 19 | Model-eval methodology depth: validity theory, benchmark construction, statistical estimation, MLPerf | n/a — peer specialist | `model-evaluation-engineer` (level 30) | Consumed in mod-106 and mod-109; depth linked out |
| 20 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer specialist | `fine-tuning-engineer` (level 30) | mod-111 uses hosted fine-tuning as an *applied* lever only |
| 21 | Alignment-risk methodology / harm modelling / red-team data generation | n/a — peer specialist | `ai-risk-engineer` (level 30) | mod-107 measures at the surface; harm-model design linked out |
| 22 | Release-assurance / regulator-facing shape | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | mod-112 documents the interface; assurance shape linked out |
| 23 | Governance / policy / model-card review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Surfaced in mod-112; policy depth linked out |
| 24 | MLOps / model registry / self-service ML platform | n/a — peer track | `ai-infra-mlops` / `ai-infra-ml-platform` | mod-111 documents the interface; platform depth linked out |
| 25 | Inference serving performance (KV cache, batching, PagedAttention, quantisation) | n/a — peer track | `ai-infra-performance` (level 30) | mod-108 & mod-109 consume the interface; serving-perf depth linked out |
| 26 | Deep ML/AI security (model extraction, eval-set exfiltration, prompt-injection at platform depth) | n/a — higher level | `ai-infra-security-learning` (level 35) | Awareness in mod-107; deep treatment owned upstream |

## Posting evidence

<!-- needs-research: populate with the ≥25 in-window postings sampled next cycle. Use the same table shape as `agentic-ai-engineer-learning/JOB_REQUIREMENTS.md`. -->

No postings were sampled this cycle. See the **Status** section above for the reason. The next autonomous research cycle should fan out across:

- **ATS aggregators** — `boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `app.workable.com`, `*.myworkdayjobs.com`, `smartrecruiters.com`.
- **Frontier-lab applied-AI teams** — OpenAI (Applied AI / Applied ML / product), Anthropic (Applied AI / Product Engineering / Agents), Google DeepMind (product / Gemini apps), Meta GenAI (Applied), Microsoft AI (Copilot / M365), Apple AIML, Amazon AGI, NVIDIA (applied), xAI, Cohere, Mistral, Reka. **Reject** pretraining / capability-research roles at these employers — those belong to `fine-tuning-engineer` / research peers.
- **Applied-AI product companies** — Notion AI, Duolingo, Klarna, Perplexity, Harvey, Sierra, Glean, Hebbia, Cursor, Codeium / Windsurf, Cognition, Character.AI, Writer, Runway, Suno, Replit, Adept, You.com, Decagon, Cresta, Rox, Ada, Jasper.
- **Vertical applied-AI teams** — Health (Included Health, Ambience, Abridge, Nuance / DAX, Hippocratic AI, K Health), Legal (Harvey, Robin AI, EvenUp, Casetext / Thomson Reuters), Finance (Hebbia, Rogo), Support (Sierra, Ada, Decagon, Cresta), Coding (Cursor, Cognition, Codeium, Sourcegraph Cody, GitHub Copilot).
- **Enterprise applied-AI teams** — JPMorgan AI, Bloomberg AI, Bank of America, ServiceNow AI, Adobe (Sensei / Firefly), Salesforce Einstein / Agentforce, IBM watsonx, Oracle AI, SAP AI, Databricks Applied AI, Snowflake Applied AI.
- **Hyperscaler applied-AI product teams** — AWS AI Applied Science / Bedrock apps, Azure AI Apps / Foundry, Vertex AI Applied.

For each posting, capture employer, exact title, URL, `date_observed`, `date_posted` (or `estimated:2026-MM`), location, 5–10 verbatim requirement bullets, 2–6 preferred-qualification bullets, salary range when published, and one short representative quote. Filter out:

- **Senior / Staff / Principal / Lead modifiers** — those inherit this packet upward to `senior-agentic-ai-engineer`, `staff-ml-engineer`, `principal-ml-engineer`, `agentic-systems-architect`.
- **First-time LLM app-development postings without production-integration depth** — those belong to `llm-application-developer` (level 25 peer).
- **Pure agent-framework internals postings** (LangGraph / CrewAI / AutoGen depth without product integration) — those belong to `agentic-ai-engineer` (level 30 peer).
- **AI evaluation methodology postings** — those belong to `ai-eval-engineer` (level 30 peer).
- **Generic ML Engineer / Data Scientist / Research Scientist titles** — those are owned by `ml-engineer`, `senior-ml-engineer`, and research-scientist peers.
- **Pre-training / foundation-research titles** — owned by `fine-tuning-engineer` and research peers.
- **Pure infra / platform / MLOps titles** — owned by `ai-infra-mlops` / `ai-infra-ml-platform` / `training-pipeline-engineer` / `ai-infra-performance`.
- **Pure governance / trust-safety titles** — owned by `ai-governance-analyst` / `ai-evaluation-engineer` / `ai-risk-engineer`.

## Ownership map — quick reference for next cycle

When backfilling postings, use this ownership decision to keep the curriculum from drifting into peer territory:

- **Applied AI Engineer** (this track, level 30, AI Engineering family) — owns *end-to-end production integration* of LLM/agent/RAG features into a product surface: spec → feature architecture → provider selection → prompting → structured outputs → RAG integration → tool calling → agent shape → guardrails → routing → cost / latency SLOs → observability → feature flags & rollouts → incident response → feature-card slice for governance. **Product-integration altitude is the differentiator against peers.** Applied AI features earn revenue.
- **LLM Application Developer** (peer, level 25, AI Engineering family) — owns *first-time* LLM app construction: what does a prompt do, what does a chain look like, how do I call a tool, how do I stand up a vector store. This curriculum picks up at level 30 and adds production-integration depth. Cross-training: point learners here first if they have never built an LLM app.
- **Agentic AI Engineer** (peer, level 30, AI Engineering family) — owns agent-framework methodology depth (LangGraph / CrewAI / AutoGen / OpenAI Agents SDK / Anthropic Agent SDK internals; multi-agent orchestration; memory-system engineering). Cross-training: point learners here for framework depth; this curriculum owns product-embedded agent features.
- **AI Evaluation Engineer** (peer, level 30, AI Engineering family) — owns AI-application evaluation program methodology depth (trace warehouse architecture, eval-data-platform slice, judge-vs-human calibration methodology at product scale, RAG-eval methodology, safety-eval methodology). Cross-training: point learners here for eval program depth; this curriculum wires evals into features and consumes the program.
- **Model Evaluation Engineer** (peer, level 30, ML Engineering family) — owns benchmark / statistical / cross-modality model-eval methodology and MLPerf serving benchmarks. Consumed in mod-106 and mod-109.
- **RAG Engineer** (peer, level 25, ML Engineering family) — owns RAG *engineering* depth (chunking, embeddings, vector DB internals, rerankers, hybrid search). This curriculum owns RAG *integration* at the product surface.
- **AI Risk Engineer** (peer, level 30) — owns alignment-risk methodology / harm modelling / red-team data generation. mod-107 measures at the surface; harm-model authoring is owned there.
- **Fine-tuning Engineer** (peer, level 30) — owns post-training depth (SFT / PEFT / RLHF / DPO). mod-111 uses hosted fine-tuning as an *applied* lever only.
- **Senior ML Engineer** (peer, level 30, ML Engineering family generalist) — consumes this curriculum's applied-AI integration playbook in cross-track release reviews.
- **AI Infra ML Platform / MLOps / Performance** (peer level-25 to 30) — hosts model routing, eval-as-a-service, inference serving; this curriculum consumes the interface.
- **Staff / Principal / Senior Agentic AI / Agentic Systems Architect** (level 40+) — inherit this packet for architectural and cross-team scope.
- **Head of AI Governance** (level 40, Governance family) — consumes applied-AI evidence in board-level reporting.
- **AI Infra Security** (level 35) — owns deep AI/ML security; surfaced as awareness in mod-107.

## Differentiation versus the closest AI Engineering peers

The AI Engineering family carries several tracks that share vocabulary. Posting research must disambiguate carefully:

| Track | Family | Level | Owns |
|---|---|---|---|
| `applied-ai-engineer` (this) | AI Engineering | 30 | Product-embedded LLM/agent/RAG feature ownership end-to-end — the engineer who ships and operates a revenue-earning AI feature |
| `llm-application-developer` | AI Engineering | 25 | First-time LLM app construction (prompts, chains, first tool, first RAG) — the on-ramp to this track |
| `agentic-ai-engineer` | AI Engineering | 30 | Agent-framework methodology depth (LangGraph / Agents SDK internals, multi-agent orchestration, memory system engineering) |
| `ai-eval-engineer` | AI Engineering | 30 | AI-application evaluation program methodology depth (trace warehouse, eval-data-platform, program ownership) |
| `rag-engineer` | ML Engineering | 25 | RAG *engineering* depth (chunking, embeddings, vector DB internals, rerankers) |

If a posting emphasises **shipping and operating an LLM feature in a real product** — routing / prompt caching / guardrails / feature-flag rollout / on-call for a live AI feature / cost per user — route it here.
If it emphasises **first-time LLM app building** without production integration depth, route it to `llm-application-developer` (level 25).
If it emphasises **agent framework internals or multi-agent research**, route it to `agentic-ai-engineer`.
If it emphasises **AI-application evaluation program ownership** across many features and teams, route it to `ai-eval-engineer`.
If it emphasises **retriever / embedding / vector DB internals**, route it to `rag-engineer`.

<!-- needs-research: after the next cycle populates the postings list, add a short summary of the top-10 skills by frequency, and note any skills that were expected but did not appear so the modules can be right-sized. -->
