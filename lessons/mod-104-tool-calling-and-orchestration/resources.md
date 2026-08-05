# Resources for mod-104 — Tool Calling and Orchestration at Product Altitude

Curated primary sources for the topics covered in the chapters. Standards, provider documentation, and canonical open-source projects first; secondary references named as such. Tool-use APIs, MCP versions, and agent frameworks change on a monthly cadence — always consult the current live docs before pinning behaviour.

## Provider tool-use / function-calling APIs (context for [chapter 01](01-tool-schema-and-side-effect-boundaries.md))

- **Anthropic — Tool use with Claude** — the canonical shape for `tools[i].input_schema`, tool-use content blocks, and `tool_result`. `https://docs.anthropic.com/en/docs/build-with-claude/tool-use`.
- **OpenAI — Function calling (Chat Completions)** — `tools[i].function.parameters`, `tool_calls` blocks, and the `tool` role. `https://platform.openai.com/docs/guides/function-calling`.
- **OpenAI — Structured Outputs** — `strict: true` on function tools and JSON-schema-constrained decoding. `https://platform.openai.com/docs/guides/structured-outputs`.
- **OpenAI — Responses API** — the newer stateful interface; tool declaration and hosted tools. `https://platform.openai.com/docs/api-reference/responses`.
- **Google Gemini — Function calling** — `tools[i].functionDeclarations` in the Gemini API. `https://ai.google.dev/gemini-api/docs/function-calling`.
- **AWS Bedrock — Tool use in Converse API** — Converse `toolConfig` for cross-provider tool-use shapes on Bedrock. `https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html`. <!-- needs-research: confirm the current canonical URL — Bedrock renames sections often. -->

## JSON Schema and typed-argument sources ([chapter 01](01-tool-schema-and-side-effect-boundaries.md))

- **JSON Schema — spec home** — draft series index and the current spec. `https://json-schema.org/specification`.
- **JSON Schema — the reference book** — Henry Andrews et al., *Learn JSON Schema*. `https://www.learnjsonschema.com/`.
- **Pydantic docs** — Python source-of-truth types for tool arguments and structured outputs. `https://docs.pydantic.dev/latest/`.
- **Zod docs** — TypeScript source-of-truth types. `https://zod.dev/`.
- **`@sinclair/typebox`** — TypeScript-first JSON Schema types, often used with tool-calling. `https://github.com/sinclairzx81/typebox`.
- **`instructor` (Python)** — Pydantic-first structured outputs and tool-argument validation for LLM APIs. `https://python.useinstructor.com/`.
- **`instructor-js`** — TypeScript equivalent. `https://github.com/instructor-ai/instructor-js`.

## Idempotency and side-effect boundaries ([chapter 01](01-tool-schema-and-side-effect-boundaries.md))

- **Stripe — Idempotent requests** — the canonical reference for idempotency-key semantics on a public API. `https://docs.stripe.com/api/idempotent_requests`.
- **Stripe — Designing robust and predictable APIs with idempotency** (engineering blog) — the design essay behind the semantics above. `https://stripe.com/blog/idempotency`.
- **AWS — Building idempotency into APIs** — AWS's reference material on the pattern. `https://docs.aws.amazon.com/prescriptive-guidance/latest/idempotency-in-apis/introduction.html`. <!-- needs-research: confirm canonical URL for the AWS Prescriptive Guidance idempotency guide. -->
- **IETF draft — The Idempotency-Key HTTP Header Field** — the proposed standardisation of the header. `https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/`.
- **RFC 9110 — HTTP Semantics** — the definition of "idempotent method" and the retry semantics permitted by each HTTP method. `https://www.rfc-editor.org/rfc/rfc9110`.
- **Google Cloud — API design guide: idempotency** — the pattern applied to Google's APIs. `https://cloud.google.com/apis/design/design_patterns#request_identification`.

## Retry, backoff, and transient failure handling ([chapter 02](02-tool-call-error-taxonomy.md))

- **AWS Architecture Blog — Exponential Backoff And Jitter** — the reference essay on backoff strategy. `https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/`.
- **Google SRE Book — Handling Overload** — circuit-breaking and load-shedding under retry storms. `https://sre.google/sre-book/handling-overload/`.
- **Google SRE Book — Addressing Cascading Failures** — the reasoning behind capping retries per surface. `https://sre.google/sre-book/addressing-cascading-failures/`.
- **`tenacity` (Python)** — retry library with backoff, jitter, and circuit-breaker primitives. `https://tenacity.readthedocs.io/`.
- **`retry` / `p-retry` (JavaScript)** — TS/JS equivalents. `https://github.com/tim-kos/node-retry` and `https://github.com/sindresorhus/p-retry`.
- **`hyx` / `polly` (Python / .NET)** — circuit-breaker + bulkhead primitives. `https://github.com/roma-glushko/hyx` and `https://www.pollydocs.org/`.

## Model Context Protocol (MCP) ([chapter 03](03-mcp-in-a-product-agent.md))

- **Model Context Protocol — Specification** — the canonical specification, versioned. `https://modelcontextprotocol.io/specification`.
- **Model Context Protocol — Overview / concepts** — introductory framing (hosts, clients, servers; tools/resources/prompts). `https://modelcontextprotocol.io/introduction`.
- **`modelcontextprotocol/python-sdk`** — Python reference implementation of the protocol. `https://github.com/modelcontextprotocol/python-sdk`.
- **`modelcontextprotocol/typescript-sdk`** — TypeScript reference implementation. `https://github.com/modelcontextprotocol/typescript-sdk`.
- **`modelcontextprotocol/servers`** — reference MCP servers (filesystem, git, Slack, Postgres, others) — useful as read-throughs, not always production-ready. `https://github.com/modelcontextprotocol/servers`.
- **MCP — Transports documentation** — the current transport list (stdio, Streamable HTTP; legacy HTTP+SSE). `https://modelcontextprotocol.io/docs/concepts/transports`. <!-- needs-research: confirm the URL and section title against the current spec version — MCP moves fast. -->
- **MCP — Authorization** — the authorization spec for hosted servers. `https://modelcontextprotocol.io/specification/basic/authorization`. <!-- needs-research: MCP's auth section was in flux (OAuth 2.1 dynamic client registration); pin the current URL and version. -->
- **JSON-RPC 2.0 specification** — the wire protocol MCP builds on. `https://www.jsonrpc.org/specification`.
- **Anthropic — Introducing the Model Context Protocol** (announcement) — the launch essay framing why the protocol exists. `https://www.anthropic.com/news/model-context-protocol`.
- **`langchain-mcp-adapters`** — LangChain's MCP client integration; consumed inside LangGraph nodes. `https://github.com/langchain-ai/langchain-mcp-adapters`.
- **OpenAI — Remote MCP support (Responses API and Agents SDK)** — how OpenAI's Agents surface consumes MCP. `https://platform.openai.com/docs/mcp`. <!-- needs-research: confirm the current OpenAI MCP docs URL — the section was renamed after initial launch. -->

## Anthropic Claude Agent SDK ([chapter 04](04-orchestrating-with-langgraph-and-agent-sdks.md))

- **Claude Agent SDK — Overview** — Anthropic's harness for Messages-API agents, successor to the original Claude Code SDK. `https://docs.anthropic.com/en/api/agent-sdk/overview`. <!-- needs-research: pin the current canonical URL — Anthropic reorganised the developer docs around Agent SDK. -->
- **`anthropic-ai/claude-agent-sdk-python`** — Python SDK repository, examples, and issue tracker. `https://github.com/anthropics/claude-agent-sdk-python`.
- **Claude Agent SDK — Hooks reference** — `PreToolUse`, `PostToolUse`, stop-condition hooks used in the chapter examples. `https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk/hooks`. <!-- needs-research: confirm URL and hook names against current SDK version. -->
- **Claude Agent SDK — MCP integration** — configuring MCP servers inside a Claude Agent SDK host. `https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk/mcp`. <!-- needs-research: same caveat — pin current URL. -->
- **Anthropic — Building effective AI agents** (engineering blog) — architectural framing on when to reach for an agent vs. a workflow. `https://www.anthropic.com/engineering/building-effective-agents`.
- **Anthropic — How to build an agent with the Claude Agent SDK** — a walkthrough. `https://www.anthropic.com/engineering/claude-agent-sdk-guide`. <!-- needs-research: confirm URL and title. -->

## OpenAI Agents SDK ([chapter 04](04-orchestrating-with-langgraph-and-agent-sdks.md))

- **OpenAI Agents SDK — Docs** — canonical entry point for Agents, Handoffs, Guardrails, Sessions. `https://openai.github.io/openai-agents-python/`.
- **`openai/openai-agents-python`** — Python SDK repository. `https://github.com/openai/openai-agents-python`.
- **`openai/openai-agents-js`** — TypeScript SDK. `https://github.com/openai/openai-agents-js`.
- **OpenAI Agents SDK — Tracing and traces.openai.com** — the built-in trace surface. `https://openai.github.io/openai-agents-python/tracing/`.
- **OpenAI Agents SDK — Guardrails** — parallel input / output checks. `https://openai.github.io/openai-agents-python/guardrails/`.
- **OpenAI Agents SDK — Handoffs** — agent-to-agent transfer. `https://openai.github.io/openai-agents-python/handoffs/`.
- **OpenAI — A practical guide to building agents** (developer blog / whitepaper) — the framing OpenAI publishes alongside the SDK. `https://cdn.openai.com/business-guides/practical-guide-to-building-agents.pdf`. <!-- needs-research: confirm current stable URL — OpenAI moves whitepapers between the blog and the docs. -->

## LangGraph and LangChain ([chapter 04](04-orchestrating-with-langgraph-and-agent-sdks.md))

- **LangGraph — Documentation** — canonical entry point for state graphs, checkpointers, human-in-the-loop, streaming. `https://langchain-ai.github.io/langgraph/`.
- **`langchain-ai/langgraph`** — repository, examples, and issue tracker. `https://github.com/langchain-ai/langgraph`.
- **LangGraph — Checkpointers reference** — `MemorySaver`, `SqliteSaver`, `PostgresSaver`, `RedisSaver`. `https://langchain-ai.github.io/langgraph/reference/checkpoints/`.
- **LangGraph — Human-in-the-loop patterns** — checkpoint-and-interrupt as a first-class shape. `https://langchain-ai.github.io/langgraph/how-tos/#human-in-the-loop`.
- **LangGraph — Multi-agent patterns** — supervisor, handoff, hierarchical agents. `https://langchain-ai.github.io/langgraph/concepts/multi_agent/`.
- **LangChain — Tool calling docs** — LangChain's cross-provider tool-calling surface. `https://python.langchain.com/docs/concepts/tool_calling/`.
- **LangSmith — Docs** — the trace + eval surface most often paired with LangGraph. `https://docs.smith.langchain.com/`.

## Alternate agent frameworks (context, not core)

- **Microsoft AutoGen** — multi-agent conversation framework. `https://microsoft.github.io/autogen/`.
- **crewAI** — role-based multi-agent orchestration. `https://docs.crewai.com/`.
- **Semantic Kernel — Agents** — Microsoft's agent framework across .NET / Python / Java. `https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/`.
- **LlamaIndex — Agents** — agent primitives on LlamaIndex retrieval surfaces. `https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/`.
- **Haystack — Agents** — pipeline-based agents on Haystack. `https://docs.haystack.deepset.ai/docs/agents`.
- **DSPy — ReAct and other program modules** — programmatic agent modules. `https://dspy.ai/`.

## Foundational agent research ([chapter 04](04-orchestrating-with-langgraph-and-agent-sdks.md))

- **ReAct — Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*.** The paper that popularised the sample-tool-loop shape. `https://arxiv.org/abs/2210.03629`.
- **Toolformer — Schick et al., *Toolformer: Language Models Can Teach Themselves to Use Tools*.** Early self-supervised tool-use training. `https://arxiv.org/abs/2302.04761`.
- **Reflexion — Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*.** The "reflect on prior tool call" pattern. `https://arxiv.org/abs/2303.11366`.
- **Toolbench / API-Bank / MetaTool** — public benchmarks for tool-use fidelity. `https://arxiv.org/abs/2307.16789`, `https://arxiv.org/abs/2304.08244`, `https://arxiv.org/abs/2310.03128`.
- **`tau-bench`** — Sierra's benchmark for evaluating agents against retail / airline customer-support tasks with real tool APIs. `https://arxiv.org/abs/2406.12045` and `https://github.com/sierra-research/tau-bench`.
- **`SWE-bench`** — Princeton benchmark for agents that resolve real GitHub issues; a good yardstick for tool-heavy coding agents. `https://www.swebench.com/`.

## Token counting, pricing, and cost tracking ([chapter 05](05-per-request-budgets-and-graceful-degradation.md))

- **Anthropic — Token counting API** — `count_tokens` on the Messages endpoint. `https://docs.anthropic.com/en/api/messages-count-tokens`.
- **Anthropic — Prompt caching** — pricing and semantics for cached-input tokens. `https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching`.
- **OpenAI — `tiktoken`** — tokenizer library used to estimate input tokens before a call. `https://github.com/openai/tiktoken`.
- **OpenAI — Prompt caching** — cached-input pricing on Chat / Responses. `https://platform.openai.com/docs/guides/prompt-caching`.
- **OpenAI — Pricing page** — the source-of-truth for per-model input / output / cached token prices. `https://openai.com/api/pricing/`.
- **Anthropic — Pricing page** — the source-of-truth for Claude pricing. `https://www.anthropic.com/pricing`.
- **Google Gemini — Pricing** — the source-of-truth for Gemini pricing. `https://ai.google.dev/pricing`.

## Observability, tracing, and evaluation surfaces ([chapters 04](04-orchestrating-with-langgraph-and-agent-sdks.md) and [05](05-per-request-budgets-and-graceful-degradation.md))

- **OpenTelemetry — GenAI semantic conventions** — the emerging standard for LLM / agent span attributes. `https://opentelemetry.io/docs/specs/semconv/gen-ai/`.
- **Langfuse — Docs** — open-source LLM observability, self-host + managed. `https://langfuse.com/docs`.
- **Arize Phoenix — Docs** — open-source LLM tracing + eval. `https://docs.arize.com/phoenix`.
- **LangSmith — Evaluations and traces** — hosted trace store paired with LangGraph. `https://docs.smith.langchain.com/`.
- **Weights & Biases Weave** — trace + eval surface, cross-framework. `https://weave-docs.wandb.ai/`.
- **Braintrust — Docs** — hosted eval and trace surface. `https://www.braintrust.dev/docs/start/`.
- **Helicone — Docs** — LLM cost + latency observability proxy. `https://docs.helicone.ai/`.
- **OpenLLMetry** — OpenTelemetry-native instrumentation for LLM applications. `https://github.com/traceloop/openllmetry`.

## Security perimeter around agents ([chapter 03](03-mcp-in-a-product-agent.md))

- **OWASP Top 10 for Large Language Model Applications** — surface-side taxonomy; `LLM01: Prompt Injection`, `LLM06: Sensitive Information Disclosure`, `LLM08: Excessive Agency` are the most relevant here. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **NIST AI RMF 1.0** — the Measure / Manage functions that map to the eval + gate loop across mod-104 → mod-106 → mod-110. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **NIST AI 600-1 — Generative AI Profile** — the GenAI-specific companion to the RMF. `https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf`.
- **MITRE ATLAS** — adversarial tactics / techniques catalog for ML systems, including LLM-integrated systems. `https://atlas.mitre.org/`.
- **Simon Willison — Prompt injection tag** — practitioner reference for prompt-injection failure modes at the tool-description and result surfaces. `https://simonwillison.net/tags/prompt-injection/`.
- **Anthropic — Claude tool-use prompt injection guidance** — the official framing on injection through tool results. `https://docs.anthropic.com/en/docs/build-with-claude/tool-use#security-considerations`. <!-- needs-research: confirm the current URL / anchor. -->
- **OAuth 2.1 (draft)** — the auth framework MCP's hosted-server flow builds on. `https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1`.

## Secondary and background reading

- **Simon Willison — LLMs tag** — practitioner writeups on tool-use failure modes, MCP integration observations, and pricing analyses. `https://simonwillison.net/tags/llms/`.
- **Chip Huyen — *AI Engineering* (book) and blog** — engineering framing for agent + tool surfaces. `https://huyenchip.com/`.
- **Eugene Yan — Applied-LLM writeups** — practitioner posts covering agent patterns and eval. `https://eugeneyan.com/writing/`.
- **Anthropic — Engineering blog** — recurrent essays on Claude tool use, MCP, and agent design. `https://www.anthropic.com/engineering`.
- **OpenAI — Cookbook** — worked examples for tool use, structured outputs, and Agents SDK. `https://cookbook.openai.com/`.

## Standards previewed here, owned in other modules

- **NIST AI RMF / AI 600-1** — the Measure and Manage loop over the eval fixtures produced in this module is developed in **mod-106** and **mod-110**.
- **OWASP Top 10 for LLM Applications** — the injection / excessive-agency / sensitive-info surfaces sketched in [chapter 03](03-mcp-in-a-product-agent.md) are developed in **mod-107**.
- **Per-turn cost and latency SLOs** referenced by the budgets in [chapter 05](05-per-request-budgets-and-graceful-degradation.md) are developed in **mod-109**.
- **Feature-flag, canary, and rollback discipline** applied to shipping the surface built across this module is developed in **mod-111**.
