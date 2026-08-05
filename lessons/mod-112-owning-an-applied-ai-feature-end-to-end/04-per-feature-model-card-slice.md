# The Per-Feature Feature-Card Slice

## Why a feature card, not just a model card

Model cards were introduced by Mitchell et al. in 2019 as a way to document the intended use, evaluation, and limitations of a *model*.<!-- needs-research: verify the canonical Model Cards for Model Reporting citation (Mitchell, Wu, Zaldivar, Barnes, Vasserman, Hutchinson, Spitzer, Raji, Gebru — FAT* 2019) and its DOI. --> Since then the industry has generalised the pattern: dataset cards, system cards, use-case cards, and various governance-adjacent variants. The common thread is a *structured evidence document* about an AI artefact, at a specific altitude, aimed at a specific reviewer.

For the applied-AI engineer, the artefact is *the shipped feature* — not the underlying model, not the platform, not the org. The reviewers are internal governance-family peer roles (`ai-governance-analyst`, `ai-evaluation-engineer`, `ai-risk-engineer`), an internal AI risk committee, and — if the feature falls under EU AI Act obligations — potentially an external auditor. The card those reviewers need is narrower than a full model card (the model card belongs to whoever trained the model) and broader than a spec (a spec does not compose evidence).

This chapter names the artefact as a **feature-card slice**: the applied engineer's contribution to the org's overall governance evidence stack. "Slice" because it is deliberately partial — the full model card, the full system card, and the full risk register are peer-role deliverables; the feature card is the *feature-level* projection that lets the peer roles do their jobs without re-collecting evidence from telemetry.

The slice satisfies three simultaneous consumers, and its shape is chosen to serve all three:

- **The internal review committee** — the group that approves a feature for GA or for expansion. Needs the "what did we build, what did we test, what could break, what did we choose *not* to defend against" summary.
- **The Governance-family peer roles** — `ai-governance-analyst` composes org-wide governance reports across features; `ai-evaluation-engineer` composes the eval program's aggregate view. Both need a machine-consumable slice they can join without re-interviewing the applied engineer.
- **The future incident retro / audit reviewer** — six months or three years from now, someone asks "what did we know about this feature when we shipped it?" The card is the answer.

## The nine sections

A working feature card is longer than the charter and one-pager — call it a 3–6 page structured document — but every section is one the applied engineer can compose from artefacts that already exist in the mod-101 through mod-111 substrates. The card is *composed*, not *authored*, if the substrates are in place.

### 1. Feature identity

- **Feature name and version.** `support-refund-composer v1.0`.
- **Surface.** Where the feature is embedded (helpdesk composer, chat panel, inline completion, IDE plugin, etc.). Named user, named seat.
- **Charter reference.** Link to the signed charter (from [chapter 01](01-feature-charter-with-kill-criteria.md)) with version.
- **Architecture reference.** Link to the signed one-pager (from [mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md)).
- **Owner and reviewers.** Applied-AI engineer, PM, governance owner. Same three roles as the charter's signature block.

### 2. Intended use and non-goals

Composed directly from the charter — the user problem, the intended use, and the explicit non-goals. This section is a citation of the charter with its version pin; if the charter has moved and the card has not been re-composed, the mismatch is a defect.

The one addition beyond the charter: **known misuse patterns**. Categories where the feature *could* be used in a way not intended, with the mitigation in place. "Users may attempt to use the refund composer to request a refund policy explanation for a different customer — the auth-boundary guard from [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md) blocks cross-tenant lookup."

### 3. System architecture

Composed from the one-pager plus the build-vs-buy trade-off notes from [chapter 03](03-build-vs-buy-across-three-surfaces.md):

- **Model(s).** Primary model, fallback model, judge model, embedding model — each with the dated version pin. `gpt-4o-2024-08-06`, not `gpt-4o`.
- **Provider(s) and region(s).** Where the traffic goes.
- **Gateway.** Which gateway, deployment shape, where it runs.
- **Eval platform.** Which platform, what fixtures live where.
- **Agent SDK / orchestration.** Framework and version.
- **Retrieval stack.** Vector DB, embedding model, index version, freshness SLO. From [mod-103](../mod-103-rag-integration-in-product/README.md).
- **Guardrails.** The layers from [mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md) that fire on this feature, in order.

Diagram optional but useful — one box per component, arrows show data flow, residency-relevant boundaries called out.

### 4. Evaluation evidence

The evidence *from the eval substrate* — the applied engineer does not run new evals to compose this card; the card summarises the ones that have run.

- **Offline eval set.** Fixture count, coverage breakdown (task-success fixtures, safety fixtures, refusal-calibration fixtures, regression fixtures per [chapter 02](02-incident-to-fixture-loop.md)). Link to the CI job that runs them.
- **Offline eval results.** Score-per-metric on the current shipped version, with the pre-registered threshold and the delta from the previous version. From [mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md).
- **Online eval loop.** Sampling rate, judge version pin, judge calibration date, current judge score on the sampled traffic. From [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md).
- **A/B experiment results.** If the feature was rolled out through an A/B, the primary metric result and the guardrail metric results. From [mod-111 chapter 03](../mod-111-shipping-experimenting-and-rollbacks/03-ab-experiment-interface-to-the-experimentation-platform.md).
- **Judge drift status.** Current calibration vs baseline, last drift alert, action taken.

The eval section is *evidence*, not *claims*. A statement of the form "the feature achieves 92% task-success" without a link to a specific CI run against a specific fixture set at a specific commit is not evidence and does not belong in the card.

### 5. Safety evidence

Composed from [mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md) artefacts:

- **Safety policy version.** Pin. From [mod-107 chapter 05](../mod-107-guardrails-and-safety-on-the-surface/05-refusal-calibration-and-owasp-mapping.md).
- **Guardrail FP/FN report.** Current numbers, delta from previous version. From [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md).
- **Over-refusal eval results.** Current over-refusal rate on the labelled fixture set, per category and aggregate.
- **Red-team findings.** Summary of the last red-team run (delegated to `ai-risk-engineer`, per [chapter 05](05-delegation-contracts-with-peer-specialists.md)) — count of findings, severity distribution, remediation status.
- **OWASP mapping.** Current row count in the mapping runbook, plus a summary table showing which categories have unremediated findings.
- **PII / data handling.** Which PII classes flow through the feature, which are scrubbed inbound, which are scrubbed outbound, retention on trace logs and fixture material.

The section is honest about what has *not* been tested. A feature without a red-team finding is not a feature that is safe; it is a feature that has not been red-teamed. Say so.

### 6. Incident evidence

Composed from the incident-to-fixture loop of [chapter 02](02-incident-to-fixture-loop.md):

- **Incident count and severity distribution** for the current review window (usually the last quarter).
- **Loop close-out rate** — of incidents in the window, what percentage produced all three artefacts (fixture, alert, runbook update)?
- **Open incidents.** Anything unmitigated or mitigated-but-not-closed-out.
- **Recurring patterns.** Any incident shape that has recurred despite a fixture; any recurring class the fixture-and-alert has *prevented* since instrumented (this is the "the substrate works" evidence).

This is the section a future incident retro will most often reach for; it is the record of what the team already knew before the incident.

### 7. Cost and latency evidence

Composed from the [mod-109](../mod-109-cost-latency-slos-for-llm-features/README.md) substrate:

- **CPSO.** Current cost-per-successful-outcome, target from the charter, delta.
- **Monthly cost.** Actual monthly cost, budget from the charter, delta. Pricing version pin.
- **Latency SLOs.** TTFT p95, e2e p95, tokens-per-second — each with target and current. From [mod-109 chapter 01](../mod-109-cost-latency-slos-for-llm-features/01-latency-vocabulary-and-per-surface-slos.md).
- **Retry / hedging behaviour.** Retry budget consumption, hedged-request rate. From [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md).

If any number is significantly out of policy, the section names it — the card is evidence, not marketing. A card that says "cost is 2× budget" is a card that is doing its job.

### 8. Human oversight and intervention

- **Human-in-the-loop points.** Where in the feature a human is required to review, approve, or override. From [mod-105 chapter 03](../mod-105-shipping-product-agents/03-human-in-the-loop-and-steering-ux.md).
- **Escalation contract.** Which categories route to a human, on what SLA, with what context.
- **Kill switch.** Where the kill switch is, how it fires, and when it was last exercised (in staging or production). From [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/README.md) and [mod-111 chapter 04](../mod-111-shipping-experimenting-and-rollbacks/04-hot-rollback-of-a-bad-prompt.md).
- **User-facing controls.** What the user can do to override, correct, or opt out of the feature.
- **Transparency affordances.** How the user knows AI is involved ("Drafted by AI" label, citation panel, "why this answer" affordance). Bridges to the EU AI Act transparency obligations in [chapter 06](06-governance-mapping.md).

### 9. Known limitations

The section that costs the most political capital to write and has the most value at review:

- **Populations the feature is not calibrated for.** Locales not in the eval set, tenant sizes not represented in the fixtures, user roles the feature has not seen in production.
- **Failure modes the eval set does not cover.** From the pre-mortem exercise in [chapter 02](02-incident-to-fixture-loop.md) — which pre-mortem shapes are not fixture-covered yet?
- **Known drift risks.** Provider alias-swap, judge drift, retrieval-corpus staleness, pricing-version drift.
- **Non-goals reiterated.** From the charter, so a reviewer skimming this section alone gets the boundary.
- **Delegation dependencies.** What the feature relies on peer roles delivering (from [chapter 05](05-delegation-contracts-with-peer-specialists.md)) that if under-delivered is a known limitation.

The pull to soft-pedal this section is strong. Resist it — a card that reads "no known limitations" is a card no reviewer trusts.

## The composition discipline

The nine sections total ~3-6 pages when composed for a mature feature. Composing them from scratch would take a week. Composing them from artefacts that already exist takes an afternoon *if* those artefacts exist.

The applied engineer's job is not to *write* the card from scratch; it is to build the feature so the card is composable. Every claim in every section maps to a link to an existing artefact:

- Charter and one-pager → sections 1, 2.
- Trade-off notes → section 3.
- Eval fixtures, CI runs, judge calibration reports → section 4.
- Safety policy, FP/FN reports, OWASP mapping runbook → section 5.
- Incident bucket, close-out tickets, fixture folder → section 6.
- Cost ledger, latency dashboards → section 7.
- Runbook, kill-switch config, HITL primitives → section 8.
- Pre-mortem doc, delegation contracts → section 9.

A feature card that requires the applied engineer to *interview themselves* to fill it in is a symptom of a substrate gap. The gap is what needs to be fixed; the card is the diagnostic.

## Format: single Markdown file, machine-consumable

The card is a single Markdown file in the repo, at a path-stable location: `docs/features/<feature-name>/feature-card-v<n>.md`. Each version is a new file, not a git-mutation of the previous file, so the review history is intact.

For the Governance-family peer roles to compose feature cards across many features, the card benefits from a small **YAML front-matter** block that machine-summarises the key fields. The narrative sections are Markdown; the front matter is a summary index the aggregator reads.

```yaml
---
feature: support-refund-composer
version: 1.0
card_version: 1.0
signed: 2026-08-04
next_review: 2026-11-04
owner: applied-ai-engineer:jferguson
pm: pm:mchang
governance_reviewer: ai-governance-analyst:oluwe
charter: docs/features/support-refund-composer/charter-v0.3.md
one_pager: docs/features/support-refund-composer/architecture-v1.0.md

models:
  primary: gpt-4o-2024-08-06
  fallback: claude-sonnet-4-5-20250929   # NB: pin dated variants; check current SKUs
  judge: gpt-4o-2024-08-06
  embedding: text-embedding-3-large

evals:
  offline_fixture_count: 217
  regression_fixture_count: 34
  task_success_score: 0.923
  task_success_target: 0.90
  over_refusal_rate: 0.021
  over_refusal_target: "<= 0.03"
  judge_calibration_date: 2026-07-28
  judge_calibration_score: 0.912

safety:
  policy_version: v2026.08.1
  owasp_mapping_findings: 6
  owasp_mapping_open: 1
  red_team_last_run: 2026-06-14
  red_team_findings_total: 12
  red_team_findings_open: 0

cost:
  cpso: 0.037
  cpso_target: 0.04
  monthly_cost: 15400
  monthly_ceiling: 18000
  pricing_baseline: 2026-07

incidents_last_quarter:
  total: 3
  p0: 0
  p1: 0
  p2: 2
  p3: 1
  loop_close_out_rate: 1.0

governance:
  nist_ai_rmf_functions: [map, measure, manage]
  eu_ai_act_transparency: applicable   # see chapter 06
  iso_42001_controls_hit: [A.5.2, A.6.1, A.7.3, A.9.2]   # see chapter 06
---
```

The front matter is where the `ai-governance-analyst` peer role runs their org-wide aggregation query; the Markdown is where the human reviewers read the narrative. Both are load-bearing; neither substitutes for the other.

## Cadence and re-composition

The feature card is composed:

- At **GA** — the initial card. Signed by all three roles.
- **Quarterly** — a refresh with updated numbers, updated incident evidence, updated safety and cost sections. Signed again.
- **On any significant change** — a new primary model, a new provider, a scope expansion (a former non-goal is now a goal), or the resolution of a known limitation. Bumps the card version.
- **On any P0 or P1 incident** — the incident-to-fixture loop closes with an update to the "known limitations" and "incident evidence" sections. Card version bumps.

A card that has not been re-composed in six months is a stale card. A stale card is a red flag at governance review — it means the feature has drifted and no one has looked at the drift.

## How the card lands with the peer roles

- **`ai-governance-analyst`** ingests the YAML front matter across all features into a governance dashboard. The card's front-matter schema is a shared contract; the applied engineer commits to keeping the field names and value shapes stable across cards, and the analyst commits to publishing a schema change with lead time. See [chapter 05](05-delegation-contracts-with-peer-specialists.md) for the delegation contract.
- **`ai-evaluation-engineer`** ingests the `evals:` block and cross-references against the eval-platform's aggregate view. Discrepancies (card claims 0.923, platform shows 0.905) surface as a defect — usually the card was composed against a stale run.
- **`ai-risk-engineer`** ingests the `safety:` block plus the OWASP mapping runbook. The card is one input to the org-wide risk register; the applied engineer is not the register's author but is the source-of-truth for feature-level findings.
- **Internal review committee** reads the Markdown. The narrative should stand alone for the committee's session (they will not open the linked artefacts in the meeting).
- **External auditor** (if the feature is EU-AI-Act-in-scope or ISO-42001-audited) reads the Markdown and follows the links selectively. The card is not the audit evidence itself; it is the *index* to the audit evidence. See [chapter 06](06-governance-mapping.md).

## The card's relationship to system cards and model cards

- The **feature card** is *this document* — the applied engineer's slice, per shipped feature.
- The **model card** belongs to whoever trained (or, for hosted models, distributes) the underlying model. OpenAI publishes model cards / system cards for GPT-4o, GPT-5, o1, etc.; Anthropic publishes system cards / model cards for Claude models. The applied engineer's job is to *cite* these, not to author them.<!-- needs-research: verify the specific naming — OpenAI calls their newer artefacts "system cards" or "GPT-4 system card"; Anthropic calls theirs "Claude ... system card"; the terminology has shifted. -->
- The **system card** is at the org / product-line altitude — a document about the composed system that hosts many features. Usually authored by the `ai-governance-analyst` or a security-team lead, drawing on the feature cards.

The applied engineer's card is the one that most needs to be a habit, because there are many features and the org has one system card. The scale is different; the shape is chosen to match.

## Anti-patterns

- **The card is authored fresh at each review.** Every quarter a week evaporates writing a card the substrates could compose in an afternoon. Fix the substrate, not the writing.
- **The card cites nothing.** Claims of scores, safety-testing results, and cost numbers without links to the source artefact. Auditors cannot verify; incidents cannot re-derive. Every claim has a link.
- **Known limitations = "None known."** No card with real content has zero known limitations. A card that claims none is a card that has not been reviewed by anyone who would push back.
- **The card is a Notion page.** Not versioned, not diffable, not survivable to a re-org. Markdown in the repo, path-stable.
- **The card does not cite the charter.** The charter is the design substrate; a card that reinvents the "what is this feature" section without linking to the charter is either replacing it (wrong) or duplicating it (wasteful).
- **Front-matter schema drifts silently.** The `ai-governance-analyst` aggregator breaks; the analyst chases down each feature owner. The schema is a shared contract with a version pin.
- **Card version = software version.** Different cadences. Card version bumps on card content changes; software version bumps on feature changes. Track separately.
- **Card is only reviewed at launch.** The next review is a discovery project because nothing has been maintained. Quarterly cadence keeps the drift small.

## Summary

- The **feature-card slice** is the applied engineer's contribution to the org's overall AI-governance evidence stack — a 3–6 page structured Markdown document, per shipped feature, at the *feature* altitude.
- It serves three consumers: the **internal review committee**, the **Governance-family peer roles**, and the **future incident retro / audit reviewer**.
- Nine sections: **identity, intended use / non-goals, system architecture, evaluation evidence, safety evidence, incident evidence, cost / latency evidence, human oversight / intervention, known limitations**. Every claim links to an existing artefact.
- The card is **composed, not authored** — the applied engineer's job is to build the feature so every section maps to an artefact the mod-101-through-mod-111 substrates already produce.
- **YAML front matter** is the machine-consumable summary the `ai-governance-analyst` aggregator reads; the Markdown body is the human-consumable narrative. Both.
- **Cadence: GA, quarterly, on significant change, on P0/P1 incident.** A card unrefreshed for six months is stale and a governance-review red flag.
- The **model card** and **system card** are peer-role artefacts the feature card *cites*; the applied engineer is not their author. Do not conflate.
- Known limitations are honest. A card that claims none is not credible.

[Chapter 05](05-delegation-contracts-with-peer-specialists.md) turns from "what evidence does the applied engineer produce" to "what does the applied engineer explicitly hand off, to whom, on what terms" — delegation contracts to the peer specialists whose depth the applied engineer plugs into.
