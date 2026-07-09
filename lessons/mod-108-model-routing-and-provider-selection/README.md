# mod-108-model-routing-and-provider-selection: Model Routing and Provider Selection

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 12 hours

## Learning objectives

- Choose a gateway architecture: LiteLLM (self-hosted), OpenRouter (managed), Portkey (guardrails + cache + fallback), Bedrock / Vertex / Foundry (hyperscaler-native) — with a trade-off note
- Route requests across providers and tiers with classifier-gated routing (cheaper model when the task fits), quality gates (fall back to frontier on low-confidence), and provider-outage fall-back
- Reason about self-hosted (vLLM / TGI / SGLang) vs managed for a given feature with cost / latency / control / residency trade-offs made explicit — consuming serving-perf depth from `ai-infra-performance`
- Compose multi-modal features (vision, audio, image / video, real-time voice via Realtime API / EVI / ElevenLabs) at the router level, so modality choice is a routing decision not a hard-coded model
- Contract-test the feature across providers so a swap does not silently regress quality, cost, or format — including golden traces per provider

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
