# Delegation Contracts with Peer Specialists

## Why "delegation" needs a written contract

The applied-AI engineer sits at the centre of a diagram where seven or eight adjacent specialist roles each own a depth the applied engineer only touches at the surface. This is not a weakness of the role; it is the *shape* of the role. Every previous chapter has named a boundary: agent-SDK depth belongs to `agentic-ai-engineer`, program-level eval to `ai-eval-engineer`, statistical rigour and benchmarks to `model-evaluation-engineer`, harm modelling to `ai-risk-engineer`, platform to `ai-infra-ml-platform`, serving performance to `ai-infra-performance`, deep security to `ai-infra-security`.

Naming the boundary is necessary but insufficient. Boundaries dissolve at incident time unless there is a written contract on either side that says: *what* the peer role owns, *what* the applied engineer commits to hand over, *what* the applied engineer expects in return, and *when* the two sides re-negotiate. Without the contract, the applied engineer either does the peer role's job badly (because they lack depth) or leaves it undone (because "someone else owns that"). Both failure modes are common and both are avoidable.

This chapter names the seven peer roles the applied-AI engineer routinely delegates to, the *shape* of a delegation contract, and a worked contract for each peer. The chapter does not attempt to describe the peer roles' own disciplines — those belong to their own tracks. It describes only the applied-engineer *side* of the interface.

## The delegation contract shape

Every delegation contract has the same six fields. Six is enough to make the interface concrete and few enough to fit on a page.

1. **Peer role** — the named role and (where possible) the specific person or team on-call for it.
2. **Scope handed off** — a bounded description of *what* is delegated. Not "safety"; "adversarial red-team programme for the support-refund-composer surface, quarterly cadence, feedback into the eval fixture set."
3. **Applied-engineer's inputs** — what the applied engineer commits to provide the peer role, in a specific format, on a specific cadence. This is where most contracts fail — the applied engineer thinks they can hand over "the feature" and the peer role asks for a trace-schema dump they were never wired to produce.
4. **Peer-role's outputs** — what the peer role commits to return, in a specific format, on a specific cadence. Findings, dashboards, fixture PRs, sign-offs.
5. **Escalation and revisit** — how the two sides handle disagreement, and when the contract is due for re-signing. Contracts unrevised for a year are contracts that describe last year's feature.
6. **Signed** — both parties, with date. A contract without a signature is a wish.

The contract lives at `docs/features/<feature>/delegations/<peer-role>.md`, in the repo, in the same folder as the charter and the feature card. Each is 1–2 pages of Markdown. Aggregated across features they become the "who owes whom what" map the eng-manager or program-manager role can read at planning time.

## The seven contracts

The applied engineer will have delegation contracts with, at minimum, the seven peer roles below. Not every feature needs every contract; a feature without an agentic control loop can skip the `agentic-ai-engineer` contract, a feature without production self-serving can skip the `ai-infra-performance` contract. But every contract that *is* present is written.

### Contract 1 — `agentic-ai-engineer`: framework depth

- **Scope handed off.** Deep expertise on the chosen agent framework (LangGraph internals, OpenAI Agents SDK internals, Semantic Kernel's planner semantics, MCP protocol nuances). Framework upgrades, breaking-change navigation, non-trivial primitive composition (checkpointing recovery semantics, subgraph reentrancy, tool-call streaming corner cases).
- **Applied-engineer inputs.** The chosen SDK (from [chapter 03](03-build-vs-buy-across-three-surfaces.md)), the feature's control-flow requirements, the `run_turn` wrapper interface, the tool schemas and their intended semantics, the trace vocabulary the feature emits.
- **Peer-role outputs.** Framework-upgrade PRs with tested compatibility notes, framework-specific primitive patterns (e.g., "how to do checkpointed retry in LangGraph v3"), review of any non-trivial DSL usage in the applied engineer's code, ownership of the framework's own bug reports upstream.
- **Escalation.** When an SDK feature the applied engineer needs is missing or broken, the applied engineer files with the peer role; the peer role handles the vendor / upstream engagement.
- **Revisit.** At every SDK-version bump and on any framework-related incident.

Working shape:

```markdown
# Delegation — agentic-ai-engineer — support-refund-composer

**Peer role:** agentic-ai-engineer (owned by the platform-agents team, on-call: name).
**Scope handed off:** LangGraph internals, upgrades, MCP protocol integration.

**Applied-engineer inputs**
- Feature commits use LangGraph via app/agent.py run_turn() wrapper (no direct LangGraph
  imports in feature code).
- Weekly: the list of LangGraph primitives the feature relies on, with test coverage.
- Per SDK-upgrade PR from the peer: rebase and re-verify against the feature's regression
  fixture set within 3 business days.

**Peer-role outputs**
- LangGraph version pin in requirements.in owned by the peer role.
- Breaking-change PRs with a compatibility note by the release date + 5 business days.
- Review of any new LangGraph primitive in app/agent.py within 2 business days of PR open.

**Escalation and revisit**
- SDK primitive missing / broken → file with peer role; they engage upstream.
- Contract revisited on every LangGraph major-version bump and after any P2+ incident
  tagged `framework`.

**Signed:** applied-ai:jferguson, agentic-ai:name, 2026-08-04. Next revisit: on trigger.
```

### Contract 2 — `ai-eval-engineer`: program-level eval

- **Scope handed off.** The org's eval *program* — cross-feature eval strategy, judge-model selection and calibration at breadth, eval-platform contract negotiation, judge-drift monitoring across the fleet, benchmark-derivation methodology from [mod-105 chapter 04](../mod-105-shipping-product-agents/04-eval-suites-from-benchmark-shapes.md) applied at the program level. The applied engineer's *feature-level* eval fixtures and judges are the applied engineer's job; the *shared* substrate (judge selection, calibration cadence, cross-feature dashboards) is the peer role's.
- **Applied-engineer inputs.** Feature-level fixture PRs, judge-usage patterns, judge-vs-ground-truth agreement numbers for the feature's judges, any observed drift signals.
- **Peer-role outputs.** The shared judge substrate (versioned templates, calibration cadence), the eval-platform contract from [chapter 03](03-build-vs-buy-across-three-surfaces.md), the drift-monitoring dashboard, cross-feature eval standards.
- **Escalation.** When the shared judge substrate needs a change (new judge shape, new calibration methodology, new eval-platform capability), the peer role owns the change. When the applied engineer needs a feature-specific fixture, the applied engineer owns it.
- **Revisit.** Quarterly, or on any eval-platform migration.

### Contract 3 — `model-evaluation-engineer`: benchmark and statistical rigour

- **Scope handed off.** Public-benchmark selection and interpretation, statistical rigour on A/B and eval comparisons (confidence intervals, multiple-comparison correction, statistical power for the feature's sample size), leaderboard-driven failure-mode diagnosis from [mod-101 chapter 05](../mod-101-applied-ai-foundations-and-feature-design/05-leaderboard-driven-failure-modes.md). Variance reduction (CUPED), sequential testing, heterogeneous treatment effects. The applied engineer's job is the *interface* — clean assignment, exposure, metric emission; the peer role's job is the *analysis*.
- **Applied-engineer inputs.** Clean A/B interface (per [mod-111 chapter 03](../mod-111-shipping-experimenting-and-rollbacks/03-ab-experiment-interface-to-the-experimentation-platform.md)), primary and guardrail metric definitions, exposure logging with no SRM, dataset access agreements.
- **Peer-role outputs.** Statistical analysis of the A/B result — point estimate, confidence interval, statistical power, guardrail-metric reads. Written interpretation for the PM. Recommendation on ship / hold / iterate.
- **Escalation.** When a metric definition is contested or a novel statistical shape (interaction, HTE) is needed, the peer role decides. When the applied engineer's interface has an SRM or an exposure bug, the peer role flags and the applied engineer fixes.
- **Revisit.** Per major-experiment, and quarterly for the methodology.

### Contract 4 — `ai-risk-engineer`: harm models and red-team data

- **Scope handed off.** Harm-model construction (what could this surface enable, and for whom), red-team programme design and execution, adversarial dataset curation, org-wide risk-register maintenance, OWASP LLM / Agentic Top 10 mapping methodology at breadth.
- **Applied-engineer inputs.** The feature's charter (in-scope harms, out-of-scope harms with rationale, from [chapter 01](01-feature-charter-with-kill-criteria.md)), the safety policy (from [mod-107 chapter 05](../mod-107-guardrails-and-safety-on-the-surface/05-refusal-calibration-and-owasp-mapping.md)), the OWASP mapping runbook, the feature's guardrail configuration.
- **Peer-role outputs.** Quarterly red-team run for the feature, findings landed in the mapping runbook with severity and remediation owner, updates to the harm model, sign-off on the safety-scope section of the charter.
- **Escalation.** When a red-team finding is P0/P1, the peer role coordinates with the applied engineer on the incident-shaped mitigation (see [chapter 02](02-incident-to-fixture-loop.md)); the finding threads through the incident-to-fixture loop.
- **Revisit.** Quarterly for the red-team cadence, on any charter-scope change, on any P0/P1 finding.

### Contract 5 — `ai-infra-mlops` / `ai-infra-ml-platform`: deploy and platform

- **Scope handed off.** The deployment substrate — the k8s / Nomad / serverless runtime the feature runs on, CI/CD pipelines, secrets management, container base images, cluster capacity planning, the shared feature-flag / prompt-registry / eval-platform infrastructure at *infrastructure* altitude. The applied engineer owns the feature's code and config; the peer role owns the platform that hosts them.
- **Applied-engineer inputs.** The feature's deployment manifest (image, resource requests, secrets required, config sources), the feature's traffic model, the feature's cadence of change.
- **Peer-role outputs.** Cluster capacity, deploy pipeline reliability, secrets-rotation SLAs, base-image CVE patches, incident-response for platform-level failures.
- **Escalation.** When the platform-level failure is limiting the feature, the peer role owns the mitigation. When a feature-level deploy is failing due to feature config, the applied engineer owns the fix.
- **Revisit.** Semi-annually, or on any platform-level incident affecting the feature.

The peer-role naming varies across orgs — sometimes `mlops`, sometimes `ml-platform`, sometimes split into `platform` (shared infra) and `mlops` (ML-specific). The contract is with whoever plays the role; the shape is the same.

### Contract 6 — `ai-infra-performance`: serving performance

- **Scope handed off.** Serving-tier performance engineering — GPU utilisation, batching policy on self-hosted models, autoscaler tuning, quantisation trade-offs, KV-cache tuning, vLLM / TGI / SGLang deployment optimisation. Applies primarily when the feature uses self-hosted inference (see [mod-108 chapter 03](../mod-108-model-routing-and-provider-selection/03-self-hosted-vs-managed-with-vllm-baseline.md)). For fully hosted (provider API) features, this contract is often thin or absent.
- **Applied-engineer inputs.** Traffic model (RPS, token distribution, batch-eligibility rate), latency SLOs (from [mod-109 chapter 01](../mod-109-cost-latency-slos-for-llm-features/01-latency-vocabulary-and-per-surface-slos.md)), cost budget, tolerance for quality regression (from any quantisation).
- **Peer-role outputs.** Serving-tier configuration meeting the SLOs, capacity plan for the traffic model, a documented quality-vs-throughput frontier for the tuning choices.
- **Escalation.** When the SLO cannot be met at the cost budget, the peer role flags the trade-off and the applied engineer takes it to the PM (from the charter's cost-budget field). When the applied engineer's traffic model shifts, the applied engineer flags and the peer role re-plans.
- **Revisit.** On any material traffic-model shift, on any provider / hardware change.

### Contract 7 — `ai-infra-security` / `application-security-engineer`: deep security

- **Scope handed off.** Authentication and authorisation architecture, secrets management, dependency hygiene at depth (SCA, SBOM), network posture (VPC, egress control), agent-specific security depth (MCP-server security review, tool-credential-rotation programme, sandboxed-tool-execution design). The applied engineer's Guard 4 (auth boundary) from [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/README.md) is downstream of this role's design; the applied engineer implements it, the peer role designs it.
- **Applied-engineer inputs.** The feature's threat model at surface altitude, the tool catalogue with per-tool auth requirements, the data classes handled, the third-party services touched (gateway, eval platform, MCP servers).
- **Peer-role outputs.** Security review sign-off before GA, the security-hardened tool-execution substrate, MCP-server allowlists, dependency-vulnerability escalation.
- **Escalation.** When a security-review finding blocks GA, the peer role owns the finding's resolution; the applied engineer implements the fix. When a new tool is added post-GA, the applied engineer files a review with the peer role before the tool ships.
- **Revisit.** Annually, on GA, and on any tool-catalogue expansion.

## The applied engineer's own scope (what does *not* delegate)

Naming the boundaries is only half the value; naming what *stays* with the applied engineer is the other half. The applied engineer explicitly owns:

- **The feature's charter, one-pager, and feature card.** No peer role can compose these; they are the applied engineer's synthesis.
- **The prompt registry, prompt versioning, and prompt-level rollout discipline.** From [mod-102 chapter 05](../mod-102-production-prompting-and-structured-outputs/05-prompts-as-versioned-artefacts.md) and [mod-111 chapter 04](../mod-111-shipping-experimenting-and-rollbacks/04-hot-rollback-of-a-bad-prompt.md).
- **The feature-level eval fixtures and the incident-to-fixture loop.** The peer `ai-eval-engineer` owns the *program*; the applied engineer owns the *feature-level fixtures* including the regression fixtures from incidents.
- **The feature's guardrail configuration, safety policy, and OWASP mapping runbook rows.** The peer `ai-risk-engineer` owns the harm model and the red-team; the applied engineer owns the surface's guardrail wiring and the mapping-runbook entries.
- **The feature's cost accounting, CPSO, and cost-budget adherence.** From [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).
- **The feature's on-call runbook, incident response, and post-incident three-artefact close-out.** From [mod-110 chapter 05](../mod-110-observability-feedback-and-incidents/05-per-feature-status-page-and-on-call-runbook.md) and [chapter 02](02-incident-to-fixture-loop.md).
- **The flag layer, canary plan, and rollback SLA for the feature.** From [mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md).
- **The delegation contracts themselves.** The applied engineer authors the contract, negotiates it with the peer, and refreshes it on cadence. The peer role is the counter-party, not the author.

Anything in this list that the applied engineer tries to hand off is a scope violation of the *role*, not just the delegation contract. The applied engineer is the person who lives at the surface where a feature meets a user, and no peer role can do that from a lower altitude.

## The pathological patterns

Two failure modes recur in practice.

**"I thought you had it"** — the applied engineer assumes the peer role is covering something; the peer role assumes the applied engineer is. Neither is. At incident time the gap surfaces. Written contracts prevent this; the *absence* of a contract for a boundary is the diagnostic. If neither side has an artefact naming ownership, ownership is undefined.

**"That's not my job"** — the peer role rejects a hand-off the applied engineer expected them to accept, or vice versa. Almost always because the scope statement in the contract is vague ("safety") rather than bounded ("adversarial red-team programme for this surface, quarterly cadence"). Bounded scope statements make the disagreement land on a concrete phrase that can be revised.

Both failure modes are resolved by the same discipline: **name the boundary, write it down, sign it, revisit it.** The contract is not bureaucratic overhead; it is what lets a small applied-AI team plug into a large org's specialist capacity without dropping either the surface or the depth.

## The delegation-contract inventory as a check

At any point the applied engineer can list the contracts for the feature — one file per peer role — and read the list. If the list contains a role whose contract has not been signed, that peer role's scope is undelegated and the applied engineer is currently on the hook for it. If the list is missing a role that the substrate depends on (e.g., a feature that uses self-hosted inference but has no `ai-infra-performance` contract), the delegation graph has a hole.

The inventory is one of the artefacts the feature card ([chapter 04](04-per-feature-model-card-slice.md)) cites: section 9 (known limitations) enumerates the delegation dependencies, and the reviewer can cross-reference against the actual contracts on file. A feature card that lists a peer-role dependency with no matching contract on file is a governance-review defect.

## Anti-patterns

- **Delegation by shrug.** "Someone in security should look at this before we ship." No name, no scope, no cadence. Everyone assumes it's happening; no one is doing it. Name the role, write the contract.
- **Contracts written but never re-signed.** Last year's contract described a different feature. The peer role has moved to a different team. The applied engineer has drifted the surface. Revisit cadence.
- **Verbal agreements at incident time.** The on-call and the peer-role responder negotiate ownership at 2 AM. Both remember it differently at the postmortem. Contracts are pre-negotiated.
- **The applied engineer authors the peer role's output.** The applied engineer red-teams their own feature "to save time," or runs their own A/B statistical analysis. The peer role's contribution is depth *plus independence*; self-review sacrifices both.
- **Handing off the surface.** The applied engineer routes user-visible incidents to the platform team. The platform team is not equipped to reason about a support-refund-composer's user; the applied engineer is. The surface stays with the surface owner.
- **Scope statements that read "safety" or "quality."** Both are too broad to delegate. Bounded scope statements ("red-team programme, quarterly, this surface, MCP-boundary focus") are what the peer role can sign onto.
- **The contract is a Slack DM.** Cannot be re-read at a governance review or an incident retro. Markdown in the repo.
- **Peer role has never seen the contract.** Written in isolation, never negotiated. The peer role is not bound by a contract they did not sign.

## Summary

- The applied-AI engineer is at the centre of a delegation graph with seven peer roles: **`agentic-ai-engineer`** (framework depth), **`ai-eval-engineer`** (program-level eval), **`model-evaluation-engineer`** (benchmark and statistical rigour), **`ai-risk-engineer`** (harm models and red-team data), **`ai-infra-mlops` / `ai-infra-ml-platform`** (deploy and platform), **`ai-infra-performance`** (serving performance), **`ai-infra-security`** (deep security).
- Each delegation is a **written contract** with six fields: peer role, scope handed off (bounded, not vague), applied-engineer inputs, peer-role outputs, escalation and revisit, signed.
- Contracts live in the repo at `docs/features/<feature>/delegations/<peer-role>.md`, versioned, signed, and re-signed on cadence.
- **What does not delegate**: the charter, the one-pager, the feature card, the prompt registry, the feature-level fixtures, the incident-to-fixture loop, the guardrail configuration, the cost accounting, the on-call runbook, the flag layer, and the delegation contracts themselves.
- Two failure modes recur: **"I thought you had it"** (absence of a contract) and **"That's not my job"** (vague scope). Both are resolved by naming the boundary, writing it, signing it, revisiting it.
- The **delegation-contract inventory** is a check: any peer-role dependency the feature card cites should have a signed contract on file. A missing contract is a governance defect.

[Chapter 06](06-governance-mapping.md) turns from "who owes whom what within the org" to "how the feature fits the external frameworks the org is held to" — NIST AI RMF Map / Measure / Manage, the EU AI Act transparency obligations that apply, and ISO/IEC 42001 controls, with the applied engineer's contribution named and the deep governance work handed off.
