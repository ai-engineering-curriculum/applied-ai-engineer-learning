# Resources for mod-109-cost-latency-slos-for-llm-features (Cost and Latency SLOs for LLM Features)

Primary-source references — official provider documentation, SRE canon, and canonical papers. Each entry names *why* the resource matters to the chapter it supports. Verify the exact URLs, pricing numbers, and API shapes at the point of use; provider docs move faster than the module's revision cadence.

## App-altitude latency instrumentation (chapter 01)

- **OpenTelemetry Semantic Conventions for Generative AI** — the emerging standard for span attributes (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*`, and evolving latency attributes). The observability substrate the mod-110 pipeline consumes. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry tracing specification** — the underlying span / attribute / propagation model any provider SDK plugs into. <https://opentelemetry.io/docs/specs/otel/trace/>
- **OpenAI streaming responses documentation** — the SSE chunk shape used to time TTFT and TPOT. <https://platform.openai.com/docs/api-reference/streaming>
- **Anthropic Messages API — streaming** — the event stream Anthropic emits during a streamed completion; TTFT is measured from the first `content_block_delta` event. <https://docs.anthropic.com/en/api/messages-streaming>
- **Google Gemini API — streaming generation** — the SSE / grpc stream shape for streaming Gemini responses. <https://ai.google.dev/gemini-api/docs/text-generation>
- **Dean & Barroso, "The Tail at Scale," CACM 2013** — the canonical paper on tail latency in large-scale services; foundation for hedging (chapter 05) and for the "measure percentiles not averages" discipline in chapter 01. <https://research.google/pubs/the-tail-at-scale/>

## Provider prompt caching (chapter 02)

### OpenAI
- **Prompt caching (OpenAI)** — automatic prefix caching; the `cached_tokens` field returned in `usage.prompt_tokens_details`. <https://platform.openai.com/docs/guides/prompt-caching>
- **OpenAI pricing page** — canonical source for base input, cached input, and output rates per model. <https://openai.com/api/pricing/>

### Anthropic
- **Prompt caching (Anthropic)** — explicit `cache_control` breakpoints, `cache_creation_input_tokens` and `cache_read_input_tokens` accounting, 5-minute base TTL and extended-TTL beta. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **Anthropic pricing page** — base input, cache-write, cache-read, and output rates per model. <https://www.anthropic.com/pricing>

### AWS Bedrock
- **Bedrock prompt caching** — the `cachePoint` marker in the Converse API and provider-native cache_control for Anthropic-on-Bedrock. <https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>
- **Bedrock pricing page** — per-model pricing including cache-read and cache-write rates where applicable. <https://aws.amazon.com/bedrock/pricing/>

### Google Gemini / Vertex AI
- **Context caching (Gemini API)** — explicit cache objects with user-controlled TTL; per-hour storage plus reduced per-token read rate. <https://ai.google.dev/gemini-api/docs/caching>
- **Vertex AI context caching** — the same feature on the Vertex AI enterprise surface. <https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview>

## Batch APIs and offline workloads (chapter 03)

- **OpenAI Batch API** — endpoint reference, JSONL input shape, poll semantics, 24-hour completion window, and pricing discount vs the synchronous endpoints. <https://platform.openai.com/docs/guides/batch>
- **OpenAI Batch API reference** — the exact request / response envelope for `POST /v1/batches` and the input-file format. <https://platform.openai.com/docs/api-reference/batch>
- **Anthropic Message Batches API** — the equivalent batch shape on the Anthropic Messages API. <https://docs.anthropic.com/en/docs/build-with-claude/message-batches>
- **Amazon Bedrock batch inference** — the S3-in / S3-out `ModelInvocationJob` shape for Bedrock batch. <https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html>
- **Google Vertex AI batch prediction** — BigQuery / GCS-source batch prediction on Vertex, including for Gemini. <https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/batch-prediction-api>

## Per-feature and per-tenant cost accounting (chapter 04)

- **OpenTelemetry Semantic Conventions for GenAI — usage attributes** — the standard names for input / output / cached-input token counts on spans. See the GenAI semconv link above.
- **OpenAI usage API** — the account-level usage endpoint used for reconciliation between your ledger and the provider bill. <https://platform.openai.com/docs/api-reference/usage>
- **Anthropic usage / cost reporting** — where to pull per-key or per-workspace spend for reconciliation. <https://docs.anthropic.com/en/api/organization-member-management>
- **AWS Cost Explorer / AWS Billing** — the substrate for Bedrock invoice reconciliation. <https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-what-is.html>
- **GCP Billing Export to BigQuery** — the substrate for Vertex AI / Gemini invoice reconciliation. <https://cloud.google.com/billing/docs/how-to/export-data-bigquery>
- **FinOps Foundation — FinOps Framework** — the vocabulary (unit economics, cost per outcome, showback / chargeback) that per-tenant accounting maps onto for finance stakeholders. <https://www.finops.org/framework/>

## Retries, hedging, SLOs, and burn-rate alerts (chapter 05)

### Retry and hedging
- **Google SRE Book — chapter on handling overload / retry budgets** — the source pattern for the retry budget in chapter 05. <https://sre.google/sre-book/handling-overload/>
- **gRPC A6 — Client Retries** — the design doc formalising exponential backoff, jitter, and retry budgets in gRPC clients. <https://github.com/grpc/proposal/blob/master/A6-client-retries.md>
- **AWS Architecture Blog — "Exponential Backoff And Jitter"** — the definitive short piece on why jitter matters. <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>
- **Dean & Barroso, "The Tail at Scale," CACM 2013** — the canonical hedging reference. <https://research.google/pubs/the-tail-at-scale/>

### SLOs and burn-rate alerts
- **Google SRE Workbook — chapter 5, "Alerting on SLOs"** — the multi-window multi-burn-rate alerting pattern that chapter 05 uses (14.4× / 1h fast, 6× / 6h slow). <https://sre.google/workbook/alerting-on-slos/>
- **Google SRE Book — chapter 4, "Service Level Objectives"** — the source vocabulary (SLI, SLO, error budget). <https://sre.google/sre-book/service-level-objectives/>
- **Prometheus — recording rules and alerting** — the practical stack most burn-rate alerts are configured against. <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>

### MLPerf and serving benchmarks (the reference frame consumed by applied engineers)
- **MLPerf Inference (MLCommons)** — the industry-standard serving-benchmark suite; the reference frame `ai-infra-performance` and `model-evaluation-engineer` publish against and applied engineers consume for plausibility checks. <https://mlcommons.org/benchmarks/inference/>
- **MLPerf Inference results archive** — dated result rounds with per-hardware / per-model throughput and latency. <https://mlcommons.org/benchmarks/inference-datacenter/>

## Adjacent standards and cross-module references

- **OpenTelemetry Semantic Conventions for GenAI (metrics)** — the emerging metrics side of the GenAI semconv, complements the tracing conventions used in chapter 01 and chapter 04. <https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/>
- **NIST AI Risk Management Framework** — the outer envelope that residency / risk conversations around provider swaps land in; consumed via `privacy-engineer` sign-off. <https://www.nist.gov/itl/ai-risk-management-framework>
- **EU AI Act — official text** — the regional-serving envelope that constrains where hedged / fallback routes may go. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>

## Peer-role handoffs (link-outs to peer track material)

- **`ai-infra-performance`** — owns MLPerf-shape benchmarking, continuous-batching schedulers, KV-cache paging, quantisation quality-vs-throughput trade-offs. The applied engineer consumes their baseline as the reference frame in chapter 05.
- **`model-evaluation-engineer`** — owns the quality evaluation regime the SLI's `structural_validity` and `outcome` fields read from; see mod-106 for the eval-suite substrate.
- **`sre` / `platform`** — owns the alerting substrate (Prometheus / Alertmanager, Datadog, PagerDuty, etc.) that burn-rate alerts land in and the on-call rotation that responds.
- **`privacy-engineer`** and **`data-security-engineer`** — own the residency envelope any hedge or fallback provider swap must respect; the routing substrate is in mod-108.
- **`ai-security-engineer`** — reads the per-provider cost attribution (chapter 04) to keep the data-flow / DPA story current.
- **Finance / cost-owner (organisational)** — reads the CPSO and per-tenant dashboards; co-owns the monthly reconciliation ritual with the applied engineer.
