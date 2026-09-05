# Resources for mod-110-observability-feedback-and-incidents (Observability, User Feedback, and Incident Response for LLM Features)

Primary-source references — official standards, provider documentation, SRE canon, and canonical tooling. Each entry names *why* the resource matters to the chapter it supports. Verify the exact URLs and API shapes at the point of use; the OTel GenAI semantic conventions and the provider docs move faster than the module's revision cadence.

## OTel Gen-AI + OpenInference span vocabulary (chapter 01)

### OpenTelemetry (vendor-neutral)

- **OpenTelemetry Semantic Conventions for Generative AI (`gen_ai.*`)** — the working attribute set for LLM operations (system, request/response model, operation name, usage, response ID). Status of individual attributes / events / metrics is **experimental** and moves faster than a normal spec cycle; verify the stability status of each attribute against the current release. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry Semantic Conventions for Gen-AI — events** — the event-based carrier for prompt / response content (`gen_ai.content.prompt`, `gen_ai.content.completion`), covered where chapter 01 discusses payload-carrier choices. <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/>
- **OpenTelemetry Semantic Conventions for Gen-AI — metrics** — the metrics side of the GenAI semconv (usage counters, latency histograms). <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/>
- **OpenTelemetry tracing specification** — the underlying span, attribute, event, and context-propagation model any provider SDK plugs into. <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenTelemetry Python SDK** — the reference SDK used in chapter 01's wrapper code shape. <https://opentelemetry.io/docs/languages/python/>

### OpenInference (Arize / Phoenix)

- **OpenInference specification** — the LLM-app-shaped semconv (LLM / RETRIEVER / TOOL / EMBEDDING / CHAIN / GUARDRAIL span kinds; `llm.input_messages.*`, `retrieval.documents.*.document.score`, `tool.parameters`). <https://github.com/Arize-ai/openinference/tree/main/spec>
- **OpenInference Python instrumentation** — the language-side auto-instrumentation packages that emit OpenInference attributes on top of an OTel SDK. <https://github.com/Arize-ai/openinference/tree/main/python>
- **Arize Phoenix** — open-source LLM observability tool that renders OpenInference traces (used as a local trace backend in exercise-01). <https://github.com/Arize-ai/phoenix>

## Sampling, PII scrubbing, and the trace budget (chapter 02)

### Sampling substrate

- **OpenTelemetry Collector — Tail Sampling Processor** — the tail sampler chapter 02 assumes (keep on error, keep on latency, per-attribute policies). <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>
- **OpenTelemetry Collector — Probabilistic Sampling Processor** — head sampling in the collector (an alternative to SDK-side head sampling). <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor>
- **OpenTelemetry — Sampling (concepts)** — the head-vs-tail sampling primer that grounds the pattern in chapter 02. <https://opentelemetry.io/docs/concepts/sampling/>

### PII scrubbing

- **OpenTelemetry Collector — Attributes Processor** — the standard export-boundary attribute-mutation stage (delete, hash, extract, redact). <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor>
- **OpenTelemetry Collector — Redaction Processor** — an OTel-Contrib processor specifically for allow-list-based attribute redaction on export. <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor>
- **Microsoft Presidio** — an open-source PII detection and anonymisation library covering named entities, phone numbers, emails, credit cards, and custom recognisers. <https://microsoft.github.io/presidio/>
- **NIST SP 800-122 — Guide to Protecting the Confidentiality of PII** — canonical taxonomy of PII sensitivity levels the scrub matrix maps onto. <https://csrc.nist.gov/pubs/sp/800/122/final>
- **GDPR — Article 32 (Security of processing)** — the DPA-facing legal basis for scrubbing before export to a data processor. <https://gdpr-info.eu/art-32-gdpr/>

### Trace-budget references

- **OpenTelemetry Semantic Conventions for GenAI (metrics)** — for the token-count metrics the trace budget's cost-per-span estimate is calibrated against. <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/>

## User feedback triage: capture, dedup, expert review (chapter 03)

### Annotation platforms

- **Argilla — documentation** — open-source LLM feedback / annotation platform used in exercise-03 for expert review; supports records, suggestions, responses, and a Python SDK. <https://argilla.io/> and <https://docs.argilla.io/>
- **Argilla GitHub repository** — the source, examples, and deployment instructions. <https://github.com/argilla-io/argilla>
- **Label Studio — documentation** — mature open-source alternative supporting text-classification, ML-backend suggestions, and workspace / project structure. <https://labelstud.io/guide/>
- **Label Studio ML backend** — the extension point for LLM-as-judge suggestions in the annotation UI (referenced in exercise-03's stretch goal). <https://labelstud.io/guide/ml.html>

### Embedding-based deduplication

- **`sentence-transformers`** — the small-embedding-model library referenced for comment dedup in exercise-03. <https://www.sbert.net/>
- **`sentence-transformers/all-MiniLM-L6-v2`** — a common small model for CPU-only comment embedding. <https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2>

### Feedback → eval dataset

- The **eval-fixture schema** the pipeline exports into is defined in the paired module, [mod-106 chapter 01 — Offline eval suites a PR can run](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md).
- The **online eval loop** whose fixtures the triage entries seed is in [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md).

## Incident playbooks for the six LLM failure modes (chapter 04)

### SRE canon

- **Google SRE Book** — the canonical volume; source of the incident-response vocabulary (severity, triage, mitigation, root cause, prevention) that the LLM playbooks specialise. <https://sre.google/sre-book/table-of-contents/>
- **Google SRE Book — Managing Incidents** — the specific chapter chapter 04's five-part shape maps onto. <https://sre.google/sre-book/managing-incidents/>
- **Google SRE Workbook — Postmortem Culture** — the postmortem discipline the game-day report (exercise-04) is designed to produce. <https://sre.google/workbook/postmortem-culture/>
- **Google SRE Book — Postmortems (blameless)** — the source pattern for the postmortem review process. <https://sre.google/sre-book/postmortem-culture/>

### Provider status pages (subscribe for outage detection)

- **OpenAI status** — canonical status source for OpenAI API availability. <https://status.openai.com/>
- **Anthropic status** — canonical status source for the Claude API. <https://status.anthropic.com/>
- **AWS Health / Bedrock** — canonical status source for Bedrock region availability. <https://health.aws.amazon.com/health/status>
- **Google Cloud status (Vertex AI, Gemini)** — canonical status source for Vertex AI and Gemini availability. <https://status.cloud.google.com/>
- **Microsoft Azure status (Azure OpenAI)** — canonical status source for Azure OpenAI availability. <https://status.azure.com/status>

### Judge drift and quality regression

- **`ai-infra-mlops` and `model-evaluation-engineer` peer references** — the judge-calibration methodology chapter 04's Playbook 3 monitors is developed in [mod-106 chapters 02 and 04](../mod-106-feature-side-evaluation/README.md) of this track.
- **Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"** — the canonical paper on LLM-as-judge calibration; the substrate the drift alert reads from. <https://arxiv.org/abs/2306.05685>

### Guardrail false-positive spike

- The **guardrail vocabulary and rollback discipline** live in [mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md) — the FP-spike playbook rolls back the guardrail config defined there.
- **Meta Llama Guard** — commonly-deployed open guardrail whose false-positive rate is a monitored SLI when it is in production. <https://github.com/meta-llama/PurpleLlama/tree/main/Llama-Guard3>

## Per-feature status page and on-call runbook (chapter 05)

### Status-page substrates

- **Statuspage (Atlassian)** — the incumbent hosted-status-page product; component / incident model referenced in chapter 05. <https://www.atlassian.com/software/statuspage>
- **Instatus** — a modern hosted status-page alternative. <https://instatus.com/>
- **Cachet (self-hosted)** — an open-source self-hostable status-page substrate. <https://cachethq.io/>

### Paging tools

- **PagerDuty** — canonical incident-response / paging platform; alert-to-runbook links, escalation policies. <https://www.pagerduty.com/docs/>
- **Opsgenie (Atlassian)** — alternative paging platform. <https://support.atlassian.com/opsgenie/>
- **Grafana OnCall** — Grafana Labs' open-source paging tool. <https://grafana.com/docs/oncall/latest/>
- **Rootly** — modern incident-management + paging platform. <https://docs.rootly.com/>
- **incident.io** — modern incident-management platform; declaration flow referenced in chapter 05. <https://incident.io/>
- **FireHydrant** — incident-management platform with a runbook engine. <https://docs.firehydrant.com/>

### Alerting substrates

- **Prometheus — Alerting Rules** — the rule syntax the alert examples in chapter 05 use. <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>
- **Alertmanager** — the routing / silencing / grouping half of the Prometheus alerting stack. <https://prometheus.io/docs/alerting/latest/alertmanager/>
- **Grafana Alerting** — the Grafana-side alerting model. <https://grafana.com/docs/grafana/latest/alerting/>
- **Datadog Monitors** — the Datadog alert primitive with `runbook_url` and playbook annotations. <https://docs.datadoghq.com/monitors/>

### SLO and burn-rate references

The SLO substrate the status-page colour derives from lives in [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md). Key upstream references:

- **Google SRE Workbook — Alerting on SLOs** — multi-window multi-burn-rate alert pattern (14.4× / 1h fast, 6× / 6h slow). <https://sre.google/workbook/alerting-on-slos/>
- **Google SRE Book — Service Level Objectives** — the SLI / SLO / error-budget vocabulary. <https://sre.google/sre-book/service-level-objectives/>

## Adjacent standards

- **NIST AI Risk Management Framework** — the outer envelope for risk-and-safety conversations around LLM feature ownership and incident classification. <https://www.nist.gov/itl/ai-risk-management-framework>
- **NIST SP 800-61 — Computer Security Incident Handling Guide** — a canonical incident-handling process reference that the LLM-feature playbooks specialise, particularly for security-adjacent incidents (jailbreak, prompt-injection escalation). <https://csrc.nist.gov/pubs/sp/800/61/r2/final>
- **EU AI Act — official text** — regional-serving envelope constraining what a provider-outage fallback playbook may do. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>

## Peer-role handoffs (link-outs to peer-track material)

- **`ai-infra-mlops`** — owns the alerting engine, paging tool, status-page hosting, incident-management substrate, and metrics/traces backend. Chapter 05 names the six interfaces explicitly.
- **`model-evaluation-engineer`** — owns the offline / online eval regime the feedback pipeline (chapter 03) feeds and the judge that the drift playbook (chapter 04, Playbook 3) monitors. See [mod-106](../mod-106-feature-side-evaluation/README.md).
- **`sre` / `platform`** — owns the on-call model and org-wide incident process the LLM-feature runbook slots into.
- **`privacy-engineer`** and **`data-security-engineer`** — own the DPA-scope constraints on what leaves the process boundary. The PII scrub matrix in chapter 02 is the applied-side implementation of what they define.
- **`ai-security-engineer`** — reads the per-provider attribution on the spans (chapter 01) to keep the data-flow / DPA story current across providers and regions.
