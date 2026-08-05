# exercise-01: Layered Moderation With FP/FN Report

**Estimated effort:** 3 hours

## Objective

Ship a *composed* moderation layer for a product surface — a **provider** moderation call plus a **classifier** guardrail — with an explicit **false-positive / false-negative report** generated from a labelled eval set. The exercise is not "wire up a moderation SDK." It is "build the artefact that will let you defensibly change a threshold next quarter."

## Learning outcomes

By the end you should be able to:

- Compose provider moderation (OpenAI / Anthropic / Gemini / Bedrock / Azure) with a classifier guardrail (Llama Guard, Prompt Guard, or provider content-safety) into a single moderation call for one moderation point on a product surface.
- Name the combining policy (`OR`, `AND`, or per-category weighted) and write it down.
- Assemble a labelled eval fixture set drawn from your incident library, from at least one public red-team corpus, and from synthetic augmentation.
- Produce a per-category FP/FN, latency, and cost report — versioned and re-runnable.
- Argue for a specific threshold change on the basis of the report's *delta*, not on absolute numbers.

## Prerequisites

- You have read [chapter 01](../01-layered-moderation-with-classifier-and-provider.md).
- A working provider account (any of OpenAI moderation, Anthropic classifier, Gemini `safetyRatings`, Bedrock Guardrails, or Azure AI Content Safety).
- A locally runnable classifier: Llama Guard (via `transformers` or an inference provider), Prompt Guard, or an equivalent — one is enough. If none is available, use a provider-hosted content-safety endpoint as the second layer and document that the "classifier" role is filled by a second provider.
- Python 3.11+ or Node 20+.
- A surface to moderate. If none from your day-job is appropriate, use the *support-agent-lite* or equivalent scaffold in `applied-ai-engineer-solutions/mod-107/scaffold/` (planned; if absent, adapt the mod-105 scaffold).

## Problem statement

Your team has enabled the provider's moderation endpoint on user input. There is no output-side moderation, no classifier layer, no evaluation of the false-positive rate against real conversations, and no runbook for "we want to tighten `self_harm` next quarter — what will that cost us?" A recent support ticket ("the assistant refused to talk about my medication schedule") sits in the queue and no one can defensibly say whether the refusal was correct.

In three hours, close the loop for **one moderation point** (pick input, output, or retrieval-side — do the whole thing well rather than a shallow pass on all three).

## Requirements

### Deliverable 1 — The moderation point and policy

Under `docs/moderation-policy.md`, one page:

- Which moderation point (`input`, `output`, or `retrieval`) you are wiring, and why that point is the highest-value first move for this surface.
- The category taxonomy you are using — either the provider's, the classifier's, or a merged set. Name each category and its intended surface behaviour.
- The combining policy: `OR`, `AND`, or per-category weighted. Justify in one paragraph.
- The streaming pattern (buffer / boundary / retract) if this is output-side moderation.
- The fail-closed / fail-open decision on classifier timeout, with reasoning.

### Deliverable 2 — The composed moderation call

Under `moderation/`:

- One module per layer (`provider.py`, `classifier.py`) exposing a common `classify(text) -> Verdict` shape.
- A `pipeline.py` that composes them per the combining policy and returns a single `ModerationVerdict` with the fields from [chapter 01](../01-layered-moderation-with-classifier-and-provider.md): `allow`, `categories`, `provider_flags`, `classifier_flags`, `reason_code`, `latency_ms`, `layer_versions`.
- A hard timeout on each layer (a few hundred ms) with the fail-closed / fail-open behaviour honoured.
- Structured logs (JSON) that record the verdict, the timings, and the layer versions.

### Deliverable 3 — The labelled eval fixture set

Under `fixtures/moderation/`, at least **200 items**. Every item is a versioned record with:

- `id`, `input` (the text under classification), `context` (the prompt that produced it, for output-side classification), `label` (`violating` or `benign`), `category` (from Deliverable 1's taxonomy, or `benign`), and `provenance` (`incident-library`, `public-corpus:<name>`, or `synthetic`).

Composition:

- ≥ 20 items from your incident library (or from the scaffold's seeded incident library if you are using it).
- ≥ 80 items from at least one **public** red-team corpus — RealToxicityPrompts, BBQ, AdvBench, MLCommons AILuminate hazards, or a comparable published set. Cite which.
- ≥ 60 items from **synthetic augmentation** (rephrasings, code-switching, hostile framings) generated from the two sets above. Keep synthetic items labelled with the same category as their seed.
- ≥ 40 items explicitly labelled `benign` in each of the categories the classifier tends to over-block (satire, medical vocabulary in legitimate contexts, discussions of policy or violence in the news).

Sensitive-content warning: some public red-team corpora contain very disturbing material. Vet the corpus you draw from and note explicitly in `docs/moderation-policy.md` what content is in the fixture set and who has access to it. The fixture set is subject to the same handling discipline as the traces in [mod-106 chapter 02](../../mod-106-feature-side-evaluation/02-trace-snapshots-as-regression-fixtures.md).

### Deliverable 4 — The FP/FN report runner

Under `bin/moderation-report`:

- Loads the pipeline, the fixture set, and the current threshold configuration.
- Runs the pipeline against every fixture, records the verdict, and computes per-category FP, FN, category confusion, latency (mean, P95), and cost per 1k calls.
- Emits `reports/YYYY-MM-DD.yaml` and `reports/YYYY-MM-DD.md`.
- Cache: `(input_hash, layer_versions)` keyed so re-runs are cheap and threshold sweeps are affordable.

### Deliverable 5 — The threshold-tune experiment

Under `experiments/`:

- Pick one category where you suspect the current threshold is wrong (either the classifier's default or the provider's default). Run the report at the current threshold; record `report-baseline.yaml`.
- Sweep the threshold across at least three values. For each, run the report; record `report-t<value>.yaml`.
- Under `experiments/analysis.md`, one page: name the current FP/FN, name the delta at each swept threshold, and *recommend* a threshold change with a one-sentence justification. This is the artefact a reviewer approves in production.

### Deliverable 6 — The "would-have-fired" replay

Under `proofs/would-have-fired.md`:

- Pick 5 recent, real (or scaffold-seeded) requests that reached the surface. Show what the moderation pipeline would have decided for each. For each, name whether the decision is correct against the safety policy, and — where wrong — which layer failed.

## Starter guidance

- **Start with one point, one category.** Wire input-side moderation for `hate` end-to-end before you add `self_harm` or the output side. Every category and every additional point is cheap once the pipeline is built.
- **Cache aggressively.** Threshold sweeps re-run against the same fixture set; cache verdicts on `(input_hash, layer_versions)` so a sweep costs pennies, not dollars.
- **Pin layer versions.** `openai/omni-moderation-latest` is not a version. Pin the dated model ID; log it in every verdict. When the provider rolls the alias, your delta will read as an unexplained shift otherwise.
- **Do not skip the benign fixtures.** A report that only measures how well you catch violations is not honest. The over-blocking is what your users experience.
- **Own the reason-code taxonomy.** Six months from now the dashboards will roll up incorrectly if `provider.hate`, `provider_hate`, and `hate_speech` are all present. Freeze the taxonomy in a single file.
- **Streaming is a rabbit hole.** If you pick output-side moderation, decide the streaming pattern up front (chapter 01's buffer / boundary / retract) and stick with the simplest that fits — usually buffer-before-render for a first cut.

## Acceptance criteria

- [ ] `docs/moderation-policy.md` names the moderation point, taxonomy, combining policy, streaming pattern (if applicable), and fail-open/fail-closed policy.
- [ ] `moderation/pipeline.py` composes provider + classifier layers, honours per-layer timeouts, and returns a `ModerationVerdict` with all the chapter-01 fields.
- [ ] `fixtures/moderation/` contains ≥ 200 items with the required composition (incident library, public corpus, synthetic augmentation, explicit benign categories).
- [ ] `bin/moderation-report` produces per-category FP, FN, category confusion, latency (mean, P95), and cost per 1k calls — machine-readable and human-readable.
- [ ] Reports pin layer versions.
- [ ] `experiments/analysis.md` recommends a threshold change on the basis of a report *delta*, not absolute numbers.
- [ ] `proofs/would-have-fired.md` shows five real-shaped requests and the correct classification of the pipeline's decision on each.
- [ ] No fixture set is used without noting its handling posture (who can read it, where it is stored).
- [ ] The scrub category from chapter 03 is applied to any trace produced by this pipeline (even the eval runner's logs should not leak the fixtures' raw text if the fixtures contain sensitive material).

## Stretch goals

Pick any two:

- **Per-language FP/FN.** Split the fixture set by language. Report per-language rates alongside the aggregate. Any provider classifier will degrade in non-English; the delta is worth publishing internally.
- **Judge-side sanity check.** For 20 disputed items (where provider and classifier disagree), run an LLM-as-judge with the safety policy in the prompt. Report agreement. Fold the disputes into a review queue.
- **Retract-on-flag streaming.** For output-side moderation, implement Pattern C from chapter 01: token streaming with a rolling classifier and a retract-on-flag UI hook. Prove the retraction is legible to the user (a policy message replaces the partial stream).
- **Cost / latency budget.** Add an explicit cost cap per 1k calls in the pre-registration file. When the pipeline breaches, the runner reports the breach as a hard failure. This is the pipeline-side version of [chapter 04](../04-tool-abuse-and-destructive-action-gates.md)'s per-tool cap.
- **Cross-provider fallback.** When the primary provider is unavailable, fall back to a secondary classifier with the same categorical output. Report the FP/FN under the fallback path separately; the two must be commensurable for the fallback to be safe.

## Deliverable

Submit as `mod-107/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/moderation-policy.md`.
- `moderation/` with `provider.py`, `classifier.py`, `pipeline.py`.
- `fixtures/moderation/` with the labelled set and a `PROVENANCE.md` naming every source.
- `bin/moderation-report` and `reports/` with at least one baseline report.
- `experiments/analysis.md` and the per-threshold reports.
- `proofs/would-have-fired.md`.
- A short `README.md` naming the surface, the moderation point, and how to run the report.
- Any stretch-goal artefacts named alongside.
