# AI Engineering · Applied AI Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum) · [Startup](https://github.com/ai-startup-curriculum) · [Workforce](https://github.com/ai-workforce-curriculum).
<!-- /aicg:site-banner -->

<!-- aicg:sponsor -->
> 💜 **[Sponsor this curriculum](https://github.com/sponsors/ai-engineering-curriculum)** — sponsorships keep the whole open-source AI Career Curriculum free and moving.
<!-- /aicg:sponsor -->

Ship and operate LLM/agent/RAG features inside real products end-to-end: production prompting, RAG integration, tool calling, product-embedded agents, feature-side evaluation, surface guardrails, model routing, cost / latency SLOs, observability, incident response, and feature-lifecycle ownership.

> **Status**: curriculum plan authored (2026-07-09). Lessons and project READMEs will be drafted by subsequent autonomous content cycles. See [`CURRICULUM.md`](CURRICULUM.md) for the planned scope and [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map (postings list deferred to the next research cycle — see `JOB_REQUIREMENTS.md` Status section).

**Level:** 30 (deep specialist — peer to `agentic-ai-engineer` and `ai-eval-engineer` at the same level; one level above the build-altitude peer `llm-application-developer` at level 25; peer to `senior-ml-engineer` at the same level on the ML Engineering ladder).
**Family:** AI Engineering.
**Planned scope:** 12 modules (156h) + 3 projects (140h) = ~296 hours.

## What this track owns

End-to-end production integration of LLM/agent/RAG features into a product surface:

- Applied AI foundations — spec → LLM-feature architecture, provider / tier / shape choice, SLO mapping, ISO/IEC 25059 & NIST AI RMF framing
- Production prompting & structured outputs — prompt caching, JSON mode / function calling, constrained decoding (Instructor / Outlines), streaming + cancellation, prompt versioning + rollback
- RAG integration at the product surface — consuming a retriever, citations / freshness / evidence UX, retrieval quality instrumentation, query rewriting
- Tool calling & orchestration at product altitude — JSON schemas, MCP servers, LangGraph / OpenAI Agents SDK / Anthropic Claude Agent SDK, per-request budgets
- Shipping product-embedded agents — coding co-pilot, customer-experience, retrieval-with-actions, browsing, sales / support / ops shapes; memory system with retention + PII policy; hand-offs & kill-switches
- Feature-side evaluation — offline suites, snapshot-replay regression fixtures, eval-gated CI, sampled online eval, judge-drift monitor, per-feature dashboards
- Guardrails & safety on the surface — layered moderation, prompt-injection defence, PII scrubbing, tool-abuse guards, OWASP LLM & Agentic Top 10 mapping
- Model routing & provider selection — LiteLLM / OpenRouter / Portkey / Bedrock / Vertex / Foundry gateway patterns, classifier-gated routing, self-host vs managed, multi-modal
- Cost & latency SLOs — TTFT / TPOT at app altitude, prompt caching, batching, per-tenant token / cost accounting, retry / hedging, burn-rate alerts
- Observability, user feedback & incident response — OpenTelemetry Gen-AI / OpenInference instrumentation, sampling + PII scrubbing, Argilla / Label Studio triage, six-failure-mode playbooks, on-call runbooks
- Shipping, experimenting & rollbacks — feature-flag driven rollout (LaunchDarkly / OpenFeature), canary + shadow launches, A/B interface, hot-rollback of a bad prompt, hosted fine-tuning as an applied lever
- Owning an applied-AI feature end-to-end — charter with kill criteria, incident-to-fixture loop, build-vs-buy across gateway / eval / agent SDK, per-feature model-card slice, delegation contracts to peer specialists

## What this track defers

- Build-altitude first-time LLM app construction → [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) (peer, level 25 — the on-ramp to this track).
- RAG engineering depth (chunking, embeddings, vector DB internals, rerankers, hybrid search) → [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) (peer, level 25).
- Agent-framework methodology depth (LangGraph / CrewAI / AutoGen internals, multi-agent orchestration) → [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (peer, level 30).
- AI-application evaluation program methodology → [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30).
- Model-eval methodology depth (validity theory, benchmark construction, statistical rigour, cross-modality methodology, MLPerf serving benchmarks) → [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family).
- Post-training depth (SFT / PEFT / RLHF / DPO) → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, level 30).
- Alignment-risk methodology, harm modelling, and red-team data generation → [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) (peer, level 30).
- Release-assurance / governance shape (audit trails, regulator-facing docs, third-party evaluator interface) → [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer, level 25, Governance family).
- MLOps / model registry / self-service ML/AI platform / inference serving → [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning), [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning), [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning).
- Classical ML fundamentals (PyTorch, sklearn, FastAPI packaging, MLflow) → [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20).
- Deep ML/AI security (model extraction, eval-set exfiltration, prompt-injection defence at platform depth) → [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning).
- Governance / policy / model-card review depth → [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning), [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning).

## Layout

```
applied-ai-engineer-learning/
├── .aicg/                    curriculum-plan.json and job-requirements.json (machine-readable catalog)
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes (to be drafted)
├── projects/project-XXX-*/   multi-module capstones (to be drafted)
├── CURRICULUM.md             role-level coverage map
├── JOB_REQUIREMENTS.md       requirements catalog with citations and ownership map
├── PREREQUISITES.md          assumed entry skills
├── VERSIONS.md               release history
└── README.md                 this file
```

## Paired Solutions Repo

[`applied-ai-engineer-solutions`](https://github.com/ai-engineering-curriculum/applied-ai-engineer-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
