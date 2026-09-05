# Resources for mod-112-owning-an-applied-ai-feature-end-to-end (Owning an Applied-AI Feature End-to-End)

Primary-source references — official standards, canonical papers, provider documentation, SRE and incident-response canon, and vendor documentation for the substrate categories the chapters compare. Each entry names *why* the resource matters to the chapter it supports. Verify exact URLs, product names, and pricing at the point of use; the substrates the applied engineer selects — gateways, eval platforms, agent SDKs — iterate faster than the module's revision cadence.

## Translating a roadmap item into a feature charter (chapter 01)

### Product-management and design canon

- **Bryar & Carr — *Working Backwards: Insights, Stories, and Secrets from Inside Amazon* (2021)** — the canonical description of Amazon's PR/FAQ discipline and the "start from the user's problem, work backwards" method the six-field charter descends from. <!-- needs-research: verify the current canonical citation and publisher URL for the Working Backwards book. --> <https://www.workingbackwards.com/>
- **Amazon — leadership principles ("Customer Obsession," "Insist on the Highest Standards")** — the operating principles that pair with PR/FAQ; useful when framing the user-problem field for reviewers unfamiliar with the vocabulary. <https://www.amazon.jobs/en/principles>
- **Marty Cagan — *Inspired: How to Create Tech Products Customers Love* (2nd ed., 2017)** — the "user problem vs solution shape" distinction chapter 01's Field 1 formalises. <https://www.svpg.com/inspired-how-to-create-products-customers-love/>
- **Teresa Torres — *Continuous Discovery Habits* (2021)** — the discovery framing behind the "what does 'solved' look like *for the user*" bullet in the user-problem field. <https://www.producttalk.org/continuous-discovery-habits/>
- **John Doerr — *Measure What Matters* (2018)** — OKR-flavoured framing of "one metric, bounded, attributable" — the discipline the success-metric field imposes. <https://www.whatmatters.com/>

### Success-metric and SLO framing

- **Google SRE Book — Service Level Objectives** — the SLO / SLI / SLA distinction that the charter's success-metric field borrows for LLM features. <https://sre.google/sre-book/service-level-objectives/>
- **Andrew Chen — "The North Star Metric" (blog / *The Cold Start Problem*, 2021)** — the framing that a *single* primary metric is a feature; multiple primaries is a policy. <https://andrewchen.com/north-star-metric-growth-startup-metrics/>
- **Amplitude — "The North Star Playbook"** — a working primer on selecting and defending a single primary metric. <https://amplitude.com/books/north-star>

### Kill-criteria and pre-registration

- **Google SRE Workbook — Alerting on SLOs** — the multi-window multi-burn-rate alert pattern the charter's kill criteria borrow for "measurable from a substrate the on-call can act on." <https://sre.google/workbook/alerting-on-slos/>
- **Google SRE Workbook — Implementing SLOs** — the canonical guide to formalising thresholds a paging system can act on. <https://sre.google/workbook/implementing-slos/>
- **AllTrials / EQUATOR Network — clinical-trial pre-registration** — the discipline of pre-registering endpoints before running the trial, which the charter's "kill criteria agreed *before* the incident, not after" bullet descends from. <http://www.alltrials.net/> <https://www.equator-network.org/>
- **Center for Open Science — pre-registration primer** — accessible introduction to pre-registration as a discipline against post-hoc rationalisation. <https://www.cos.io/initiatives/prereg>

## The incident-to-fixture loop (chapter 02)

### Incident-response canon

- **Google SRE Book — Managing Incidents** — the incident-response vocabulary (roles, comms, decision protocol) chapter 02's incident lifecycle plugs into. <https://sre.google/sre-book/managing-incidents/>
- **Google SRE Book — Effective Troubleshooting** — the diagnosis discipline the "capture 5–20 traces, pin the good ones for contrast" pattern borrows from. <https://sre.google/sre-book/effective-troubleshooting/>
- **Google SRE Workbook — Postmortem Culture** — the blameless-postmortem discipline that the three-artefact close-out extends for LLM features. <https://sre.google/workbook/postmortem-culture/>
- **Google SRE Book — Emergency Response** — the "have a plan before you need it" chapter that grounds the "pin traces first, mitigate second" habit. <https://sre.google/sre-book/emergency-response/>
- **PagerDuty — Incident Response documentation** — the widely-used incident-response reference; incident commander roles, comms, and runbook link semantics. <https://response.pagerduty.com/>
- **PagerDuty — Postmortem documentation** — postmortem shape and follow-up-action discipline. <https://postmortems.pagerduty.com/>
- **Etsy — "Blameless PostMortems and a Just Culture" (John Allspaw, 2012)** — the foundational blog post on blameless postmortems. <https://www.etsy.com/codeascraft/blameless-postmortems/>

### Tracing and observability substrate

- **OpenTelemetry — Generative AI semantic conventions** — the OTel Gen-AI attribute vocabulary (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, prompt / completion events) chapter 02's trace-pinning stage relies on. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenInference — semantic conventions** — the OpenInference span vocabulary aligned with OTel Gen-AI; the primary format the Phoenix / Arize eval ecosystem consumes. <https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md>
- **OpenTelemetry Collector** — the substrate for centralising the trace stream and applying redaction pipelines. <https://opentelemetry.io/docs/collector/>

### PII redaction for fixture material

- **Microsoft Presidio** — open-source PII detection and anonymisation library; the reference scrubber chapter 02's redaction stage assumes is applied to fixture material. <https://microsoft.github.io/presidio/>
- **GDPR — Article 5 (principles relating to processing of personal data)** — the "data minimisation" and "storage limitation" bases for the redaction discipline. <https://gdpr-info.eu/art-5-gdpr/>
- **GDPR — Article 32 (security of processing)** — the legal basis for the reversible-mapping key custody discipline. <https://gdpr-info.eu/art-32-gdpr/>

### Fixture-as-regression discipline

- **Google — "Testing Overview" (Software Engineering at Google, ch. 11)** — the general software-engineering discipline of "every bug becomes a test" chapter 02 specialises for LLM features. <https://abseil.io/resources/swe-book/html/ch11.html>
- **PyTest — parametrised fixtures documentation** — the mechanical shape most Python-based eval-gated CI jobs use to load fixture files. <https://docs.pytest.org/en/stable/how-to/parametrize.html>

## Build vs buy: gateway, eval platform, agent SDK (chapter 03)

### Gateway options

- **LiteLLM — documentation** — open-source Python SDK + proxy speaking the OpenAI API shape across 100+ providers; the reference OSS gateway option. <!-- needs-research: verify LiteLLM's current provider count and the state of its managed offering at publication time. --> <https://docs.litellm.ai/>
- **LiteLLM — GitHub repository** — source, examples, provider adapter catalogue. <https://github.com/BerriAI/litellm>
- **OpenRouter — documentation** — hosted routing service with per-request auto-fallback; the reference hosted-gateway option for prototypes. <!-- needs-research: verify OpenRouter's fallback semantics and current pricing model at publication time. --> <https://openrouter.ai/docs>
- **Portkey — documentation** — commercial gateway with observability, guardrail hooks, per-tenant budgets, and audit logs. <!-- needs-research: verify Portkey's current feature set and pricing structure at publication time. --> <https://portkey.ai/docs>
- **AWS Bedrock — documentation** — AWS-hosted access to a curated multi-provider model catalogue plus routing / guardrails / knowledge-base services. <https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html>
- **AWS Bedrock — supported foundation models** — the current model catalogue for the AWS-native gateway option. <!-- needs-research: verify the current provider list at publication time — the catalogue has expanded through 2024–2026. --> <https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html>
- **Google Vertex AI Model Garden** — GCP's model catalogue (Gemini, Claude on Vertex, Llama, Mistral, and others). <!-- needs-research: verify the Model Garden's third-party model list at publication time. --> <https://cloud.google.com/model-garden>
- **Microsoft Azure AI Foundry — documentation** — Microsoft's umbrella-branded evolution of Azure AI Studio + Azure OpenAI Service, covering OpenAI plus third-party models plus evaluation. <!-- needs-research: verify the current Azure AI Foundry naming and product boundaries at publication time. --> <https://learn.microsoft.com/en-us/azure/ai-foundry/>
- **OpenAI — API reference (client shape reference)** — the OpenAI API shape every serious gateway speaks; the "call the gateway through an OpenAI-shaped client" exit-door discipline. <https://platform.openai.com/docs/api-reference/introduction>

### Eval platform options

- **Braintrust — documentation** — commercial evaluation platform (fixtures, judge library, dataset management, CI integration). <!-- needs-research: verify Braintrust's current product boundaries at publication time. --> <https://www.braintrust.dev/docs>
- **Weights & Biases Weave — documentation** — W&B's LLM-focused subproduct (trace logging, evals, dataset management). <!-- needs-research: verify Weave's positioning vs core W&B experiment tracking at publication time. --> <https://weave-docs.wandb.ai/>
- **Langfuse — documentation** — open-source (self-hostable) LLM tracing + eval + prompt management, with a managed cloud tier. <!-- needs-research: verify Langfuse's current OSS licensing and the boundary between managed vs self-hosted at publication time. --> <https://langfuse.com/docs>
- **Arize Phoenix — documentation** — open-source LLM observability + eval; OpenInference is Phoenix's native format. <!-- needs-research: verify Phoenix's feature set and its relationship to the paid Arize AX platform at publication time. --> <https://docs.arize.com/phoenix>
- **Promptfoo — documentation** — open-source CLI-first prompt / model / RAG eval tool for eval-gated CI on prompt changes. <https://www.promptfoo.dev/docs/intro/>
- **DeepEval — documentation** — open-source Python-first eval framework with pytest-style assertions on LLM outputs. <https://docs.confident-ai.com/>
- **OpenAI Evals — GitHub** — OpenAI's open-source evals framework; the historical reference and still a common baseline. <https://github.com/openai/evals>
- **Google Vertex AI — Model Evaluation** — GCP-native eval product bundled with Vertex AI. <https://cloud.google.com/vertex-ai/generative-ai/docs/models/evaluation-overview>
- **Azure AI Foundry — Evaluation** — Azure-native eval product bundled with AI Foundry. <!-- needs-research: verify the current URL and product name at publication time. --> <https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/evaluation-approach-gen-ai>

### Agent SDK options

- **OpenAI — Agents SDK documentation** — OpenAI's official multi-agent framework; successor / companion to the earlier Assistants API. <!-- needs-research: verify the current OpenAI Agents SDK naming and its relationship to the Assistants API at publication time. --> <https://openai.github.io/openai-agents-python/>
- **OpenAI — Agents SDK GitHub repository** — reference implementation and examples. <https://github.com/openai/openai-agents-python>
- **Anthropic — Claude Agent SDK documentation** — Anthropic's official agent SDK with native Claude tool-use semantics and MCP integration. <!-- needs-research: verify the current Claude Agent SDK naming and version at publication time — Anthropic has iterated between "Claude SDK", "Claude Agent SDK", and "Claude Code SDK". --> <https://docs.claude.com/en/api/agent-sdk/overview>
- **Anthropic — Model Context Protocol (MCP) specification** — the tool / resource / prompt protocol the Anthropic SDK integrates natively and other SDKs increasingly support. <https://modelcontextprotocol.io/specification>
- **LangGraph — documentation** — LangChain's graph-based agent framework with explicit state machines, checkpointing, and human-in-the-loop primitives. <!-- needs-research: verify LangGraph's current major-version and API stability posture at publication time. --> <https://langchain-ai.github.io/langgraph/>
- **LangGraph — GitHub repository** — source and reference examples. <https://github.com/langchain-ai/langgraph>
- **Agno — documentation** — open-source multi-agent framework (formerly Phidata); focus on lean agent composition. <!-- needs-research: verify Agno's current positioning and its relationship to the Phidata brand at publication time. --> <https://docs.agno.com/>
- **Microsoft Semantic Kernel — documentation** — Microsoft's agent SDK (.NET / Python / Java) with deep Azure AI Foundry integration. <!-- needs-research: verify the current SK vs AutoGen convergence story at publication time. --> <https://learn.microsoft.com/en-us/semantic-kernel/overview/>
- **Microsoft AutoGen — documentation** — Microsoft's research-oriented multi-agent framework; converging with Semantic Kernel. <https://microsoft.github.io/autogen/>

### Build-vs-buy and trade-off note methodology

- **Michael Nygard — "Documenting Architecture Decisions" (Architecture Decision Records)** — the ADR pattern the trade-off note descends from. <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>
- **ThoughtWorks Technology Radar — "Lightweight Architecture Decision Records"** — the widely-adopted lightweight-ADR articulation. <https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records>
- **Joel Spolsky — "Things You Should Never Do, Part I" (2000)** — the canonical essay on the "let's just rewrite it" fallacy; the reference for chapter 03's "build option is defensible only when…" bar. <https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/>

## The per-feature feature-card slice (chapter 04)

### Model card and datasheet lineage

- **Mitchell, Wu, Zaldivar, Barnes, Vasserman, Hutchinson, Spitzer, Raji, Gebru — "Model Cards for Model Reporting" (FAT* 2019)** — the canonical model-card paper the feature-card slice descends from. <https://arxiv.org/abs/1810.03993>
- **Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé III, Crawford — "Datasheets for Datasets" (Communications of the ACM, 2021)** — the sibling documentation pattern for training / evaluation datasets, useful when a feature card cites a dataset. <https://arxiv.org/abs/1803.09010>
- **Google — Model Card Toolkit** — Google's open-source toolkit for generating structured model cards with JSON schema; a machine-consumable reference for the feature-card front matter. <https://research.google/pubs/model-card-toolkit-a-scalable-and-comprehensive-model-card-generation-toolkit/>
- **Hugging Face — Model Cards** — Hugging Face's model-card pattern (as required documentation on the Hub); the dominant practical form in the OSS ecosystem. <https://huggingface.co/docs/hub/model-cards>

### System cards and provider practice

- **OpenAI — GPT-4 System Card (2023)** — the reference system-card artefact at the provider altitude, useful as a model for what the applied engineer's card is *not* (system card is provider-level; feature card is deployment-level). <https://cdn.openai.com/papers/gpt-4-system-card.pdf>
- **OpenAI — GPT-4o System Card (2024)** — the follow-up system card. <!-- needs-research: verify the current-generation system card and its canonical URL at publication time; OpenAI publishes new system cards with each major model release. --> <https://openai.com/index/gpt-4o-system-card/>
- **Anthropic — Claude model system cards** — Anthropic's system-card / model-card publications for the Claude family. <!-- needs-research: verify the current canonical URL for the Claude system card index at publication time. --> <https://www.anthropic.com/news>
- **Google DeepMind — Gemini technical reports** — Google's provider-level model documentation. <!-- needs-research: verify the current Gemini technical report URL at publication time. --> <https://deepmind.google/technologies/gemini/>
- **Meta — Llama model cards** — Meta's model-card artefacts for the Llama family; reference for open-weights documentation practice. <https://github.com/meta-llama/llama-models/tree/main/models>
- **Partnership on AI — About ML Reference Documents** — cross-org documentation-standards project informing the feature-card discipline. <https://partnershiponai.org/workstream/about-ml/>

### Feature-card front-matter and aggregation

- **YAML — specification (v1.2.2)** — the format for the feature-card front-matter block. <https://yaml.org/spec/1.2.2/>
- **JSON Schema — specification** — the schema-validation vocabulary useful when the org's aggregator wants to validate the feature-card front-matter shape. <https://json-schema.org/specification>

## Delegation contracts with peer specialists (chapter 05)

### RACI and org-design references

- **Project Management Institute — *PMBOK Guide* (RACI chapters)** — the canonical statement of the Responsible / Accountable / Consulted / Informed matrix that the delegation contract's "who owns what" section formalises. <!-- needs-research: verify the current PMBOK Guide edition at publication time. --> <https://www.pmi.org/pmbok-guide-standards>
- **Google SRE Book — On-Call** — the on-call responsibility model; the reference for the "on-call counter-party" field in each contract. <https://sre.google/sre-book/being-on-call/>
- **Google SRE Workbook — Working with Other Teams** — the cross-team collaboration chapter, primary source for the "written contract, not verbal agreement" discipline. <https://sre.google/workbook/> <!-- needs-research: verify the direct chapter URL if one exists in the Workbook TOC. -->
- **Team Topologies — Skelton & Pais (2019)** — the framework for defining team interaction modes ("collaboration," "X-as-a-service," "facilitating") that the peer-role delegation contracts operationalise. <https://teamtopologies.com/book>

### Peer-role tracks (link-outs for scope alignment)

- **`agentic-ai-engineer` track — org curriculum** — the peer track owning framework depth (LangGraph internals, OpenAI Agents / Claude Agent SDK internals, MCP nuance). <!-- needs-research: link the specific published track when available. -->
- **`ai-eval-engineer` and `model-evaluation-engineer` tracks — org curriculum** — the peer tracks owning program-level eval and benchmark / statistical rigour. <!-- needs-research: link the specific published tracks when available. -->
- **`ai-risk-engineer` track — org curriculum** — the peer track owning harm modelling, red-team programme, and adversarial dataset curation. <!-- needs-research: link the specific published track when available. -->
- **`ai-infra-mlops` / `ai-infra-ml-platform` / `ai-infra-performance` / `ai-infra-security` tracks — org curriculum** — the peer tracks owning deploy, serving performance, and deep security. <!-- needs-research: link the specific published tracks when available. -->
- **`ai-governance-analyst` track — org curriculum** — the peer track that composes the org-wide governance evidence from feature-level artefacts. <!-- needs-research: link the specific published track when available. -->

## Governance mapping — NIST AI RMF, EU AI Act, ISO/IEC 42001 (chapter 06)

### NIST AI Risk Management Framework

- **NIST — AI Risk Management Framework (AI RMF 1.0, January 2023)** — the primary voluntary framework the chapter's Map / Measure / Manage citation table cites. <https://www.nist.gov/itl/ai-risk-management-framework>
- **NIST — AI RMF 1.0 core document (PDF)** — the AI RMF specification itself. <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf>
- **NIST — AI RMF Playbook** — accompanying implementation guidance for practitioners. <https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook>
- **NIST — Generative AI Profile (AI 600-1, July 2024)** — the GAI companion document specialising the AI RMF for generative-AI systems; the more directly-applicable document for LLM features. <!-- needs-research: verify the current published version of NIST AI 600-1 at publication time. --> <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- **NIST AI Resource Center (AIRC)** — the umbrella site that hosts current AI-RMF updates, crosswalks, and profile revisions. <https://airc.nist.gov/>

### EU AI Act

- **EU — Regulation (EU) 2024/1689 (the AI Act) — official text on EUR-Lex** — the primary legal text. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **European Commission — AI Act landing page** — official overview of the Act, its risk categorisation, and staged applicability dates. <!-- needs-research: verify the current staged-applicability dates and any implementing acts published since the Act's entry into force. --> <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>
- **European AI Office** — the Commission body coordinating implementation, GPAI oversight, and codes of practice. <https://digital-strategy.ec.europa.eu/en/policies/ai-office>
- **AI Act Explorer (Future of Life Institute)** — a searchable, article-by-article reader for the Act text; useful for locating the specific transparency articles the applied engineer maps. <https://artificialintelligenceact.eu/>
- **AI Act — Article 50 (transparency obligations)** — the article containing the primary transparency obligations that most limited-risk LLM features are subject to. <!-- needs-research: confirm the current article numbering and cross-reference to any implementing acts at publication time. --> <https://artificialintelligenceact.eu/article/50/>

### ISO/IEC 42001

- **ISO/IEC 42001:2023 — Information technology — Artificial intelligence — Management system** — the international standard for AI management systems (AIMS); the primary source the Annex A control mapping cites. <https://www.iso.org/standard/81230.html>
- **ISO/IEC 23894:2023 — Information technology — Artificial intelligence — Guidance on risk management** — the companion risk-management guidance that pairs with 42001 and cross-references NIST AI RMF conceptually. <https://www.iso.org/standard/77304.html>
- **ISO/IEC 22989:2022 — Artificial intelligence concepts and terminology** — the underlying vocabulary standard cited by 42001. <https://www.iso.org/standard/74296.html>
- **ISO/IEC 5338:2023 — AI system life cycle processes** — the life-cycle standard that Annex A control A.9 (AI system life cycle) is coordinated with. <https://www.iso.org/standard/81118.html>
- **BSI — ISO/IEC 42001 explainer (accredited body summary)** — accessible overview of the AIMS structure and Annex A control families for readers who cannot access the paid ISO PDF. <!-- needs-research: verify the current BSI (or equivalent accredited-body) summary URL at publication time. --> <https://www.bsigroup.com/en-GB/products-and-services/standards/iso-iec-42001-artificial-intelligence-management-system/>

### Related governance and standards cross-references

- **OWASP — Top 10 for LLM Applications** — the surface-level threat taxonomy the safety-scope field maps against. <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- **OWASP — Agentic AI Threat Modeling** — the newer OWASP work on agent-shaped threats. <!-- needs-research: verify the exact current title and URL at publication time; the OWASP Agentic project has evolved. --> <https://genai.owasp.org/>
- **MITRE ATLAS — Adversarial Threat Landscape for AI Systems** — MITRE's tactic / technique catalogue for adversarial attacks against AI systems; a peer taxonomy to OWASP LLM Top 10, useful when the mapping needs a red-team-oriented framing. <https://atlas.mitre.org/>
- **UK AI Safety Institute — publications** — the UK AISI's public evaluations of frontier models; a reference the `ai-risk-engineer` peer role cites for methodology. <!-- needs-research: verify the AISI's current name — it may have been renamed to "UK AI Security Institute" — and the current publications URL. --> <https://www.aisi.gov.uk/>
- **NIST — U.S. AI Safety Institute Consortium (AISIC)** — the U.S. NIST AI Safety Institute's public work programme; a companion the org's governance function tracks. <https://www.nist.gov/aisi>

## Cross-module and cross-track references

- **[mod-101 — Applied AI foundations and feature design](../mod-101-applied-ai-foundations-and-feature-design/README.md)** — the one-page architecture doc and trade-off note the charter (chapter 01) and build-vs-buy notes (chapter 03) compose with.
- **[mod-102 — Production prompting and structured outputs](../mod-102-production-prompting-and-structured-outputs/README.md)** — the prompt-versioning discipline the applied engineer explicitly does not delegate.
- **[mod-103 — RAG integration in product](../mod-103-rag-integration-in-product/README.md)** — the retrieval stack the feature card's system-architecture section cites.
- **[mod-104 — Tool calling and orchestration](../mod-104-tool-calling-and-orchestration/README.md)** and **[mod-105 — Shipping product agents](../mod-105-shipping-product-agents/README.md)** — the agent-SDK substrate the build-vs-buy decision in chapter 03 covers.
- **[mod-106 — Feature-side evaluation](../mod-106-feature-side-evaluation/README.md)** — the eval substrate every fixture, judge, and CI gate in chapters 02 and 04 depends on.
- **[mod-107 — Guardrails and safety on the surface](../mod-107-guardrails-and-safety-on-the-surface/README.md)** — the safety-scope substrate the charter (chapter 01) names and the feature card (chapter 04) audits.
- **[mod-108 — Model routing and provider selection](../mod-108-model-routing-and-provider-selection/README.md)** — the gateway substrate the build-vs-buy decision in chapter 03 covers.
- **[mod-109 — Cost, latency, and SLOs for LLM features](../mod-109-cost-latency-slos-for-llm-features/README.md)** — the CPSO / SLO substrate the charter's cost-budget field and the feature card's cost / latency section read from.
- **[mod-110 — Observability, feedback, and incidents](../mod-110-observability-feedback-and-incidents/README.md)** — the trace and runbook substrate the incident-to-fixture loop in chapter 02 plugs into.
- **[mod-111 — Shipping, experimenting, and rollbacks](../mod-111-shipping-experimenting-and-rollbacks/README.md)** — the flag / canary / rollback substrate the charter's kill-criteria field and the feature card's human-oversight section depend on.

## Peer-role handoffs (link-outs to peer-track material)

- **`agentic-ai-engineer`** — deep framework expertise (LangGraph internals, OpenAI Agents / Claude Agent SDK internals, MCP nuance); the peer role for chapter 05 Contract 1.
- **`ai-eval-engineer`** — the program-level eval substrate; the peer role for chapter 05 Contract 2.
- **`model-evaluation-engineer`** — benchmark selection and statistical rigour; the peer role for chapter 05 Contract 3.
- **`ai-risk-engineer`** — harm modelling, red-team programme, adversarial dataset curation; the peer role for chapter 05 Contract 4.
- **`ai-infra-mlops` / `ai-infra-ml-platform`** — the deploy substrate; the peer role for chapter 05 Contract 5.
- **`ai-infra-performance`** — serving-tier performance for self-hosted inference; the peer role for chapter 05 Contract 6.
- **`ai-infra-security` / `application-security-engineer`** — authz architecture, secrets, MCP-server security; the peer role for chapter 05 Contract 7.
- **`ai-governance-analyst`** — the primary counter-party for the feature card's front matter and the governance mapping in chapter 06.
- **`fine-tuning-engineer`** — SFT vs PEFT vs DPO vs RLHF methodology depth, cited from [mod-111 chapter 05](../mod-111-shipping-experimenting-and-rollbacks/05-hosted-fine-tuning-as-an-applied-lever.md).
- **`privacy-engineer` / `data-security-engineer`** — regulatory posture (DPIA, cross-border transfer, DPA authority interactions); the reviewers on the trade-off notes and the feature card's data-handling section.
- **Product manager** — the counter-party for the charter's success-metric field and the "is this trade acceptable" call at the feature card's cost / latency section.
