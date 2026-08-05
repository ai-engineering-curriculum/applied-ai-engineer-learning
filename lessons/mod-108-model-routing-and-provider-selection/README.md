# mod-108-model-routing-and-provider-selection: Model Routing and Provider Selection

**Estimated effort:** 12 hours

Every applied-AI surface past its first ship has the same latent problem: three features, three copies of the retry loop, three billing surfaces, three teams each rediscovering that Anthropic's tool-use shape is not OpenAI's. Model routing is not a research topic — it is a *shipping discipline* that decides which model serves which request, on which provider, on which infrastructure, and with what fall-back when any of the above fails. Peer roles own the deep work (`ai-infra-performance` for serving-perf, `model-evaluation-engineer` for benchmark-shape evals, `privacy-engineer` for the residency envelope). The applied-AI engineer owns the *product-facing* discipline: the gateway seam, the router state machine, the self-host decision, the modality-aware route table, and the contract tests that make a provider swap safe.

The module carries a single through-line: **every model choice is a routing decision made at one deliberate seam, with a written trade-off and a contract test to prove the swap does not silently regress.** Every chapter builds one piece of that seam.

## Learning objectives

- Choose a gateway architecture — LiteLLM (self-hosted), OpenRouter (managed), Portkey (guardrails + cache + fallback), Bedrock / Vertex / Foundry (hyperscaler-native) — with a written trade-off note that names the decision, the constraints, and the revisit trigger.
- Route requests across providers and tiers with **classifier-gated routing** (cheaper model when the task fits), **quality gates** (fall back to frontier on low-confidence or malformed output), and **provider-outage fallback** (per-provider circuit breakers with a routing-decision log).
- Reason about **self-hosted (vLLM / TGI / SGLang) vs managed** for a given route with cost / latency / control / residency / feature-velocity trade-offs made explicit — consuming serving-perf baselines owned by `ai-infra-performance`.
- Compose **multi-modal features** (vision, audio, image / video, real-time voice via Realtime API / EVI / ElevenLabs) at the router level, so modality choice is a routing decision — including a *composed* fallback for realtime voice (STT → text-LLM → TTS) when the native realtime API is unavailable.
- Contract-test the feature across providers so a swap does not silently regress quality, cost, or format — including **per-provider golden traces**, hard-fail / soft-warn diff bands, and CI wiring that blocks merges on shape breaks.

## Chapters

1. [Gateway Architecture and the Trade-Off Note](01-gateway-architecture-and-the-trade-off-note.md) — the four canonical shapes (LiteLLM, OpenRouter, Portkey, hyperscaler-native), the six trade-off axes, common compositions, and the one-page trade-off note as the load-bearing artefact.
2. [Classifier-Gated Routing, Quality Gates, and Provider-Outage Fallback](02-classifier-gated-routing-and-fallback.md) — the three axes (difficulty, quality, availability), the classifier schema and route table, the quality-gate spec, the circuit-breaker skeleton, and the routing-decision log.
3. [Self-Hosted vs Managed: The vLLM / TGI / SGLang Baseline](03-self-hosted-vs-managed-with-vllm-baseline.md) — the five decision axes, the peer-team baseline the applied engineer *consumes*, the per-route decision doc with break-even bands, and the hybrid patterns worth naming.
4. [Multi-Modal Routing: Vision, Audio, Image / Video, and Realtime Voice](04-multi-modal-routing-vision-audio-realtime.md) — the modality-aware route table, per-modality gotchas (vision-native vs OCR, streaming STT, image tokenisation, video async, realtime duplex), and cross-modal composition patterns.
5. [Cross-Provider Contract Testing and Per-Provider Golden Traces](05-cross-provider-contract-testing.md) — what is under contract, the fixture-set + runner + assertions shape, per-provider goldens pinned to dated model IDs, and the CI wiring that gates merges.

## Exercises

- [exercise-01 — gateway architecture trade-off note](exercises/exercise-01-gateway-architecture-trade-off-note.md).
- [exercise-02 — classifier-gated routing and fallback](exercises/exercise-02-classifier-gated-routing-and-fall-back.md).
- [exercise-03 — self-host vs managed decision with vLLM baseline](exercises/exercise-03-self-host-vs-managed-decision-with-vllm-baseline.md).
- [exercise-04 — multi-modal routing: vision, audio, realtime](exercises/exercise-04-multi-modal-routing-vision-audio-realtime.md).
- [exercise-05 — cross-provider contract tests](exercises/exercise-05-cross-provider-contract-tests.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## Cross-module context

- The **structured-output validators** [chapter 02](02-classifier-gated-routing-and-fallback.md)'s quality gates depend on live in **mod-102**.
- The **tool-call shape** [chapter 05](05-cross-provider-contract-testing.md)'s contract tests exercise per provider is defined in **mod-104**.
- The **surface / agent shape** [chapter 04](04-multi-modal-routing-vision-audio-realtime.md)'s realtime-voice fallback lives inside is developed in **mod-105 chapter 01**.
- The **kill switch** and confirmation gates the router honours as a hard `refuse` route live in **mod-107 chapter 04**.
- The **PII scrubbing** [chapter 04](04-multi-modal-routing-vision-audio-realtime.md) runs on STT output and the router runs on requests before dispatch is developed in **mod-107 chapter 03**.
- The **offline-eval, pre-registration, and CI-gating substrate** [chapter 05](05-cross-provider-contract-testing.md)'s contract-test job plugs into lives in **mod-106**, especially chapter 03.
- The **cost / latency SLO substrate** the route table's per-route `max_p95_ms` / `max_cost_per_call_usd` feed lives in **mod-109**.
- The **observability spans** the router emits (`route.chosen_route`, `route.chosen_provider`, `route.escalated_from`, `route.quality_gate_result`) are consumed by the trace pipeline in **mod-110**.
- The **rollout / rollback discipline** [chapter 05](05-cross-provider-contract-testing.md)'s contract report gates is developed in **mod-111**.
- End-to-end ownership of a shipped feature with a routed, contract-tested, cost-aware surface is the capstone in **mod-112**.

## Peer-role handoffs

Every chapter names its peer-role boundary. In short:

- **`ai-infra-performance`** owns the deep serving-perf work — continuous-batching schedulers, KV-cache paging, tensor / pipeline / expert parallelism, quantisation quality-vs-throughput trade-offs, MLPerf-shape benchmarking. The applied engineer *consumes* their baseline (chapter 03) and asks for a refresh when the traffic profile shifts.
- **`ml-platform-engineer`** owns the GPU capacity plan and autoscaling for any self-hosted route.
- **`model-evaluation-engineer`** owns the deeper quality eval regime (public benchmarks, LLM-as-judge suites, calibration). The applied engineer's contract-tests (chapter 05) are a *thin* pipe-test; the deep quality bar is theirs.
- **`privacy-engineer`** and **`data-security-engineer`** own the residency envelope. The router's route table and the self-host decision *serve* the envelope; they do not define it.
- **`sre` / `platform`** consume the router's contract report as a signal in the release-gating pipeline (mod-111).
- **`ai-security-engineer`** consumes the router's evidence of what data leaves the boundary and to which provider.

Naming the boundaries out loud is deliberate: the applied engineer's routing discipline is *shipping-side* on purpose; the peer roles' work is *depth-side* on purpose. The two together are what a routed, self-hostable, multi-modal, contract-tested surface actually ships with.
