# exercise-05: Hosted Fine-Tuning as an Applied Lever

**Estimated effort:** 3 hours (2 hours + provider training time, which runs in the background)

## Objective

Run one hosted fine-tune from end to end on a *narrow shape problem* — style, structured-output reliability, or small-model distillation — using OpenAI SFT, Bedrock custom models, or Vertex tuning. Ship the fine-tuned model through the same offline eval → shadow → canary regime as any other variant. The exercise is not "learn LoRA." It's "prove you can identify a shape problem, curate a dataset, run a hosted job, ship the result safely, and know when to stop and bring in `fine-tuning-engineer`."

## Learning outcomes

By the end you should be able to:

- Diagnose whether a real feature's remaining quality gap is a *shape* problem fine-tuning can address, or a *knowledge* / *hallucination* problem it cannot.
- Curate a training set of ≥ 500 real-traffic examples with human-checked labels, deduplicated against your frozen eval set, PII-scrubbed.
- Kick off, monitor, and validate a hosted fine-tuning job on OpenAI, Bedrock, or Vertex.
- Compare the fine-tuned model against the base on offline eval, shadow, and canary — with the same gate discipline as any other variant.
- Model the two-line cost decision: `cost_base` per outcome vs `cost_ft + amortised_training` per outcome.
- Name the specific questions in this fine-tune that would justify escalating to `fine-tuning-engineer`.

## Prerequisites

- You have read [chapter 05](../05-hosted-fine-tuning-as-an-applied-lever.md).
- The flag layer ([exercise-01](exercise-01-feature-flag-driven-prompt-rollout.md)), canary regime ([exercise-02](exercise-02-canary-and-shadow-launch-with-gates.md)), and rollback substrate ([exercise-04](exercise-04-hot-rollback-of-a-bad-prompt.md)) are in place.
- Provider access with fine-tuning enabled: **OpenAI** (`gpt-4o-mini` or `gpt-4.1-mini` family currently supports SFT via the API), **Bedrock** (Titan / Llama / some Claude variants), or **Vertex AI** (Gemini Flash tuning). The exercise assumes OpenAI SFT for code samples; the shape carries over to Bedrock and Vertex.
- The mod-106 frozen offline eval set exists for the feature. This is *the* number you'll judge the fine-tune against.
- The mod-110 feedback triage pipeline exists — it's the source of the training examples.
- A billing budget: OpenAI SFT training on 500–1000 examples typically costs $5–$30 depending on model tier; Bedrock and Vertex have similar orders of magnitude. Confirm the budget with your PM before running.

## Problem statement

Your feature is the support-ticket summariser from prior exercises. The current model (`gpt-4o-2024-08-06` or the equivalent for your provider) produces summaries that are usually good, but 4% of the time it outputs the summary in *markdown* instead of the schema-required JSON, and the JSON-repair fallback adds ~250 ms of latency to those turns plus a 20% extra cost on the affected turns. The prompt team has tried three prompt-engineering fixes over two months and moved the failure rate from 6% to 4%; further prompt tweaks appear to have flat returns.

You suspect this is a **structured-output reliability shape problem** — the exact case chapter 05 names as fine-tuning-good-at. Over three hours (plus training time in the background): curate a training set, run one hosted fine-tune on the smaller model tier of your provider, ship it through offline eval + shadow + canary, and either promote or reject with numbers.

## Requirements

### Deliverable 1 — Diagnosis doc

Under `docs/tune-diagnosis.md`, a 1-page argument that this is a shape problem fine-tuning can address:

- What is the *observed* failure? Cite metrics: the 4% JSON-failure rate, the latency and cost delta on affected turns.
- What has already been tried? Cite the three prompt-engineering iterations and their observed effect.
- Why is this a shape problem, not a knowledge/hallucination problem? Argue against the alternatives: is the problem retrieval-shaped? tool-shaped? model-tier-shaped?
- What does success look like? A specific number: e.g., "JSON-failure rate ≤ 0.5% on the frozen eval set, at ≤ 1.2× the base model's per-token cost."
- What is the abort criterion? "If offline eval shows a JSON-failure rate above 1%, or if a scenario floor drops by more than 3 points, abort."

Freeze this doc before curation. It is the "why" that justifies the training run.

### Deliverable 2 — Training set curation

Under `data/training-set.jsonl`:

- **≥ 500** high-quality examples in the provider's expected format (OpenAI: `{"messages": [...]}` per line; Bedrock: provider-specific JSONL; Vertex: JSONL or CSV per Vertex's spec).
- Examples are sourced from **real user turns** via the mod-110 feedback triage pipeline. Prefer turns tagged as *thumbs-up with clean JSON output* — that's the target behaviour.
- Every output is **human-reviewed** against the schema. If the model's original output had subtle schema violations, fix them for the training example. Do NOT accept the model's original output uncritically as the label — you'll cap the fine-tune's quality at the base model's.
- The set is **deduplicated against the frozen eval set** — check by input hash, near-duplicate (Jaccard > 0.7 on token n-grams), and by source case ID. Any leakage invalidates the eval.
- The set is **PII-scrubbed** using the mod-107 chapter 03 scrubbing pipeline before upload. Even the training data. Document the scrubber's config in `docs/data-audit.md`.
- The set covers the mod-106 scenario taxonomy in **roughly the proportions the production traffic shows** — no oversampling of the happy path.

Under `docs/data-audit.md`:

- Sample-count table by scenario.
- Provenance: which pipeline pulled which turns, on what date, filtered how.
- Scrubber output: number of fields scrubbed by category (email, name, ID, secret).
- Leakage check result: 0 exact matches, 0 near-duplicates, 0 shared case IDs against the eval set.
- Human-review coverage: what fraction of examples got human relabelling; who reviewed.

### Deliverable 3 — Run the hosted fine-tune

Under `scripts/tune.sh` (or a notebook `scripts/tune.ipynb`):

- Upload the training set to the provider (OpenAI files API, Bedrock S3, Vertex GCS/BigQuery).
- Submit the fine-tuning job with these choices, and document each in `docs/tune-config.md`:
  - **Base model**: pick the smaller-tier fine-tunable model your provider offers (OpenAI: `gpt-4o-mini` or `gpt-4.1-mini`; Bedrock: Llama or Titan Express; Vertex: `gemini-2.5-flash`). Justify: smaller tier is cheaper to serve, faster, and more responsive to fine-tuning for shape problems.
  - **Hyperparameters**: use the provider's default epochs / learning rate multiplier / batch size unless you have a specific reason not to. Applied engineers do not tune hyperparameters; document that you're using defaults.
  - **Validation split**: hand the provider a hold-out slice of your training-shape data (usually 10%) for training-loss monitoring. **Do NOT hand them the mod-106 eval set** — that stays unpolluted.
- Poll until the job completes. Record the training loss curve, the validation loss curve, and the final `fine_tuned_model` ID (or Bedrock ARN, or Vertex resource name).
- Register the fine-tuned model ID in the prompt/model registry from [exercise-04](exercise-04-hot-rollback-of-a-bad-prompt.md), with metadata: `base_model`, `training_set_hash`, `job_id`, `cost_usd_train`.

### Deliverable 4 — Offline eval

Run the mod-106 frozen offline eval set against both the base model and the fine-tuned model. Under `docs/tune-eval.md`:

- Composite score for each variant.
- Per-scenario scores in a table.
- **JSON-failure rate** specifically (this is the primary success criterion from Deliverable 1).
- Any regression on any scenario ≥ 5 points is a **red flag** — call it out and decide whether to abort per your diagnosis doc.

If the fine-tune fails the offline eval gates you set in Deliverable 1, stop. Write up why the gates failed in the retro; do not proceed to shadow. That's a legitimate outcome — most first-attempt fine-tunes need a second iteration on the dataset.

### Deliverable 5 — Shadow launch

If offline eval passes, wire the fine-tuned model as a **shadow** on 10% of turns using the flag layer from exercise-01 and the shadow substrate from exercise-02. Run for at least 24 hours (adjust for your feature's traffic). Under `docs/tune-shadow.md`:

- Judge win-rate (fine-tune vs base) on paired turns.
- JSON-failure rate observed on shadow turns.
- Latency observation (informational; shadow is async, so this is the shadow's own p95, not the user-perceived latency).
- Cost observation: the CPSO of the shadow, tagged under `app.feature = "support_agent.tune_shadow"`.

### Deliverable 6 — Canary and ship

If shadow passes, canary the fine-tune through the same gate regime as exercise-02: 1% → 5% → 25% → 100%, with per-stage promotion artefacts under `docs/tune-canary-log.md`. At each stage, all four gate families are evaluated. If any gate fails, either roll back (per exercise-04) or hold.

### Deliverable 7 — Cost model

Under `docs/tune-cost.md`, the two-line comparison:

```
Base model per outcome:  $<A>
  = input_price * avg_input_tokens
  + output_price * avg_output_tokens

Fine-tune per outcome:   $<B>
  = ft_input_price * avg_input_tokens
  + ft_output_price * avg_output_tokens
  + training_cost_usd / expected_lifetime_outcomes

Delta:                   $<B - A>
  Δ%:                    <(B - A) / A * 100>%
  Break-even at:         <training_cost / (A - B)> outcomes if B < A
```

Include the assumed lifetime (in outcomes) for the amortised training cost — a fine-tune that lasts 3 months at your feature's rate produces a specific outcome count; use that. If you cannot amortise the training cost under a reasonable lifetime assumption (because the provider will deprecate the base model too soon), name it as an explicit risk.

### Deliverable 8 — Retro and peer-role note

Under `docs/tune-retro.md`, one page:

- Did the diagnosis hold? Was the problem in fact a shape problem?
- What was the total elapsed engineering time?
- What was the eval / shadow / canary outcome?
- Would you re-run the fine-tune when the base model updates? Under what trigger?
- **What specifically would justify escalating to `fine-tuning-engineer`?** Name concrete future scenarios: "if the JSON-failure rate returns to 2% after this fine-tune, we should try PEFT with a larger dataset — that's a `fine-tuning-engineer` conversation." Or: "if we want a DPO run on preference data next quarter, that's the peer role."

## Starter guidance

- **Pick a shape problem, not a knowledge problem.** The Deliverable-1 diagnosis is the deliverable that most often gets skipped and most often causes the fine-tune to fail. If you can't articulate a shape hypothesis in one page, don't run the training job.
- **Do NOT contaminate the eval set.** The mod-106 frozen eval is the *only* number that matters for the promotion decision. If your training set overlaps with the eval set, the fine-tune will look great and regress in production. Run the deduplication check and check it in as evidence.
- **Human-review the training labels.** If you accept model outputs as labels uncritically, you cap the fine-tune at the base model's quality. For a JSON-reliability fine-tune specifically, review the JSON structure of every label — this is exactly the failure mode you're trying to fix.
- **Do not tune hyperparameters as an applied engineer.** Use the provider's defaults. If defaults produce a bad fine-tune, that's a signal to work on your dataset (more common) or to bring in `fine-tuning-engineer` (rarer), not to sweep learning rates.
- **Fine-tune the smaller tier of the model family, not the top tier.** Applied fine-tuning wins are shape wins on smaller models; frontier fine-tunes are gated and often not a self-service surface. The smaller tier also makes distillation a viable secondary win.
- **Reserve budget for at least two training runs.** Almost every first attempt reveals a dataset problem (a class of examples was under-represented, some labels were inconsistent). Plan for iteration.

## Acceptance criteria

- [ ] `docs/tune-diagnosis.md` argues the problem is a shape problem, cites metrics, names the success and abort criteria, and is frozen before curation.
- [ ] `data/training-set.jsonl` has ≥ 500 examples in the provider's expected format, deduplicated against the eval set (0 leakage), PII-scrubbed, and human-reviewed.
- [ ] `docs/data-audit.md` documents provenance, sample counts by scenario, scrubber output, leakage check, and human-review coverage.
- [ ] `docs/tune-config.md` names the base model, hyperparameter choices (defaults, justified), and validation split.
- [ ] The fine-tune job completed; `fine_tuned_model` ID is registered in the model registry with training-set hash and cost.
- [ ] `docs/tune-eval.md` compares base vs fine-tune on composite, per-scenario, and JSON-failure metrics against the frozen eval set.
- [ ] If eval passed: `docs/tune-shadow.md` and `docs/tune-canary-log.md` cover the shadow and canary stages with gate outcomes.
- [ ] `docs/tune-cost.md` shows the two-line comparison with amortised training cost.
- [ ] `docs/tune-retro.md` closes the loop and names specific `fine-tuning-engineer` escalation triggers.

## Stretch goals

Pick any one:

- **Distillation angle.** Frame the fine-tune as a distillation — train `gpt-4o-mini` (or your provider's smaller tier) on outputs from `gpt-4o` (or the top tier), aiming for ≥ 5× cost reduction with ≤ 3 points quality loss. Report the trade in `tune-cost.md` and be explicit about the label ceiling.
- **Alternative-provider check.** Run the same fine-tune shape on a second provider (e.g., you did OpenAI; also try Bedrock's fine-tune of a Llama variant). Compare the eval and cost numbers. Note the operational differences.
- **DPO comparison (peer-role scope).** With `fine-tuning-engineer`'s guidance, prepare a preference dataset (thumbs-up vs thumbs-down pairs from the mod-110 feedback pipeline) and run a DPO fine-tune alongside the SFT one. Compare. This is intentionally a peer-role-supported step; document the interface.
- **Monthly re-eval scheduler.** Wire the offline eval to re-run against the fine-tuned model monthly, with an alert on quality drift crossing a pre-registered threshold. Document the trigger and the intended response (usually: schedule a re-fine-tune).

## Deliverable

Submit as `mod-111/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `data/training-set.jsonl` (or a pointer to storage if PII policy forbids checking in real data) and `docs/data-audit.md`.
- `scripts/tune.sh` or the notebook that ran the training job.
- `docs/tune-diagnosis.md`, `docs/tune-config.md`, `docs/tune-eval.md`, `docs/tune-shadow.md` (if reached), `docs/tune-canary-log.md` (if reached), `docs/tune-cost.md`, `docs/tune-retro.md`.
- The fine-tuned model ID and its entry in the model registry.
- Trace exports from at least the eval-run and shadow-window, showing spans tagged with the fine-tuned model.
- A short `README.md` naming the provider, the base model, the fine-tuned model ID, and the ship / no-ship decision.
