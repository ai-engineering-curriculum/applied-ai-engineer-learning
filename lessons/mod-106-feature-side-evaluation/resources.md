# Resources for mod-106 — Feature-Side Evaluation: Offline, Online, and In CI

Curated primary sources for the topics covered across the five chapters. Provider documentation, standards, canonical open-source projects, and the papers behind LLM-as-judge and RAG-triad discipline come first; secondary references are named as such. The eval-runner space in particular moves on a monthly cadence — consult the current live docs before pinning behaviour, and add an accessed-on date to any citation you carry into a design review.

## Cross-cutting framing (all chapters)

- **Anthropic — Building effective AI agents** (engineering blog) — the "workflow vs. agent" framing that informs which surfaces need which eval shape. `https://www.anthropic.com/engineering/building-effective-agents`.
- **OpenAI — A practical guide to building agents** (whitepaper) — includes the applied engineer's evaluation section. `https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf`.
- **Chip Huyen — *AI Engineering* (book)** — the "evaluation-driven development" chapter is the closest published statement of this module's discipline. `https://huyenchip.com/`.
- **Eugene Yan — Evals patterns and practitioner writeups** — long-form field notes on LLM eval design. `https://eugeneyan.com/writing/`.
- **Hamel Husain — Your AI Product Needs Evals** — the widely-cited practitioner post that maps to this module's "eval as CI concern" framing. `https://hamel.dev/blog/posts/evals/`.
- **Shreya Shankar et al. — Who Validates the Validators?** — practitioner + academic study on LLM-as-judge calibration and drift in shipped systems. `https://arxiv.org/abs/2404.12272`.
- **Simon Willison — Evals tag** — practitioner reference for eval failure modes, provider changes, and judge behaviour. `https://simonwillison.net/tags/evals/`.

## Chapter 01 — Offline eval suites a PR can run

### LLM-as-judge foundations and bias literature

- **Zheng et al. — *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*** — the foundational reference for LLM-as-judge, position bias, and swap-agreement mitigation. `https://arxiv.org/abs/2306.05685`.
- **Wang et al. — *Large Language Models are not Fair Evaluators*** — early paper documenting position bias in pairwise judge settings. `https://arxiv.org/abs/2305.17926`.
- **Dubois et al. — *Length-Controlled AlpacaEval*** — the canonical treatment of length / verbosity bias and how to control for it. `https://arxiv.org/abs/2404.04475`.
- **Panickssery et al. — *LLM Evaluators Recognize and Favor Their Own Generations*** — self-preference bias in judge models. `https://arxiv.org/abs/2404.13076`.
- **Zeng et al. — *Evaluating Large Language Models at Evaluating Instruction Following*** — a broad look at judge reliability; introduces the LLMBar meta-evaluation benchmark. `https://arxiv.org/abs/2310.07641`.
- **Anthropic — Research publications index** — provider-side research posts on model behaviour and safety, including judge-adjacent material on bias and evaluation. `https://www.anthropic.com/research`.

### Rubric grading and structured evaluation

- **G-Eval — Liu et al., *G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment*** — the rubric-with-chain-of-thought judge pattern that many rubric harnesses derive from. `https://arxiv.org/abs/2303.16634`.
- **DeepEval — G-Eval metric documentation** — the practitioner implementation of the pattern. `https://deepeval.com/docs/metrics-llm-evals`.
- **Braintrust — Autoevals library** — a small library of common rubric / judge scorers, useful as reference implementations. `https://github.com/braintrustdata/autoevals`.

### Programmatic correctness (code, SQL, JSON, tool state)

- **HumanEval — Chen et al., *Evaluating Large Language Models Trained on Code*** — the canonical code-correctness harness reference. `https://arxiv.org/abs/2107.03374`.
- **MBPP — Austin et al., *Program Synthesis with Large Language Models*** — a companion code-correctness benchmark. `https://arxiv.org/abs/2108.07732`.
- **BIRD-SQL — Li et al., *Can LLM Already Serve as A Database Interface?*** — the reference for programmatic SQL correctness across realistic schemas. `https://arxiv.org/abs/2305.03111`, `https://bird-bench.github.io/`.
- **JSON Schema — Specification** — the substrate for structured-output correctness checks. `https://json-schema.org/specification`.
- **Pydantic — Documentation** — Python source-of-truth types for the correctness harness. `https://docs.pydantic.dev/latest/`.

### RAG triad (context / groundedness / answer relevance)

- **TruLens — RAG triad documentation** — one of the earliest canonical formulations of the three-metric decomposition (context relevance, groundedness, answer relevance). `https://www.trulens.org/getting_started/core_concepts/rag_triad/`.
- **Arize Phoenix — RAG evaluation cookbook** — Phoenix's parallel formulation, with worked examples. `https://arize.com/docs/phoenix/cookbook/evaluation/evaluate-rag`.
- **Ragas — Documentation** — an OSS eval library organised around faithfulness, answer relevancy, and context precision / recall. `https://docs.ragas.io/`.
- **Es et al. — *RAGAS: Automated Evaluation of Retrieval Augmented Generation*** — the Ragas paper, with the metric definitions. `https://arxiv.org/abs/2309.15217`.

### Safety spot checks and adversarial catalogues

- **OWASP — Top 10 for LLM Applications** — the canonical taxonomy for injection, sensitive-info disclosure, insecure output handling, excessive agency, etc. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **MITRE ATLAS** — adversarial-technique catalogue for ML systems, including LLM-integrated ones. `https://atlas.mitre.org/`.
- **NIST AI 600-1 — Generative AI Profile** — GenAI-specific companion to the RMF; useful for framing which safety categories a surface must cover. `https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf`.
- **Anthropic — Constitutional Classifiers** — provider-side research on defending against universal jailbreaks and prompt-injection attempts. `https://www.anthropic.com/research/constitutional-classifiers`.
- **Simon Willison — Prompt injection tag** — practitioner field notes on new injection variants and their failure modes. `https://simonwillison.net/tags/prompt-injection/`.

### Pre-registration and evaluation discipline

- **Center for Open Science — Preregistration** — the general-science discipline of pre-registering hypotheses and thresholds, transplanted into ML eval by this chapter. `https://www.cos.io/initiatives/prereg`.
- **Google — Rules of Machine Learning** — includes rules on eval discipline that transfer cleanly to LLM-eval. `https://developers.google.com/machine-learning/guides/rules-of-ml`.
- **Anthropic — Model cards and system cards** — a practitioner reference for what a released model / feature's eval report should contain. `https://platform.claude.com/docs/en/resources/overview`.

## Chapter 02 — Trace snapshots as regression fixtures

### Provider dated / pinned model IDs

- **Anthropic — Models overview** — model IDs and the alias-vs-pinned-snapshot contract ("every Claude model ID is a pinned snapshot, including the dateless IDs used from the 4.6 generation on"). `https://platform.claude.com/docs/en/about-claude/models`.
- **OpenAI — Models reference** — snapshot IDs and the moving-alias behaviour behind `gpt-4o`, `gpt-4.1`, etc. `https://platform.openai.com/docs/models`.
- **Google — Gemini models** — dated model IDs and the stable / preview / latest / experimental version patterns. `https://aistudio.google.com/docs/models`.
- **Azure OpenAI — Model versioning** — Microsoft's take on pin-vs-alias behaviour on Azure (now under Microsoft Foundry). `https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/model-versions`.

### Trace / observability substrates the snapshot pipeline sits on

- **OpenTelemetry — GenAI semantic conventions** — the emerging standard for LLM span attributes referenced across the module; moved out of the main semconv tree into its own repository. `https://github.com/open-telemetry/semantic-conventions-genai`.
- **OpenInference — Semantic conventions** — Phoenix / Arize's parallel spec that shipped ahead of the OTel one, with a broader tool-call surface. `https://github.com/Arize-ai/openinference/tree/main/spec`.
- **Langfuse — Tracing documentation** — an OSS observability substrate that captures the lineage fields this chapter names. `https://langfuse.com/docs/tracing`.
- **LangSmith — Observability documentation** — LangChain's equivalent, tightly integrated with LangGraph. `https://docs.langchain.com/langsmith/observability`.
- **Braintrust — Tracing documentation** — Braintrust's trace + experiment substrate. `https://www.braintrust.dev/docs/guides/tracing`. <!-- needs-research: confirm URL. -->

### PII detection and redaction

- **Microsoft Presidio — Documentation** — the canonical OSS PII detector for structured + free-text redaction. `https://microsoft.github.io/presidio/`.
- **Google Cloud — Sensitive Data Protection (Cloud DLP)** — hosted PII detection with strong regex + ML surfaces. `https://cloud.google.com/sensitive-data-protection/docs`.
- **AWS — Comprehend PII detection** — hosted alternative. `https://docs.aws.amazon.com/comprehend/latest/dg/pii.html`.
- **Azure — AI Language PII detection** — Microsoft-hosted alternative. `https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview`.
- **`llm-guard` — PII scanner** — an OSS library specifically for LLM-input and output PII scrubbing. `https://llm-guard.com/input_scanners/anonymize/`. <!-- needs-research: confirm URL. -->
- **`presidio-anonymizer` — Custom operators** — reference for building the stable typed-placeholder scheme this chapter names. `https://microsoft.github.io/presidio/anonymizer/`.

### Data-subject rights (the deletion cascade)

- **GDPR — Regulation (EU) 2016/679** — Articles 15 (access) and 17 (erasure) are the ones the deletion-cascade contract implements. `https://eur-lex.europa.eu/eli/reg/2016/679/oj`.
- **California — CCPA / CPRA** — the California statute for equivalent user-deletion rights. `https://oag.ca.gov/privacy/ccpa`.
- **HIPAA — HHS overview** — the regulatory frame most relevant to special-category (health) data referenced in the snapshot-never default. `https://www.hhs.gov/hipaa/index.html`.
- **PCI DSS v4.0** — the payment-card standard relevant to any surface where a snapshot could include a card number. `https://www.pcisecuritystandards.org/document_library/`. <!-- needs-research: confirm current version URL. -->

### Fixture design and versioning

- **Google — Testing on the Toilet: Fixtures** — background reference for fixture-set discipline transplanted to eval. `https://testing.googleblog.com/`. <!-- needs-research: pin a specific post. -->
- **Jest / Vitest — Snapshot testing** — the software-testing sibling of trace-snapshot regression, useful for framing. `https://jestjs.io/docs/snapshot-testing`, `https://vitest.dev/guide/snapshot.html`.
- **DVC — Data Version Control** — the OSS tool for versioning datasets / fixture sets alongside code. `https://dvc.org/doc`.
- **Git LFS** — the low-tech alternative for versioning large fixture blobs. `https://git-lfs.com/`.

## Chapter 03 — Eval-gated CI end-to-end

### Promptfoo

- **Promptfoo — Documentation** — the canonical entry point. `https://www.promptfoo.dev/docs/intro/`.
- **Promptfoo — Assertions reference** — the full assertion catalogue (exact match, contains, JSON validity, Python / JavaScript callbacks, LLM-graded rubrics, factuality, cost / latency thresholds). `https://www.promptfoo.dev/docs/configuration/expected-outputs/`. <!-- needs-research: confirm URL and section slug. -->
- **Promptfoo — GitHub Action** — the CI integration used in the chapter's example. `https://www.promptfoo.dev/docs/integrations/github-action/`. <!-- needs-research: confirm URL. -->
- **`promptfoo/promptfoo`** — reference repository. `https://github.com/promptfoo/promptfoo`.

### DeepEval

- **DeepEval — Documentation** — the pytest-first eval library. `https://docs.confident-ai.com/`. <!-- needs-research: confirm URL — the docs domain has moved. -->
- **DeepEval — Metrics reference** — G-Eval, Faithfulness, AnswerRelevancy, ContextualRelevancy, Hallucination, Bias, Toxicity, custom judges. `https://docs.confident-ai.com/docs/metrics-introduction`. <!-- needs-research: confirm URL. -->
- **`confident-ai/deepeval`** — reference repository. `https://github.com/confident-ai/deepeval`.
- **Confident AI — Hosted dashboard** — the managed surface pairing with the OSS library. `https://www.confident-ai.com/`.

### Braintrust

- **Braintrust — Documentation** — the hosted eval + trace platform. `https://www.braintrust.dev/docs/start/`.
- **Braintrust — Eval API reference** — `Eval(...)`, scorers, experiments. `https://www.braintrust.dev/docs/reference/api/Eval`. <!-- needs-research: confirm URL. -->
- **Braintrust — GitHub Action** — the CI integration for the diff-comment view. `https://www.braintrust.dev/docs/guides/ci`. <!-- needs-research: confirm URL. -->
- **`braintrustdata/braintrust-proxy`** — the OSS proxy used to instrument arbitrary providers for Braintrust. `https://github.com/braintrustdata/braintrust-proxy`.

### Langfuse

- **Langfuse — Documentation** — the OSS observability + eval platform. `https://langfuse.com/docs`.
- **Langfuse — Datasets and evaluations** — the eval-runner surface. `https://langfuse.com/docs/datasets/overview`. <!-- needs-research: confirm URL. -->
- **Langfuse — Prompt management** — the versioned-prompt surface referenced in the chapter. `https://langfuse.com/docs/prompts/get-started`.
- **`langfuse/langfuse`** — reference repository. `https://github.com/langfuse/langfuse`.

### Arize Phoenix

- **Arize Phoenix — Documentation** — the OSS observability + eval framework from Arize. `https://arize.com/docs/phoenix/`.
- **Phoenix — Evals module** — the eval-runner surface, with strong RAG-triad tooling. `https://arize.com/docs/phoenix/evaluation/how-to-evals/running-pre-tested-evals`. <!-- needs-research: confirm URL. -->
- **Phoenix — OpenInference tracing** — the OTel-native trace substrate. `https://arize.com/docs/phoenix/tracing/llm-traces`. <!-- needs-research: confirm URL. -->
- **`Arize-ai/phoenix`** — reference repository. `https://github.com/Arize-ai/phoenix`.

### Weights & Biases Weave

- **Weights & Biases Weave — Documentation** — W&B's LLM-eval-and-trace surface. `https://weave-docs.wandb.ai/`.
- **Weave — Evaluations reference** — the `Evaluation` API and scorer conventions. `https://weave-docs.wandb.ai/guides/core-types/evaluations`. <!-- needs-research: confirm URL. -->
- **`wandb/weave`** — reference repository. `https://github.com/wandb/weave`.

### LangSmith

- **LangSmith — Documentation** — LangChain's hosted trace + eval surface. `https://docs.smith.langchain.com/`.
- **LangSmith — Datasets and experiments** — the eval-runner surface with first-class LangGraph integration. `https://docs.smith.langchain.com/evaluation`. <!-- needs-research: confirm URL. -->
- **LangSmith — CI/CD integration guide** — the CI wiring path. `https://docs.smith.langchain.com/evaluation/how_to_guides/ci`. <!-- needs-research: confirm URL. -->

### Runner-adjacent tools

- **Inspect AI (UK AI Safety Institute)** — the evaluation framework the UK AISI publishes; useful comparison for the runner shape. `https://inspect.ai-safety-institute.org.uk/`.
- **OpenAI Evals** — OpenAI's OSS eval framework, an early reference implementation of the config-driven eval-runner pattern. `https://github.com/openai/evals`.
- **Anthropic — Evaluating prompts in the Console** — the Anthropic-native lightweight eval surface. `https://docs.anthropic.com/en/docs/test-and-evaluate/eval-tool`. <!-- needs-research: confirm URL. -->

### GitHub Actions / CI substrate

- **GitHub Actions — Documentation** — the CI substrate used in the chapter's example workflow. `https://docs.github.com/en/actions`.
- **GitHub Actions — Path filters** — the `paths:` trigger this chapter uses to keep the eval from firing on unrelated PRs. `https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters`.
- **GitHub Actions — Encrypted secrets** — the mechanism for scoping the judge API key to the deep gate only. `https://docs.github.com/en/actions/security-guides/encrypted-secrets`.
- **GitLab CI — `rules:` and `only:`** — the equivalent trigger surface for GitLab-hosted repos. `https://docs.gitlab.com/ee/ci/yaml/#rules`.

### Judge-cost discipline and caching

- **Anthropic — Prompt caching** — provider-side caching that reduces judge cost on repeated evaluations. `https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching`. <!-- needs-research: confirm URL. -->
- **OpenAI — Prompt caching** — OpenAI's equivalent. `https://platform.openai.com/docs/guides/prompt-caching`.
- **Google — Context caching for Gemini** — Google's equivalent. `https://ai.google.dev/gemini-api/docs/caching`. <!-- needs-research: confirm URL. -->

## Chapter 04 — Sampled online eval loop with judge-drift monitor

### Online / production evaluation frameworks

- **Braintrust — Online scoring documentation** — the hosted-runner path for live-traffic scoring. `https://www.braintrust.dev/docs/guides/online-scoring`. <!-- needs-research: confirm URL. -->
- **Langfuse — Online evaluations documentation** — the OSS-runner path. `https://langfuse.com/docs/scores/model-based-evals`. <!-- needs-research: confirm URL. -->
- **LangSmith — Online evaluators** — the hosted equivalent. `https://docs.smith.langchain.com/observability/how_to_guides/online_evaluations`. <!-- needs-research: confirm URL. -->
- **Arize Phoenix — Online evaluation** — Phoenix's approach to live scoring. `https://arize.com/docs/phoenix/evaluation/concepts-evals/llm-as-a-judge`. <!-- needs-research: confirm URL. -->

### Sampling and stratification

- **Alex Strick van Linschoten — Sampling for LLM evaluation** — practitioner writeup on stratified sampling for online eval. `https://mlops.systems/posts/2024-06-16-sampling-for-llm-evaluation.html`. <!-- needs-research: confirm URL and date. -->
- **Google SRE Workbook — Monitoring** — background reference on sampling design for high-volume signals; the intuition transfers to eval sampling. `https://sre.google/workbook/monitoring/`.

### Judge drift, calibration, and re-baselining

- **Zheng et al. — *Judging LLM-as-a-Judge*** (re-cited) — the calibration-panel discipline traces back to Section 4 on human agreement. `https://arxiv.org/abs/2306.05685`.
- **Shankar et al. — *Who Validates the Validators?*** (re-cited) — a working reference for the applied-engineer's calibration-panel workflow. `https://arxiv.org/abs/2404.12272`.
- **Anthropic — Model deprecation policy** — the provider-side contract for how long a pinned judge model stays available. `https://docs.anthropic.com/en/docs/resources/model-deprecations`. <!-- needs-research: confirm URL. -->
- **OpenAI — Deprecations** — the parallel contract for OpenAI-hosted judge models. `https://platform.openai.com/docs/deprecations`.
- **Google — Gemini model deprecations** — the parallel contract for Google-hosted judge models. `https://ai.google.dev/gemini-api/docs/models/deprecations`. <!-- needs-research: confirm URL. -->
- **Hamel Husain — LLM Eval Cookbook: judge drift section** — practitioner notes on judge upgrades. `https://hamel.dev/blog/posts/evals/`. <!-- needs-research: pin the specific post that covers judge drift. -->

### Alerting and on-call substrate

- **Google SRE Book — Practical Alerting** — the canonical reference for the "sustained aggregate drop on named segments" alert design. `https://sre.google/sre-book/practical-alerting/`.
- **Prometheus — Alerting rules** — the substrate many teams write these rules on. `https://prometheus.io/docs/alerting/latest/overview/`.
- **Grafana — Alerting documentation** — the alternative substrate. `https://grafana.com/docs/grafana/latest/alerting/`.
- **PagerDuty — Documentation** — the on-call notification substrate. `https://support.pagerduty.com/main/docs`.
- **Charity Majors — On observability, incidents, and operational maturity** — practitioner writing on alert-design discipline that transfers to online eval. `https://charity.wtf/tag/incidents/`.

### Cost accounting for eval spend

- **OpenAI — Usage dashboard and cost API** — provider-side spend visibility for the judge budget. `https://platform.openai.com/docs/guides/production-best-practices/setting-up-your-organization`. <!-- needs-research: confirm URL. -->
- **Anthropic — Usage and cost tracking** — parallel provider-side surface. `https://docs.anthropic.com/en/docs/build-with-claude/token-counting`. <!-- needs-research: confirm URL. -->
- **OpenMeter / Helicone / Portkey** — third-party LLM cost-tracking surfaces sometimes used alongside the eval-runner store. `https://www.openmeter.io/`, `https://www.helicone.ai/`, `https://portkey.ai/`. <!-- needs-research: confirm current URLs. -->

## Chapter 05 — Cross-persona feature dashboards

### Dashboarding tools

- **Grafana — Documentation** — the canonical OSS dashboard tool, referenced for the on-call surface. `https://grafana.com/docs/grafana/latest/`.
- **Metabase — Documentation** — a BI-tool option often used for PM-facing dashboards. `https://www.metabase.com/docs/latest/`.
- **Looker Studio (Google) — Documentation** — hosted BI alternative. `https://support.google.com/looker-studio/answer/6283323`.
- **Notion — Embedded databases and pages** — the surface many PMs actually live in. `https://www.notion.so/help/intro-to-databases`.
- **Superset (Apache)** — the OSS BI tool sometimes used as a Metabase alternative. `https://superset.apache.org/docs/intro`.
- **Runner-native dashboards** — every named runner in chapter 03 (Braintrust, Langfuse, LangSmith, Phoenix, Weave) ships a dashboard surface; the "meet the persona in their tool" discipline sometimes lands there rather than in a BI tool.

### Dashboard design references

- **Stephen Few — *Information Dashboard Design*** — the canonical practitioner book on dashboard design, applicable across personas. Not free; well worth the price. `https://www.perceptualedge.com/library.php`. <!-- needs-research: confirm current publisher URL. -->
- **Nielsen Norman Group — Dashboard design articles** — practitioner reference on chart legibility, scanning patterns, and drill-down design. `https://www.nngroup.com/topic/dashboards/`. <!-- needs-research: confirm URL. -->
- **Google SRE Book — Being on-call** — the reader-persona framing for the on-call dashboard. `https://sre.google/sre-book/being-on-call/`.
- **Google SRE Workbook — Alerting on SLOs** — the framing that ties the on-call dashboard to SLO burn. `https://sre.google/workbook/alerting-on-slos/`.

### Governance and launch-decision discipline

- **Google SRE Workbook — Postmortem culture** — the discipline the dashboard's incident-retro linkage participates in. `https://sre.google/workbook/postmortem-culture/`.
- **Google — Rules of Machine Learning: Rules on eval discipline** (re-cited) — includes the "chart the metric the launch decision depends on" discipline. `https://developers.google.com/machine-learning/guides/rules-of-ml`.
- **NIST AI RMF 1.0 — Measure and Manage functions** — the governance frame the safety-reviewer dashboard sits inside. `https://www.nist.gov/itl/ai-risk-management-framework`.

### Peer-role dashboards this chapter explicitly does NOT build

- **Bootstrap CIs / sequential testing** — Efron & Tibshirani, *An Introduction to the Bootstrap* is the canonical reference. `https://www.crcpress.com/An-Introduction-to-the-Bootstrap/Efron-Tibshirani/p/book/9780412042317`. <!-- needs-research: confirm current publisher URL. -->
- **Sequential testing in A/B experiments** — Johari et al., *Peeking at A/B Tests: Why It Matters, and What to Do About It* (KDD 2017) — the peer role's reference for sequential-test math. `https://dl.acm.org/doi/10.1145/3097983.3097992`. <!-- needs-research: confirm URL. -->
- **Judge-vs-human calibration heatmaps** — Cohen's kappa, Krippendorff's alpha, and the practitioner literature the peer role owns. `https://en.wikipedia.org/wiki/Cohen%27s_kappa`, `https://en.wikipedia.org/wiki/Krippendorff%27s_alpha`.
- **Comprehensive safety benchmarks** — HELM, MLCommons AILuminate, and the peer eval role's benchmark surface. `https://crfm.stanford.edu/helm/`, `https://mlcommons.org/benchmarks/ai-safety/`. <!-- needs-research: confirm AILuminate URL. -->

## Standards and topics previewed here, owned in other modules

- **Prompt discipline, structured outputs, rubric prompting patterns** consumed by the LLM-as-judge and rubric-grading modes in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) are developed in **mod-102**.
- **Retriever configuration and the retriever-lineage surface** pinned in [chapter 02](02-trace-snapshots-as-regression-fixtures.md) — plus the RAG-triad decomposition in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) — are developed in **mod-103**.
- **Tool schemas, tool-call replay, and the tool-catalogue hash** in [chapter 02](02-trace-snapshots-as-regression-fixtures.md) are developed in **mod-104**.
- **Agent-shaped eval fixtures** (τ-bench, SWE-bench Verified, WebArena, AgentDojo shape templates) referenced by [chapter 01](01-offline-eval-suites-a-pr-can-run.md)'s programmatic tier and [chapter 02](02-trace-snapshots-as-regression-fixtures.md)'s snapshot design are developed in **mod-105 chapter 04**.
- **Safety and guardrails** — the broader story the safety spot checks in [chapter 01](01-offline-eval-suites-a-pr-can-run.md) and the safety-reviewer dashboard in [chapter 05](05-cross-persona-feature-dashboards.md) narrow the scope of — is developed in **mod-107**.
- **Model routing decisions** captured in the trace lineage ([chapter 02](02-trace-snapshots-as-regression-fixtures.md)) and per-arm sampling in the online loop ([chapter 04](04-sampled-online-eval-loop.md)) are governed by **mod-108**.
- **Cost and latency SLOs** referenced across [chapter 01](01-offline-eval-suites-a-pr-can-run.md)'s pre-registration file and [chapter 04](04-sampled-online-eval-loop.md)'s alerting shape are developed in **mod-109**.
- **Trace collection, span attributes, and the observability substrate** the online loop in [chapter 04](04-sampled-online-eval-loop.md) samples from — and that the on-call dashboard in [chapter 05](05-cross-persona-feature-dashboards.md) queries — is developed in **mod-110**.
- **Feature-flag, canary, and rollback discipline** that the on-call dashboard's rollback panel ([chapter 05](05-cross-persona-feature-dashboards.md)) integrates with is developed in **mod-111**.
- **End-to-end ownership of a shipped feature** with this module's eval loop wired in is the capstone in **mod-112**.

## Peer-role handoffs

- **`ai-eval-engineer` role curriculum** — the peer role that owns judge-vs-human calibration methodology, bootstrap CIs, sequential testing, and the "how do we know the numbers mean what we think they mean" surface. The applied engineer *reads* those outputs; the peer role maintains them.
- **`model-evaluation-engineer` role curriculum** — the peer role that owns benchmark construction from first principles, cross-model comparability, and judge-model behavioural characterisation. The applied engineer's pre-registered judge pin (chapters 01, 04) is downstream of the peer role's judge-selection work.
- **`ai-safety-engineer` role curriculum** — the peer role that owns the comprehensive safety programme this module's safety spot checks (chapter 01) and safety-reviewer dashboard (chapter 05) are the applied-side view of.
