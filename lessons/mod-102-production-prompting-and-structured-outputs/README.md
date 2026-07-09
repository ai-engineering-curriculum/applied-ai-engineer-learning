# mod-102 — Production Prompting and Structured Outputs

**Estimated effort:** 14 hours

Turn the shape + SLO + provider decision from mod-101 into a shippable prompt-and-output layer. Learn to write prompts that stay cached across every edit a live product makes, produce schema-safe outputs the rest of the product can consume, stream cleanly with cancellation that reaches all the way to the provider, retry-and-repair schema failures without amplifying cost or opening a jailbreak channel, and version prompts as first-class artefacts with pinned regression fixtures and hot rollback.

## Learning objectives

- Design production prompts (system / instruction / few-shot / rubric) that survive prompt-caching boundaries on OpenAI, Anthropic, Gemini, and Bedrock without silent invalidation.
- Use JSON mode, function / tool calling, and constrained decoding (Instructor + Pydantic, Outlines) to produce schema-safe outputs the rest of the product can consume.
- Implement streaming and cancellation end-to-end (backend → SSE / WebSocket → UI), including partial-output rendering and mid-stream tool-call handling.
- Design retry, repair, and self-correction strategies for structured outputs that fail schema validation, with bounded budgets and explicit fall-back.
- Version prompts as first-class artefacts (git / prompt registry) with dataset-pinned regression fixtures and rollback semantics.

## Lecture chapters

1. [Cache-Aware Prompt Design](01-cache-aware-prompt-design.md) — how each of OpenAI, Anthropic, Gemini, and Bedrock actually caches, the four-block layering, six named anti-patterns that break real deployments, and the telemetry that catches silent invalidation.
2. [Schema-Safe Structured Outputs](02-schema-safe-structured-outputs.md) — JSON mode, function / tool calling and Structured Outputs, Instructor + Pydantic, Outlines on self-hosted models, and the semantic-validation layer that sits on top.
3. [Streaming and Cancellation, End to End](03-streaming-and-cancellation.md) — the four hops from provider to UI, SSE vs WebSocket, partial JSON, mid-stream tool calls, and cancellation that actually stops upstream billing.
4. [Retry, Repair, and Bounded Fallback](04-retry-repair-and-fallback.md) — five failure classes, per-class budgets, sanitised repair messages, and explicit fallback as a designed product outcome.
5. [Prompts as Versioned Artefacts](05-prompts-as-versioned-artefacts.md) — the five guarantees a prompt-as-artefact must give you, git vs registry substrates, regression fixtures and the promotion gate, and rollback in minutes.

## Exercises

- [exercise-01 — Prompt Cache Aware System Prompt](exercises/exercise-01-prompt-cache-aware-system-prompt.md)
- [exercise-02 — Structured Outputs With Instructor And Outlines](exercises/exercise-02-structured-outputs-with-instructor-and-outlines.md)
- [exercise-03 — Streaming With Cancellation End To End](exercises/exercise-03-streaming-with-cancellation-end-to-end.md)
- [exercise-04 — Retry And Repair For Schema Failures](exercises/exercise-04-retry-and-repair-for-schema-failures.md)
- [exercise-05 — Prompt Versioning And Rollback](exercises/exercise-05-prompt-versioning-and-rollback.md)

## Structure

```
mod-102-production-prompting-and-structured-outputs/
├── 01-cache-aware-prompt-design.md
├── 02-schema-safe-structured-outputs.md
├── 03-streaming-and-cancellation.md
├── 04-retry-repair-and-fallback.md
├── 05-prompts-as-versioned-artefacts.md
├── exercises/                 per-exercise prompts (solutions live in the paired -solutions repo)
├── labs/                      long-form hands-on labs
├── quizzes/                   knowledge checks
├── resources.md               curated external references
└── README.md                  this file
```

## What comes next in the track

- mod-103 wires the retrieval-with-citations shape into the prompt / output / streaming layer built here.
- mod-104 lands the tool-calling surface; the schema-safe-output work here is the wire format for arguments and results.
- mod-105 composes the pieces into a product-shaped autonomous agent.
- mod-106 turns the regression-fixture pattern (chapter 05) into a full eval-gated CI harness.
- mod-107 layers surface-side guardrails around the prompt and the output.
- mod-109 attaches the cost-per-successful-outcome accounting to the retry-and-cache metrics named in chapters 01 and 04.
- mod-110 turns rollback (chapter 05) into an on-call runbook with incident-to-fixture loops.
- mod-111 turns the environment-binding + experiment split from chapter 05 into a canary and shadow-launch harness.
