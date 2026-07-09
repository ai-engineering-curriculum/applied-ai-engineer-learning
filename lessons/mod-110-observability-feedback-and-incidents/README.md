# mod-110-observability-feedback-and-incidents: Observability, User Feedback, and Incident Response for LLM Features

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 13 hours

## Learning objectives

- Instrument LLM features with OpenTelemetry Gen-AI + OpenInference span attributes: prompt / response / model version / tokens / cost / latency / hashed user id / retrieved chunks / tool calls — at the granularity a debugger needs
- Adopt a sampling policy that keeps trace cost inside a product budget (head sampling, tail sampling on error / quality signal, PII scrubbing) — trace budgets are a real line item at applied-AI product scale
- Wire user feedback into a triage pipeline: thumbs-up / thumbs-down, structured feedback, free-text — deduped and routed into an eval dataset with owner-assigned triage, using Argilla or Label Studio for expert review
- Design LLM-feature incident playbooks for the six recurring failure modes: bad prompt push, provider outage, judge drift, price hike, silent quality regression, guardrail false-positive spike
- Compose a per-feature status page and on-call runbook a rotation can actually use — including the interfaces to the platform / SRE team owned by `ai-infra-mlops`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
