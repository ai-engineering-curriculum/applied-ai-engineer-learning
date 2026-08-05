# mod-105-shipping-product-agents: Shipping Product-Embedded Agents

**Estimated effort:** 16 hours

An "agent" in a product is a specific product surface with a specific set of trade-offs — not a generic capability you drop into a chat window. This module is about the choices the applied-AI engineer makes to ship one: the **shape** of the surface (coding co-pilot vs. CX vs. retrieval-with-actions vs. browsing vs. sales/support/ops), the **memory system** that gives users continuity without breaking the deletion / export contract, the **steering UX** (progress, cancel, undo, resume, hand-off, kill-switch) that lets a user trust the agent enough to give it a real task, the **eval suite** stolen in shape from the best public benchmarks, and the **shipped-agent references** (Cursor Composer, GitHub Copilot Workspace, Sierra, Harvey) whose design choices are the current state of the art.

The module deliberately separates what the applied-AI engineer owns (the shipped product agent) from what the agentic-AI engineer peer role owns (the agent engine and its research-facing benchmarks). Both roles exist in every one of the four shipped products this module studies; both are hiring; neither replaces the other.

## Learning objectives

- Choose the right agent shape for a real product surface: coding co-pilot (inline / chat / composer / workspace), customer-experience agent, retrieval-with-actions assistant, browsing agent, sales / support / ops assistant — with a written case for each.
- Design the memory system a product agent actually needs: session context, per-user memory, long-term memory with retention and PII policy, and the deletion / export contract.
- Wire human-in-the-loop hand-offs, kill-switches, and progress / cancel / undo / resume UX so users can trust and steer the agent.
- Consume `τ-bench`, SWE-bench Verified, WebArena, and AgentDojo as *shape templates* for internal task suites — while keeping the product-agent eval focused on real product tasks, not benchmark maximisation.
- Study four public applied-agent patterns (Cursor Composer, GitHub Copilot Workspace, Sierra CX agents, Harvey legal agents) and extract the design choices that ship — separating what belongs to the applied engineer from what belongs to the agentic-AI engineer peer.

## Chapters

1. [Agent Shape for a Real Product Surface](01-agent-shape-for-a-real-product-surface.md) — five shapes (coding co-pilot, CX, retrieval-with-actions, browsing, internal assistant), a one-hour decision procedure, and the applied-vs-agentic split introduced.
2. [Memory System with Retention and PII Policy](02-memory-system-with-retention-and-pii-policy.md) — three layers (session, per-user, workspace), a disciplined per-user fact schema, per-turn retrieval, the deletion / export pipeline, and the provider-side memory trap.
3. [Human-in-the-Loop and Steering UX](03-human-in-the-loop-and-steering-ux.md) — six affordances (progress, cancel, undo, resume, hand-off, kill-switch), the runtime signal shape that supports them, confirmation UX for high-blast-radius writes, and the operator's kill-switch runbook.
4. [Eval Suites from Benchmark Shapes](04-eval-suites-from-benchmark-shapes.md) — τ-bench, SWE-bench Verified, WebArena, AgentDojo as shape templates; composing an internal suite; the applied-vs-agentic eval split.
5. [Public Applied-Agent Patterns](05-public-applied-agent-patterns.md) — Cursor Composer, GitHub Copilot Workspace, Sierra CX, Harvey; design choices worth stealing; anti-patterns visible in surfaces that tried to imitate.

## Exercises

- [exercise-01 — agent shape decision for real surface](exercises/exercise-01-agent-shape-decision-for-real-surface.md).
- [exercise-02 — memory system with retention and PII policy](exercises/exercise-02-memory-system-with-retention-and-pii-policy.md).
- [exercise-03 — human-in-the-loop and kill-switch UX](exercises/exercise-03-human-in-the-loop-and-kill-switch-ux.md).
- [exercise-04 — product-agent eval suite derived from benchmark shapes](exercises/exercise-04-product-agent-eval-suite-derived-from-benchmark-shapes.md).
- [exercise-05 — public applied-agent pattern audit](exercises/exercise-05-public-applied-agent-pattern-audit.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- Structured-output validation and prompt discipline consumed by the memory extractor in [chapter 02](02-memory-system-with-retention-and-pii-policy.md) lives in **mod-102**.
- Retrieval quality and permissions-aware retrieval consumed by Shape C (retrieval-with-actions) in [chapter 01](01-agent-shape-for-a-real-product-surface.md) lives in **mod-103**.
- Tool schemas, error taxonomy, MCP integration, orchestration, and per-request budgets consumed by every shape in [chapters 01](01-agent-shape-for-a-real-product-surface.md), [03](03-human-in-the-loop-and-steering-ux.md), and [05](05-public-applied-agent-patterns.md) live in **mod-104**.
- Product-agent eval fixtures introduced in [chapter 04](04-eval-suites-from-benchmark-shapes.md) — dataset curation, LLM-as-judge calibration, and CI gating — are developed in **mod-106**.
- Prompt injection, tool-output attack posture, and jailbreak resistance sketched in [chapter 04](04-eval-suites-from-benchmark-shapes.md) (AgentDojo template) are developed in **mod-107**.
- Model routing and provider selection referenced in [chapter 05](05-public-applied-agent-patterns.md) (Harvey's multi-provider stack) is developed in **mod-108**.
- Per-turn cost, latency SLOs, and long-task budget shape from [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-109**.
- Trace / span / feedback surfaces for the steering signals in [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-110**.
- Feature flags, canary rollouts, and rollback discipline for the kill-switch runbook in [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-111**.
- End-to-end ownership of a shipped product-agent surface, using the shape choice from this module, is the capstone in **mod-112**.
