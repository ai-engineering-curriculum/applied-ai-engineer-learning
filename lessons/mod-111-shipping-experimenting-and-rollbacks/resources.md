# Resources for mod-111-shipping-experimenting-and-rollbacks (Shipping, Experimenting, and Rollbacks for LLM Features)

Primary-source references — official standards, provider documentation, SRE canon, and canonical papers. Each entry names *why* the resource matters to the chapter it supports. Verify the exact URLs, API shapes, and pricing at the point of use; the flag SDKs, experimentation platforms, and hosted fine-tuning surfaces move faster than the module's revision cadence.

## Feature flags as the LLM shipping substrate (chapter 01)

### Vendor-neutral standard

- **OpenFeature specification (CNCF)** — the vendor-neutral SDK / API specification for feature flag evaluation, with the `EvaluationContext`, `targeting_key`, and provider model chapter 01 uses. <https://openfeature.dev/specification/>
- **OpenFeature — home / documentation** — landing page and language SDK entry points (Python, Node, Go, Java, .NET, and others). <https://openfeature.dev/>
- **OpenFeature GitHub organization** — reference implementations, providers, and the specification repository. <https://github.com/open-feature>
- **flagd (OpenFeature reference provider)** — the CNCF OpenFeature reference flag provider used in exercise-01's free-path setup; runs locally, backed by a JSON file. <https://flagd.dev/>

### Commercial and self-hostable providers

- **LaunchDarkly — documentation** — the reference commercial flag / experimentation platform; server-side SDKs, targeting rules, the SSE-driven in-memory cache model chapter 01 discusses. <https://docs.launchdarkly.com/>
- **LaunchDarkly — server-side SDKs overview** — the SDK shape (in-memory store, streaming updates, `variation`/`identify` call surface) chapter 01 assumes. <https://docs.launchdarkly.com/sdk/server-side>
- **GrowthBook — documentation** — open-source, self-hostable feature flagging + experimentation platform with an OpenFeature provider. <https://docs.growthbook.io/>
- **Split — documentation** — commercial flag / experimentation platform; used as an alternative reference in chapter 03. <https://help.split.io/hc/en-us>
- **ConfigCat — documentation** — hosted feature-flag service with a free tier suitable for exercise-01. <https://configcat.com/docs/>
- **Flagsmith — documentation** — open-source self-hostable feature-flag service. <https://docs.flagsmith.com/>
- **Statsig — feature gates documentation** — Statsig's flag surface; the same SDK powers assignment for experiments (chapter 03). <https://docs.statsig.com/>

### Progressive delivery canon

- **Google SRE Book — Release Engineering** — the canonical chapter on release cadence, hermeticity, and the rollout / rollback discipline the LLM flag substrate specialises. <https://sre.google/sre-book/release-engineering/>
- **Martin Fowler — Feature Toggles (aka Feature Flags)** — the widely-cited taxonomy of release toggles vs experiment toggles vs ops toggles vs permissioning toggles, which chapter 01's four flag shapes derive from. <https://martinfowler.com/articles/feature-toggles.html>

## Canary and shadow launches with pre-registered gates (chapter 02)

### SRE canon on canaries and rollouts

- **Google SRE Book — Canarying Releases (SRE Workbook)** — the canonical primer on canary sizing, observation windows, and promotion criteria that chapter 02 specialises for LLM gates. <https://sre.google/workbook/canarying-releases/>
- **Google SRE Book — Reliable Product Launches at Scale** — the launch checklist / gated-rollout discipline underlying chapter 02's plan doc. <https://sre.google/sre-book/reliable-product-launches/>
- **Google SRE Workbook — Alerting on SLOs** — the multi-window multi-burn-rate alert pattern (14.4× / 1h fast, 6× / 6h slow) chapter 02 adapts for auto-rollback triggers. <https://sre.google/workbook/alerting-on-slos/>

### Progressive delivery tooling

- **Argo Rollouts — documentation** — Kubernetes-native progressive delivery controller supporting canary, blue-green, and analysis-driven promotion — the infra substrate flag-driven canaries often plug into. <https://argoproj.github.io/argo-rollouts/>
- **Flagger — documentation** — CNCF progressive delivery operator for automated canary promotion with metric analysis gates. <https://docs.flagger.app/>
- **Continuous Delivery Foundation — Progressive Delivery WG** — cross-vendor working group on the progressive delivery vocabulary. <https://cd.foundation/>

### Sample-size and power references

- **Evan Miller — Sample Size Calculator (proportions)** — the widely-used online calculator whose formula chapter 02's `n ≈ 16 × p × (1 - p) / Δ²` heuristic approximates. <https://www.evanmiller.org/ab-testing/sample-size.html>
- **StatsModels — power calculations** — Python library with proportion, mean, and chi-squared power calculations for verifying the canary heuristic against a more rigorous number. <https://www.statsmodels.org/stable/stats.html#power-and-sample-size-calculations>

### Shadow-launch precedent

- **GitHub Engineering — Move Fast and Fix Things (Scientist)** — the "science" pattern of running a candidate implementation alongside the control and comparing outputs off the critical path — the direct precedent for the shadow pattern in chapter 02. <https://github.blog/2016-02-03-scientist/>
- **`github/scientist` (Ruby reference)** — the reference implementation. <https://github.com/github/scientist>

## A/B experiments: the interface to the experimentation platform (chapter 03)

### Experimentation platforms

- **Statsig — experiments documentation** — commercial experimentation platform; assignment, exposure, and guardrail-metric APIs. <https://docs.statsig.com/experiments-plus/create-new/>
- **Eppo — documentation** — commercial warehouse-native experimentation platform; assignment SDK and metric definition model. <https://docs.geteppo.com/>
- **GrowthBook — experiments documentation** — the open-source experimentation surface referenced in exercise-03's code samples. <https://docs.growthbook.io/experiments>
- **LaunchDarkly Experimentation — documentation** — the experimentation add-on on top of the LaunchDarkly flag platform. <https://docs.launchdarkly.com/home/experimentation>
- **Split Experimentation — documentation** — Split's experimentation surface. <https://help.split.io/hc/en-us/categories/360002696612-Experimentation>
- **Optimizely — Full Stack documentation** — server-side experimentation SDK, historically foundational. <https://docs.developers.optimizely.com/full-stack-experimentation/docs/welcome>

### Canonical experimentation literature

- **Kohavi, Tang, Xu — *Trustworthy Online Controlled Experiments* (Cambridge, 2020)** — the canonical practitioner's book on running A/B tests correctly, covering assignment, exposure, guardrails, and the pitfalls chapter 03 names. <https://experimentguide.com/>
- **Fabijan, Gupchup, Gupta et al. — "Diagnosing Sample Ratio Mismatch in Online Controlled Experiments" (KDD 2019)** — the primary paper on the SRM discipline chapter 03 treats as the "your interface is broken" alarm. <https://dl.acm.org/doi/10.1145/3292500.3330722>
- **Deng, Xu, Kohavi, Walker — "Improving the sensitivity of online controlled experiments by utilizing pre-experiment data" (WSDM 2013)** — the CUPED paper; the canonical variance-reduction technique chapter 03 names as peer-role territory. <https://dl.acm.org/doi/10.1145/2433396.2433413>
- **Johari, Koomen, Pekelis, Walsh — "Peeking at A/B Tests: Why It Matters, and What to Do About It" (KDD 2017)** — the primer on why "check the p-value every day" is broken and what always-valid inference does about it. <https://dl.acm.org/doi/10.1145/3097983.3097992>
- **Ron Kohavi — publications page** — Kohavi's index of experimentation papers, including the SRM, primary-metric-selection, and org-experimentation-maturity references. <https://exp-platform.com/>
- **Microsoft — Experimentation Platform documentation** — the ExP team's public documentation on running experiments at scale. <https://www.microsoft.com/en-us/research/group/experimentation-platform-exp/>

### Sequential testing and always-valid inference

- **Howard, Ramdas, McAuliffe, Sekhon — "Time-uniform, nonparametric, nonasymptotic confidence sequences" (Annals of Statistics 2021)** — foundational reference for always-valid inference / confidence sequences underlying modern sequential testing in experimentation. <https://arxiv.org/abs/1810.08240>

## Hot rollback of a bad prompt (chapter 04)

### Prompt-registry substrates

- **LangSmith — Prompt Hub documentation** — versioned prompt store with a pull / push API; a working reference for the append-only registry chapter 04 describes. <https://docs.smith.langchain.com/prompt_engineering/how_to_guides>
- **Humanloop — documentation** — hosted prompt-management platform with versioning, deployment environments, and audit logs. <https://humanloop.com/docs>
- **PromptLayer — documentation** — hosted prompt-management surface with versioning and rollback semantics. <https://docs.promptlayer.com/>
- **Braintrust — documentation** — evaluation + prompt-management platform with a prompt registry surface. <https://www.braintrust.dev/docs>
- **OpenAI — Prompt objects (API reference)** — OpenAI's platform-managed prompt object with versioning primitives. <!-- needs-research: verify current URL and API-surface status for OpenAI's Prompts API at authoring time --> <https://platform.openai.com/docs/api-reference/prompts>

### Rollback and incident-response canon

- **Google SRE Book — Managing Incidents** — the incident-response vocabulary (roles, comms, decision protocol) chapter 04's rollback runbook slots into. <https://sre.google/sre-book/managing-incidents/>
- **Google SRE Workbook — Postmortem Culture** — the blameless postmortem discipline that the drill retro (deliverable 7) feeds. <https://sre.google/workbook/postmortem-culture/>
- **Google SRE Book — Emergency Response** — the "have a plan before you need it" chapter that grounds the monthly rollback drill. <https://sre.google/sre-book/emergency-response/>
- **PagerDuty — Incident Response documentation** — the widely-used incident-response reference; runbook link semantics, escalation policies, and drill cadences. <https://response.pagerduty.com/>

### Feature-flag rollback semantics

- **LaunchDarkly — feature-flag targeting and rollout** — the specific SDK / API mechanics of flipping a multivariate flag; latency of change propagation via SSE. <https://docs.launchdarkly.com/home/flags/targeting>
- **LaunchDarkly — audit log** — the audit trail chapter 04 names as the forensic source for post-incident retros. <https://docs.launchdarkly.com/home/account-security/audit-log>
- **OpenFeature — hooks and evaluation lifecycle** — the standard extension points every rollback verification hook uses. <https://openfeature.dev/specification/sections/hooks/>

## Hosted fine-tuning as an applied lever (chapter 05)

### Provider fine-tuning surfaces

- **OpenAI — Fine-tuning guide** — official supervised fine-tuning (SFT) documentation, including the JSONL messages format, the `fine_tuning/jobs` API, and the list of currently fine-tunable models. <https://platform.openai.com/docs/guides/fine-tuning>
- **OpenAI — Direct Preference Optimization (DPO) fine-tuning** — DPO-shape fine-tuning on OpenAI, for teams with preference pairs. <!-- needs-research: verify current availability and URL of OpenAI's DPO fine-tuning surface at authoring time — this moves --> <https://platform.openai.com/docs/guides/fine-tuning>
- **Anthropic — Claude fine-tuning on Amazon Bedrock** — the Bedrock-hosted path for fine-tuning Claude models. <!-- needs-research: verify current model list and availability window --> <https://docs.anthropic.com/en/docs/build-with-claude/fine-tuning>
- **Amazon Bedrock — Custom Models** — Bedrock's custom-model fine-tuning surface (Titan, Llama, and other supported base models), plus continued pre-training. <https://docs.aws.amazon.com/bedrock/latest/userguide/custom-models.html>
- **Amazon Bedrock — Provisioned Throughput** — the provisioned-throughput pricing model chapter 05 discusses for Bedrock custom models. <https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html>
- **Google Vertex AI — Supervised tuning for Gemini** — Vertex's supervised tuning surface on Gemini models. <https://cloud.google.com/vertex-ai/generative-ai/docs/models/tune-gemini-overview>
- **Google Vertex AI — Tuning overview** — landing page for the full tuning surface (supervised, RLHF, adapter tuning). <https://cloud.google.com/vertex-ai/generative-ai/docs/models/tune-models>
- **Azure OpenAI — Fine-tuning** — the Azure-hosted OpenAI fine-tuning path for teams on Azure. <https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/fine-tuning>

### Foundational method references (peer-role depth)

- **Hu, Shen, Wallis et al. — "LoRA: Low-Rank Adaptation of Large Language Models" (2021)** — the canonical LoRA paper underlying PEFT surfaces; peer-role territory but a required background read. <https://arxiv.org/abs/2106.09685>
- **Dettmers, Pagnoni, Holtzman, Zettlemoyer — "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)** — the QLoRA extension for memory-efficient tuning. <https://arxiv.org/abs/2305.14314>
- **Rafailov, Sharma, Mitchell et al. — "Direct Preference Optimization: Your Language Model is Secretly a Reward Model" (2023)** — the DPO paper, i.e., the preference-optimisation alternative to RLHF referenced in chapter 05 as a `fine-tuning-engineer` handoff. <https://arxiv.org/abs/2305.18290>
- **Ouyang, Wu, Jiang et al. — "Training language models to follow instructions with human feedback" (InstructGPT, 2022)** — the canonical RLHF paper, the reference for what "alignment tax" and "safety re-tuning" mean in chapter 05. <https://arxiv.org/abs/2203.02155>
- **Houlsby, Giurgiu, Jastrzebski et al. — "Parameter-Efficient Transfer Learning for NLP" (Adapters, 2019)** — the foundational adapter-tuning paper that PEFT descends from. <https://arxiv.org/abs/1902.00751>

### Distillation and dataset pitfalls

- **Hinton, Vinyals, Dean — "Distilling the Knowledge in a Neural Network" (2015)** — the foundational distillation paper, primer for chapter 05's "small-model distillation" applied lever. <https://arxiv.org/abs/1503.02531>
- **Shumailov, Shumaylov, Zhao et al. — "AI models collapse when trained on recursively generated data" (Nature, 2024)** — the model-collapse paper chapter 05 cites in the anti-patterns; the deep version of "you cannot exceed your labels." <https://www.nature.com/articles/s41586-024-07566-y>

### Compliance and PII scrubbing for training data

- **OpenAI — Enterprise privacy / data usage policies** — the DPA-facing statement that fine-tuning data is not used to train foundation models; the counterpart chapter 05 references when scrubbing before upload. <https://openai.com/enterprise-privacy/>
- **Anthropic — Trust Center / data usage** — Anthropic's equivalent commitments for enterprise usage. <https://trust.anthropic.com/>
- **GDPR — Article 32 (Security of processing)** — the DPA-facing legal basis for scrubbing training data before upload to a fine-tuning provider. <https://gdpr-info.eu/art-32-gdpr/>
- **Microsoft Presidio** — open-source PII detection / anonymisation library referenced across the track for pre-upload scrubbing. <https://microsoft.github.io/presidio/>

## Cross-module and cross-track references

- **[mod-102 — Production prompting and structured outputs](../mod-102-production-prompting-and-structured-outputs/README.md)** — the prompt-versioning discipline the registry (chapter 04) stores.
- **[mod-106 — Feature-side evaluation](../mod-106-feature-side-evaluation/README.md)** — the offline eval, judge, and drift substrates the quality gates in chapters 02, 03, and 05 read from.
- **[mod-107 — Guardrails and safety on the surface](../mod-107-guardrails-and-safety-on-the-surface/README.md)** — the guardrail vocabulary the kill switch (chapter 01) and safety gates (chapter 02) plug into.
- **[mod-108 — Model routing and provider selection](../mod-108-model-routing-and-provider-selection/README.md)** — the provider-selection substrate the `model_id` flag layer (chapter 01) resolves against.
- **[mod-109 — Cost, latency, and SLOs](../mod-109-cost-latency-slos-for-llm-features/README.md)** — the CPSO ledger, per-surface latency spec, and multi-window multi-burn-rate alert pattern the cost / latency gates (chapter 02) and auto-rollback (chapter 04) read from.
- **[mod-110 — Observability, feedback, and incidents](../mod-110-observability-feedback-and-incidents/README.md)** — the OTel Gen-AI + OpenInference span vocabulary every chapter emits `app.experiment.arm`, `app.prompt.id`, `app.prompt.template_hash`, `app.shadow.*` on top of.
- **[mod-112 — Owning an applied AI feature end-to-end](../mod-112-owning-an-applied-ai-feature-end-to-end/README.md)** — the capstone that composes the flag substrate, canary regime, A/B interface, rollback path, and hosted-tune surface into feature ownership.

## Peer-role handoffs (link-outs to peer-track material)

- **`senior-ml-engineer`** — owns the deep experimentation methodology (CUPED, sequential testing, heterogeneous treatment effects, delta-method estimators); chapter 03 names the four families of work that trigger the handoff.
- **`model-evaluation-engineer`** — owns the LLM-judge design and calibration that the quality gates (chapters 02, 03, 05) read from; see [mod-106](../mod-106-feature-side-evaluation/README.md).
- **`fine-tuning-engineer`** — owns SFT vs PEFT vs DPO vs RLHF method selection, advanced dataset curation, catastrophic-forgetting audits, alignment re-tuning, and open-weights fine-tuning on private infra; chapter 05 names the specific escalation triggers.
- **`ai-infra-mlops`** — owns the flag-platform infra (or the vendor relationship), the experimentation-platform data pipeline, and the alerting substrate the auto-rollback fires into.
- **`sre` / `platform`** — owns the org-wide incident process, the paging system, and the rollback-drill cadence the LLM-feature runbook (chapter 04) plugs into.
- **`privacy-engineer`** and **`data-security-engineer`** — own the DPA / residency envelope any shadow-launch cross-provider call and any fine-tuning upload must respect.
- **Product manager** — owns the "is this trade acceptable" decision when a launch shows a mixed result (primary metric up, guardrail metric up).
