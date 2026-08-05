# mod-112-owning-an-applied-ai-feature-end-to-end: Owning an Applied-AI Feature End-to-End

**Estimated effort:** 12 hours

Every previous module in this track builds a *substrate* — prompting discipline, RAG plumbing, tool calling, agent shape, feature-side eval, guardrails, routing, cost / latency SLOs, observability, shipping and rollback. Each is a piece of the machine an applied-AI feature runs on. This module is the **owner's manual** for the feature the substrates run under: the artefacts, boundaries, and cadences that turn a roadmap line item into a governance-review-ready feature and keep it that way.

The through-line is that ownership is *artefactual*, not vibes-based. An applied-AI engineer who owns a feature end-to-end has, on file, a signed charter with kill criteria; a three-artefact close-out on every incident (fixture, alert, runbook); signed trade-off notes for the gateway, eval platform, and agent SDK chosen; a composable per-feature card that hands cleanly to the Governance-family peers; a signed delegation contract with each peer specialist whose depth the applied engineer plugs into; and a governance-mapping citation table connecting the feature to NIST AI RMF, EU AI Act transparency, and ISO/IEC 42001 controls. Any of these missing is a scope gap. All present is what the role's title actually means.

This module deliberately does not re-teach the substrates. It is the *composition layer* — how the substrates meet the org, how the applied engineer meets the peer roles, and how the feature meets external frameworks. Peer-role boundaries are named up-front so the applied engineer stays inside the surface they actually own; the deep work belongs to the specialist tracks. That is the capstone posture the rest of the track has been building toward.

## Learning objectives

- Translate a product roadmap into an applied-AI feature charter: user problem, success metric, non-goals, safety scope, cost budget, kill criteria.
- Run the incident-to-fixture loop: turn a real production incident into a permanent regression fixture, an alert, and a runbook update.
- Make build-vs-buy decisions across the three surfaces an applied engineer touches monthly: gateway (LiteLLM / OpenRouter / Portkey / Bedrock / Vertex / Azure AI Foundry), eval platform (Braintrust / Weave / Langfuse / Phoenix / Promptfoo / DeepEval), and agent SDK (OpenAI Agents / Anthropic Claude Agent / LangGraph / Agno / Semantic Kernel).
- Compose a per-feature model-card / feature-card slice from the eval + incident + cost + safety evidence that satisfies internal review and hands cleanly to `ai-governance-analyst` / `ai-evaluation-engineer` (Governance family).
- Author delegation contracts to peer specialists: `agentic-ai-engineer` (framework depth), `ai-eval-engineer` (program-level eval), `model-evaluation-engineer` (benchmark / statistical rigour), `ai-risk-engineer` (harm models & red-team data), `ai-infra-mlops` / `ai-infra-ml-platform` (deploy & platform), `ai-infra-performance` (serving perf), `ai-infra-security` (deep security).
- Map the feature to NIST AI RMF Map / Measure / Manage, the EU AI Act transparency obligations that apply, and ISO/IEC 42001 controls — knowing governance shape is owned by the peer Governance-family tracks.

## Chapters

1. [Translating a Roadmap Item into a Feature Charter](01-feature-charter-with-kill-criteria.md) — the six-field feature charter: user problem, success metric, non-goals, safety scope, cost budget, kill criteria. Signed by PM, engineer, and governance owner. Versioned in the repo. The design substrate every subsequent chapter cites.
2. [The Incident-to-Fixture Loop](02-incident-to-fixture-loop.md) — five stages (capture, redact, fixture, alert, runbook), three merged artefacts per incident, cross-substrate wiring, and the pre-mortem sibling that buys fixture coverage before the first user hits the feature.
3. [Build vs Buy: Gateway, Eval Platform, Agent SDK](03-build-vs-buy-across-three-surfaces.md) — the three monthly-cadence decisions; buy-option catalogue and decision matrix per surface; the exit-door discipline that keeps every choice reversible; the signed trade-off note that lands the decision.
4. [The Per-Feature Feature-Card Slice](04-per-feature-model-card-slice.md) — the nine-section evidence document; composable from mod-101-through-mod-111 artefacts; YAML front matter for Governance-family aggregation; the cadence that keeps the card from going stale.
5. [Delegation Contracts with Peer Specialists](05-delegation-contracts-with-peer-specialists.md) — the six-field contract; the seven peer roles the applied engineer routinely delegates to; what stays with the applied engineer; the delegation-contract inventory as a scope-gap check.
6. [Governance Mapping — NIST AI RMF, EU AI Act, ISO/IEC 42001](06-governance-mapping.md) — mapping vs compliance, the frameworks' feature-level slice, the citation-table shape, the Governance-family hand-off, the read audiences.

## Exercises

- [exercise-01 — feature charter with kill criteria](exercises/exercise-01-feature-charter-with-kill-criteria.md).
- [exercise-02 — incident-to-regression-fixture walkthrough](exercises/exercise-02-incident-to-regression-fixture-walkthrough.md).
- [exercise-03 — build-vs-buy matrix: gateway, eval, agent SDK](exercises/exercise-03-build-vs-buy-matrix-gateway-eval-agent.md).
- [exercise-04 — per-feature model-card slice](exercises/exercise-04-per-feature-model-card-slice.md).
- [exercise-05 — delegation contracts with peer specialists](exercises/exercise-05-delegation-contracts-with-peer-specialists.md).

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: primary standards, referenced tools, and canonical papers.

## Cross-module context

- The **charter and one-pager** in [chapter 01](01-feature-charter-with-kill-criteria.md) sit at two altitudes; the [mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md) one-pager and the [mod-101 chapter 03](../mod-101-applied-ai-foundations-and-feature-design/03-provider-and-tier-selection.md) trade-off note are directly composed with the charter.
- The **incident-to-fixture loop** in [chapter 02](02-incident-to-fixture-loop.md) plugs into telemetry from [mod-110](../mod-110-observability-feedback-and-incidents/README.md), fixtures from [mod-106](../mod-106-feature-side-evaluation/README.md), PII scrubbing from [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md), and runbook shape from [mod-110 chapter 05](../mod-110-observability-feedback-and-incidents/05-per-feature-status-page-and-on-call-runbook.md).
- **Build-vs-buy** in [chapter 03](03-build-vs-buy-across-three-surfaces.md) elaborates on the gateway decisions in [mod-108](../mod-108-model-routing-and-provider-selection/README.md), the eval platforms behind [mod-106](../mod-106-feature-side-evaluation/README.md), and the agent SDKs of [mod-104](../mod-104-tool-calling-and-orchestration/README.md) / [mod-105](../mod-105-shipping-product-agents/README.md).
- The **feature card** in [chapter 04](04-per-feature-model-card-slice.md) composes evidence from every previous module — offline eval ([mod-106](../mod-106-feature-side-evaluation/README.md)), guardrails ([mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md)), cost accounting ([mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md)), observability ([mod-110](../mod-110-observability-feedback-and-incidents/README.md)), rollout ([mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md)).
- The **delegation contracts** in [chapter 05](05-delegation-contracts-with-peer-specialists.md) are the artefact form of the peer-role boundaries named in every previous module's "peer-role handoffs" section.
- The **governance mapping** in [chapter 06](06-governance-mapping.md) is where the artefact stack meets external frameworks, with `ai-governance-analyst` as the primary Governance-family counter-party.

## Peer-role handoffs

The capstone-module version of the boundaries, gathered in one place so the applied engineer can see the full delegation graph:

- **`agentic-ai-engineer`** — deep framework expertise (LangGraph internals, OpenAI Agents SDK internals, MCP protocol nuance). The applied engineer wraps the SDK in a thin `run_turn` seam and delegates framework-depth work per the [chapter 05](05-delegation-contracts-with-peer-specialists.md) contract shape.
- **`ai-eval-engineer`** — the program-level eval substrate (judge selection, calibration, cross-feature dashboards). The applied engineer owns the feature-level fixtures; the peer role owns the shared substrate.
- **`model-evaluation-engineer`** — benchmark selection, statistical rigour (variance reduction, sequential testing, HTE), leaderboard-methodology critique. The applied engineer runs the A/B *interface*; the peer role runs the analysis.
- **`ai-risk-engineer`** — harm modelling, red-team programme, adversarial dataset curation, org-wide risk register. The applied engineer's OWASP mapping and safety-scope feeds the peer's register.
- **`ai-infra-mlops` / `ai-infra-ml-platform`** — the deploy substrate (k8s / Nomad / serverless, CI/CD, secrets, cluster capacity). The applied engineer owns the feature's config; the peer role owns the platform.
- **`ai-infra-performance`** — serving-tier optimisation (GPU utilisation, batching, quantisation, KV-cache tuning) for self-hosted inference from [mod-108 chapter 03](../mod-108-model-routing-and-provider-selection/03-self-hosted-vs-managed-with-vllm-baseline.md).
- **`ai-infra-security` / `application-security-engineer`** — authz architecture, MCP-server security review, tool-credential rotation, sandboxed-execution design. The applied engineer's Guard 4 from [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/README.md) is implemented on the peer's design.
- **`ai-governance-analyst`** — composes org-wide governance evidence; primary counter-party for the feature-card front matter and the governance mapping in [chapter 06](06-governance-mapping.md).
- **`fine-tuning-engineer`** — SFT vs PEFT vs DPO vs RLHF methodology depth; owns everything below the applied engineer's hosted-fine-tune surface from [mod-111 chapter 05](../mod-111-shipping-experimenting-and-rollbacks/05-hosted-fine-tuning-as-an-applied-lever.md).
- **`privacy-engineer` / `data-security-engineer`** — regulatory posture (DPIA, cross-border transfer, DPA authority interactions). The applied engineer scrubs and stays inside the envelope; the peer role designs the envelope.

Every peer-role name here appears in a written delegation contract for every non-trivial feature. The absence of a contract for one of these roles is the scope gap the applied engineer either closes or explicitly names in the feature card's "known limitations" section.
