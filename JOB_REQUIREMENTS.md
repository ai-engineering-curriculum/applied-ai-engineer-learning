# Job Requirements — Applied AI Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25)
**Track:** `applied-ai-engineer-learning`
**Research window:** 2026-05-05 → 2026-08-05 (last 90 days)
**Today:** 2026-08-05

This file documents the requirements catalog used to seed and now maintain the Applied AI Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed additions (empty) live in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — 2026-08 cycle: live-posting pass complete, no curriculum additions

This cycle completed the first live posting pass for the track. **30 in-window postings** were sampled across ATS aggregators (Greenhouse, Lever, Ashby, Workday, Avature) and direct careers pages, spanning frontier labs (OpenAI, Anthropic, Cohere, Google DeepMind / Google Cloud, xAI), applied-AI product companies (Notion, Ramp, Cognition, Sierra, Glean, Zapier, Cresta, Snorkel, Splitero, Gradial, Hasbro, Future, Hightouch, Appian, commercetools, Cargomatic, Human Agency, General Matter, Intuit), enterprise (Stripe, Oscar Health, GitLab), and hyperscaler applied-AI product teams (Google Cloud CES Applied AI, SpaceX Applied AI).

Titles accepted: `Applied AI Engineer`, `Applied LLM Engineer`, `Applied Generative AI Engineer`, `AI Engineer` (when the JD emphasised shipping LLM features in a product), `Applied AI Software Engineer`, and `Software Engineer, Agents / AI Agent` (when the JD emphasised product-embedded agents rather than agent-framework internals). Postings carrying `Senior / Staff / Principal / Lead / Manager / Director / Head` modifiers, generic ML / Data Science / Research Scientist titles, pretraining / foundation-research roles, pure MLOps / platform / infra roles, and pure peer-track shapes (LLM Application Developer at level 25, Agentic AI Engineer, AI Evaluation Engineer, RAG Engineer) were filtered out.

Every requirement in the catalog now carries at least one in-window posting citation in [`.aicg/job-requirements.json`](.aicg/job-requirements.json). **No net-new modules, exercises, or projects are proposed this cycle** — the continuity-bias gate (≥3 postings AND ≥30% frequency AND not incrementally coverable by an existing module) was cleared by zero candidate additions. The empty [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is the expected output when the market shape matches the existing curriculum, as it does this cycle.

## Authoritative references (grounding)

The requirements catalog is anchored to the following public references, which are what postings hire against and which the modules consume as their primary source material. Full list, IDs, and per-module use notes live in [`.aicg/job-requirements.json → authoritative_references`](.aicg/job-requirements.json).

- **Provider APIs & SDKs** — [OpenAI Platform](https://platform.openai.com/docs), [Anthropic API](https://docs.anthropic.com/), [Google Gemini API](https://ai.google.dev/gemini-api/docs), [Google Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs), [AWS Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/), [Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/).
- **Agent SDKs & orchestration** — [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), [Anthropic Claude Agent SDK](https://docs.anthropic.com/en/api/agent-sdk/overview), [LangChain](https://python.langchain.com/docs/introduction/) / [LangGraph](https://langchain-ai.github.io/langgraph/), [LlamaIndex](https://docs.llamaindex.ai/), [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction).
- **Structured-output libraries** — [Instructor](https://python.useinstructor.com/), [Outlines](https://dottxt-ai.github.io/outlines/), [Vercel AI SDK](https://ai-sdk.dev/docs/introduction).
- **LLM gateways / routing** — [LiteLLM](https://docs.litellm.ai/), [OpenRouter](https://openrouter.ai/docs/quickstart), [Portkey](https://portkey.ai/docs/introduction).
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

## Methodology

1. Sourced ~60 candidate postings via `WebSearch` across ATS aggregators (`boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `*.myworkdayjobs.com`) plus direct careers pages, mixing all six accepted title variants against three company segments (frontier labs + applied-AI product cos, enterprise + verticals, hyperscalers + startups).
2. `WebFetch` extracted the verbatim requirement bullets, preferred qualifications, salary range, and a representative one-sentence scope quote from each candidate. Postings whose title turned out to carry a senior modifier or belong to a peer track on inspection were dropped (notably one Bloomberg role tagged `Senior Software Engineer - Artificial Intelligence` was excluded from frequency counting even though the surrounding JD matched, per the filter rules).
3. Each accepted posting was tagged with the requirement keys it evidences, drawn from a fixed vocabulary aligned to the 12 planned modules. Emerging themes outside that vocabulary were tagged with a `candidate-` prefix and reviewed against the continuity-bias gate.
4. Mapped every non-prerequisite requirement to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that covers it.
5. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. See the sibling packets and `ownership_rule` block in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) for the full delegation matrix.
6. Ran the continuity-bias gate: no theme cleared ≥3 postings AND ≥30% frequency AND lacked incremental coverage in an existing module. [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is intentionally empty for this cycle.

## Requirement themes → curriculum ownership

The table lists each requirement theme, its posting frequency this cycle (`n/30`), its planned owner per the level hierarchy, and the curriculum coverage path.

| # | Theme | Freq (n/30) | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Applied AI foundations & feature design: spec → LLM-feature architecture, provider / tier / shape choice, SLO mapping | 29/30 (97%) | `applied-ai-engineer` (this) | [`mod-101-applied-ai-foundations-and-feature-design`](lessons/mod-101-applied-ai-foundations-and-feature-design) |
| 2 | Production prompting & structured outputs: prompt caching, JSON mode / function calling, constrained decoding, streaming, prompt versioning | 21/30 (70%) | `applied-ai-engineer` | [`mod-102-production-prompting-and-structured-outputs`](lessons/mod-102-production-prompting-and-structured-outputs) |
| 3 | RAG integration in product: consuming a retriever design, citations / freshness / evidence UX, retrieval quality instrumentation | 12/30 (40%) | `applied-ai-engineer` | [`mod-103-rag-integration-in-product`](lessons/mod-103-rag-integration-in-product) |
| 4 | Tool calling & orchestration at product altitude: JSON schemas, MCP servers (host + custom-server), LangGraph / Agents SDK graphs, token / step / cost budgets | 23/30 (77%) | `applied-ai-engineer` | [`mod-104-tool-calling-and-orchestration`](lessons/mod-104-tool-calling-and-orchestration) |
| 5 | Shipping product agents: coding co-pilot / customer-experience / retrieval-with-actions / browsing / support shapes, memory, hand-offs, kill-switches | 26/30 (87%) | `applied-ai-engineer` | [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 6 | Feature-side evaluation: offline suites, judge with bias controls, snapshot-replay, eval-gated CI, sampled online eval, per-feature dashboards | 18/30 (60%) | `applied-ai-engineer` | [`mod-106-feature-side-evaluation`](lessons/mod-106-feature-side-evaluation) |
| 7 | Guardrails & safety on the surface: content moderation, prompt-injection defence, PII scrubbing, tool-abuse, OWASP LLM & Agentic Top 10 mapping | 4/30 (13%) | `applied-ai-engineer` | [`mod-107-guardrails-and-safety-on-the-surface`](lessons/mod-107-guardrails-and-safety-on-the-surface) |
| 8 | Model routing & provider selection: gateway (LiteLLM / OpenRouter / Portkey), classifier-gated routing, fallbacks, self-host vs managed, multi-modal | 7/30 (23%) | `applied-ai-engineer` | [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) |
| 9 | Cost & latency SLOs for LLM features: TTFT / TPOT / p50 / p95, prompt caching, batching, streaming UX budget, per-tenant cost | 4/30 (13%) | `applied-ai-engineer` | [`mod-109-cost-latency-slos-for-llm-features`](lessons/mod-109-cost-latency-slos-for-llm-features) |
| 10 | Observability, feedback & incident response: OpenTelemetry GenAI / OpenInference, span-attribute schema, sampling, incident playbooks, user-feedback triage | 10/30 (33%) | `applied-ai-engineer` | [`mod-110-observability-feedback-and-incidents`](lessons/mod-110-observability-feedback-and-incidents) |
| 11 | Shipping, experimenting & rollbacks: feature flags (LaunchDarkly / OpenFeature), canary / shadow, A/B, hot-rollback of a bad prompt, applied fine-tuning as a lever | 8/30 (27%) | `applied-ai-engineer` | [`mod-111-shipping-experimenting-and-rollbacks`](lessons/mod-111-shipping-experimenting-and-rollbacks) |
| 12 | Multi-modal & voice features: vision, image / video, real-time voice, TTS / STT — as a product-integration pattern rolled into routing & agent shape | 3/30 (10%) | `applied-ai-engineer` | Merged into [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) and [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 13 | Owning an applied-AI feature end-to-end: build-vs-buy across gateway / eval / agent SDK, incident-to-fixture loop, feature-card slice for governance | 28/30 (93%) | `applied-ai-engineer` | [`mod-112-owning-an-applied-ai-feature-end-to-end`](lessons/mod-112-owning-an-applied-ai-feature-end-to-end) |
| 14 | Engineering craft (Python, packaging, HTTP, testing, git, Docker) | n/a — prerequisite | `ai-infra-junior-engineer` (level 10) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 15 | Classical ML / PyTorch / sklearn / FastAPI packaging / MLflow | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 16 | First-time LLM app construction (basic prompting, first chain, first tool call) | n/a — prerequisite | `llm-application-developer` (level 25) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught — this curriculum picks up at production-integration altitude |
| 17 | RAG engineering depth (chunking, embeddings, vector DB internals, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-103 owns RAG *integration*; RAG engineering linked out |
| 18 | Agent-framework depth (LangGraph / CrewAI / AutoGen internals, multi-agent) | n/a — peer track | `agentic-ai-engineer` (level 30) | mod-104 & mod-105 own *product-integrated* agents; framework depth linked out |
| 19 | AI eval program methodology depth | n/a — peer specialist | `ai-eval-engineer` (level 30) | mod-106 wires feature-side evals; program shape linked out |
| 20 | Model-eval methodology depth: validity theory, benchmark construction, statistical estimation, MLPerf | n/a — peer specialist | `model-evaluation-engineer` (level 30) | Consumed in mod-106 and mod-109; depth linked out |
| 21 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer specialist | `fine-tuning-engineer` (level 30) | mod-111 uses hosted fine-tuning as an *applied* lever (see candidate frequency below); depth linked out |
| 22 | Alignment-risk methodology / harm modelling / red-team data generation | n/a — peer specialist | `ai-risk-engineer` (level 30) | mod-107 measures at the surface; harm-model design linked out |
| 23 | Release-assurance / regulator-facing shape | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | mod-112 documents the interface; assurance shape linked out |
| 24 | Governance / policy / model-card review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Surfaced in mod-112; policy depth linked out |
| 25 | MLOps / model registry / self-service ML platform | n/a — peer track | `ai-infra-mlops` / `ai-infra-ml-platform` | mod-111 documents the interface; platform depth linked out |
| 26 | Inference serving performance (KV cache, batching, PagedAttention, quantisation) | n/a — peer track | `ai-infra-performance` (level 30) | mod-108 & mod-109 consume the interface; serving-perf depth linked out |
| 27 | Deep ML/AI security (model extraction, eval-set exfiltration, prompt-injection at platform depth) | n/a — higher level | `ai-infra-security-learning` (level 35) | Awareness in mod-107; deep treatment owned upstream |

### Emerging candidate themes (below continuity-bias gate, monitoring)

Themes surfaced during this cycle's requirement tagging that do NOT meet the ≥30% frequency gate but are worth tracking next cycle. All are already covered incrementally in the existing curriculum — no new content is proposed.

| Candidate theme | Freq (n/30) | Where existing curriculum already covers it |
|---|---|---|
| Hosted fine-tuning as an applied lever (SFT for style / format / small distillation) | 6/30 (20%) | mod-111 already lists hosted fine-tuning as an applied lever; deeper SFT / PEFT / RLHF / DPO methodology deferred to `fine-tuning-engineer` |
| MCP server authoring (not just consuming vendor servers) | 3/30 (10%) | mod-104 chapter 03 covers hosts, servers, transports, and writing a custom host directly against the specification |
| Voice / conversational AI (TTS, STT, contact-center integration) | 3/30 (10%) | mod-108 covers voice as a routing decision; mod-105 covers voice agents as an agent shape |
| Full-stack shipping across frontend + backend + LLM layer | 6/30 (20%) | mod-112 owns feature-lifecycle ownership; the full-stack skill itself is upstream engineering craft (`ai-infra-junior-engineer`) |

If any of these clears the gate next cycle, the delta should either extend the referenced module with an exercise or split out a dedicated module — do NOT duplicate what already exists.

## Posting evidence

30 in-window postings sampled between 2026-05-05 and 2026-08-05. Full verbatim requirement bullets, preferred qualifications, salary ranges, and requirement-key tags are in [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json).

| # | Employer | Title | Location | Date observed | Post ID |
|---|---|---|---|---|---|
| 1 | OpenAI | Applied AI Engineer, Codex Core Agent | San Francisco, CA | 2026-08-05 | `post-slice-a-01` |
| 2 | OpenAI | Applied AI Engineer, GTM Growth Engineering | San Francisco, CA | 2026-08-05 | `post-slice-a-02` |
| 3 | Anthropic | Applied AI Engineer (Startups) | US (SF / NYC / Seattle) | 2026-08-05 | `post-slice-a-03` |
| 4 | Anthropic | Applied AI Engineer, Enterprise Tech | SF / NYC / Seattle | 2026-08-05 | `post-slice-a-04` |
| 5 | Anthropic | Forward Deployed Engineer, Applied AI | Syracuse, Italy | 2026-08-05 | `post-slice-a-05` |
| 6 | Ramp | Applied AI Engineer | NYC / SF | 2026-08-05 | `post-slice-a-06` |
| 7 | Cohere | Applied AI Engineer – Agentic Workflows | Seoul, KR (hybrid) | 2026-08-05 | `post-slice-a-07` |
| 8 | Cognition | Applied AI Engineer | NYC / SF (in-person) | 2026-08-05 | `post-slice-a-08` |
| 9 | Notion | Software Engineer, AI Platform | SF (Hybrid) / NYC | 2026-08-05 | `post-slice-a-09` |
| 10 | Zapier | Engineer, Applied AI | Remote (US / CA) | 2026-08-05 | `post-slice-a-10` |
| 11 | Sierra | Software Engineer, Agent | Toronto, ON (in-person) | 2026-08-05 | `post-slice-a-11` |
| 12 | Google DeepMind / Google Cloud | Forward Deployed Engineer II, Applied AI | US (high-travel) | 2026-08-05 | `post-slice-a-12` |
| 13 | Stripe | AI Engineer | Chicago | 2026-08-05 | `post-slice-b-01` |
| 14 | Oscar Health | Applied AI Engineer | New York, NY | 2026-08-05 | `post-slice-b-02` |
| 15 | Cresta | Solutions Engineer, AI Agent | Remote (US) | 2026-08-05 | `post-slice-b-04` |
| 16 | Snorkel AI | Applied AI Engineer – Enterprise Solutions | NYC / Redwood City / SF (Hybrid) / Remote | 2026-08-05 | `post-slice-b-05` |
| 17 | Hasbro | Applied AI Engineer | Boston, MA | 2026-08-05 | `post-slice-b-06` |
| 18 | Future | Applied AI Engineer | Remote | 2026-08-05 | `post-slice-b-07` |
| 19 | GitLab | AI Engineer | Remote, US | 2026-08-05 | `post-slice-b-08` |
| 20 | Glean | Software Engineer, Agents | Bangalore, IN | 2026-08-05 | `post-slice-b-09` |
| 21 | Intuit | Applied AI Engineer, Marketing Futures | San Diego, CA | 2026-08-05 | `post-slice-b-10` |
| 22 | Splitero | Applied AI Engineer | Remote | 2026-08-05 | `post-slice-c-02` |
| 23 | Gradial | Applied AI Engineer | Seattle, WA | 2026-08-05 | `post-slice-c-03` |
| 24 | General Matter | Applied AI Engineer | Los Angeles, CA | 2026-08-05 | `post-slice-c-04` |
| 25 | SpaceX | Application Software Engineer, Applied AI | Vandenberg, CA | 2026-08-05 | `post-slice-c-05` |
| 26 | Human Agency | Applied AI Engineer | Remote (US / CA) | 2026-08-05 | `post-slice-c-06` |
| 27 | Hightouch | Software Engineer, AI Agents | Remote (NA) | 2026-08-05 | `post-slice-c-07` |
| 28 | Appian | Applied AI Engineer | McLean, VA | 2026-08-05 | `post-slice-c-08` |
| 29 | commercetools | AI Engineer | Munich, DE (Hybrid) / Berlin / Boston / Durham / London / Valencia | 2026-08-05 | `post-slice-c-09` |
| 30 | Cargomatic | AI Engineer – Agentic Automation | Remote | 2026-08-05 | `post-slice-c-10` |

### Top skills by frequency (this cycle)

The top-10 skills across the 30 sampled postings, in descending order:

1. **Applied AI foundations & feature design** — 97%
2. **Owning an applied-AI feature end-to-end** — 93%
3. **Shipping product-embedded agents** — 87%
4. **Tool calling & orchestration (JSON schemas, MCP, LangGraph / Agents SDK)** — 77%
5. **Production prompting & structured outputs** — 70%
6. **Feature-side evaluation (offline / online, eval-gated CI, judge design)** — 60%
7. **RAG integration at the product surface** — 40%
8. **Observability, feedback & incident response** — 33%
9. **Shipping, experimenting & rollbacks (feature flags, canary, A/B, rollback)** — 27%
10. **Model routing & provider selection (gateway, classifier-routing, fallbacks)** — 23%

### Skills expected but under-represented this cycle

Some themes that the module plan invested in showed up less frequently than the module hours suggest. This is a monitoring signal, not a demotion signal — postings under-index on operational concerns (guardrails, cost / latency SLOs, multi-modal) because those requirements live in the SLO / on-call side of the role that JDs rarely enumerate line-by-line.

- **Guardrails & safety on the surface** — 13% (mod-107 is 14 hours). Posting-language proxies (`safety`, `guardrails`, `prompt injection`, `PII`) are present in postings like GitLab, SpaceX, Hasbro, and Cargomatic but rarely called out as a top-line required skill. Retain mod-107 at current shape; watch for the theme to move up as OWASP LLM Top 10 & Agentic Top 10 adoption spreads.
- **Cost & latency SLOs for LLM features** — 13% (mod-109 is 12 hours). Notion, Zapier, GitLab, and SpaceX name it explicitly; most postings assume it. Retain mod-109 at current shape.
- **Multi-modal & voice features** — 10% (merged into mod-108 + mod-105). Google Cloud FDE, Cresta, and Hasbro cited it; consistent with the current merged-coverage decision.
- **Applied fine-tuning as a lever** — 20% as a candidate theme (mod-111 covers it inline). Frequency is higher than expected but still below the gate; not worth splitting into a dedicated module this cycle.

## Ownership map — quick reference

When routing a posting to a curriculum, use this ownership decision to keep the curriculum from drifting into peer territory:

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
