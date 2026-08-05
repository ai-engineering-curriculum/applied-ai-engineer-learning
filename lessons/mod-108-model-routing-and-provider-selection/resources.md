# Resources for mod-108-model-routing-and-provider-selection (Model Routing and Provider Selection)

Primary-source references — official documentation, project repositories, and canonical papers. Each entry names *why* the resource matters to the chapter it supports. Verify the exact URLs and version notes at the point of use; the resource landscape here moves faster than the module's revision cadence.

## Gateway architectures (chapter 01)

### LiteLLM

- **LiteLLM repository (BerriAI)** — the OSS proxy + Python SDK that normalises ~100 provider APIs to the OpenAI shape. Source of truth for the config surface, the router config, and the fallback / retry primitives referenced in chapter 01. <https://github.com/BerriAI/litellm>
- **LiteLLM proxy documentation** — the config-file shape used by production LiteLLM deployments. <https://docs.litellm.ai/>

### OpenRouter

- **OpenRouter documentation** — the managed marketplace gateway with a unified OpenAI-shape API across hundreds of models. Model catalogue, per-model pricing, and the request-level provider-preference parameters live here. <https://openrouter.ai/docs>
- **OpenRouter model list** — canonical source for the current model slugs used in route tables. <https://openrouter.ai/models>

### Portkey

- **Portkey AI Gateway (OSS repo)** — the self-hostable version of Portkey's gateway. Where the config DSL for fallbacks, caching, and guardrails is documented. <https://github.com/Portkey-AI/gateway>
- **Portkey documentation** — the managed offering, virtual-key issuance, and observability primitives. <https://docs.portkey.ai/>

### Hyperscaler-native

- **Amazon Bedrock documentation** — the unified API across Anthropic, Meta, Mistral, Cohere, Amazon Titan / Nova, and others; Bedrock Guardrails; PrivateLink and IAM patterns. <https://docs.aws.amazon.com/bedrock/>
- **Google Vertex AI Model Garden** — Gemini plus Claude on Vertex, Meta, Mistral, and open-weights catalogue; safety filters. <https://cloud.google.com/vertex-ai/docs>
- **Azure AI Foundry** — the successor to Azure OpenAI + Azure AI Studio; OpenAI models plus partner models under Azure IAM and residency. <https://learn.microsoft.com/en-us/azure/ai-foundry/>

## Routing, gating, and fallback (chapter 02)

- **LiteLLM router documentation** — the config-driven router primitives (`model_list`, `router_settings`, `fallbacks`, `context_window_fallbacks`) that map cleanly onto chapter 02's route table shape. <https://docs.litellm.ai/docs/routing>
- **RouteLLM (Ong et al., 2024) — "RouteLLM: Learning to Route LLMs with Preference Data"** — the canonical learned-routing paper referenced as Shape 1c (bandit / learned routing). Useful when a team's traffic volume justifies moving past static rules. arXiv: <https://arxiv.org/abs/2406.18665>
- **OpenAI structured outputs / JSON mode documentation** — the validators that drive the structural-validity quality gate. <https://platform.openai.com/docs/guides/structured-outputs>
- **Anthropic Messages API — tool use** — the tool-call shape referenced when contrasting Anthropic's `content: [{type: "tool_use", ...}]` against OpenAI's `tool_calls` array. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>

## Self-hosted inference (chapter 03)

- **vLLM** — PagedAttention, continuous batching, OpenAI-compatible HTTP server. Repository: <https://github.com/vllm-project/vllm> — documentation: <https://docs.vllm.ai/>
- **PagedAttention paper (Kwon et al., SOSP 2023) — "Efficient Memory Management for Large Language Model Serving with PagedAttention"** — the original vLLM paper. Useful for understanding why the throughput numbers in the baseline concurrency sweep look the way they do. <https://arxiv.org/abs/2309.06180>
- **Text Generation Inference (Hugging Face)** — TGI repository and documentation. Continuous batching, tensor parallelism, quantisation support. <https://github.com/huggingface/text-generation-inference>
- **SGLang** — RadixAttention, structured-output-first frontend, prefix caching. Repository: <https://github.com/sgl-project/sglang>
- **MLPerf Inference benchmarks (MLCommons)** — the reference frame `ai-infra-performance` publishes serving-perf numbers against. Consumed by applied engineers as a sanity check, not authored by them. <https://mlcommons.org/benchmarks/inference/>

## Multi-modal routing (chapter 04)

### Vision

- **OpenAI vision documentation** — image tokenisation, `detail: low/high` presets, and the request shape. <https://platform.openai.com/docs/guides/vision>
- **Anthropic vision (Messages API images)** — image encoding, size limits, and per-image cost. <https://docs.anthropic.com/en/docs/build-with-claude/vision>
- **Google Gemini vision** — multimodal input handling on the Gemini API. <https://ai.google.dev/gemini-api/docs/vision>
- **Azure AI Document Intelligence** — enterprise document-AI with structured field extraction. <https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/>
- **Google Document AI** — Google's document-AI service on Vertex. <https://cloud.google.com/document-ai/docs>
- **AWS Textract** — AWS document-AI for OCR and forms extraction. <https://docs.aws.amazon.com/textract/>

### Audio (STT / TTS)

- **OpenAI audio (Whisper / TTS)** — cloud Whisper for transcription and the TTS endpoint. <https://platform.openai.com/docs/guides/speech-to-text> and <https://platform.openai.com/docs/guides/text-to-speech>
- **Whisper (OpenAI research paper)** — Radford et al., "Robust Speech Recognition via Large-Scale Weak Supervision." <https://arxiv.org/abs/2212.04356>
- **Deepgram documentation** — streaming STT with sub-second first-partial, diarization, and language coverage. <https://developers.deepgram.com/docs/>
- **AssemblyAI Universal-2 documentation** — long-form transcription with diarization and speaker labels. <https://www.assemblyai.com/docs>
- **ElevenLabs documentation** — TTS and streaming audio primitives referenced as a common TTS route. <https://elevenlabs.io/docs>

### Image / video generation

- **OpenAI image generation documentation** — `gpt-image-1` and prior image models. <https://platform.openai.com/docs/guides/images>
- **Stable Diffusion / Stability AI documentation** — SDXL and successor OSS image models. <https://platform.stability.ai/docs>
- **Black Forest Labs Flux (repository)** — open-weights image generation model referenced as a route alternative. <https://github.com/black-forest-labs/flux>
- **Runway Gen-3 documentation** — video generation as a managed API. <https://docs.dev.runwayml.com/>
- **Google Veo (Vertex AI)** — video generation on Vertex. <https://cloud.google.com/vertex-ai/generative-ai/docs/video/generate-videos>

### Realtime voice

- **OpenAI Realtime API** — duplex WebSocket / WebRTC realtime voice, referenced as the primary realtime route in chapter 04. <https://platform.openai.com/docs/guides/realtime>
- **Hume Empathic Voice Interface (EVI)** — realtime voice with prosody / affect passthrough. <https://dev.hume.ai/docs/empathic-voice-interface-evi/overview>
- **ElevenLabs Conversational AI** — realtime voice agent stack. <https://elevenlabs.io/docs/conversational-ai>

## Contract testing and per-provider goldens (chapter 05)

- **OpenAI Chat Completions / Responses API reference** — the canonical response envelope shape most contract tests normalise to. <https://platform.openai.com/docs/api-reference>
- **Anthropic Messages API reference** — the response envelope tests compare against for Anthropic-fallback routes. <https://docs.anthropic.com/en/api/messages>
- **Google Gemini API reference** — the response shape and safety-rating envelope for Gemini-fallback routes. <https://ai.google.dev/api>
- **JSON Schema** — the standard the structural-validity gate and structured-output fixtures reference. <https://json-schema.org/specification>
- **OpenTelemetry Semantic Conventions for GenAI** — the emerging standard for the span attributes the router emits (model name, tokens, cost, latency). Consumed by the mod-110 observability substrate. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>

## Adjacent standards and cross-module references

- **MCP (Model Context Protocol)** — the tool-boundary standard that mod-104 chapter 03 develops; relevant here because MCP servers behind a gateway are part of the routing substrate. <https://modelcontextprotocol.io/>
- **EU AI Act — official journal text** — the residency / regional-serving envelope some routes have to honour. Consumed via `privacy-engineer` sign-off, not owned by the applied engineer. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- **NIST AI Risk Management Framework** — the outer envelope any residency / risk conversation lands in. <https://www.nist.gov/itl/ai-risk-management-framework>

## Peer-role handoffs (link outs to peer track material)

- **`ai-infra-performance`** — the track that owns the deep serving-perf work (vLLM / TGI / SGLang tuning, continuous batching, KV-cache paging, tensor / pipeline / expert parallelism, quantisation quality vs throughput). The applied engineer reads the baseline; the peer role produces it.
- **`ml-platform-engineer`** — GPU capacity planning, autoscaling, spot / reserved / on-demand mix, disaster-recovery for the inference tier.
- **`model-evaluation-engineer`** — the deeper benchmark-shape eval suites; the contract tests in chapter 05 are the thin pipe-test layer, this role's suites are the deep quality bar.
- **`privacy-engineer`** and **`data-security-engineer`** — the residency envelope the gateway and self-host decisions serve.
- **`ai-security-engineer`** — third-party API dependency posture, data-flow reviews, DPA management.
- **`sre` / `platform`** — on-call for any self-hosted inference tier, and the release-gating substrate the contract report feeds (mod-111).
