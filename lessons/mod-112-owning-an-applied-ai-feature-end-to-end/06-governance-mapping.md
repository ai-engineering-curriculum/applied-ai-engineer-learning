# Governance Mapping — NIST AI RMF, EU AI Act, ISO/IEC 42001

## Why the applied engineer maps at all

The three frameworks in this chapter's title are the ones the applied-AI engineer will most frequently see in review documents, procurement questionnaires, and audit follow-ups. None of them is the applied engineer's *primary* discipline — that belongs to the Governance-family roles (`ai-governance-analyst`, `ai-evaluation-engineer`, `ai-risk-engineer`) and to the org's legal / compliance function. What the applied engineer owes those roles is a **mapping**: for each framework the org is subject to, which of the framework's specific controls or obligations does *this feature* touch, and where in the feature's substrate is the evidence?

Without the mapping, the peer roles are re-interviewing the applied engineer every audit cycle. With it, the peer role reads the feature card ([chapter 04](04-per-feature-model-card-slice.md)), follows the mapping, and traces evidence back to the substrates the applied engineer already runs. The mapping is the discipline that keeps governance and engineering in the same repo instead of on different platforms with different truths.

This chapter does *not* teach the applied engineer to *own* the frameworks. Ownership of "does the org's overall AI programme comply with the EU AI Act" belongs to the legal / compliance leadership, informed by the Governance-family peers. What the applied engineer owns is the **feature-level slice** of each framework — which paragraphs of NIST AI RMF, which articles of the EU AI Act, and which Annex A controls of ISO/IEC 42001 does *this feature* actually touch.

The chapter is deliberately shorter than the standards themselves. The intent is to give the applied engineer enough footing to compose the mapping and hand it off, not to reproduce the frameworks. Primary sources are cited; the applied engineer reads them before signing anything material.

## NIST AI Risk Management Framework — Map / Measure / Manage

The NIST AI Risk Management Framework (AI RMF 1.0, published January 2023) is a voluntary framework for managing risks in AI systems.<!-- needs-research: verify the current NIST AI RMF version at publication time; a 1.1 or Generative AI Profile revision may have shipped. The Generative AI Profile (NIST AI 600-1) was released in 2024 and is directly relevant to LLM features. --> It structures AI risk work around four functions: **Govern**, **Map**, **Measure**, **Manage**. Govern is the org-wide function; the applied engineer's contribution runs almost entirely through the other three.

- **Map** — establishing the *context* the AI system operates in: who uses it, for what purpose, with what data, subject to what constraints, what the impacts could be.
- **Measure** — analysing, assessing, benchmarking, and monitoring the risks. Metrics, evaluations, and their interpretation.
- **Manage** — acting on the assessed risks: prioritising, allocating resources, treating (mitigate / transfer / avoid / accept), and documenting the decisions.

The applied engineer's mapping to Map / Measure / Manage is direct:

### Map — the applied engineer's contribution

The Map function asks *what is the context this feature operates in?* The applied engineer's already-written artefacts cover most of it:

- **Purpose and users** — from the charter's user problem and success metric ([chapter 01](01-feature-charter-with-kill-criteria.md)).
- **Non-goals and out-of-scope harms** — from the charter's non-goals and safety scope.
- **Data flow and residency** — from the one-pager's Section 5 data-flow paragraph ([mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md)) and the trade-off notes' residency review.
- **Stakeholders** — from the charter's signature block (PM, engineer, governance owner) plus the delegation contracts ([chapter 05](05-delegation-contracts-with-peer-specialists.md)) naming peer-role stakeholders.
- **Foreseeable impacts** — from the safety scope's in-scope, handed-off, and out-of-scope harms lists.

The applied engineer's Map deliverable is not a separate document; it is the citation table showing which sections of the charter / one-pager / feature card satisfy each Map subcategory. The `ai-governance-analyst` composes the org-wide Map view by aggregating these citation tables.

### Measure — the applied engineer's contribution

The Measure function asks *what are we measuring, and does the measurement work?* The applied engineer's eval and telemetry substrates do the measuring:

- **Evaluation** — offline eval fixtures ([mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md)), online eval loop ([mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md)), judge calibration.
- **Monitoring** — the telemetry substrate ([mod-110](../mod-110-observability-feedback-and-incidents/README.md)), the per-feature dashboards ([mod-106 chapter 05](../mod-106-feature-side-evaluation/05-cross-persona-feature-dashboards.md)), the burn-rate alerts.
- **Testing for identified risks** — the FP/FN report ([mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md)), the injection-defence eval, the red-team programme output (delegated per [chapter 05](05-delegation-contracts-with-peer-specialists.md)).
- **Assessment of measurement quality** — judge-vs-ground-truth calibration, judge-drift monitoring.

The applied engineer's Measure deliverable is the feature-card sections 4 (evaluation evidence) and 5 (safety evidence). The peer `ai-evaluation-engineer` composes the program-level Measure view; the applied engineer feeds it.

### Manage — the applied engineer's contribution

The Manage function asks *how are we acting on the risks we've measured?* The applied engineer's rollout and incident-response substrates are where the acting happens:

- **Risk prioritisation** — the charter's kill criteria and the safety scope's rank ordering.
- **Risk treatment** — the guardrails ([mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md)), the flag layer and canary regime ([mod-111 chapters 01–02](../mod-111-shipping-experimenting-and-rollbacks/README.md)), the routing fallback ([mod-108](../mod-108-model-routing-and-provider-selection/README.md)).
- **Response** — the incident-to-fixture loop ([chapter 02](02-incident-to-fixture-loop.md)), the per-feature runbook, the kill switch, the rollback SLA.
- **Documentation** — the feature card, the trade-off notes, the delegation contracts.

The applied engineer's Manage deliverable is the feature-card sections 6 (incident evidence), 8 (human oversight), and 9 (known limitations) plus the sign-off records.

### The NIST GAI Profile

In 2024 NIST published the **Generative AI Profile (NIST AI 600-1)** — a companion document that specialises the AI RMF for generative AI systems.<!-- needs-research: verify the current NIST AI 600-1 version at publication time and its canonical citation. --> It enumerates GAI-specific risks (hallucination, confabulation, harmful content generation, IP infringement, environmental impacts, data privacy, etc.) and cross-references them to the Map / Measure / Manage functions. For an LLM-integrated feature, the Profile is the more directly-applicable document; the applied engineer maps to Profile-listed risks explicitly, not just to the four-function framework.

The applied engineer's job is to read the current Profile, identify which GAI-specific risks the feature is exposed to (usually a subset of the enumerated list), and cite the substrates where each risk is measured and managed. This is exactly the same shape as the base AI-RMF mapping — the difference is that the Profile's risk list is closer to the day-to-day failure modes the applied engineer already knows from [mod-110 chapter 04](../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md).

## EU AI Act — transparency obligations

The EU AI Act (Regulation (EU) 2024/1689) is the EU's binding legal framework for AI, adopted in 2024 with staged applicability dates.<!-- needs-research: verify the current staged-applicability dates at publication time; the Act's provisions phase in over 2024–2026 with specific dates for prohibited practices, GPAI obligations, high-risk system obligations, etc. The specific citation Regulation (EU) 2024/1689 should be confirmed. --> The Act is not a voluntary framework — for AI systems placed on the EU market or affecting EU persons, it is law, and its risk-based categorisation determines what obligations apply.

The Act's risk categories:

- **Unacceptable-risk (prohibited)** — a small list of practices (e.g., social scoring by public authorities, some biometric categorisations, some emotion recognition in specific contexts). Not applicable to most applied-AI features; if this category is a live question, the delegation to legal counsel is not optional.
- **High-risk** — systems in specified domains (biometric identification, critical infrastructure, education, employment, essential services, law enforcement, migration, administration of justice, democratic processes). Extensive obligations: conformity assessment, risk management system, data governance, technical documentation, human oversight, accuracy / robustness / cybersecurity, post-market monitoring.
- **Limited-risk (transparency)** — systems that interact with people (chatbots), generate synthetic content (generative AI outputs), or perform emotion recognition / biometric categorisation. Transparency obligations apply.
- **Minimal-risk** — everything else. No specific obligations under the Act.

The applied-AI engineer's most common encounter with the Act is the **limited-risk transparency category**, because most product-embedded LLM features fall there. The high-risk category is real and non-negligible if the feature is in a covered domain, but the majority of applied-AI features (support chat, IDE assistants, drafting tools, internal-tooling assistants) sit in the limited-risk band. If the applied engineer's feature *might* be high-risk, that determination is not one to make alone — it goes to legal counsel via the `ai-governance-analyst` peer role.

### The transparency obligations that apply to most LLM features

The Act's transparency provisions for limited-risk systems require, at a minimum:

- **Disclosure of AI interaction.** Users must be informed that they are interacting with an AI system, unless it is obvious from the context. In practice: "You're chatting with an AI assistant," a visible "AI-powered" label, or an equivalent affordance. The applied engineer's feature-card Section 8 (human oversight and intervention) documents where and how this is done.
- **Disclosure of AI-generated content.** For synthetic content (audio, video, images, text) that could be mistaken for authentic, the fact that the content is AI-generated must be disclosed in a machine-readable form (watermarking, provenance metadata). Text is a partial exception — the current text of the Act treats AI-generated text with more nuance, but the direction is toward disclosure obligations. The applied engineer's job is to know whether the feature produces content in a category subject to disclosure and to route the design question to `ai-governance-analyst`.<!-- needs-research: the Act's specific text-disclosure obligations vs image/audio/video-disclosure obligations differ; verify current interpretation and the text-specific carve-outs at publication time. Also verify how GPAI obligations (Article 50-adjacent) apply upstream. -->
- **Emotion recognition / biometric categorisation** transparency — narrower, applies to features that categorise people; not applicable to most product features.

The applied engineer's mapping deliverable is a table in the feature card (or a linked companion) with one row per applicable transparency obligation, the substrate that satisfies it, and the design affordance the user sees:

```markdown
## EU AI Act transparency mapping (v0.1)

| Obligation                       | Applies? | Design affordance                                          | Substrate reference           |
|----------------------------------|----------|------------------------------------------------------------|-------------------------------|
| Disclosure of AI interaction     | Yes      | "Drafted by AI" label on every AI-drafted reply in composer| mod-105 ch03; UI spec §4.2    |
| Disclosure of AI-generated content | Partial | Drafted content is agent-reviewed before send;
                                              the user sees the AI-drafted state; no synthetic
                                              audio / video / image content is produced.       | Feature-card §8               |
| Emotion / biometric transparency | No       | Not applicable — no emotion recognition, no biometrics.    | —                             |
```

Every row has a substrate reference. Every "not applicable" has a rationale. The `ai-governance-analyst` peer role composes the org-wide compliance view; the applied engineer's contribution is one feature's mapping.

### GPAI (general-purpose AI) obligations

The Act includes obligations on providers of general-purpose AI models (GPAI) — the foundation-model providers themselves. Those obligations are *upstream* of the applied engineer; the applied engineer is a *deployer* (a downstream user of a GPAI model). Deployer obligations exist and are lighter than provider obligations; the applied engineer's job is to know which category the feature's use falls into (usually deployer) and to consume the provider's GPAI-compliance documentation as an input to the feature card.

## ISO/IEC 42001 — AI management system

**ISO/IEC 42001:2023** is the international standard for AI management systems (AIMS).<!-- needs-research: confirm ISO/IEC 42001's publication date (2023) and current amendment status. --> It is structured like other ISO management-system standards (27001 for information security, 9001 for quality) — an AIMS is the org's management framework for its AI work, with a set of controls in Annex A.

The applied engineer is not the *owner* of the org's AIMS; that belongs to the person leading the org's ISO 42001 certification (usually a Governance-family lead or a compliance function). What the applied engineer owes the AIMS is **evidence at the feature altitude that satisfies the applicable Annex A controls**.

The Annex A controls that a feature-level applied engineer's substrates most directly touch:

- **A.5 — Policies related to AI.** The safety policy from [mod-107 chapter 05](../mod-107-guardrails-and-safety-on-the-surface/05-refusal-calibration-and-owasp-mapping.md) is the feature-level artefact that satisfies the policy control for this feature.
- **A.6 — Internal organisation.** The delegation contracts from [chapter 05](05-delegation-contracts-with-peer-specialists.md) document the internal organisation for this feature.
- **A.7 — Resources for AI systems.** The one-pager's provider / cost / compute paragraph and the FinOps cost budget from the charter.
- **A.8 — AI system impact assessment.** The Map section of the AI-RMF citation table above; the safety-scope harm inventory; the risk-treatment mapping.
- **A.9 — AI system life cycle.** The design, development, deployment, and decommissioning discipline — the flag layer and rollback SLA from [mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md), the eval-gated CI from [mod-106](../mod-106-feature-side-evaluation/README.md), the feature-card cadence.
- **A.10 — Data for AI systems.** The data-flow paragraph, the PII scrubbing, the residency envelope.

The applied engineer's mapping deliverable is a table with one row per applicable Annex A control, the substrate that provides the evidence, and the last time the substrate was validated.<!-- needs-research: verify the specific ISO/IEC 42001 Annex A control numbering and titles from the current published standard; the labels above are directional descriptions, not verbatim from the standard, and should be reconciled against the standard before external audit. -->

```markdown
## ISO/IEC 42001 Annex A mapping (v0.1)

| Control                          | Evidence substrate                                  | Last validated     |
|----------------------------------|-----------------------------------------------------|--------------------|
| A.5 — AI policies                | Safety policy v2026.08.1 (mod-107 ch05)             | 2026-07-30 review  |
| A.6 — Internal organisation      | Delegation contracts (all 7 peer roles signed)      | 2026-08-04         |
| A.7 — Resources                  | Charter cost budget; one-pager §4                   | 2026-07-14 sign-off|
| A.8 — Impact assessment          | Safety scope; NIST AI-RMF Map citation table        | 2026-08-04         |
| A.9 — Life cycle                 | Flag layer; canary regime; eval-gated CI            | Running            |
| A.10 — Data                      | One-pager §5; mod-107 ch03 scrubbing                | 2026-07-30 review  |
```

The `ai-governance-analyst` peer role composes the org-wide 42001 evidence pack from these feature-level tables. The applied engineer is not audited *personally* — the org is — but the auditor's evidence request will traverse these tables to the substrates, and a substrate that cannot produce evidence on demand is one the auditor flags.

## The mapping's home in the artefact stack

Two placement options for the mapping:

- **Inline in the feature card.** The feature card gets three new sections (or three tables in Section 5 — safety evidence — expanded). Simpler; the mapping is where the reviewer reads it.
- **Companion file.** `docs/features/<feature>/governance-mapping.md` alongside the card. Better when the org's Governance-family peers want a single-file schema across features for aggregation.

Both patterns work. The choice depends on the aggregation tooling the peer roles use; the applied engineer decides in consultation with `ai-governance-analyst`. The important property is *stable* — reviewers and auditors know where to find it, and the front matter of the feature card (from [chapter 04](04-per-feature-model-card-slice.md)) references its version.

## Cadence and re-mapping

The mapping is re-composed:

- **On feature GA** — the initial mapping. Signed by the applied engineer and the `ai-governance-analyst`.
- **Annually** — a mandatory re-review. Frameworks evolve (NIST publishes profile revisions, the EU AI Act's implementing acts are ongoing, ISO 42001 has amendment cycles).
- **On any framework revision** — a NIST Profile update, an EU AI Act implementing act, an ISO 42001 amendment. The `ai-governance-analyst` flags the trigger; the applied engineer refreshes the affected rows.
- **On any change in feature classification** — if the feature's scope expands into a domain that might change EU AI Act risk category (e.g., a low-risk feature is embedded in an HR workflow, potentially crossing into high-risk territory). Re-classify with legal counsel.

A mapping that has not been touched in a year is stale by default. A mapping that references a superseded framework version is a red flag at any external audit.

## Who reads the mapping

- **`ai-governance-analyst`** — composes the org-wide compliance view; feeds it to the compliance function.
- **`ai-risk-engineer`** — cross-references with the org-wide risk register and the OWASP mapping runbook.
- **Internal AI risk committee** — reads the mapping's evidence pointers when reviewing a feature for GA or expansion.
- **External auditor** (for ISO 42001 certification, EU AI Act conformity, or customer-driven due diligence) — traverses the mapping to the substrates. The mapping's job here is to be a *navigable index* to the evidence, not to duplicate the evidence.
- **Procurement questionnaires** (customer-facing) — the answers to "how do you handle AI-generated content?" and "what's your risk management framework?" are composed from the mapping. The applied engineer is not the questionnaire author, but a mapping that cannot answer common procurement questions is one that will be renegotiated at every deal.

## The scope of "mapping" versus "compliance"

A recurring source of confusion in review meetings: the applied engineer's *mapping* is not the same thing as the org's *compliance*.

- **The mapping** is a citation table that connects framework obligations to feature-level substrates. It is *composable* from the feature's existing artefacts.
- **Compliance** is a formal claim (usually by an accountable executive, sometimes with external audit) that the org's AI programme meets the framework's requirements. It is composed from many mappings plus additional org-wide artefacts (policies, board oversight, incident-response programme, etc.).

The applied engineer contributes to compliance by producing accurate mappings. The applied engineer does not *claim* compliance for the org — that is the compliance function's job, backed by legal counsel and (where applicable) external auditors. Overstating the applied engineer's scope here is a governance failure of its own; the applied engineer's role is to make the compliance function's job possible, not to substitute for it.

## Anti-patterns

- **Mapping the framework once, at GA, and never re-visiting.** Frameworks evolve. A mapping to AI RMF 1.0 in 2026 when NIST has published a 1.1 or a new Profile is a mapping the auditor will flag first.
- **Copying framework text verbatim into the mapping.** The mapping is a citation of *this feature*'s evidence to the framework, not a summary of the framework. If it reads like the framework itself, it is not doing its job.
- **Marking an obligation "not applicable" without a rationale.** "N/A" fails audit. "N/A — this feature does not generate synthetic audio, video, or images; text disclosure is handled by the 'Drafted by AI' label per feature-card §8" passes audit.
- **Substrate references that don't resolve.** The mapping says "see mod-107 ch05," but the safety-policy file has been renamed and the link is dead. Mapping validation catches this before the auditor does.
- **The applied engineer signing the compliance claim.** Not the applied engineer's role. Sign the mapping; do not sign the compliance.
- **The mapping missing the peer-role sign-off.** The `ai-governance-analyst` reviews and signs every mapping. An unsigned mapping is a draft.
- **High-risk classification made without legal counsel.** The EU AI Act's high-risk categorisation is a legal determination. The applied engineer flags the *possibility*; the compliance function decides.
- **The mapping is a separate system that no artefact references.** Nobody reads it. The feature card's front matter references the mapping version; the mapping references the substrates; the substrates reference the eval runs and dashboards. The graph is connected.

## Summary

- The applied-AI engineer's job on governance frameworks is **mapping**, not compliance. Mapping is the feature-level citation of framework obligations to the substrates that provide evidence; compliance is a formal claim by the org's compliance function, backed by legal counsel.
- **NIST AI RMF Map / Measure / Manage** — the applied engineer's Map contribution comes from the charter, one-pager, and safety scope; Measure from the eval and telemetry substrates; Manage from guardrails, flag layer, incident-to-fixture loop, and runbook. The **NIST GAI Profile (AI 600-1)** specialises this for LLM features and is the more directly-applicable document.
- **EU AI Act** — most product-embedded LLM features fall in the **limited-risk transparency** band. The applied engineer's obligations: disclosure of AI interaction, disclosure of AI-generated content where applicable. **High-risk classification is a legal determination** — flagged by the applied engineer, decided by the compliance function.
- **ISO/IEC 42001** — the applied engineer contributes feature-level evidence to the Annex A controls (A.5 policies, A.6 organisation, A.7 resources, A.8 impact assessment, A.9 life cycle, A.10 data). The `ai-governance-analyst` composes the org-wide evidence pack.
- Every mapping is a **citation table** — obligation, applicability, substrate reference, last validation. "N/A" entries carry a rationale. Substrate references resolve.
- The mapping lives in the feature card or in a companion file at a path-stable location. It is signed by the applied engineer and the `ai-governance-analyst`.
- **Cadence: GA, annually, on framework revision, on classification change.** A mapping unrefreshed for a year is stale.
- The mapping is read by the peer Governance-family roles, the internal AI risk committee, external auditors (where applicable), and procurement questionnaires. Its job is to be a *navigable index* to the evidence, not to duplicate the evidence.
- Framework primary sources (NIST AI RMF 1.0 and Generative AI Profile, EU AI Act Regulation (EU) 2024/1689, ISO/IEC 42001:2023) are read by the applied engineer before signing any material mapping. The Governance-family peer roles guide the interpretation; the applied engineer runs the substrate that produces the evidence.

This chapter closes mod-112. The applied engineer who has completed the module holds the artefact stack a feature needs from roadmap-line to governance-review-ready: a signed charter, a versioned incident-to-fixture loop, signed build-vs-buy trade-off notes, a composable feature card, signed delegation contracts with each peer specialist, and a governance mapping citation table. The next module in the track — or the reader's next feature — starts from this substrate.
