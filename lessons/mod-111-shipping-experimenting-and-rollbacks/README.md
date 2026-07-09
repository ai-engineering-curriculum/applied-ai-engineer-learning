# mod-111-shipping-experimenting-and-rollbacks: Shipping, Experimenting, and Rollbacks for LLM Features

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 13 hours

## Learning objectives

- Wire feature flags (LaunchDarkly or OpenFeature) so a new prompt / model / chain / agent can be turned on for a cohort without a redeploy
- Run canary and shadow launches with per-cohort quality / safety / cost gates — pre-registered, not eyeballed
- Design A/B experiments for LLM features with the *interface* to the A/B platform owned by `senior-ml-engineer` / `model-evaluation-engineer`, and know where deep methodology (variance reduction, sequential testing) begins
- Hot-rollback a bad prompt without redeploying the service: prompt registry semantics, rollback SLA, blast-radius containment
- Use hosted fine-tuning (OpenAI SFT, Anthropic tuned models where available, Bedrock custom models, Vertex tuning) as an *applied* lever — style, format, small distillation — knowing SFT / PEFT / RLHF / DPO methodology depth is owned by `fine-tuning-engineer`

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
