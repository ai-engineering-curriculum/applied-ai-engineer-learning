# mod-107-guardrails-and-safety-on-the-surface: Guardrails and Safety on the Product Surface

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Layer input / output moderation on a production surface using provider moderation (OpenAI / Anthropic / Gemini) plus a classifier guardrail (Llama Guard, Purple Llama, provider content-safety) with an explicit false-positive / false-negative report
- Defend against prompt injection at the surface: input sanitisation, tool-boundary isolation, retrieval-side indirect-injection detection, and structured-output escape-hatch controls — grounded in AgentDojo / InjecAgent shape
- Wire PII / secret scrubbing on inputs, prompts, outputs, and traces so telemetry does not become a data-leak channel
- Add tool-abuse and unauthorised-tool-use guards: allow-listed tool sets per feature, cost / rate-limit caps per tool per tenant, and destructive-action confirmation gates
- Calibrate refusal / over-refusal at the product surface against a written safety policy and map every finding to OWASP LLM Top 10 and OWASP Agentic AI Top 10 with an owner assigned — deferring harm-model / red-team-data generation to `ai-risk-engineer`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
