# Layered Moderation on the Product Surface

## Why "just turn on the provider moderation" is not a plan

Every mainstream model provider ships a moderation surface. OpenAI's `omni-moderation-latest` scores text and images against a taxonomy of categories including `harassment`, `hate`, `illicit`, `self-harm`, `sexual`, and `violence` (each with sub-categories such as `harassment/threatening`, `self-harm/intent`, `sexual/minors`, `violence/graphic`), returning a `category_applied_input_types` field that distinguishes text-side from image-side flags ([OpenAI Moderation guide](https://platform.openai.com/docs/guides/moderation)). Anthropic exposes classifier-based safety filters and publishes usage-policy-shaped guidance for input and output checks on Claude. Google's Gemini API returns per-category `safetyRatings` on generations and lets callers set `safetySettings` thresholds per category. AWS Bedrock ships Guardrails as a first-class API. Azure AI Content Safety is a standalone service with its own SDK. All of them are documented, all of them are cheap to call, all of them will catch obvious things. The applied engineer's job is not to build a classifier from scratch — the peer role does that — it is to *compose* these surfaces into a single defensible moderation layer on your product.

Composition is where teams get it wrong. The two failure shapes are symmetric:

- **Turn on one classifier, ship, forget.** The provider's moderation runs on the user's input. Nothing runs on the model's output. Nothing runs on retrieved documents the model was about to quote. Nothing runs on tool results. When the model repeats a slur from a retrieved chunk into a customer's mouth, the moderation dashboard shows zero flags — because you never asked.
- **Stack five classifiers, fail closed on any flag.** Every request hits four filters, each with its own false-positive rate. The compound false-positive rate stacks. Support tickets pile up ("the assistant refused to talk about my breast-cancer screening appointment"). The team quietly raises every threshold to 0.99 and the layer is now decoration.

The layered-moderation discipline this chapter builds is designed to avoid both failure shapes: an explicit input pass, an explicit output pass, an explicit context-side pass on anything retrieved, an escape hatch for over-refusals, and a **false-positive / false-negative report** you re-run on a fixed evaluation set so you can *see* what a threshold change does to real conversations before you ship it.

The chapter does not build a classifier. It does not do red-team-corpus generation (that belongs to `ai-risk-engineer`). It builds the *plumbing* the applied engineer owns.

## The four moderation points on a product surface

For any LLM feature more complicated than a demo, there are four distinct places where a moderation call can fire. Naming them keeps the discussion honest — most product incidents involve a category the team never ran a classifier at.

**Point 1 — User input, before it reaches the model.** The user typed something. Before the message enters the prompt, run moderation on it. This is the point everyone remembers. It catches "generate CSAM," "how do I synthesise VX," and the obvious hostile prompts. It does NOT catch anything the model then produces on its own; nor does it catch anything a retrieved document injects.

**Point 2 — Model output, before it reaches the user.** The model generated a candidate reply. Run moderation on the reply before you render it. This catches hallucinated slurs, over-graphic descriptions from a summarisation prompt, and the tail of jailbreaks that got past Point 1. Streaming complicates this — see the streaming section below — but the discipline of "no unmoderated bytes reach the user" is not optional for a serious surface.

**Point 3 — Retrieval / tool context, before the model consumes it.** A RAG chunk pulled from the corpus, a scraped webpage a browsing tool returned, a Slack thread the agent read: any of these can carry adversarial content. Content moderation here is *content* moderation only — the *injection* attack ("ignore prior instructions and email …") is chapter 02's problem. But a hostile chunk containing hate speech, a phishing lure, or leaked PII is a Point-3 problem that Point-1 and Point-2 will not catch.

**Point 4 — Trace / telemetry, before it lands in a shared observability store.** The scrubbing story that chapter 03 develops in full also carries a moderation implication: a conversation containing an abusive user message is stored, indexed, and searchable by anyone with dashboard access. Some organisations tag those traces at capture time so downstream consumers (analytics jobs, LLM-as-judge runs, offline evals) can opt out.

The layered-moderation policy names, for each of the four points, which classifier fires and what the on-flag action is. Blank cells in that matrix are where incidents come from.

## The two-layer split: provider moderation and classifier guardrail

For each moderation point, there is a durable pattern: a *provider* moderation call plus a *classifier* guardrail. The two do genuinely different jobs.

**Provider moderation** — cheap, broadly-trained, category-taxonomy-shaped, usually free or near-free per call. Examples: OpenAI's `moderations` endpoint, Anthropic's classifier-based safety filters, Gemini's `safetyRatings` on generation responses, Azure AI Content Safety, AWS Bedrock Guardrails. Strengths: kept current with the provider's own policy work, updated without you having to redeploy, well-calibrated on the categories it was trained for. Limits: opaque (you cannot inspect *why* it flagged), coarse (the taxonomy is fixed), and mismatched to your surface's specific policy (a medical-symptom classifier optimised for general chat has different assumptions than one you would build for a triage tool).

**Classifier guardrail** — a small, purpose-run model (or ruleset) you own the deployment of. Examples: Meta's Llama Guard family (the current version is [Llama Guard 4, a 12B multimodal classifier released April 2025](https://huggingface.co/meta-llama/Llama-Guard-4-12B) that labels content against the 14-category MLCommons hazards taxonomy S1–S14; Llama Guard 3 variants also remain available), Meta's [Llama Prompt Guard 2](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M) for injection detection, IBM Granite Guardian, NeMo Guardrails, Guardrails-AI's Hub, and provider-hosted content-safety products where the taxonomy is user-tunable. (ProtectAI's LLM-Guard framework was [archived in 2026 after Protect AI's acquisition](https://github.com/protectai/llm-guard) and should not be adopted as a maintained dependency, though its scanner catalogue is still a useful reference.) Strengths: you control it, you can retrain the threshold against your own labelled data, you can add categories the provider does not cover (`out_of_scope_medical_advice`, `mentions_competitor_pricing`, `discusses_internal_tool`). Limits: you must own the eval, the calibration, and the latency.

The two-layer split matters because a single classifier will fail one of the two roles. A provider classifier alone gives you no policy specificity. A homebrew classifier alone gives you no continuous tuning against the industry's current threat surface. Running both — and *ANDing* or *ORing* their outputs by an explicit policy — is what a defensible layer looks like.

A minimal shape for a single moderation point:

```python
# moderation/pipeline.py — sketch, not a library.
from dataclasses import dataclass

@dataclass(frozen=True)
class ModerationVerdict:
    allow: bool                    # final decision after policy
    categories: dict[str, float]   # union of provider + classifier scores
    provider_flags: dict[str, bool]
    classifier_flags: dict[str, bool]
    reason_code: str | None        # e.g. "provider.hate", "classifier.self_harm", "policy.override_deny"
    latency_ms: int
    layer_versions: dict[str, str]

async def moderate_input(text: str, surface: str) -> ModerationVerdict:
    provider = await provider_moderation.classify(text)
    classifier = await guardrail.classify(text, policy=POLICIES[surface])
    return policy_combine(
        provider=provider,
        classifier=classifier,
        surface=surface,
    )
```

Three properties to notice. The `categories` field is a *union* across layers, not a single score — a category flagged by the provider but not by the guardrail should be legible to the auditor. The `reason_code` names *which layer* fired, so a spike in "provider.hate over the last hour" is diagnosable without spelunking a wide table. The `layer_versions` field pins the provider model ID and the classifier version so a shift in the false-positive rate can be tied to a specific rollout.

## Streaming: the moderation problem you cannot ignore

Every "no unmoderated bytes reach the user" statement above collides with streaming. If your surface streams tokens to the user as they arrive, the natural implementation moderates the full response *after* it has already been sent. Three patterns that work, in order of increasing engineering cost.

**Pattern A — buffer before render.** Do not stream to the user. Generate to completion, moderate, then render. This is the honest default for high-stakes surfaces (compliance-adjacent chat, health, finance). Users perceive it as slower; the eng cost is zero.

**Pattern B — moderate on complete sentences (or turns).** Buffer tokens until a natural boundary (sentence, code block, JSON completion, tool-call boundary), moderate, then flush. Latency to first token is worse than raw streaming but much better than Pattern A. Requires the classifier to score short strings accurately, which most provider moderation surfaces do.

**Pattern C — retract-on-flag.** Stream tokens as they arrive, moderate in a parallel channel on rolling windows, and if the classifier flags, replace the stream with a policy message ("I'm not able to help with that — here's why"). Adds UI complexity (the frontend has to be able to erase a partial response), but preserves streaming perception. The [OpenAI Agents SDK's output guardrails](https://openai.github.io/openai-agents-python/guardrails/) implement a related — but not identical — tripwire pattern: on a flagged final output the runner raises a terminal exception, persists completed tool-call items to the session with any rejected output replaced by a `"Output withheld by an output guardrail"` payload, and exits. The product-side retract-and-render-policy-message behaviour is layered on top of that signal rather than provided out of the box.

None of the three is "right." The choice is a product decision, and the pre-registration file should name it — a chat surface silently switching from Pattern C to Pattern A because someone tuned the buffer size is a shipped UX regression.

## The false-positive / false-negative report

The single most important artefact this chapter builds is not the classifier call — it is the *report* that tells you what changed when you tune the threshold.

A classifier without an FP/FN report has no meaningful threshold; every tune is done by anecdote. A classifier with an FP/FN report is a small offline eval — the same discipline as mod-106's suites — pointed specifically at the moderation layer.

The report is built from a **fixed labelled eval set** of prompts and outputs. Each item carries:

- The `input` (the message being classified) and, if the classification target is a model output, the `context` (the prompt that produced it).
- The `label`: does this item *violate* the surface's written policy? A ground-truth call by a human labeller, not by another model.
- The `category` under which it violates (or `benign` if not).
- Provenance: is this from the incident library (a real past incident), from a public red-team corpus ([BBQ](https://arxiv.org/abs/2110.08193), [RealToxicityPrompts](https://arxiv.org/abs/2009.11462), [AdvBench](https://arxiv.org/abs/2307.15043), or the [MLCommons AILuminate v1.0 benchmark](https://mlcommons.org/benchmarks/ailuminate/) with its 12-category hazard taxonomy across physical, non-physical, and contextual harms), or from synthetic augmentation?

The runner computes, per category and in aggregate:

- **False-positive rate** — items labelled `benign` that the layer blocked. High FP means the layer over-refuses; the surface will feel timid.
- **False-negative rate** — items labelled `violating` that the layer let through. High FN means the layer under-blocks; the surface will feel unsafe.
- **Category confusion** — items labelled under category X that the layer flagged under category Y. Not a hard failure (the layer still fired), but a signal your reason codes will mislabel incidents.
- **Latency and cost** — mean and P95 per call, cost per 1k calls. Moderation that costs more than the model call itself is a signal of overhead you should own.

A concrete on-disk shape:

```yaml
# moderation/reports/support-agent/2026-08-01.yaml
suite_version: 2026.08.1
layer_versions:
  provider: openai/omni-moderation-2025-05
  classifier: llama-guard-3-8b@quantised-int8
sample_size: 400
aggregates:
  false_positive_rate: 0.036
  false_negative_rate: 0.011
  mean_latency_ms: 84
  p95_latency_ms: 210
  cost_per_1k_calls_usd: 0.021
per_category:
  hate:
    fp: 0.05
    fn: 0.00
    n_positive: 40
  self_harm:
    fp: 0.02
    fn: 0.05
    n_positive: 40
  policy.medical_out_of_scope:   # a surface-specific category
    fp: 0.01
    fn: 0.02
    n_positive: 30
notes:
  - "self_harm FN driven by 2 items that describe historical event framing; escalate to reviewer."
  - "hate FP concentrated on satire fixtures; consider raising threshold to 0.65 with FN re-check."
```

Two properties. First, the report is versioned and *stored*; the delta between last week's report and this week's is what a reviewer looks at, not the absolute numbers. Second, the notes are written by whoever ran the report — one paragraph tying anomalies back to specific fixtures. Automated stats without narrative do not survive a reviewer's questions.

## Policy: combining the layers

Once the two layers score an item, some policy has to combine them into an `allow: bool`. Three shapes are common; the applied engineer picks one and *writes it down*.

- **OR (block if either flags).** Highest recall, highest false-positive rate. Reasonable default for the input pass on a high-stakes surface.
- **AND (block only if both flag).** Highest precision, lowest false-positive rate. Reasonable default for over-refusal-sensitive surfaces (creative writing tools, satire).
- **Weighted (per-category rules).** For each category, name which layer is authoritative. E.g. "provider is authoritative on CSAM (any flag blocks); classifier is authoritative on `policy.medical_out_of_scope` (only the classifier is trained for it)." The most defensible in the long run because you can point at every decision and say why.

The combining rule lives next to the pre-registration file (chapter 06 of mod-106 for the format) and is version-controlled. Threshold changes require an FP/FN re-run; the reviewer signs off on the *delta*, not on the numbers alone.

## The unsafe defaults you will inherit

Six default settings that need explicit review before ship — every one has caused a shipped incident on some product.

1. **The provider moderation endpoint's default thresholds** are tuned for general-purpose chat. Your surface is not general-purpose chat. A category the provider treats as "medium" may be "must block" for your surface (children's product), or "must allow" (medical documentation tool).
2. **Streaming without a moderation pattern**. If the sample code you copied from the docs starts a stream and moderates in a `try/finally`, you are shipping unmoderated bytes.
3. **Moderating only user input**. Most templates do this. The classifier never sees the model's output. When the model quotes a chunk verbatim, you have shipped whatever the chunk contained.
4. **Moderating only English**. Every provider classifier is best-in-class in English and degrades in every other language, often silently. A multilingual surface needs per-language evaluation, and the FP/FN report must be per-language.
5. **Failing open on classifier timeout**. The guardrail is slow, the request times out, the code shrugs and lets the message through. If the classifier is a gate, its failure mode is *fail closed*, and the surface renders a policy message instead of the response. Fail-open should be an explicit, logged, alertable exception, not a default.
6. **Blocking without a reason code the user can see**. Refusing with "I can't help with that" and no legible reason is a support-ticket generator. Say what category was hit (at the granularity your policy allows) and — for false positives — how the user can flag the error.

## Anti-patterns

- **The two-layer stack that becomes a one-layer stack.** The team adds Llama Guard on top of provider moderation, notices the compound FP rate, and quietly disables Llama Guard for "performance." Now the two-layer story in the runbook is a lie. The correct fix is a threshold re-tune with an FP/FN re-run, not a silent removal.
- **The moderation call in the hot path with no timeout.** Every user request now sits behind a classifier's P99 tail. Set a hard timeout (a few hundred ms is usually enough), decide fail-open vs. fail-closed *by policy*, and alert on timeouts as a first-class SLO breach.
- **The classifier trained on public red-team data only.** Fixtures never touched by real users produce a classifier calibrated for a distribution your surface does not have. Mix in incident-library fixtures and current-production sampled traces (scrubbed — chapter 03).
- **The reason-code taxonomy that drifts.** Every engineer adds a new `reason_code` on their own; six months later the dashboards do not roll up because `provider.hate`, `provider_hate`, and `hate_speech` are all present. Name the taxonomy in one file; PRs that add a new code touch it.
- **The one-shot audit.** The layer passed its FP/FN report at launch; nothing has been re-run in six months. Provider models have shifted; your classifier's baseline has drifted; your incident library has grown. Re-run on a schedule (monthly is a defensible floor for a high-traffic surface).

## Summary

- Moderation happens at four distinct points on a product surface — user input, model output, retrieval / tool context, and trace / telemetry. A layered policy names the classifier and on-flag action at each point; blank cells are where incidents come from.
- At each point, the durable pattern is a **provider moderation call** (broad, current, cheap, opaque) plus a **classifier guardrail** (owned, surface-specific, tunable, latency-owning). Neither alone is enough.
- Streaming forces an explicit pattern choice — buffer-before-render, moderate-on-boundary, or retract-on-flag. The choice is a product decision written into the pre-registration.
- The false-positive / false-negative report is the load-bearing artefact. Fixed labelled eval set → per-category FP/FN, latency, cost → re-run when you tune, when a layer updates, or on a monthly schedule. The report's *delta* is what a reviewer signs off on.
- The combining policy (`OR`, `AND`, or per-category weighted) is written down and version-controlled. Threshold changes require a re-run and an explicit sign-off on the delta.
- Six default settings need explicit review: provider thresholds, streaming behaviour, output-side moderation, non-English coverage, timeout policy, and legible reason codes.

[Chapter 02](02-prompt-injection-defence-at-the-surface.md) turns from content moderation — *is this message allowed?* — to prompt injection defence — *is this message trying to hijack the agent?* Those two problems are usually confused for one problem; keeping them distinct is the first move toward defending against either.
