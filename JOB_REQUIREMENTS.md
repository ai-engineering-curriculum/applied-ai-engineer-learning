# Job Requirements — Applied AI Engineer

**Role level:** 30 (deep specialist, AI Engineering family — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25)
**Track:** `applied-ai-engineer-learning`
**Research window:** 2026-06-07 → 2026-09-05 (last 90 days)
**Today:** 2026-09-05

This file documents the requirements catalog used to seed and now maintain the Applied AI Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed additions (empty) live in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Status — 2026-09 cycle: second live pass, no curriculum additions

The 2026-08 cycle's 30 postings (`post-slice-a-01` … `post-slice-c-10`, all observed 2026-08-05) remain within the 90-day window ending 2026-09-05 and are retained as the base sample. **6 additional postings were freshly observed this cycle**, bringing the total sample to **36 in-window postings**:

| Post ID | Employer | Title | Location | Date posted |
|---|---|---|---|---|
| `post-slice-d-01` | Cognition | Product Engineer | SF / NYC | 2026-08-06 |
| `post-slice-d-02` | Clera | Applied AI Engineer – AI & Automation | Berlin, DE | 2026-09-04 |
| `post-slice-d-03` | Monte Carlo | Applied AI Engineer | Remote (Americas) | 2026-08-27 |
| `post-slice-d-04` | Anthropic | Applied AI Engineer | Tokyo, JP | n/a (in-window observed) |
| `post-slice-d-05` | Anthropic | Applied AI Engineer, Beneficial Deployments (Life Sciences) | SF / NYC | n/a (in-window observed) |
| `post-slice-d-06` | Anthropic | Applied AI Engineer, Enterprise | Munich, DE | n/a (in-window observed) |

Titles accepted this cycle (unchanged from 2026-08): `Applied AI Engineer`, `Applied LLM Engineer`, `Applied Generative AI Engineer`, `Product AI Engineer` / `AI Product Engineer`, `AI Software Engineer`, `AI Engineer` (when the JD emphasised shipping LLM features in a product), `Applied AI Software Engineer`, `Software Engineer, Agents / AI Agent`, and `Forward Deployed Engineer, Applied AI`. Postings carrying `Senior / Staff / Principal / Lead / Manager / Director / Head` modifiers, generic ML / Data Science / Research Scientist titles, pretraining / foundation-research roles, pure MLOps / platform / infra roles, and pure peer-track shapes (LLM Application Developer at level 25, Agentic AI Engineer, AI Evaluation Engineer, RAG Engineer) were filtered out. Fetch attempts against ~15 additional candidate URLs (Paraform, Cohere Agents & Automations, Ramp Fullstack, xAI, Webflow, BLEN, Speak, AGI Inc, Brain Co, Raspberry AI, Camber, three OpenAI variant slugs) returned 404s, closed listings, or JS-rendered shells — those are excluded rather than force-fitted (see next-cycle note in `.aicg/job-requirements.json` for the browser-capable-fetcher follow-up).

Every requirement in the catalog carries at least one in-window posting citation in [`.aicg/job-requirements.json`](.aicg/job-requirements.json), with per-requirement `evidence_post_ids` and `evidence_frequency` recomputed against the 36-posting sample. **No net-new modules, exercises, or projects are proposed this cycle** — the continuity-bias gate (≥3 postings AND ≥30% frequency AND not incrementally coverable by an existing module) was cleared by zero candidate additions for the second cycle in a row. The empty [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is the expected output when the market shape matches the existing curriculum, as it does this cycle.

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

1. **2026-08 baseline** — Sourced ~60 candidate postings via `WebSearch` across ATS aggregators (`boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `*.myworkdayjobs.com`) plus direct careers pages, mixing all six accepted title variants against three company segments (frontier labs + applied-AI product cos, enterprise + verticals, hyperscalers + startups). 30 postings passed the filter (one Bloomberg role tagged `Senior Software Engineer - Artificial Intelligence` was excluded from frequency counting even though the surrounding JD matched, per the filter rules).
2. **2026-09 fresh pass** — Re-swept the same aggregator set (extended to `Product AI Engineer` / `AI Product Engineer` variants explicitly per the packet). Sourced ~30 candidate URLs, hit ~15 with usable JD bodies (Ashby / OpenAI-careers / some Anthropic-syndicated URLs were blocked or JS-only for the fetcher and are logged for a browser-capable re-fetch next cycle), and accepted **6 fresh in-window postings** (`post-slice-d-01` … `post-slice-d-06`). Combined with the 30 retained postings from the 2026-08 pass (all still date-observed within 90 days), the sample is now 36.
3. `WebFetch` extracted the verbatim requirement bullets, preferred qualifications, salary range, and a representative one-sentence scope quote from each accepted posting.
4. Each accepted posting was tagged with the requirement keys it evidences, drawn from a fixed vocabulary aligned to the 12 planned modules. Emerging themes outside that vocabulary were tagged with a `candidate-` prefix and reviewed against the continuity-bias gate. Two new candidate themes were declared this cycle (`candidate-context-engineering`, `candidate-life-sciences-domain`); one prior candidate (`candidate-agent-observability`) was formalised in the emerging_candidates block; frequencies for all existing candidates were recomputed against the 36-posting sample.
5. Mapped every non-prerequisite requirement to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that covers it.
6. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. See the sibling packets and `ownership_rule` block in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) for the full delegation matrix.
7. Ran the continuity-bias gate: no theme cleared ≥3 postings AND ≥30% frequency AND lacked incremental coverage in an existing module. `candidate-customer-facing-advisory` cleared the frequency gate (33%) but is classified as role-shape / soft-skill (FDE variants), not a technical curriculum topic. [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) is intentionally empty for this cycle for the second consecutive pass.

## Requirement themes → curriculum ownership

The table lists each requirement theme, its posting frequency this cycle (`n/36`), its cycle-over-cycle trajectory (2026-08 → 2026-09), its planned owner per the level hierarchy, and the curriculum coverage path.

| # | Theme | Freq (n/36) | Δ vs 2026-08 | Owner role | Coverage |
|---|---|---|---|---|---|
| 1 | Applied AI foundations & feature design: spec → LLM-feature architecture, provider / tier / shape choice, SLO mapping | 35/36 (97%) | 97% → 97% | `applied-ai-engineer` (this) | [`mod-101-applied-ai-foundations-and-feature-design`](lessons/mod-101-applied-ai-foundations-and-feature-design) |
| 2 | Production prompting & structured outputs: prompt caching, JSON mode / function calling, constrained decoding, streaming, prompt versioning | 25/36 (69%) | 70% → 69% | `applied-ai-engineer` | [`mod-102-production-prompting-and-structured-outputs`](lessons/mod-102-production-prompting-and-structured-outputs) |
| 3 | RAG integration in product: consuming a retriever design, citations / freshness / evidence UX, retrieval quality instrumentation | 13/36 (36%) | 40% → 36% | `applied-ai-engineer` | [`mod-103-rag-integration-in-product`](lessons/mod-103-rag-integration-in-product) |
| 4 | Tool calling & orchestration at product altitude: JSON schemas, MCP servers (host + custom-server), LangGraph / Agents SDK graphs, token / step / cost budgets | 25/36 (69%) | 77% → 69% | `applied-ai-engineer` | [`mod-104-tool-calling-and-orchestration`](lessons/mod-104-tool-calling-and-orchestration) |
| 5 | Shipping product agents: coding co-pilot / customer-experience / retrieval-with-actions / browsing / support shapes, memory, hand-offs, kill-switches | 32/36 (89%) | 87% → 89% | `applied-ai-engineer` | [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 6 | Feature-side evaluation: offline suites, judge with bias controls, snapshot-replay, eval-gated CI, sampled online eval, per-feature dashboards | 20/36 (56%) | 60% → 56% | `applied-ai-engineer` | [`mod-106-feature-side-evaluation`](lessons/mod-106-feature-side-evaluation) |
| 7 | Guardrails & safety on the surface: content moderation, prompt-injection defence, PII scrubbing, tool-abuse, OWASP LLM & Agentic Top 10 mapping | 4/36 (11%) | 13% → 11% | `applied-ai-engineer` | [`mod-107-guardrails-and-safety-on-the-surface`](lessons/mod-107-guardrails-and-safety-on-the-surface) |
| 8 | Model routing & provider selection: gateway (LiteLLM / OpenRouter / Portkey), classifier-gated routing, fallbacks, self-host vs managed, multi-modal | 7/36 (19%) | 23% → 19% | `applied-ai-engineer` | [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) |
| 9 | Cost & latency SLOs for LLM features: TTFT / TPOT / p50 / p95, prompt caching, batching, streaming UX budget, per-tenant cost | 4/36 (11%) | 13% → 11% | `applied-ai-engineer` | [`mod-109-cost-latency-slos-for-llm-features`](lessons/mod-109-cost-latency-slos-for-llm-features) |
| 10 | Observability, feedback & incident response: OpenTelemetry GenAI / OpenInference, span-attribute schema, sampling, incident playbooks, user-feedback triage | 11/36 (31%) | 33% → 31% | `applied-ai-engineer` | [`mod-110-observability-feedback-and-incidents`](lessons/mod-110-observability-feedback-and-incidents) |
| 11 | Shipping, experimenting & rollbacks: feature flags (LaunchDarkly / OpenFeature), canary / shadow, A/B, hot-rollback of a bad prompt, applied fine-tuning as a lever | 8/36 (22%) | 27% → 22% | `applied-ai-engineer` | [`mod-111-shipping-experimenting-and-rollbacks`](lessons/mod-111-shipping-experimenting-and-rollbacks) |
| 12 | Multi-modal & voice features: vision, image / video, real-time voice, TTS / STT — as a product-integration pattern rolled into routing & agent shape | 3/36 (8%) | 10% → 8% | `applied-ai-engineer` | Merged into [`mod-108-model-routing-and-provider-selection`](lessons/mod-108-model-routing-and-provider-selection) and [`mod-105-shipping-product-agents`](lessons/mod-105-shipping-product-agents) |
| 13 | Owning an applied-AI feature end-to-end: build-vs-buy across gateway / eval / agent SDK, incident-to-fixture loop, feature-card slice for governance | 34/36 (94%) | 93% → 94% | `applied-ai-engineer` | [`mod-112-owning-an-applied-ai-feature-end-to-end`](lessons/mod-112-owning-an-applied-ai-feature-end-to-end) |
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

| Candidate theme | Freq (n/36) | Δ vs 2026-08 | Where existing curriculum already covers it |
|---|---|---|---|
| Hosted fine-tuning as an applied lever (SFT for style / format / small distillation) | 6/36 (17%) | 20% → 17% | mod-111 already lists hosted fine-tuning as an applied lever; deeper SFT / PEFT / RLHF / DPO methodology deferred to `fine-tuning-engineer` |
| MCP server authoring (not just consuming vendor servers) | 4/36 (11%) | 10% → 11% | mod-104 chapter 03 covers hosts, servers, transports, and writing a custom host directly against the specification (Monte Carlo added `Building and maintaining MCP servers` this cycle) |
| Voice / conversational AI (TTS, STT, contact-center integration) | 3/36 (8%) | 10% → 8% | mod-108 covers voice as a routing decision; mod-105 covers voice agents as an agent shape |
| Full-stack shipping across frontend + backend + LLM layer | 8/36 (22%) | 20% → 22% | mod-112 owns feature-lifecycle ownership; the full-stack skill itself is upstream engineering craft (`ai-infra-junior-engineer`) — Cognition and Clera add fresh Python/TypeScript/React and NestJS+React evidence |
| Named `context engineering` as a distinct skill | 3/36 (8%) | new theme this cycle (Anthropic postings use `prompting, context engineering, and agent architectures` as a fixed phrase) | mod-102 already covers context-window assembly, prompt caching, and per-request budgets; monitor for a rename or a first-class chapter promotion if frequency rises further |
| Named agent observability / eval-in-production | 2/36 (6%) | new callout this cycle (Monte Carlo posting is a strong evidence point) | mod-106 covers sampled online eval + judge-drift monitor; mod-110 covers span-attribute schema for tool calls and trajectories |
| Life-sciences / biomedical vertical (genomics, drug discovery) | 1/36 (3%) | new theme this cycle (Anthropic Beneficial Deployments) | Not directly covered — vertical-domain expertise sits outside the technical curriculum. Watch for 2+ additional postings next cycle before noting in PREREQUISITES.md as a domain-specialisation on-ramp |

Candidate `candidate-customer-facing-advisory` (FDE / Solutions-Engineer / workshop-facilitator shape) rose from 30% to **33%** across the two cycles (12/36 postings now, driven by three Anthropic FDE-flavoured postings this cycle in Tokyo, Life Sciences, and Munich). This is the first candidate to clear the ≥30% frequency gate. Verdict remains **at-gate-but-out-of-scope-for-technical-curriculum** — it is a role-shape / soft-skill theme concentrated in FDE variants of the role, not a technical curriculum topic. If a future cycle wants to teach it, the right home is a role-shape addendum in `PREREQUISITES.md` (career on-ramp), not a new module in this track.

If any technical candidate clears the gate next cycle, the delta should either extend the referenced module with an exercise or split out a dedicated module — do NOT duplicate what already exists.

## Posting evidence

36 in-window postings — the 30 sampled between 2026-05-05 and 2026-08-05 (all still date-observed within the 90-day window ending 2026-09-05) plus 6 freshly sampled between 2026-08-06 and 2026-09-05. Full verbatim requirement bullets, preferred qualifications, salary ranges, and requirement-key tags are in [`.aicg/job-requirements.json → postings`](.aicg/job-requirements.json).

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
| 31 | Cognition | Product Engineer | San Francisco, CA / New York City, NY | 2026-09-05 | `post-slice-d-01` |
| 32 | Clera | Applied AI Engineer – AI & Automation | Berlin, DE (on-site) | 2026-09-05 | `post-slice-d-02` |
| 33 | Monte Carlo | Applied AI Engineer | Remote (Americas) | 2026-09-05 | `post-slice-d-03` |
| 34 | Anthropic | Applied AI Engineer | Tokyo, JP | 2026-09-05 | `post-slice-d-04` |
| 35 | Anthropic | Applied AI Engineer, Beneficial Deployments (Life Sciences) | San Francisco, CA / New York City, NY | 2026-09-05 | `post-slice-d-05` |
| 36 | Anthropic | Applied AI Engineer, Enterprise | Munich, DE | 2026-09-05 | `post-slice-d-06` |

### Top skills by frequency (this cycle)

The top-10 skills across the 36 sampled postings, in descending order (with cycle trajectory):

1. **Applied AI foundations & feature design** — 97% (97% → 97%)
2. **Owning an applied-AI feature end-to-end** — 94% (93% → 94%)
3. **Shipping product-embedded agents** — 89% (87% → 89%)
4. **Tool calling & orchestration (JSON schemas, MCP, LangGraph / Agents SDK)** — 69% (77% → 69%)
5. **Production prompting & structured outputs** — 69% (70% → 69%)
6. **Feature-side evaluation (offline / online, eval-gated CI, judge design)** — 56% (60% → 56%)
7. **RAG integration at the product surface** — 36% (40% → 36%)
8. **Customer-facing advisory / FDE role shape** — 33% (30% → 33%) *(role-shape theme, not a technical topic — retained out of curriculum scope; see emerging candidates table)*
9. **Observability, feedback & incident response** — 31% (33% → 31%)
10. **Shipping, experimenting & rollbacks (feature flags, canary, A/B, rollback)** — 22% (27% → 22%)

### Skills expected but under-represented this cycle

Some themes that the module plan invested in showed up less frequently than the module hours suggest. This is a monitoring signal, not a demotion signal — postings under-index on operational concerns (guardrails, cost / latency SLOs, multi-modal) because those requirements live in the SLO / on-call side of the role that JDs rarely enumerate line-by-line.

- **Guardrails & safety on the surface** — 11%, down from 13% (mod-107 is 14 hours). Posting-language proxies (`safety`, `guardrails`, `prompt injection`, `PII`) are present in postings like GitLab, SpaceX, Hasbro, and Cargomatic but rarely called out as a top-line required skill. Retain mod-107 at current shape; watch for the theme to move up as OWASP LLM Top 10 & Agentic Top 10 adoption spreads.
- **Cost & latency SLOs for LLM features** — 11%, down from 13% (mod-109 is 12 hours). Notion, Zapier, GitLab, and SpaceX name it explicitly; most postings assume it. Retain mod-109 at current shape.
- **Multi-modal & voice features** — 8%, down from 10% (merged into mod-108 + mod-105). Google Cloud FDE, Cresta, and Hasbro cited it; consistent with the current merged-coverage decision.
- **Model routing & provider selection** — 19%, down from 23% (mod-108 is 12 hours). None of the 6 fresh postings named a specific gateway; a broader dip. Second cycle below the 30% gate — retain but watch; if the trajectory continues into a third cycle, consider tighter integration of routing into mod-101 rather than a stand-alone module.
- **Shipping, experimenting & rollbacks** — 22%, down from 27% (mod-111 is 13 hours). Second cycle below the 30% gate. Retain (feature-flag / rollout maturity is table-stakes and typically assumed rather than enumerated); if the trajectory continues into a third cycle, revisit module scope.
- **Applied fine-tuning as a lever** — 17% as a candidate theme, down from 20% (mod-111 covers it inline). Still below the gate; still not worth splitting into a dedicated module.

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
