# mod-104-tool-calling-and-orchestration: Tool Calling and Orchestration at Product Altitude

**Estimated effort:** 14 hours

Tool calling is where a product agent stops being a chat wrapper and starts being a system that can act. This module is about designing that action surface so it's safe under production traffic. The applied-AI engineer owns four artefacts: the **tool schema** (what the model can send), the **error taxonomy** (what happens when a call fails), the **orchestration graph** (how a sequence of calls stays maintainable), and the **per-request budget** (how the surface stays economically honest). MCP is the plugin interface that lets tools you don't own participate in that surface without a rewrite.

## Learning objectives

- Design tool interfaces (JSON schemas, argument validation, side-effect boundaries) that a model can call reliably under production traffic.
- Handle tool-call errors: model-side hallucinated arguments, backend-side transient failures, and permanent failures — with retry / repair / user-facing surface for each.
- Wire MCP servers into a product agent (Claude Agent SDK, LangGraph MCP client, custom host) so third-party tools plug in without a rewrite.
- Orchestrate multi-step tool sequences with LangGraph and/or the OpenAI Agents SDK / Anthropic Claude Agent SDK, keeping the graph shape maintainable as the feature grows.
- Enforce per-request budgets on token count, step count, wall-clock, and dollar cost — with graceful degradation when a budget is hit, not silent truncation.

## Chapters

1. [Tool Schema and the Side-Effect Boundary](01-tool-schema-and-side-effect-boundaries.md) — the schema as a public API against a non-deterministic caller: strict JSON Schema, argument validation, idempotency keys, dry-run and undo, and naming as first-class.
2. [The Tool-Call Error Taxonomy](02-tool-call-error-taxonomy.md) — three failure sources (model-side, backend transient, backend permanent), the bounded retry / repair posture for each, and the user-facing surface for every class.
3. [MCP in a Product Agent](03-mcp-in-a-product-agent.md) — Model Context Protocol as a plugin interface: transports, catalog filtering, description overrides, the security perimeter, and MCP-boundary SLOs.
4. [Orchestrating with LangGraph and the Agent SDKs](04-orchestrating-with-langgraph-and-agent-sdks.md) — graph shape as a design artefact; loop-shaped SDKs vs. graph frameworks; the five graph-shape principles; testing an agent graph.
5. [Per-Request Budgets and Graceful Degradation](05-per-request-budgets-and-graceful-degradation.md) — the four budgets (tokens, steps, wall-clock, dollars); pre-flight vs. reactive enforcement; the graceful-degradation path; the silent-truncation anti-pattern.

## Exercises

- [exercise-01 — tool schema design and argument validation](exercises/exercise-01-tool-schema-design-and-argument-validation.md).
- [exercise-02 — tool error handling taxonomy](exercises/exercise-02-tool-error-handling-taxonomy.md).
- [exercise-03 — MCP server and client integration](exercises/exercise-03-mcp-server-and-client-integration.md).
- [exercise-04 — LangGraph vs. Agents SDK bakeoff](exercises/exercise-04-langgraph-vs-agents-sdk-bakeoff.md).
- [exercise-05 — per-request budget enforcement](exercises/exercise-05-per-request-budget-enforcement.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- Structured-output validation, prompt caching, and the `describe_errors` shape reused in [chapter 01](01-tool-schema-and-side-effect-boundaries.md) and [chapter 02](02-tool-call-error-taxonomy.md) live in **mod-102**.
- Retrieval as a tool boundary — the applied-AI / peer-track split modelled in [chapter 03](03-mcp-in-a-product-agent.md) — is developed in depth in **mod-103**.
- Model routing and provider selection referenced in [chapter 04](04-orchestrating-with-langgraph-and-agent-sdks.md) — cheap classifier feeding a strong reasoner — live in **mod-108**.
- Prompt-injection defence for tool descriptions and result content in [chapter 03](03-mcp-in-a-product-agent.md) is developed in **mod-107**.
- Per-turn cost and latency SLOs consumed by the budgets in [chapter 05](05-per-request-budgets-and-graceful-degradation.md) live in **mod-109**.
- Agent-graph traces, spend-vs-cap dashboards, and early-stop alerting land in **mod-110**.
- Eval fixtures for tool-selection, argument-fidelity, and degradation-path tests live in **mod-106**.
- Shipping, versioning, and rollback of an agent surface built on this module lives in **mod-105** and **mod-111**.
