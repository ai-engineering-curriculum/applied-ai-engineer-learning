# Applied AI Engineer Curriculum

<!-- aicg:orientation:start -->
> **Start here:** [Module 00: Your Relationship to the Model](lessons/00-your-relationship-to-the-model/README.md). Where this role sits relative to the model, what it owns, and who it works with across the sibling curricula.
<!-- aicg:orientation:end -->

**Role level:** 30 (deep specialist — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25; peer to `senior-ml-engineer` at the same level on the ML Engineering ladder)
**Family:** AI Engineering
**Status:** planned — modules and projects below are the planned scope authored from [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Lessons and projects will be drafted by subsequent autonomous content cycles.

## Overview

This track teaches applied AI engineering end-to-end for a deep specialist who ships and operates LLM/agent/RAG features inside real products: reading a product spec into a shippable LLM-feature architecture → production prompting with caching, structured outputs, streaming, and versioning → RAG integration at the product surface with citations, freshness, and evidence UX → tool calling and orchestration at product altitude (LangGraph, OpenAI Agents SDK, Anthropic Claude Agent SDK, MCP) → shipping product-embedded agents (coding co-pilot, customer-experience, retrieval-with-actions, browsing, sales / support / ops assistants) → feature-side evaluation loops (offline suites, eval-gated CI, sampled online eval, per-feature dashboards) → guardrails and safety on the surface (OWASP LLM Top 10 & Agentic Top 10 mapping) → model routing and provider selection across gateway architectures → cost / latency SLOs at app altitude → observability, user feedback, and incident response → shipping / experimenting / rollbacks (feature flags, canary / shadow, hot-rollback of a bad prompt) → owning an applied-AI feature end-to-end across product, infra, safety, and governance.

It is a **specialist track at level 30**, positioned in the **AI Engineering family**. The differentiator against the peer `llm-application-developer-learning` (level 25) is **production-integration altitude**: this curriculum picks up where first-time LLM app construction ends and adds the depth a staff-track candidate would run — routing, evaluations, guardrails, cost / latency SLOs, incident response, and feature-lifecycle ownership. The differentiator against the peer `agentic-ai-engineer-learning` (level 30) is **product embedding**: agent-framework methodology depth is owned by that peer; this owns *product-embedded* agent features. The differentiator against the peer `ai-eval-engineer-learning` (level 30) is **feature altitude**: AI-evaluation program methodology depth is owned by that peer; this owns feature-side eval wiring and consumes the program.

Total planned commitment: **156 hours across 12 modules** + **140 hours across 3 projects** = **~296 hours**.

## Ownership rule

Following the project-wide ownership rule, this curriculum:

- **Owns** end-to-end production integration of LLM/agent/RAG features into a product surface — spec-to-architecture, production prompting, RAG integration, tool calling, product agents, feature-side eval, surface-level guardrails, model routing, cost / latency SLOs, observability, feature flags & rollouts, incident response, and feature-lifecycle ownership.
- **Defers down** to [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) (peer, level 25) for first-time LLM app construction; to [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (peer, level 25) for RAG *engineering* depth; to [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20) for classical ML / PyTorch / sklearn / packaging fundamentals; and to [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) for engineering-craft prerequisites.
- **Defers sideways** to [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (peer, level 30) for agent-framework internals and multi-agent depth; to [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30) for AI-application evaluation program methodology; to [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family) for benchmark and statistical methodology depth; to [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, level 30) for post-training depth; to [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30) for alignment-risk / harm-model / red-team-data depth; to [`nlp-engineer-learning`](https://github.com/ml-engineering-curriculum/nlp-engineer-learning) (peer, level 25) for classical NLP depth; to [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family) and [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning) (peer, level 25) for release-assurance / compliance / model-card review depth; to [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, level 25), [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) (peer, level 30), and [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) (peer, level 30) for platform, MLOps, and inference-serving depth.
- **Defers up** to [`senior-agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/senior-agentic-ai-engineer-learning), [`agentic-systems-architect-learning`](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning), [`staff-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning), [`principal-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/principal-ml-engineer-learning), [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning), and [`chief-ai-officer-learning`](https://github.com/ai-governance-curriculum/chief-ai-officer-learning) for architectural, leadership, and executive scope; to [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) for deep ML/AI security (model extraction, eval-set exfiltration, prompt-injection defence at platform depth).

See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map and the cited public references the catalog is grounded in.

## Module Plan

| Module | Title | Hours | Status |
|---|---|---|---|
| mod-101-applied-ai-foundations-and-feature-design | Applied AI Foundations: From Product Spec to LLM Feature Architecture | 10 | planned |
| mod-102-production-prompting-and-structured-outputs | Production Prompting and Structured Outputs | 14 | planned |
| mod-103-rag-integration-in-product | RAG Integration at the Product Surface | 12 | planned |
| mod-104-tool-calling-and-orchestration | Tool Calling and Orchestration at Product Altitude | 14 | planned |
| mod-105-shipping-product-agents | Shipping Product-Embedded Agents | 16 | planned |
| mod-106-feature-side-evaluation | Feature-Side Evaluation: Offline, Online, and In CI | 14 | planned |
| mod-107-guardrails-and-safety-on-the-surface | Guardrails and Safety on the Product Surface | 14 | planned |
| mod-108-model-routing-and-provider-selection | Model Routing and Provider Selection | 12 | planned |
| mod-109-cost-latency-slos-for-llm-features | Cost and Latency SLOs for LLM Features | 12 | planned |
| mod-110-observability-feedback-and-incidents | Observability, User Feedback, and Incident Response for LLM Features | 13 | planned |
| mod-111-shipping-experimenting-and-rollbacks | Shipping, Experimenting, and Rollbacks for LLM Features | 13 | planned |
| mod-112-owning-an-applied-ai-feature-end-to-end | Owning an Applied-AI Feature End-to-End | 12 | planned |

## Project Plan

| Project | Title | Hours | Status |
|---|---|---|---|
| project-101-vertical-product-agent-capstone | Vertical Product Agent Capstone | 40 | planned |
| project-102-eval-and-guardrail-hardened-release | Eval- and Guardrail-Hardened Release for an LLM Feature | 45 | planned |
| project-103-applied-ai-feature-platform-slice | Applied-AI Feature Platform Slice for a Product Family | 55 | planned |

## Module summaries

### mod-101 — Applied AI Foundations
Translate a product spec into a shippable LLM-feature architecture: choose the feature shape (chat / assistant / co-pilot / autonomous agent / one-shot generation / retrieval-with-citations / structured extraction), map user-visible SLOs (task success, groundedness, safety, cost per successful outcome, TTFT / TPOT / p95, freshness) to concrete metrics, pick provider and model tier for the surface with an explicit trade-off note, and use ISO/IEC 25059 quality dimensions as a checklist. Reject the four leaderboard-driven failure modes in review.

### mod-102 — Production Prompting and Structured Outputs
Prompt design that survives prompt-caching boundaries on OpenAI, Anthropic, Gemini, and Bedrock; JSON mode, function / tool calling, and constrained decoding (Instructor + Pydantic, Outlines) for schema-safe outputs; streaming and cancellation end-to-end with partial-output rendering; retry / repair / self-correction for schema failures with bounded budgets; prompts as first-class artifacts with dataset-pinned regression fixtures and rollback semantics.

### mod-103 — RAG Integration at the Product Surface
Consume a retriever + vector store from the `rag-engineer` peer without owning the internals; define the freshness / coverage / permission contract with the RAG layer; design citations, evidence panels, and 'why this answer' UX (including the 'no good evidence' case); instrument retrieval quality at the surface (recall@k on a gold set, live hit rate, RAG-triad); apply feature-side levers (query rewriting, classification-gated retrieval, reranker consumption) only when the eval justifies them.

### mod-104 — Tool Calling and Orchestration
Design tool interfaces (JSON schemas, argument validation, side-effect boundaries) that a model can call reliably in production; handle tool-call errors across model / backend / permanent-failure classes; wire MCP servers into a product agent via Claude Agent SDK, LangGraph MCP client, or custom host; orchestrate multi-step tool sequences with LangGraph and/or OpenAI Agents SDK / Anthropic Claude Agent SDK; enforce per-request budgets on tokens / steps / wall-clock / dollar cost with graceful degradation.

### mod-105 — Shipping Product-Embedded Agents
Pick the right agent shape (coding co-pilot / customer-experience / retrieval-with-actions / browsing / sales / support / ops) for a real surface; design the memory system a product agent actually needs — session, per-user, long-term — with retention and PII policy and a deletion / export contract; wire human-in-the-loop hand-offs, kill-switches, and progress / cancel / undo / resume UX; consume τ-bench, SWE-bench Verified, WebArena, and AgentDojo as *shape templates* for internal task suites; extract design choices from four public applied-agent patterns (Cursor, GitHub Copilot Workspace, Sierra, Harvey) separating what belongs here from what belongs to the `agentic-ai-engineer` peer.

### mod-106 — Feature-Side Evaluation
Offline eval suites (LLM-as-judge with position / length controls, rubric grading, code / task correctness, RAG-triad, safety spot checks) with pre-registered thresholds; production trace snapshots (PII scrubbed) as regression fixtures with model / prompt / retriever / decoding / seed lineage; eval-gated CI on Promptfoo, DeepEval, Braintrust / Weave / Langfuse / Phoenix / LangSmith runners; sampled online eval loop with judge-drift monitor; per-feature dashboards a PM, on-call engineer, and safety reviewer can each read — deferring statistical rigour to `ai-eval-engineer` / `model-evaluation-engineer`.

### mod-107 — Guardrails and Safety on the Surface
Layered input / output moderation (provider moderation + Llama Guard / classifier guard + programmable rails via NeMo Guardrails / Guardrails AI) with explicit FP / FN reports; prompt-injection defence grounded in AgentDojo / InjecAgent shape; PII / secret scrubbing across inputs, prompts, outputs, and traces; tool-abuse and unauthorised-tool-use guards; refusal / over-refusal calibration against a written safety policy; every finding mapped to OWASP Top 10 for LLM Applications and OWASP Agentic AI Top 10 with an owner assigned — deferring harm-model / red-team-data generation to `ai-risk-engineer`.

### mod-108 — Model Routing and Provider Selection
Choose the gateway (LiteLLM / OpenRouter / Portkey / Bedrock / Vertex / Foundry) with a trade-off note; route across providers and tiers with classifier-gated routing, quality gates, and provider-outage fall-back; self-host (vLLM / TGI / SGLang) vs managed for a given feature; compose multi-modal features (vision, audio, image / video, real-time voice) at the router level so modality is a routing decision; contract-test the feature across providers so a swap does not regress silently.

### mod-109 — Cost and Latency SLOs for LLM Features
Measure app-altitude latency the way users feel it (TTFT, TPOT, streaming p50 / p95, retry latency, tool-call round-trip); land provider prompt caching as a design constraint on system-prompt shape and versioning cadence; move offline workloads onto batch APIs with a written eligibility rubric; build per-feature and per-tenant cost / token accounting so 'cost per successful outcome' is a first-class metric; retry / hedging strategies with an SLO budget error rate and burn-rate alert — consuming MLPerf as a reference frame owned upstream.

### mod-110 — Observability, User Feedback, and Incident Response
Instrument LLM features with OpenTelemetry Gen-AI + OpenInference span attributes at debugging granularity; sampling policy (head, tail on error / quality signal, PII scrubbing) that keeps trace cost inside budget; user-feedback triage in Argilla or Label Studio with owner-assigned routing into an eval dataset; playbooks for the six recurring LLM-feature failure modes (bad prompt push, provider outage, judge drift, price hike, silent quality regression, guardrail FP spike); per-feature status page and on-call runbook with the interface to the platform / SRE team owned by `ai-infra-mlops`.

### mod-111 — Shipping, Experimenting, and Rollbacks
Feature flags (LaunchDarkly or OpenFeature) so a new prompt / model / chain / agent turns on for a cohort without a redeploy; canary and shadow launches with per-cohort quality / safety / cost gates; A/B experiment interface with deep methodology owned by `senior-ml-engineer` / `model-evaluation-engineer`; hot-rollback of a bad prompt without redeploying the service; hosted fine-tuning as an *applied* lever (style, format, small distillation) knowing SFT / PEFT / RLHF / DPO methodology depth is owned by `fine-tuning-engineer`.

### mod-112 — Owning an Applied-AI Feature End-to-End
Translate a product roadmap into a feature charter with success metric, non-goals, safety scope, cost budget, and kill criteria; run the incident-to-fixture loop; make build-vs-buy decisions across gateway (LiteLLM / OpenRouter / Portkey / Bedrock / Vertex / Foundry), eval platform (Braintrust / Weave / Langfuse / Phoenix / Promptfoo / DeepEval), and agent SDK (OpenAI Agents / Anthropic Claude Agent / LangGraph / Agno / Semantic Kernel); compose a per-feature model-card / feature-card slice from eval + incident + cost + safety evidence; author delegation contracts to the peer specialists; map the feature to NIST AI RMF, applicable EU AI Act obligations, and ISO/IEC 42001 controls.

## Assessment

Each module ships **1 quiz** plus the exercises and a lab listed in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Each project ships a portfolio-grade README, a reproducibility bundle (prompts, tool schemas, model / provider / decoding / seed hashes, eval fixtures, budget report, safety report), and an explicit rubric covering product-fit, integration quality, evaluation coverage, safety coverage, cost / latency SLO adherence, incident-readiness, and documentation.

## Where to go after this curriculum

- **`senior-agentic-ai-engineer-learning`** — next step on the AI Engineering ladder for architectural / multi-team scope.
- **`agentic-systems-architect-learning`** (level 48) — architectural depth for agentic systems.
- **`staff-ml-engineer-learning` / `principal-ml-engineer-learning`** — cross-team / architectural scope on the ML ladder.
- **`agentic-ai-engineer-learning`** — peer level-30 specialist that owns agent-framework depth; a natural cross-training track.
- **`ai-eval-engineer-learning`** — peer level-30 specialist that owns AI-application eval program methodology.
- **`fine-tuning-engineer-learning`** — peer level-30 specialist for post-training depth.
- **`ai-risk-engineer-learning`** — peer level-30 specialist for alignment-risk methodology and red-team data generation.
- **`ai-infra-ml-platform-learning`** / **`ai-infra-performance-learning`** — self-service ML / AI platform engineering and inference-serving performance.
- **`ai-infra-security-learning`** — deep ML/AI security, including model extraction and eval-set integrity.
- **`ai-governance-analyst-learning`** / **`head-of-ai-governance-learning`** / **`chief-ai-officer-learning`** — governance / compliance / model-card review depth and executive-level AI strategy.

<!-- needs-research: backfill industry-frequency evidence into JOB_REQUIREMENTS.md once the autonomous research loop runs with WebSearch / WebFetch exercised against live job boards; demote any module or exercise whose underlying requirement does not show up in ≥3 in-window postings. -->
