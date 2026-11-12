# Build vs Buy: Gateway, Eval Platform, Agent SDK

## Why these three, and why every month

An applied-AI engineer touches many substrates over a feature's lifetime, but three of them come up in almost every quarterly plan: the **gateway** that fronts the model providers, the **eval platform** that hosts the offline / online eval loops, and the **agent SDK** the feature's multi-turn logic is written against. Each is a category with a healthy commercial ecosystem *and* a healthy open-source ecosystem *and* a plausible "we can just write this ourselves" option. Each decision is reversible-with-cost, revisit-eligible, and load-bearing for the substrates every other mod-1xx chapter builds on.

The applied engineer is not writing new frameworks in these categories. The engineer is *selecting* one, integrating it, keeping the exit door open, and revisiting the choice on a cadence. Getting the selection wrong is expensive but rarely catastrophic; the catastrophic outcome is not selecting at all — running with an in-house half-implementation that eats headcount forever.

This chapter names, for each of the three surfaces:

- The **buy** options an applied engineer will encounter and what each is good at.
- The **build** option and the price of taking it.
- The **decision matrix** — the four to six axes that actually move the choice.
- The **exit door** — what portability shape keeps the choice reversible.
- The **revisit cadence** — when the decision is due for re-examination.

The chapter deliberately does *not* declare a winner in any of the three categories. Winners change with a release cycle; disciplines don't.

## Surface 1 — The gateway

The gateway is the layer between the feature's code and the model providers. It handles routing, retries, fallbacks, rate limiting, per-key cost accounting, PII scrubbing, response caching, and observability plumbing. It is the substrate [mod-108](../mod-108-model-routing-and-provider-selection/README.md) covers in depth.

### The buy options

- **LiteLLM.** Open-source Python proxy and SDK. Speaks the OpenAI API shape across 100+ providers, includes routing, retries, cost tracking, and a proxy mode with virtual keys. Self-hosted; managed offering exists. Best fit when you want a single OpenAI-shaped client across many providers and are comfortable running a proxy.
- **OpenRouter.** Hosted routing service. Single API, priced as a percentage platform fee on top of pass-through inference costs, includes automatic per-request fallback between providers when a route fails. Best fit for prototypes, side-projects, and orgs that want zero infra and are OK with a hosted third-party in the model call path.
- **Portkey.** Gateway with SDK and proxy modes, focused on production observability and governance: routing, caching, guardrails hooks, per-tenant budgets, audit logs. The core gateway is open-source; the managed platform is tiered by log volume. Best fit for orgs that want a managed control plane for LLM traffic.
- **AWS Bedrock (as gateway).** AWS-hosted access to a curated model catalogue (Anthropic, Meta, Mistral, Cohere, AI21, Amazon's own Titan/Nova, Stability AI, and additional providers added over time) plus routing / guardrails / knowledge base services. Best fit for AWS-native shops with strict data-residency requirements or with existing AWS-committed spend.
- **Google Vertex AI (as gateway).** GCP-hosted equivalent — Model Garden with Gemini and Gemma, Anthropic Claude on Vertex, Meta Llama, Mistral, and others; native integration with Vertex evaluation and pipelines. Best fit for GCP-native shops.
- **Microsoft Foundry** (rebranded from Azure AI Foundry effective January 2026, which itself consolidated Azure AI Studio and the Azure OpenAI Service). Azure-hosted unified catalogue of OpenAI + third-party models + evaluation + agent runtime. Best fit for Microsoft-committed enterprises.

### The build option

Writing your own gateway means implementing: multi-provider adapters, exponential-backoff retries with jitter, per-key rate limiting, response caching, cost tallying against a pricing table, span emission in the OTel Gen-AI shape from [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md), streaming translation across provider quirks, and — if you want fallback — a routing policy engine.

The build option is defensible in exactly one shape: **the feature has a hard requirement (residency, latency, security) that no buy-option meets, and the team has committed at least one engineer as long-term owner of the gateway itself.** In every other shape, a partial home-grown gateway competes with a full-featured buy option and loses on evolution: the buy vendor ships new provider support in a week; you ship it in a sprint.

### The decision matrix

Six axes actually move the gateway choice.

1. **Provider coverage.** How many providers does the gateway abstract, and do the ones your feature will use in the next 12 months appear? Coverage of five providers is not helpful if the feature needs Anthropic on AWS Bedrock and the gateway only speaks direct-API Anthropic.
2. **Deployment shape (proxy vs SDK vs both).** Proxy mode is a network hop; SDK mode is a library. Proxy is friendlier to non-Python languages and centralises policy; SDK is lower latency and simpler to reason about. The decision compounds with the org's polyglot posture.
3. **Fallback semantics.** Per-request auto-fallback (the request that just failed on provider A retries on B), pool-level failover (all traffic moves to B on a health signal), or none. Different failure modes call for different semantics — see [mod-108 chapter 03](../mod-108-model-routing-and-provider-selection/03-self-hosted-vs-managed-with-vllm-baseline.md).
4. **Governance features.** Per-key / per-tenant budgets, virtual keys, PII scrubbing, moderation hooks, audit logs. If any of these must be implemented downstream, the gateway is only doing half its job and the *other* half becomes your build cost.
5. **Observability integration.** Does the gateway emit OTel Gen-AI spans natively? Does it export cost / latency metrics in a shape your telemetry stack ingests? Emitting the mod-110 span attributes without an adapter is worth engineering weeks per feature.
6. **Data-plane residency.** Where does the traffic *physically* transit? A hosted gateway (OpenRouter, Portkey managed) inserts a third-party into the data plane; a self-hosted proxy (LiteLLM, self-hosted Portkey) does not. This is a `privacy-engineer` conversation with a `data-security-engineer` cross-check — see [chapter 05](05-delegation-contracts-with-peer-specialists.md).

### The exit door

Portability from a gateway is the ability to **switch to a different gateway (or none) in one sprint** with the feature still behaving. Two disciplines keep that door open.

- **Call the gateway through an OpenAI-shaped client.** Every serious gateway speaks the OpenAI API shape as its primary interface. If your feature code is written to the OpenAI SDK and points at the gateway's base URL, swapping gateways is a config change. Writing against a gateway-proprietary SDK (a "PortkeyClient" or "LiteLLMClient") is the anti-pattern; it moves the exit door behind a locked SDK.
- **Ship model IDs, prompts, and retry policy from your side of the seam.** Do not let the gateway own the prompt registry, the model ID pin, or the retry-budget config. Those are feature-level artefacts, and moving them into the gateway's UI means the exit involves re-implementing them.

### Revisit cadence

Every 6 months, or on any of three triggers: (a) a provider you care about is missing from the gateway's roster, (b) the cost the gateway itself imposes (proxy latency, per-request margin, hosting) exceeds the value it delivers, (c) a new gateway shipped a capability the incumbent has not (e.g., first-class OTel Gen-AI export). Every revisit produces a short trade-off note in the same shape as [mod-101 chapter 03](../mod-101-applied-ai-foundations-and-feature-design/03-provider-and-tier-selection.md) — signed by the applied engineer plus whoever owns the FinOps and privacy sides.

## Surface 2 — The eval platform

The eval platform is where offline fixtures run, where LLM judges are hosted and versioned, where online-eval sampling flows land, where dashboards for judge scores live, and where the eval-gated CI reads its gates from. It is the substrate [mod-106](../mod-106-feature-side-evaluation/README.md) covers in depth.

### The buy options

- **Braintrust.** Commercial eval platform. Fixtures, LLM-judge library, dataset management, experiment comparisons, integrations to CI. Strong on the "author fixtures and compare model runs" workflow. Best fit for teams that want a batteries-included eval product and are comfortable with a SaaS control plane.
- **Weights & Biases Weave.** W&B's LLM-focused subproduct — trace logging, evals, dataset management, judge scaffolding — a layer alongside the core W&B experiment-tracking product. Best fit for teams already in the W&B ecosystem, or wanting model-experiment tracking and LLM-eval tracking in one platform.
- **Langfuse.** MIT-licensed open-source core, available self-hosted or via a tiered managed cloud. LLM tracing + eval + prompt management. Popular with LangChain-heavy stacks; increasingly framework-neutral. Best fit for teams that want the option to self-host and to see the code.
- **Arize Phoenix.** Self-hostable LLM observability + eval (from Arize AI), source-available under the Elastic License 2.0 and companion to the commercial Arize AX platform. OpenInference span standard is Phoenix's native format. Best fit for teams already committed to OpenInference tracing and wanting a self-hostable eval companion.
- **Promptfoo.** Open-source (MIT) CLI-first eval tool for prompts, models, and RAG pipelines, with first-class LLM-as-a-judge assertions (`llm-rubric` and friends). Focused on the "run a matrix of variants and compare" workflow. Best fit for eval-gated CI on prompt changes and for teams that want to keep eval config in the repo, not a SaaS UI.
- **DeepEval.** Apache-2.0 Python-first eval framework maintained by Confident AI (which also sells a hosted platform on top). Pytest-style assertions on LLM outputs, judge scaffolding, RAG-specific metrics. Best fit for eval as code inside an existing pytest suite.
- **Managed cloud offerings.** OpenAI Evals (as a hosted service), Vertex AI Gen AI Evaluation, and the Microsoft Foundry evaluation service (formerly Azure AI Foundry Evaluation) — each cloud provider bundles an eval product with its platform. Best fit when the feature is already single-cloud and the cloud's eval product covers the needed judge and dataset shapes.

### The build option

Writing your own eval platform means implementing: a fixture format, a run-a-fixture-against-a-model harness, a judge invocation layer with retries and caching, a comparison UI or artefact, a datastore for run history, dashboards for score-over-time, and hooks into CI for gating. Additionally: judge-vs-ground-truth calibration substrate ([mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md)), which is not trivial.

The build option is defensible when the org already has strong ML-experiment tracking infrastructure that eval slots into (a peer `ai-eval-engineer` role on the Governance-family side has usually driven this, and the applied engineer plugs feature-level fixtures into it). It is *not* defensible for a solo applied engineer at feature-launch time; the six-month cost of building it dwarfs the six-month cost of paying a vendor.

### The decision matrix

1. **Fixture format portability.** Does the platform's fixture format export cleanly to a text file that can live in the repo? If fixtures live only in the vendor's UI, the exit door is closed. `eval/fixtures/regressions/*.yaml` from [chapter 02](02-incident-to-fixture-loop.md) is the reference — you should be able to run the same fixtures against any of the buy-options within a day of switching.
2. **Judge substrate.** Does the platform host judge templates as first-class artefacts (versioned, calibrated, drift-monitored)? An eval platform without a judge substrate is a runner-and-a-UI and you will build the judge substrate yourself.
3. **Online-eval sampling.** Does the platform accept production trace samples for periodic evaluation ([mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md))? If not, you are running two eval systems (offline in the platform, online in your own code) and the drift-monitoring story falls apart.
4. **CI integration.** Is there a CLI or API to fail a PR based on eval scores, and to attach the eval diff to the PR? If not, the [mod-106 chapter 03](../mod-106-feature-side-evaluation/03-eval-gated-ci-end-to-end.md) discipline is a manual step and it will atrophy.
5. **Framework neutrality.** Does the platform work whether the feature is written in the OpenAI SDK, LangChain, LlamaIndex, or a custom orchestrator? Vendor-locked eval platforms that only ingest their own tracer's format are a switching-cost trap.
6. **Cost model.** Per-evaluation, per-judge-token, per-seat, or self-hosted infra. Small teams under-index on this and get surprised at the invoice when fixture counts grow past 500.

### The exit door

- **Fixtures in the repo, in a plain-text format** (YAML or JSON), independent of the platform. The platform ingests them; the platform is not the source of truth.
- **Judge prompts in the repo**, versioned, with the pin and the calibration-set path checked in. If the platform loses your judge on a version upgrade, you re-instantiate from the repo.
- **A one-page migration runbook** listing every artefact type (fixtures, judges, run history, dashboards) and where each one is on the current platform, so a future migration is a checklist, not a discovery project.

### Revisit cadence

Every 12 months, or on triggers: (a) the platform's cost outpaces value, (b) a category of eval you need (e.g., agentic-trace evals, multi-turn conversation eval) is not supported and a competitor supports it, (c) the platform's judge substrate has drifted or its calibration workflow has been deprecated. This is a longer cadence than the gateway because eval platform switching costs are higher — the fixtures port, but the run history rarely does.

## Surface 3 — The agent SDK

The agent SDK is the framework the feature's multi-turn, tool-using logic is written against. It is the substrate [mod-104](../mod-104-tool-calling-and-orchestration/README.md) and [mod-105](../mod-105-shipping-product-agents/README.md) cover in depth. This decision is the most contested of the three, because the SDK ecosystem changes fast and the choice binds the feature's control-flow model, error-handling shape, and observability semantics.

### The buy options

- **OpenAI Agents SDK.** OpenAI's official agents framework and the recommended path for new agent development following the deprecation of the older Assistants API (which stopped accepting requests in August 2026). Tightly integrated with OpenAI tool-call semantics, function schemas, structured outputs. Best fit when the feature is committed to OpenAI as the primary provider and wants the shortest path to a shipped agent.
- **Anthropic Claude Agent SDK.** Anthropic's official agent SDK (renamed from the Claude Code SDK in September 2025 to reflect its broader scope). Native Claude tool-use semantics, native Model Context Protocol (MCP) integration, computer-use tooling. Best fit when the feature is committed to Claude and wants the shortest path to an MCP-first agent.
- **LangGraph.** LangChain's graph-based agent framework; reached 1.0 in October 2025 with a stated no-breaking-changes commitment until 2.0. Explicit state machine, deterministic control flow, checkpointing, human-in-the-loop primitives. Best fit for features whose control flow is complex enough to *want* to be a graph (branching workflows, long-running processes, human review loops).
- **Agno.** Open-source Python framework for agents and multi-agent teams, rebranded from Phidata (same project, new name and import path). Focus on lean agent composition and multi-agent teams. Best fit for teams wanting a lighter-weight primitive stack than LangGraph.
- **Microsoft Agent Framework** (.NET / Python). Microsoft's unified agent SDK, GA since April 2026, which merged the former Semantic Kernel and AutoGen projects into a single stack — Semantic Kernel as the foundation, AutoGen-style multi-agent orchestration on top — with both predecessor projects now in maintenance mode. Deep Microsoft-ecosystem integration (Microsoft Foundry, Copilot). Best fit for Microsoft-committed enterprises.

### The build option

Writing your own agent runtime means implementing: a turn loop, a tool-call dispatcher, a tool-schema registry, structured-output parsing with repair, memory / state management, streaming, cancellation, retries, and observability. Additionally: any framework-specific primitives (checkpointing, human-in-the-loop, sub-agent handoffs).

The build option is defensible when the feature's control flow is so specific that every SDK is a poor fit *and* the team has committed a long-term owner. It is much *less* defensible than the gateway or eval-platform build cases, because agent SDKs iterate fastest and the build-option's opportunity cost — features you cannot ship because you are maintaining a framework — is largest.

The middle-ground move is to build a *thin wrapper* around a chosen SDK that exposes only the primitives your features need. This is often the right call when the SDK is chosen but its API shape is unstable — the wrapper insulates feature code from breaking changes.

### The decision matrix

1. **Provider neutrality.** Is the SDK single-provider (OpenAI Agents, Anthropic Claude Agent), or multi-provider (LangGraph, Agno, Semantic Kernel)? Single-provider SDKs deliver the sharpest primitives *for that provider* but bind the feature to the provider more tightly than the gateway alone does.
2. **Control-flow model.** Turn loop, ReAct agent, graph state machine, event-driven. Different features call for different models. A feature with a linear happy path and 3 tools is over-served by LangGraph; a feature with branching workflows and human review is under-served by a bare turn loop.
3. **Observability semantics.** Does the SDK emit spans in the OTel Gen-AI + OpenInference vocabulary from [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)? Framework-native tracing that does not translate to your stack means you are writing the translator.
4. **Tool-call semantics.** Function-calling shape, structured-output support, MCP integration, tool-schema validation. Frameworks vary widely in how they handle a tool that returns an error or a schema-invalid response — see [mod-104 chapter 02](../mod-104-tool-calling-and-orchestration/exercises/exercise-02-tool-error-handling-taxonomy.md) for the taxonomy.
5. **Human-in-the-loop and interruption.** Does the SDK have first-class primitives for pausing a run, requesting human approval, and resuming? [mod-105 chapter 03](../mod-105-shipping-product-agents/03-human-in-the-loop-and-steering-ux.md) is where this matters most.
6. **Community and roadmap velocity.** How often does the SDK ship? How many contributors? What is the vendor's track record on API stability? A framework that breaks its API every quarter is one whose upgrades will eat sprints.

### The exit door

- **The tool schemas, prompts, and business logic live in *your* code**, not in the SDK. Every SDK invites you to put orchestration decisions inside its DSL; every migration is bounded by how much of your logic sits in that DSL. Keep the DSL surface thin.
- **Trace emission is normalised to OTel Gen-AI**, either by the SDK's own emitter or by a translator you own. On migration, dashboards continue to work.
- **A framework wrapper pattern for the hot path.** A `run_turn(feature, request) -> response` function in `app/agent.py` that internally uses the SDK is a much smaller migration than a codebase full of framework primitives.

### Revisit cadence

Every 6 months for the first year of a feature (fastest-moving substrate), then annually. Triggers: (a) the SDK misses a primitive the feature needs and a competitor has it (e.g., first-class MCP support, first-class computer-use tooling), (b) the SDK has ended support for a version you depend on, (c) the org's provider posture has changed enough that a single-provider SDK is no longer a fit.

## The three surfaces together — a working posture

The three decisions compound, and one posture works for the majority of applied-AI teams:

- **Gateway** — buy, self-hosted (LiteLLM as the OSS default; a managed option if the FinOps case pencils out). Speak OpenAI API shape at every call site. Own the model-ID pin, prompt registry, and retry policy on your side of the seam.
- **Eval platform** — buy, managed for the UI and run history; fixtures and judge prompts in the repo. Fixture format is portable. Revisit every 12 months.
- **Agent SDK** — buy, single-vendor if the provider posture is committed; multi-vendor (LangGraph / Agno / Semantic Kernel) if not. Wrap in a thin `run_turn` seam. Revisit every 6 months for the first year.

This is a posture, not a prescription. The applied engineer's job is not to memorise this list — it is to run the decision matrices against the feature's actual constraints, write the trade-off note, sign it with the FinOps and privacy peers, and revisit on cadence.

## Writing the trade-off note

The output of a build-vs-buy decision on any of the three surfaces is a **trade-off note** in the shape of [mod-101 chapter 03](../mod-101-applied-ai-foundations-and-feature-design/03-provider-and-tier-selection.md). Working shape:

```markdown
# Gateway trade-off note — v0.2 — 2026-07-14

## Decision
Buy: LiteLLM (self-hosted proxy). Reject build.

## Context
- Feature: support-refund-composer (charter v0.3).
- Provider posture: primary Anthropic (Sonnet 4.5), fallback OpenAI (gpt-4o dated).
- Residency: US-only in v0; EU tenants deferred.
- Traffic model: 200k requests/day at 100% rollout.

## Options considered
- Build: rejected — no engineer committed to long-term ownership; provider count grows.
- LiteLLM (self-hosted): chosen — OpenAI-shape client, covers both providers, proxy mode centralises PII scrub, exports OTel Gen-AI spans.
- Portkey (managed): rejected v0 — data-plane residency review pending with `privacy-engineer`; revisit at 6-month mark.
- Bedrock (as gateway): rejected — Anthropic-on-Bedrock is available but adds AWS regional constraints we do not want to inherit for a US-first launch.

## Decision matrix summary
[table with the six axes, one row per option, one column per axis]

## Exit door
- Feature calls the gateway via OpenAI Python SDK pointed at LiteLLM base URL.
- Model IDs and prompt IDs stored in prompt-registry (chapter 04-mod-111), not in LiteLLM config.
- Switching to Portkey: est. 3 days (config + observability re-verification).

## Revisit
- Next scheduled: 2027-01-14.
- Triggers: pricing shift, new provider needed, EU tenant onboarding.

## Signed
applied-ai-engineer:jferguson, finops:kfujita, privacy:oluwe, 2026-07-14.
```

Every column, every option, every sign-off. The note is the artefact the next applied engineer inherits when this one moves teams; it is what protects the feature from an accidental re-litigation of a settled decision.

## Anti-patterns

- **Committing to build without a long-term owner.** The half-built gateway becomes an eternal side project; the eval platform is "we should really finish"; the agent runtime is "we'll upstream when we can." Build only when an owner is committed for a year.
- **Calling the gateway through a vendor-specific SDK.** The exit door is closed. Every gateway serious about production speaks OpenAI-API shape; use it.
- **Eval fixtures that live only in the vendor's UI.** The exit door is closed. Fixtures in the repo, always.
- **Agent-SDK primitives sprayed throughout the codebase.** Every framework migration is a codebase migration. Wrap the SDK; keep the surface thin.
- **Decision that never revisits.** The gateway market ships something new every quarter. A choice made in 2024 in 2026 has probably drifted from optimal. Cadence.
- **Skipping the trade-off note.** The decision is folklore. New team member cannot tell why LiteLLM was chosen over Portkey. The next re-litigation re-runs the same analysis from scratch.
- **Managed offerings without a residency review.** Every managed gateway / eval / agent platform inserts a third party into the data plane. The `privacy-engineer` sign-off on the trade-off note is not optional; see [chapter 05](05-delegation-contracts-with-peer-specialists.md).

## Summary

- Three surfaces the applied engineer selects, integrates, and revisits: the **gateway**, the **eval platform**, and the **agent SDK**. Each has a healthy commercial ecosystem, a healthy open-source ecosystem, and a plausible-but-costly build option.
- **Gateway** buy options: LiteLLM, OpenRouter, Portkey, Bedrock, Vertex, Azure AI Foundry. Decision matrix: provider coverage, deployment shape, fallback semantics, governance features, observability integration, data-plane residency. Exit door: OpenAI-shaped client + repo-owned model IDs / prompts / retries. Revisit every 6 months.
- **Eval platform** buy options: Braintrust, Weave, Langfuse, Phoenix, Promptfoo, DeepEval, cloud-native offerings. Decision matrix: fixture format portability, judge substrate, online-eval sampling, CI integration, framework neutrality, cost model. Exit door: fixtures + judges in the repo; migration runbook on file. Revisit every 12 months.
- **Agent SDK** buy options: OpenAI Agents, Anthropic Claude Agent, LangGraph, Agno, Semantic Kernel. Decision matrix: provider neutrality, control-flow model, observability semantics, tool-call semantics, human-in-the-loop primitives, community and roadmap velocity. Exit door: thin `run_turn` wrapper, OTel-Gen-AI-normalised tracing. Revisit every 6 months for the first year, annually after.
- **Build is defensible** only when: no buy-option meets a hard requirement *and* a long-term owner is committed. In every other shape, buy.
- Every build-vs-buy decision produces a signed **trade-off note** on the [mod-101 chapter 03](../mod-101-applied-ai-foundations-and-feature-design/03-provider-and-tier-selection.md) shape: decision, context, options, matrix, exit door, revisit, sign-off.
- Managed offerings insert a third-party into the data plane. `privacy-engineer` sign-off is a required column on the trade-off note.

[Chapter 04](04-per-feature-model-card-slice.md) turns from "picking substrates" to "composing evidence about the shipped feature" — the per-feature model-card / feature-card slice that satisfies internal review and hands cleanly to the Governance-family peer roles.
