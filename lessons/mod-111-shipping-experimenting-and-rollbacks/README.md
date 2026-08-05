# mod-111-shipping-experimenting-and-rollbacks: Shipping, Experimenting, and Rollbacks for LLM Features

**Estimated effort:** 13 hours

Every LLM feature is one bad prompt-push, one silent model alias-swap, or one over-eager rollout away from a user-visible incident. Applied-AI engineers who ship without a rollout substrate learn this in production: the "quick prompt tweak" merged at 4 PM Friday makes it into general availability by Monday morning because there was no way to say *5% first, watch these metrics, promote only if they pass*. The rollback of that same prompt takes ninety minutes because the prompt lives in a deploy artefact and the deploy pipeline is running someone else's slow test. The A/B intended to measure whether the new chain-of-thought helps has 51/49 exposure counts and no one knows why.

The through-line of this module is a **shipping discipline** — the runtime substrate that lets a change to a prompt, a model, a chain, or an agent become a **first-class thing you can turn on for a cohort, observe against pre-registered gates, promote to more traffic, roll back in seconds, or split-test against the current variant — all without a code deploy**. The five chapters build that substrate piece by piece. The mod-110 observability substrate ([mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)) is the sensor; the mod-106 eval substrate ([mod-106](../mod-106-feature-side-evaluation/README.md)) is the quality yardstick; this module is the *actuator*.

Peer-role boundaries are named up-front. The deep statistical methodology of experimentation — variance reduction (CUPED), sequential testing, heterogeneous treatment effects, delta-method estimators — is owned by `senior-ml-engineer` and `model-evaluation-engineer`. The methodology of fine-tuning — SFT vs PEFT vs DPO vs RLHF selection, dataset curation depth, catastrophic-forgetting audits, alignment tax — is owned by `fine-tuning-engineer`. The applied-AI engineer owns the **surface**: the flag layer, the canary plan, the interface into the experimentation platform, the rollback path, and the hosted-fine-tune surface. This module makes each of those surfaces concrete.

## Learning objectives

- Wire feature flags (LaunchDarkly or OpenFeature) so a new prompt / model / chain / agent can be turned on for a cohort without a redeploy.
- Run canary and shadow launches with per-cohort quality / safety / cost gates — **pre-registered**, not eyeballed.
- Design A/B experiments for LLM features with the *interface* to the A/B platform owned by `senior-ml-engineer` / `model-evaluation-engineer`, and know where deep methodology (variance reduction, sequential testing) begins.
- Hot-rollback a bad prompt without redeploying the service: prompt registry semantics, rollback SLA, blast-radius containment.
- Use hosted fine-tuning (OpenAI SFT, Anthropic tuned models where available, Bedrock custom models, Vertex tuning) as an *applied* lever — style, format, small distillation — knowing SFT / PEFT / RLHF / DPO methodology depth is owned by `fine-tuning-engineer`.

## Chapters

1. [Feature Flags as the LLM Shipping Substrate](01-feature-flags-as-the-llm-shipping-substrate.md) — LaunchDarkly and OpenFeature; the four flag shapes (boolean release, multivariate, config, kill-switch); the `targeting_key` / `default_value` / attributes contract; wiring the variant onto the trace via `app.experiment.arm`; the fail-closed kill-switch every feature ships with.
2. [Canary and Shadow Launches with Pre-Registered Gates](02-canary-and-shadow-launches-with-pre-registered-gates.md) — canary vs shadow, and when to use each; the four gate families (quality, safety, cost, latency); pre-registering thresholds before the flag flip; sample-size math for canary observation windows; the automated rollback wiring on hard gates; the promotion-decision artefact per rollout stage.
3. [A/B Experiments: The Interface to the Experimentation Platform](03-ab-experiment-interface-to-the-experimentation-platform.md) — assignment, exposure, metric emission, and guardrail wiring as the applied engineer's four responsibilities; sample-ratio mismatch as the platform's "your interface is broken" alarm; where the peer-role boundary sits (CUPED, sequential testing, HTE, delta-method estimators).
4. [Hot Rollback of a Bad Prompt](04-hot-rollback-of-a-bad-prompt.md) — the append-only prompt registry; the TTD / TTDe / TTE decomposition of the rollback SLA; the flag hierarchy for blast-radius containment; fail-closed defaults on every flag layer; the one-page runbook and the monthly rollback drill.
5. [Hosted Fine-Tuning as an Applied Lever](05-hosted-fine-tuning-as-an-applied-lever.md) — the four use cases fine-tuning is *good* at (style / voice, structured-output reliability, small-model distillation, narrow-domain specialisation) and the four it is *bad* at; the 500-example dataset floor; the same canary regime as any other variant; the two-line cost model with amortised training; the `fine-tuning-engineer` boundary.

## Exercises

- [exercise-01 — feature-flag-driven prompt rollout](exercises/exercise-01-feature-flag-driven-prompt-rollout.md).
- [exercise-02 — canary and shadow launch with gates](exercises/exercise-02-canary-and-shadow-launch-with-gates.md).
- [exercise-03 — A/B experiment interface to the experimentation platform](exercises/exercise-03-ab-experiment-interface-to-the-experimentation-platform.md).
- [exercise-04 — hot rollback of a bad prompt](exercises/exercise-04-hot-rollback-of-a-bad-prompt.md).
- [exercise-05 — hosted fine-tuning as an applied lever](exercises/exercise-05-hosted-fine-tuning-as-an-applied-lever.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references — LaunchDarkly, OpenFeature, provider fine-tuning docs, SRE canon on release engineering, canonical experimentation references.

## Cross-module context

- The **observability substrate** every rollout instruments against — the OTel Gen-AI span attributes (`app.experiment.arm`, `app.prompt.id`, `app.prompt.template_hash`, `app.feature`, `app.tenant_id`), the burn-rate alerts, the trace UI — lives in **mod-110**.
- The **eval substrate** the quality gates read from — offline eval sets, LLM-judge, judge-drift monitor, feedback triage pipeline — lives in **mod-106**.
- The **cost / latency substrate** the cost and latency gates read from — CPSO, per-surface latency spec, per-tenant accounting ledger, burn-rate alerts — lives in **mod-109**.
- The **model routing substrate** the flag layer's `model_id` value is served through — provider selection, residency, fallback — lives in **mod-108**.
- The **guardrail substrate** the kill switch trips into — layered moderation, PII scrubbing, human-in-the-loop escalation — lives in **mod-107**.
- The **prompt versioning discipline** the prompt registry stores — stable-prefix / volatile-suffix design, prompt-version doc with cache-shape table — lives in **mod-102** and **mod-109 chapter 02**.
- End-to-end ownership — a feature with a flag layer, a canary plan, an A/B interface, a rollback SLA, and a fine-tune where applicable — is the capstone in **mod-112**.

## Peer-role handoffs

- **`senior-ml-engineer`** — owns the deep experimentation methodology (variance reduction, sequential testing, heterogeneous treatment effects, delta-method estimators). Applied engineers hand them a clean interface (assignment / exposure / metric / guardrail) and the primary metric definition; peer role runs the analysis.
- **`model-evaluation-engineer`** — owns the LLM-judge design and calibration substrate the shadow-launch and canary quality gates read from. Applied engineers consume the judge's scores and drift alerts, not the judge's internals.
- **`fine-tuning-engineer`** — owns SFT vs PEFT vs DPO vs RLHF method selection, advanced dataset curation, catastrophic-forgetting audits, alignment tuning, and open-weights fine-tuning on private infra. Applied engineers run hosted SFT with clean pairs on a scoped shape problem and route the deep problems to the peer role.
- **`sre` / `platform`** — owns the alerting substrate the burn-rate alerts fire into, the paging system routing, and the on-call rotation ergonomics. Applied engineers own the LLM-feature-specific alert definitions and runbooks.
- **`privacy-engineer`** and **`data-security-engineer`** — own the DPA and residency envelope any fine-tune upload or shadow-launch cross-provider call must respect. Applied engineers scrub before upload and stay inside the envelope.
- **Product manager** — owns the "is this trade acceptable" decision when a launch shows a mixed result (primary metric up 1.2% but CPSO up 14%). Applied engineers make the trade legible; the PM decides whether to ship.

Naming these boundaries out loud is deliberate. Shipping discipline is where the *applied* engineer earns their keep; the peer roles' depth is what the applied engineer plugs into on the far side of the seam.
