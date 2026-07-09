# Resources for mod-102 — Production Prompting and Structured Outputs

Curated primary sources for the topics covered in the chapters. Standards and provider documentation first; secondary references are named as such. Cross-provider details (caching semantics, JSON Schema subsets, streaming event shapes) change on a monthly cadence — always consult the current live docs before pinning behaviour.

## Prompt caching — the four majors ([chapter 01](01-cache-aware-prompt-design.md))

- **OpenAI — *Prompt Caching* guide** — automatic prefix caching, minimum shared-prefix size, cached-token accounting on the response. `https://platform.openai.com/docs/guides/prompt-caching`. <!-- needs-research: verify the current URL and confirm the minimum-tokens number and TTL in the live guide. -->
- **Anthropic — *Prompt caching* docs** — explicit `cache_control` breakpoints (up to 4), ephemeral TTL defaults, longer-TTL opt-in, write premium and read discount. `https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching`.
- **Google Gemini — *Context caching* (Vertex AI)** — `CachedContent` resource, name-referenced, per-hour storage billing, minimum tokens. `https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview`.
- **AWS Bedrock — *Prompt caching*** — model-specific caching semantics on Bedrock (Claude uses Anthropic breakpoints; other model families vary). `https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html`. <!-- needs-research: confirm which model families currently support prompt caching on Bedrock and whether the semantics have converged across them. -->
- **Anthropic Cookbook — *Prompt caching examples*** — worked examples of long-context caching in Python / TypeScript SDKs. `https://github.com/anthropics/anthropic-cookbook`.

## JSON mode, function calling, and Structured Outputs ([chapter 02](02-schema-safe-structured-outputs.md))

- **OpenAI — *Structured Outputs* docs** — `response_format={"type": "json_schema", ...}` with `strict: true`, the supported JSON Schema subset, and function-calling `strict` mode. `https://platform.openai.com/docs/guides/structured-outputs`.
- **OpenAI — *Function calling* guide** — `tools` parameter, tool-call return shape, best practices for tool definitions. `https://platform.openai.com/docs/guides/function-calling`.
- **Anthropic — *Tool use* docs** — declaring tools, receiving tool-use blocks, providing tool results back, streaming shapes. `https://docs.anthropic.com/en/docs/build-with-claude/tool-use`.
- **Google Gemini — *Function calling* docs** — Gemini's `FunctionDeclaration` and `FunctionCall` shape. `https://ai.google.dev/gemini-api/docs/function-calling`.
- **AWS Bedrock — *Converse API tool use*** — the model-neutral tool-use surface on Bedrock's Converse API. `https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html`.
- **Instructor** — Pydantic-model-based structured outputs adapter for OpenAI, Anthropic, and other SDKs, with retry-on-validation-error. `https://python.useinstructor.com/` (Python) and `https://instructor-ai.github.io/instructor-js/` (JavaScript / TypeScript).
- **Outlines** — grammar / regex / JSON-schema constrained decoding on self-hosted models. `https://dottxt-ai.github.io/outlines/`.
- **Pydantic v2 docs** — validators, `@model_validator`, `@field_validator`, `Field(pattern=...)`, discriminated unions. `https://docs.pydantic.dev/latest/`.
- **`partial-json-parser`** — permissive JSON parser for progressively rendered streaming JSON. `https://github.com/promplate/partial-json-parser` (Python) and the JS port linked from the same repo.

## Constrained decoding on self-hosted inference

- **vLLM — *Guided decoding*** — vLLM's structured / guided output support (Outlines, xgrammar backends). `https://docs.vllm.ai/en/latest/features/structured_outputs.html`. <!-- needs-research: verify current URL and the list of guided-decoding backends supported by the latest vLLM release. -->
- **SGLang — *Structured outputs / constrained generation*** — SGLang's structured decoding surface. `https://docs.sglang.ai/backend/structured_outputs.html`. <!-- needs-research: pin current URL for SGLang's structured-outputs docs. -->
- **Text Generation Inference (TGI) — *Guidance*** — Hugging Face TGI's guidance / grammar support. `https://huggingface.co/docs/text-generation-inference/index`.
- **`llguidance` and `xgrammar`** — grammar engines used by modern inference servers. `https://github.com/microsoft/llguidance` and `https://github.com/mlc-ai/xgrammar`.

## Streaming, SSE, and WebSocket ([chapter 03](03-streaming-and-cancellation.md))

- **MDN — *Using server-sent events*** — canonical browser-side SSE reference. `https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events`.
- **WHATWG HTML Living Standard — *Server-sent events*** — the SSE spec. `https://html.spec.whatwg.org/multipage/server-sent-events.html`.
- **RFC 6455 — *The WebSocket Protocol*** — the protocol spec. `https://www.rfc-editor.org/rfc/rfc6455`.
- **OpenAI — *Streaming API responses*** — the current streaming guide (Chat Completions and Responses API). `https://platform.openai.com/docs/guides/streaming-responses`. <!-- needs-research: confirm canonical URL for the current streaming guide across Chat Completions and Responses; OpenAI has separated the two guides recently. -->
- **Anthropic — *Streaming Messages* docs** — event types (`message_start`, `content_block_delta` with `input_json_delta`, etc.) and SDK patterns. `https://docs.anthropic.com/en/api/messages-streaming`.
- **Starlette — *StreamingResponse* and *EventSourceResponse* (via `sse-starlette`)** — async SSE on ASGI. `https://www.starlette.io/responses/#streamingresponse` and `https://github.com/sysid/sse-starlette`.
- **FastAPI — *Streaming responses*** — practical streaming with FastAPI on top of Starlette. `https://fastapi.tiangolo.com/advanced/custom-response/`.
- **MDN — *AbortController*** — client-side cancellation of `fetch` and `EventSource`. `https://developer.mozilla.org/en-US/docs/Web/API/AbortController`.

## Retry, repair, and self-correction ([chapter 04](04-retry-repair-and-fallback.md))

- **Google SRE Book — *Chapter 22: Addressing Cascading Failures*** — the reasoning behind bounded retry, backoff-with-jitter, and circuit-breaker patterns. `https://sre.google/sre-book/addressing-cascading-failures/`.
- **AWS Architecture Blog — *Exponential Backoff and Jitter*** — the canonical "jitter is not optional" post; the base of most retry-with-jitter implementations. `https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/`.
- **Instructor — *Retries* docs** — `max_retries` and validation-error-feedback shape. `https://python.useinstructor.com/concepts/retrying/`.
- **Anthropic — *Handling API errors and retries*** — provider-side retry recommendations and idempotency guidance. `https://docs.anthropic.com/en/api/errors`.
- **OpenAI — *Error handling and rate limits*** — status codes, `Retry-After`, `x-request-id`. `https://platform.openai.com/docs/guides/error-codes` and `https://platform.openai.com/docs/guides/rate-limits`.

## Prompt versioning, prompt registries, and rollback ([chapter 05](05-prompts-as-versioned-artefacts.md))

- **AWS Bedrock — *Prompt management*** — Bedrock's provider-native prompt-versioning surface. `https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html`. <!-- needs-research: confirm canonical URL for Bedrock Prompt Management. -->
- **Google Vertex AI — *Prompt management / Prompt Optimizer*** — Vertex's prompt versioning and optimisation surface. `https://cloud.google.com/vertex-ai/generative-ai/docs/learn/prompts/prompt-optimizer`. <!-- needs-research: pin the current canonical URL for the Vertex prompt-management surface. -->
- **LangSmith — *Prompt Hub* / *Prompts* docs** — LangChain's hosted prompt registry with versioning and eval integration. `https://docs.smith.langchain.com/prompt_engineering`.
- **LangFuse — *Prompts* docs** — open-source LLM engineering platform with prompt versioning and rollout. `https://langfuse.com/docs/prompts/get-started`.
- **PromptLayer — *Prompt Registry*** — prompt-versioning-focused product with A/B testing and rollback. `https://docs.promptlayer.com/features/prompt-registry`.
- **Weights & Biases Weave — *Prompt management*** — W&B's LLM observability + prompt versioning surface. `https://weave-docs.wandb.ai/`. <!-- needs-research: confirm the current URL for Weave's prompt-management docs. -->
- **Humanloop and Vellum** — commercial prompt-management + eval platforms; useful references for what a "full" registry offers. `https://humanloop.com/docs` and `https://docs.vellum.ai/`.

## Provider SDKs cited in code samples

- **OpenAI Python SDK** — `https://github.com/openai/openai-python`.
- **OpenAI Node SDK** — `https://github.com/openai/openai-node`.
- **Anthropic Python SDK** — `https://github.com/anthropics/anthropic-sdk-python`.
- **Anthropic TypeScript SDK** — `https://github.com/anthropics/anthropic-sdk-typescript`.
- **Google Generative AI Python SDK / Vertex AI SDK** — `https://github.com/googleapis/python-aiplatform` and `https://github.com/googleapis/python-genai`.
- **AWS SDK (boto3) Bedrock Runtime** — `https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime.html`.

## Streaming and rendering references for the UI hop

- **`marked`** — Markdown parser with streaming-friendly options in JavaScript. `https://marked.js.org/`.
- **`mistune`** — permissive Markdown parser in Python. `https://mistune.lepture.com/`.
- **React docs — *useTransition* and *Suspense* for streaming UI patterns** — background on rendering under partial data. `https://react.dev/reference/react/useTransition`.

## Secondary and background reading

- **Simon Willison — *Structured data extraction from unstructured content using LLM Schemas*** — practitioner writeups on structured-output pipelines. `https://simonwillison.net/tags/structuredextraction/`.
- **Simon Willison — *Prompt injection* series** — surface-side prompt-injection reading relevant to sanitised repair messages (chapter 04) and adversarial fixtures (chapter 05). `https://simonwillison.net/tags/prompt-injection/`.
- **Karpathy — *Software 2.0*** — the framing for treating prompts as versioned artefacts alongside code. `https://karpathy.medium.com/software-2-0-a64152b37c35`.

## Standards previewed here, owned in other modules

- **NIST AI Risk Management Framework (AI RMF 1.0)** — the Measure and Manage functions map to the eval-and-rollback loop in chapter 05; owned in mod-106 and mod-110. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **OWASP Top 10 for Large Language Model Applications** — surface-side taxonomy relevant to the sanitised repair message (chapter 04); owned in mod-107. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
