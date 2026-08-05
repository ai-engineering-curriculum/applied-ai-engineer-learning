# mod-107-guardrails-and-safety-on-the-surface: Guardrails and Safety on the Product Surface

**Estimated effort:** 14 hours

Safety on an LLM product surface is a *shipping discipline first and a research discipline second*. The peer roles on this org chart — `ai-risk-engineer`, `ai-safety-engineer`, `ai-security-engineer`, `privacy-engineer` — own the deep work: harm modelling, red-team programme design, jailbreak-corpus curation, model-side safety tuning, the DPIA process, the compliance envelope. The applied-AI engineer owns the *surface's* discipline: composing moderation, defending against injection, scrubbing PII on the way to telemetry, gating destructive tool calls, calibrating refusal against a written policy, and mapping every finding to a taxonomy with an owner.

The module carries a single through-line: **defence in depth on the specific product surface you ship**, not defence in general. Every chapter — every exercise — is pointed at that discipline. Where the applied engineer's work ends and a peer role's begins is named explicitly in [chapter 05](05-refusal-calibration-and-owasp-mapping.md) and in the peer-role boundaries exercise.

## Learning objectives

- Layer input / output moderation on a production surface using provider moderation (OpenAI / Anthropic / Gemini) plus a classifier guardrail (Llama Guard, Purple Llama, provider content-safety) with an explicit false-positive / false-negative report.
- Defend against prompt injection at the surface: input sanitisation, tool-boundary isolation, retrieval-side indirect-injection detection, and structured-output escape-hatch controls — grounded in AgentDojo / InjecAgent shape.
- Wire PII / secret scrubbing on inputs, prompts, outputs, and traces so telemetry does not become a data-leak channel.
- Add tool-abuse and unauthorised-tool-use guards: allow-listed tool sets per feature, cost / rate-limit caps per tool per tenant, and destructive-action confirmation gates.
- Calibrate refusal / over-refusal at the product surface against a written safety policy and map every finding to OWASP LLM Top 10 and OWASP Agentic AI Top 10 with an owner assigned — deferring harm-model / red-team-data generation to `ai-risk-engineer`.

## Chapters

1. [Layered Moderation on the Product Surface](01-layered-moderation-with-classifier-and-provider.md) — the four moderation points (input, output, retrieval, telemetry), the provider + classifier two-layer split, streaming patterns, the false-positive / false-negative report as the load-bearing artefact.
2. [Prompt Injection Defence at the Surface](02-prompt-injection-defence-at-the-surface.md) — injection vs. moderation, the four defensive layers (input sanitisation, tool-boundary isolation, structured-output controls, retrieval-side detection), the AgentDojo / InjecAgent-shape eval fixture set.
3. [PII and Secret Scrubbing Across the Stack](03-pii-and-secret-scrubbing-across-the-stack.md) — the four scrubbing points, the five data categories, the detector-and-replacement matrix, the tracing-SDK integration, the DSAR pipeline extension, and the scrubber's own FP/FN report.
4. [Tool-Abuse Guards and Destructive-Action Gates](04-tool-abuse-and-destructive-action-gates.md) — per-feature tool allow-lists, per-tool per-tenant caps (rate / cost / argument-shape volume), the four-way confirmation-policy classification, auth-boundary discipline, and the kill switch.
5. [Refusal Calibration and the OWASP Mapping Runbook](05-refusal-calibration-and-owasp-mapping.md) — the written safety policy, the runtime refusal-category taxonomy, the over-refusal / under-refusal eval, drift monitoring, the OWASP LLM Top 10 + Agentic AI Top 10 mapping runbook, and the peer-role handoffs.

## Exercises

- [exercise-01 — layered moderation with FP/FN report](exercises/exercise-01-layered-moderation-with-fp-fn-report.md).
- [exercise-02 — prompt injection defence with AgentDojo-shape eval](exercises/exercise-02-prompt-injection-defence-with-agentdojo-shape.md).
- [exercise-03 — PII and secret scrubbing on traces](exercises/exercise-03-pii-and-secret-scrubbing-on-traces.md).
- [exercise-04 — tool-abuse guards and destructive-action gates](exercises/exercise-04-tool-abuse-guards-and-destructive-action-gates.md).
- [exercise-05 — OWASP LLM and Agentic mapping runbook](exercises/exercise-05-owasp-llm-and-agentic-mapping-runbook.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- The tool-schema, argument-validation, and idempotency discipline that [chapter 04](04-tool-abuse-and-destructive-action-gates.md)'s guards depend on lives in **mod-104**.
- The MCP transport and tool-boundary discussion that [chapter 02](02-prompt-injection-defence-at-the-surface.md)'s Layer 2 builds on lives in **mod-104 chapter 03**.
- The human-in-the-loop UX pattern that [chapter 04](04-tool-abuse-and-destructive-action-gates.md)'s confirmation gates specialise lives in **mod-105 chapter 03**.
- The memory-layer deletion pipeline that [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md)'s DSAR extension composes with lives in **mod-105 chapter 02**.
- The offline-eval, pre-registration, and CI-gating discipline that [chapter 01](01-layered-moderation-with-classifier-and-provider.md)'s FP/FN report and [chapter 05](05-refusal-calibration-and-owasp-mapping.md)'s over-refusal eval plug into lives in **mod-106**.
- The trace-snapshot pipeline whose PII scrubbing is fed by [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md) lives in **mod-106 chapter 02**.
- Cost and latency SLOs that [chapter 04](04-tool-abuse-and-destructive-action-gates.md)'s caps roll into live in **mod-109**.
- Trace collection, span attributes, and the observability substrate that [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md)'s span-processor plugs into — and the refusal-drift dashboard from [chapter 05](05-refusal-calibration-and-owasp-mapping.md) queries — live in **mod-110**.
- The kill-switch propagation and rollout / rollback discipline that [chapter 04](04-tool-abuse-and-destructive-action-gates.md) previews are developed in **mod-111**.
- End-to-end ownership of a shipped feature with this module's guardrails wired in is the capstone in **mod-112**.

## Peer-role handoffs

Every chapter names its peer-role boundary explicitly; [chapter 05](05-refusal-calibration-and-owasp-mapping.md) collects them in one place. In short:

- **`ai-risk-engineer`** owns harm-model construction, the red-team programme at depth, and the cross-product risk register. The applied engineer's OWASP mapping (chapter 05) *feeds* the register; the applied engineer does not maintain it alone.
- **`ai-safety-engineer`** owns model-side safety tuning, jailbreak-corpus curation, and refusal-behaviour alignment. The applied engineer *consumes* the improved model behaviour and calibrates the surface around it.
- **`ai-security-engineer` / `application-security-engineer`** own the deeper security posture — authn/authz substrate, secrets management programme, MCP server security review, tool-credential rotation. The applied engineer's chapter-04 Guard 4 (auth boundary) is downstream of this work.
- **`privacy-engineer` / `data-security-engineer`** own the regulatory foundations — GDPR / CCPA / HIPAA / GLBA / PCI substrate, the DPIA process, cross-border transfer analysis. The applied engineer's chapter-03 scrubbing runs *inside* their envelope.
- **`incident-commander` / `security-response`** own coordinated incident response. The applied engineer's kill switch (chapter 04) and OWASP-mapped remediation tickets are inputs to their playbook.

Naming the boundaries out loud is deliberate: the applied engineer's guardrails are shallow-and-fast on purpose; the peer roles' work is deep-and-slow on purpose. The two together are what a serious surface actually ships with.
