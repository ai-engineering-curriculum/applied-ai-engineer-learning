# Job Requirements — Applied AI Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25)
**Track:** `applied-ai-engineer-learning`
**Research window:** 2026-08-06 → 2026-11-04 (last 90 days)
**Today:** 2026-11-04

This file documents the requirements catalog used to seed and now maintain the Applied AI Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed additions (empty) live in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — 2026-11 cycle: fourth live pass, no curriculum additions

The 2026-08 cycle's 30 postings (`post-slice-a-01` … `post-slice-c-10`, observed 2026-08-05) are now 91 days old on the 2026-11-04 cycle date and fall **outside the 90-day window**; per methodology they are retired from the in-window sample and preserved in the raw JSON as historical reference only. The 2026-09 cycle's 6 postings (`post-slice-d-01` … `post-slice-d-06`, observed 2026-09-05, 60 days old) and the 2026-10 cycle's 8 postings (`post-slice-e-01` … `post-slice-e-08`, observed 2026-10-05, 30 days old) are retained as the carry-forward base (14 postings). **15 additional postings were freshly observed this cycle**, bringing the total in-window sample to **29 postings** (clearing the ≥25-minimum):

| Post ID | Employer | Title | Location | Date observed |
|---|---|---|---|---|
| `post-slice-f-01` | Anthropic | Applied AI Engineer, Public Sector | Washington, DC | 2026-11-04 |
| `post-slice-f-02` | Anthropic | Applied AI Engineer, Startups (Physical AI) | San Francisco, CA | 2026-11-04 |
| `post-slice-f-03` | Anthropic | Applied AI Engineer, Startups (Life Sciences) | SF / NYC | 2026-11-04 |
| `post-slice-f-04` | Anthropic | Applied AI Engineer, Startups | London, UK | 2026-11-04 |
| `post-slice-f-05` | Anthropic | Applied AI Engineer, Beneficial Deployments (Lab Automation) | London, UK | 2026-11-04 |
| `post-slice-f-06` | Anthropic | Applied AI Engineer, Enterprise | SF / NYC | 2026-11-04 |
| `post-slice-f-07` | Fanatics Inc. | Applied AI Engineer | New York, NY | 2026-11-04 |
| `post-slice-f-08` | Rackner | AI/ML Engineer — Generative AI Mission Systems | Remote (US) | 2026-11-04 |
| `post-slice-f-09` | HeyGen | Applied AI Engineer, Video Agent | SF / Palo Alto / LA / Toronto | 2026-11-04 |
| `post-slice-f-10` | Addepar | Software Engineer - Customer Facing Applied AI | Remote, USA | 2026-11-04 |
| `post-slice-f-11` | Engelhart | Applied AI Engineer | Berlin, DE | 2026-11-04 |
| `post-slice-f-12` | Future | Applied AI Engineer | Remote (continental US) | 2026-11-04 |
| `post-slice-f-13` | Seso | Software Engineer, AI/Agents | Remote (US) | 2026-11-04 |
| `post-slice-f-14` | Pallet | Software Engineer, Agent Delivery | San Francisco, CA | 2026-11-04 |
| `post-slice-f-15` | Iterative Health | Applied AI Engineer | Cambridge, MA / NYC | 2026-11-04 |

Titles accepted this cycle (unchanged from prior cycles): `Applied AI Engineer`, `Applied LLM Engineer`, `Applied Generative AI Engineer`, `Product AI Engineer` / `AI Product Engineer`, `AI Software Engineer`, `AI Engineer` (when the JD emphasised shipping LLM features in a product), `Applied AI Software Engineer`, `Software Engineer, Agents / AI Agent`, and `Forward Deployed Engineer, Applied AI`. Postings carrying `Senior / Staff / Principal / Lead / Manager / Director / Head` modifiers were filtered out — this cycle that rule excluded Snorkel Staff Applied AI Engineer Pre-Sales, Omada Health Senior Applied AI Engineer, Komodo Health Senior Applied AI Engineer, Twin Health Senior AI Engineer, AssetWatch Sr. Applied AI Engineer, Appian Principal/Lead/Sr Applied AI Engineer variants, Bolt.new (Stackblitz) Staff Applied AI Engineer, Weedmaps Senior Software Engineer AI Platform, Honeycomb Senior Software Engineer II Agentic Intelligence, and Addepar Sr. Software Engineer Customer Facing Applied AI. Fetch attempts that 404'd or redirected to employer listing indexes rather than individual postings this cycle: Hightouch Software Engineer AI Agents 5542602004 (404 — retired between search surfacing and fetch), Anthropic 5116274008 and 5248983008 (both redirected to Anthropic jobs index — retired/removed between search surfacing and fetch), Snorkel 5503593004, Quillbot 8045409, Komodo 8512178002, CHAOS Industries 5213650007, Appian 7946195, Carrot 6011067004 (all redirected to employer listing indexes — employer-root fetch pattern known-unreliable on Greenhouse, next-cycle must deep-link job IDs). Ashby (Mistral, Snorkel landing pages), Lever (Egen), openai.com/careers, and tealhq.com continued to block the fetcher (403 / JS-only index) for the fourth consecutive cycle — mounting evidence that this is a tooling gap requiring resolution before next pass. **Anthropic concentration this cycle is elevated (12 of 29 postings = 41%, up from 10 of 44 = 23% last cycle) because Anthropic has the highest open Applied AI Engineer headcount of any single employer right now AND their Greenhouse board is the most-reliably fetchable**; next cycle should actively balance.

Every requirement in the catalog carries at least one in-window posting citation in [`.aicg/job-requirements.json`](.aicg/job-requirements.json), with per-requirement `evidence_post_ids` and `evidence_frequency` recomputed against the 29-posting sample. **No net-new modules, exercises, or projects are proposed this cycle** — the continuity-bias gate (≥3 postings AND ≥30% frequency AND not incrementally coverable by an existing module) was cleared by zero candidate additions for the fourth cycle in a row. The empty [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is the expected output when the market shape matches the existing curriculum, as it does this cycle.

Two forward-looking signals surfaced this cycle, both still **below** the gate but worth watching next cycle:
- `candidate-context-engineering` jumped from 11% → 24% (climbing fast — all three fresh Anthropic Startups variants repeat the fixed phrase 'prompting, context engineering, agent architectures, evaluation frameworks, and deployment at scale'). If it clears 30% next cycle, rename mod-102 chapter 01 from 'Production prompting' to 'Prompting and context engineering' — do NOT propose a new module.
- `candidate-life-sciences-domain` climbed from 5% → 17% (Anthropic Startups Life Sciences + Lab Automation + Iterative Health are fresh). If it holds or rises, add a short domain-specialisation on-ramp to PREREQUISITES.md pointing at vertical-expertise resources — NOT a new module.

Four requirements dropped sharply after their evidence aged out of the window: `req-model-routing-and-provider-selection` (16% → 0%), `req-cost-latency-slos-for-llm-features` (9% → 0%), `req-shipping-experimenting-and-rollbacks` (20% → 3%), and `req-observability-feedback-and-incidents` (32% → 21%). These are the classic 'rarely enumerated but table-stakes' operational themes that drop out when an aging batch of postings leaves the window. See the under-represented table below for retention rationale — **no demotions proposed**.

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

1. **2026-08 baseline** — Sourced ~60 candidate postings via `WebSearch` across ATS aggregators (`boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `*.myworkdayjobs.com`) plus direct careers pages, mixing all six accepted title variants against three company segments (frontier labs + applied-AI product cos, enterprise + verticals, hyperscalers + startups). 30 postings passed the filter — all now out of window as of 2026-11-04.
2. **2026-09 fresh pass** — Re-swept the same aggregator set. Sourced ~30 candidate URLs, hit ~15 with usable JD bodies (Ashby / OpenAI-careers / some Anthropic-syndicated URLs were blocked or JS-only for the fetcher), and accepted **6 fresh in-window postings** (`post-slice-d-01` … `post-slice-d-06`).
3. **2026-10 fresh pass** — Re-swept the same aggregator set; added `AI Product Engineer` and `Software Engineer, Applied AI` as explicit search variants. Sourced ~25 candidate URLs, hit 11 with usable JD bodies. Accepted **8 fresh in-window postings** (`post-slice-e-01` … `post-slice-e-08`).
4. **2026-11 fresh pass** — Re-swept the same aggregator set; also sourced the landedjobs/llm-engineer-jobs GitHub release tags (2026-10-26 / 2026-10-28 / 2026-10-30) as a secondary funnel. Sourced ~45 candidate URLs, hit 15 with usable JD bodies on individual-job Greenhouse URLs (employer-root Greenhouse URLs and Ashby/Lever/Workday/openai.com-careers continued to block the fetcher for the fourth consecutive cycle). Accepted **15 fresh in-window postings** (`post-slice-f-01` … `post-slice-f-15`). 2026-08 cycle postings aged out of the 90-day window; sample now 29 postings.
5. `WebFetch` extracted the verbatim requirement bullets, preferred qualifications, salary range, and a representative one-sentence scope quote from each accepted posting.
6. Each accepted posting was tagged with the requirement keys it evidences, drawn from a fixed vocabulary aligned to the 12 planned modules. Emerging themes outside that vocabulary were tagged with a `candidate-` prefix and reviewed against the continuity-bias gate. No new candidate themes were declared this cycle (existing ones continue to track); frequencies for all requirements and candidates were recomputed against the 29-posting sample.
7. Mapped every non-prerequisite requirement to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that covers it.
8. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. See the sibling packets and `ownership_rule` block in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) for the full delegation matrix.
9. Ran the continuity-bias gate: no theme cleared ≥3 postings AND ≥30% frequency AND lacked incremental coverage in an existing module. `candidate-customer-facing-advisory` cleared the frequency gate (52%, up from 34%) for the third cycle running but remains classified as role-shape / soft-skill (FDE variants), not a technical curriculum topic. `candidate-context-engineering` (24%) and `candidate-life-sciences-domain` (17%) are both climbing fast but remain below the technical-curriculum gate. [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is intentionally empty for the fourth consecutive pass.

## Requirement themes → curriculum ownership

The table lists each requirement theme, its posting frequency this cycle (`n/29`), its cycle-over-cycle trajectory (2026-08 → 2026-09 → 2026-10 → 2026-11), its planned owner per the level hierarchy, and the curriculum coverage path.

| # | Theme | Freq (n/29) | Trajectory | Owner role | Coverage |
|---|---|---|---|---|---|
| 1 | Applied AI foundations & feature design: spec → LLM-feature architecture, provider / tier / shape choice, SLO mapping | 29/29 (100%) | 97% → 97% → 98% → 100% | `applied-ai-engineer` (this) | [`mod-101-applied-ai-foundations-and-feature-design`](lessons/mod-101-applied-ai-foundations-and-feature-design) |
| 2 | Production prompting & structured outputs: prompt caching, JSON mode / function calling, constrained decoding, streaming, prompt versioning | 22/29 (76%) | 70% → 69% → 73% → 76% | `applied-ai-engineer` | [`mod-102-production-prompting-and-structured-outputs`](lessons/mod-102-production-prompting-and-structured-outputs) |
| 3 | RAG integration in product: consuming a retriever design, citations / freshness / evidence UX, retrieval quality instrumentation | 10/29 (34%) | 40% → 36% → 41% → 34% | `applied-ai-engineer` | [`mod-103-rag-integration-in-product`](lessons/mod-103-rag-integration-in-product) |
| 4 | Tool calling & orchestration at product altitude: JSON schemas, MCP servers (host + custom-server), LangGraph / Agents SDK graphs, token / step / cost budgets | 25/29 (86%) | 77% → 69% → 75% → 86% | `applied-ai-engineer` | [`mod-104-tool-calling-and-orchestration`](lessons/mod-104-tool-calling-and-orchestration) |
| 5 | Shipping product agents: coding co-pilot / customer-experience / retrieval-with-actions / browsing / support shapes, memory, hand-offs, kill-switches | 29/29 (100%) | 87% → 89% → 91% → 100% | `applied-ai-engineer` | [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 6 | Feature-side evaluation: offline suites, judge with bias controls, snapshot-replay, eval-gated CI, sampled online eval, per-feature dashboards | 20/29 (69%) | 60% → 56% → 59% → 69% | `applied-ai-engineer` | [`mod-106-feature-side-evaluation`](lessons/mod-106-feature-side-evaluation) |
| 7 | Guardrails & safety on the surface: content moderation, prompt-injection defence, PII scrubbing, tool-abuse, OWASP LLM & Agentic Top 10 mapping | 3/29 (10%) | 13% → 11% → 14% → 10% | `applied-ai-engineer` | [`mod-107-guardrails-and-safety-on-the-surface`](lessons/mod-107-guardrails-and-safety-on-the-surface) |
| 8 | Model routing & provider selection: gateway (LiteLLM / OpenRouter / Portkey), classifier-gated routing, fallbacks, self-host vs managed, multi-modal | 0/29 (0%) | 23% → 19% → 16% → 0% | `applied-ai-engineer` | [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) |
| 9 | Cost & latency SLOs for LLM features: TTFT / TPOT / p50 / p95, prompt caching, batching, streaming UX budget, per-tenant cost | 0/29 (0%) | 13% → 11% → 9% → 0% | `applied-ai-engineer` | [`mod-109-cost-latency-slos-for-llm-features`](lessons/mod-109-cost-latency-slos-for-llm-features) |
| 10 | Observability, feedback & incident response: OpenTelemetry GenAI / OpenInference, span-attribute schema, sampling, incident playbooks, user-feedback triage | 6/29 (21%) | 33% → 31% → 32% → 21% | `applied-ai-engineer` | [`mod-110-observability-feedback-and-incidents`](lessons/mod-110-observability-feedback-and-incidents) |
| 11 | Shipping, experimenting & rollbacks: feature flags (LaunchDarkly / OpenFeature), canary / shadow, A/B, hot-rollback of a bad prompt, applied fine-tuning as a lever | 1/29 (3%) | 27% → 22% → 20% → 3% | `applied-ai-engineer` | [`mod-111-shipping-experimenting-and-rollbacks`](lessons/mod-111-shipping-experimenting-and-rollbacks) |
| 12 | Multi-modal & voice features: vision, image / video, real-time voice, TTS / STT — as a product-integration pattern rolled into routing & agent shape | 2/29 (7%) | 10% → 8% → 9% → 7% | `applied-ai-engineer` | Merged into [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) and [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 13 | Owning an applied-AI feature end-to-end: build-vs-buy across gateway / eval / agent SDK, incident-to-fixture loop, feature-card slice for governance | 29/29 (100%) | 93% → 94% → 95% → 100% | `applied-ai-engineer` | [`mod-112-owning-an-applied-ai-feature-end-to-end`](lessons/mod-112-owning-an-applied-ai-feature-end-to-end) |
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

Themes surfaced during requirement tagging that do NOT meet the ≥30% frequency gate but are worth tracking next cycle. All are already covered incrementally in the existing curriculum — no new content is proposed.

| Candidate theme | Freq (n/29) | Trajectory | Where existing curriculum already covers it |
|---|---|---|---|
| Hosted fine-tuning as an applied lever (SFT for style / format / small distillation) | 1/29 (3%) | 20% → 17% → 14% → 3% | mod-111 already lists hosted fine-tuning as an applied lever; deeper SFT / PEFT / RLHF / DPO methodology deferred to `fine-tuning-engineer`. Four cycles below the gate and drifting lower. HeyGen Video Agent (f-09) is the only in-window citation |
| MCP server authoring (not just consuming vendor servers) | 1/29 (3%) | 10% → 11% → 9% → 3% | mod-104 chapter 03 covers hosts, servers, transports, and writing a custom host directly against the specification. Fresh citations continue to frame MCP in a consume-side rather than authoring posture |
| Voice / conversational AI (TTS, STT, contact-center integration) | 1/29 (3%) | 10% → 8% → 9% → 3% | mod-108 covers voice as a routing decision; mod-105 covers voice agents as an agent shape. Dscout (e-02) is the only in-window citation; HeyGen (f-09) is video-modality rather than voice |
| Full-stack shipping across frontend + backend + LLM layer | 7/29 (24%) | 20% → 22% → 25% → 24% | mod-112 owns feature-lifecycle ownership; the full-stack skill itself is upstream engineering craft (`ai-infra-junior-engineer`). HeyGen (TypeScript+React+Python) and Seso (TypeScript+Node.js/React) carry fresh evidence. Stable at ~25%, still below the 30% gate |
| Named `context engineering` as a distinct skill | 7/29 (24%) | 8% → 11% → 24% | mod-102 already covers context-window assembly, prompt caching, and per-request budgets. Three fresh Anthropic Startups variants (Physical AI, Life Sciences, London) repeat the fixed phrase 'prompting, context engineering, agent architectures, evaluation frameworks, and deployment at scale'. **Climbing fast — if next cycle clears 30%, rename mod-102 chapter 01 from 'Production prompting' to 'Prompting and context engineering' rather than add a new module** |
| Named agent observability / eval-in-production | 4/29 (14%) | 6% → 9% → 14% | mod-106 covers sampled online eval + judge-drift monitor; mod-110 covers span-attribute schema for tool calls and trajectories. Future (f-12) adds the first in-sample explicit stack naming — 'Langfuse, OpenTelemetry, Datadog'. Climbing but still below gate |
| Life-sciences / biomedical vertical (genomics, clinical data, drug discovery) | 5/29 (17%) | 3% → 5% → 17% | Not directly covered — vertical-domain expertise sits outside the technical curriculum. Anthropic Life Sciences (f-03), Lab Automation (f-05), and Iterative Health (f-15) all join Anthropic Beneficial Deployments (d-05) and Savant Bio (e-05). **Passes the threshold for a PREREQUISITES.md note next cycle — add a short domain-specialisation on-ramp pointing at vertical-expertise resources, NOT a new module** |
| Enterprise deployment patterns (IT integration, tenancy, professional-services delivery) | 7/29 (24%) | 8% → 11% → 24% | mod-112 owns build-vs-buy and interface-to-platform decisions; deep enterprise-integration architecture belongs to platform / architect tracks. Anthropic Public Sector (air-gapped/classified), Anthropic Enterprise SF/NYC, Rackner (classified/restricted), and Pallet (world's largest supply chain) add fresh evidence. Climbing fast — if next cycle clears 30%, propose an exercise in mod-112 |
| Comfort using AI coding tools as part of the engineer's own workflow | 2/29 (7%) | 2% → 7% | Not covered as a technical topic — this is an engineering-craft meta-skill (how the engineer uses AI tools in their own loop). Addepar (f-10) is the second posting to name AI-assisted dev tools (Claude Code, Codex, or similar) as a required qualification; Dscout (e-02) was the first. Monitor; if the theme hits 3+ citations, note in PREREQUISITES.md rather than adding curriculum |

Candidate `candidate-customer-facing-advisory` (FDE / Solutions-Engineer / workshop-facilitator shape) rose from 30% to 33% to 34% to **52%** across four cycles (15/29 postings now). It cleared the frequency gate for the third consecutive cycle. This cycle's jump is partly because the 2026-08 non-FDE postings (Ramp, OpenAI GTM, Cohere, SpaceX etc.) aged out and 9 of 15 fresh postings carry FDE/advisory framing (the six Anthropic variants are all customer-facing technical-advisor shapes, plus Addepar Customer Facing Applied AI, Seso's FDE preference, and Pallet Agent Delivery). Verdict remains **at-gate-but-out-of-scope-for-technical-curriculum** — this is a role-shape / soft-skill theme concentrated in FDE variants of the role, not a technical curriculum topic. If a future cycle wants to teach it, the right home is a role-shape addendum in `PREREQUISITES.md` pointing at the `forward-deployed-engineer-learning` level-35 peer track as a career on-ramp (NOT a new module in this track).

If any technical candidate clears the gate next cycle, the delta should either extend the referenced module with an exercise or split out a dedicated module — do NOT duplicate what already exists.

## Posting evidence

29 in-window postings — the 6 sampled 2026-09-05, the 8 sampled 2026-10-05, and 15 freshly sampled between 2026-10-06 and 2026-11-04 (all date-observed within the 90-day window ending 2026-11-04). The 2026-08-05 postings (30 originally sampled) have aged out of the 90-day window and are retained in [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json) as historical reference only — they are not counted in the in-window sample this cycle. Full verbatim requirement bullets, preferred qualifications, salary ranges, and requirement-key tags for every posting are in that JSON.

| # | Employer | Title | Location | Date observed | Post ID |
|---|---|---|---|---|---|
| 1 | Cognition | Product Engineer | San Francisco, CA / New York City, NY | 2026-09-05 | `post-slice-d-01` |
| 2 | Clera | Applied AI Engineer – AI & Automation | Berlin, DE (on-site) | 2026-09-05 | `post-slice-d-02` |
| 3 | Monte Carlo | Applied AI Engineer | Remote (Americas) | 2026-09-05 | `post-slice-d-03` |
| 4 | Anthropic | Applied AI Engineer | Tokyo, JP | 2026-09-05 | `post-slice-d-04` |
| 5 | Anthropic | Applied AI Engineer, Beneficial Deployments (Life Sciences) | SF / NYC | 2026-09-05 | `post-slice-d-05` |
| 6 | Anthropic | Applied AI Engineer, Enterprise | Munich, DE | 2026-09-05 | `post-slice-d-06` |
| 7 | Pagaya | Applied AI Engineer, Office of the CEO | New York, NY | 2026-10-05 | `post-slice-e-01` |
| 8 | Dscout | Applied AI Engineer – India | Remote – India | 2026-10-05 | `post-slice-e-02` |
| 9 | Anthropic | Applied AI Engineer, Enterprise | Paris, FR | 2026-10-05 | `post-slice-e-03` |
| 10 | Anthropic | Forward Deployed Engineer | Paris, FR | 2026-10-05 | `post-slice-e-04` |
| 11 | Savant Bio | AI Engineer — Applied AI | New York, NY | 2026-10-05 | `post-slice-e-05` |
| 12 | Anthropic | Applied AI Engineer | Sydney, AU | 2026-10-05 | `post-slice-e-06` |
| 13 | Sharebite | AI Engineer | Remote | 2026-10-05 | `post-slice-e-07` |
| 14 | Sony Interactive Entertainment | AI Software Engineer – D2C – SPOC | San Diego, CA | 2026-10-05 | `post-slice-e-08` |
| 15 | Anthropic | Applied AI Engineer, Public Sector | Washington, DC | 2026-11-04 | `post-slice-f-01` |
| 16 | Anthropic | Applied AI Engineer, Startups (Physical AI) | San Francisco, CA | 2026-11-04 | `post-slice-f-02` |
| 17 | Anthropic | Applied AI Engineer, Startups (Life Sciences) | SF / NYC | 2026-11-04 | `post-slice-f-03` |
| 18 | Anthropic | Applied AI Engineer, Startups | London, UK | 2026-11-04 | `post-slice-f-04` |
| 19 | Anthropic | Applied AI Engineer, Beneficial Deployments (Lab Automation) | London, UK | 2026-11-04 | `post-slice-f-05` |
| 20 | Anthropic | Applied AI Engineer, Enterprise | SF / NYC | 2026-11-04 | `post-slice-f-06` |
| 21 | Fanatics Inc. | Applied AI Engineer | New York, NY | 2026-11-04 | `post-slice-f-07` |
| 22 | Rackner | AI/ML Engineer — Generative AI Mission Systems | Remote (US) | 2026-11-04 | `post-slice-f-08` |
| 23 | HeyGen | Applied AI Engineer, Video Agent | SF / Palo Alto / LA / Toronto | 2026-11-04 | `post-slice-f-09` |
| 24 | Addepar | Software Engineer - Customer Facing Applied AI | Remote, USA | 2026-11-04 | `post-slice-f-10` |
| 25 | Engelhart | Applied AI Engineer | Berlin, DE | 2026-11-04 | `post-slice-f-11` |
| 26 | Future | Applied AI Engineer | Remote (continental US) | 2026-11-04 | `post-slice-f-12` |
| 27 | Seso | Software Engineer, AI/Agents | Remote (US) | 2026-11-04 | `post-slice-f-13` |
| 28 | Pallet | Software Engineer, Agent Delivery | San Francisco, CA (5d/wk in-office) | 2026-11-04 | `post-slice-f-14` |
| 29 | Iterative Health | Applied AI Engineer | Cambridge, MA / NYC | 2026-11-04 | `post-slice-f-15` |

### Top skills by frequency (this cycle)

The top-10 skills across the 29 in-window postings, in descending order (with cycle trajectory):

1. **Applied AI foundations & feature design** — 100% (97% → 97% → 98% → 100%)
2. **Shipping product-embedded agents** — 100% (87% → 89% → 91% → 100%)
3. **Owning an applied-AI feature end-to-end** — 100% (93% → 94% → 95% → 100%)
4. **Tool calling & orchestration (JSON schemas, MCP, LangGraph / Agents SDK)** — 86% (77% → 69% → 75% → 86%)
5. **Production prompting & structured outputs** — 76% (70% → 69% → 73% → 76%)
6. **Feature-side evaluation (offline / online, eval-gated CI, judge design)** — 69% (60% → 56% → 59% → 69%)
7. **Customer-facing advisory / FDE role shape** — 52% (30% → 33% → 34% → 52%) *(role-shape theme, not a technical topic — retained out of curriculum scope; see emerging candidates table)*
8. **RAG integration at the product surface** — 34% (40% → 36% → 41% → 34%)
9. **Full-stack shipping across frontend + backend + LLM layer** — 24% (20% → 22% → 25% → 24%) *(candidate theme, still below gate; see emerging candidates table)*
10. **Named `context engineering`** — 24% (new in-top-10 entry, up from 11% last cycle) *(candidate theme, climbing fast — watch next cycle)*

### Skills expected but under-represented this cycle

Some themes that the module plan invested in showed up less frequently than the module hours suggest. This is a monitoring signal, not a demotion signal — four requirements dropped sharply this cycle after their 2026-08 evidence aged out of the 90-day window.

- **Model routing & provider selection** — **0%**, down from 16% (mod-108 is 12 hours). Four cycles below the 30% gate; dropped to zero this cycle after all seven historically-tagged postings (Cohere, Notion, Zapier, Hasbro, GitLab, Appian, Cargomatic) aged out. None of the 15 fresh postings explicitly named a gateway (LiteLLM / OpenRouter / Portkey). Same 'operationally assumed, rarely enumerated' pattern as req-cost-latency. Retain at planned scope for now; **joint-signal demotion criterion**: if a fifth cycle also returns 0 AND a targeted named-gateway search returns 0, fold routing into mod-101 as a chapter rather than keeping a stand-alone module.
- **Cost & latency SLOs for LLM features** — **0%**, down from 9% (mod-109 is 12 hours). Four cycles below the gate; dropped to zero this cycle after Notion, Zapier, GitLab, and SpaceX aged out. No fresh posting names TTFT / TPOT / prompt caching / batch API / cost-per-outcome explicitly. **Joint demotion criterion with mod-108** — if both stay at 0 for five consecutive cycles, fold mod-108 and mod-109 into a single '~20h Operational SLOs & Gateway' module.
- **Guardrails & safety on the surface** — 10%, down from 14% (mod-107 is 14 hours). Four cycles below the gate. Rackner (f-08) adds fresh guardrail-adjacent evidence (secure SDLC, DevSecOps, evaluating hallucinations / grounding / reliability) alongside Anthropic Paris Enterprise (e-03, 'enterprise security, compliance, and architecture') and Sony SPOC (e-08, 'trust and safety'). Classic 'operationally table-stakes, rarely enumerated' pattern — the on-call / safety team owns the alerting channel for it. Retain mod-107 at current shape.
- **Observability, feedback & incident response** — 21%, down from 32% (mod-110 is 13 hours). Dropped below the 30% gate this cycle after 8 of 14 historically-tagged postings aged out. Future (f-12) adds the first in-sample explicit stack naming ('Langfuse, OpenTelemetry, Datadog' as preferred observability/tracing tools) — a strong hit on the OpenTelemetry Gen-AI stack mod-110 was designed around. Anthropic Enterprise SF/NYC (f-06) adds 'transcript analysis' as a required bullet. Retain at current scope.
- **Shipping, experimenting & rollbacks** — 3%, down from 20% (mod-111 is 13 hours). Four cycles below the 30% gate. Only Dscout (e-02, 'experiments (A/B, staged rollouts, production replay)') survives in-window. No fresh posting names feature flags / canary / hot-rollback explicitly. Same table-stakes pattern — retain for now; if a fifth cycle also stays at ≤5%, revisit whether to shrink mod-111 by folding hosted-fine-tuning chapters into mod-108/mod-112.
- **Multi-modal & voice features** — 7%, down from 9% (merged into mod-108 + mod-105). HeyGen Video Agent (f-09) adds a dedicated video-modality agent posting alongside Sony SPOC (e-08). No split proposed; if video-agent postings double next cycle, re-evaluate.
- **Applied fine-tuning as a lever** — 3% as a candidate theme, down from 14% (mod-111 covers it inline). Four cycles below the gate and drifting lower. Not worth splitting into a dedicated module.

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
