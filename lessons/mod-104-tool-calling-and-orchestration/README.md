# mod-104-tool-calling-and-orchestration: Tool Calling and Orchestration at Product Altitude

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Design tool interfaces (JSON schemas, argument validation, side-effect boundaries) that a model can call reliably under production traffic
- Handle tool-call errors: model-side hallucinated arguments, backend-side transient failures, and permanent failures — with retry / repair / user-facing surface for each
- Wire MCP servers into a product agent (Claude Agent SDK, LangGraph MCP client, custom host) so third-party tools plug in without a rewrite
- Orchestrate multi-step tool sequences with LangGraph and/or the OpenAI Agents SDK / Anthropic Claude Agent SDK, keeping the graph shape maintainable as the feature grows
- Enforce per-request budgets on token count, step count, wall-clock, and dollar cost — with graceful degradation when a budget is hit, not silent truncation

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
