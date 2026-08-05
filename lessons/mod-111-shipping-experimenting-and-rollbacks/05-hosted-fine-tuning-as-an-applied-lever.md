# Hosted Fine-Tuning as an Applied Lever

## Where this chapter fits

Every applied-AI engineer eventually hits a wall that prompting cannot cheaply climb. The system prompt is 4000 tokens and the model still forgets the output format half the time. The reasoning chain is right but the *voice* is wrong — brand-standard, tone-of-voice, or industry-jargon regressions that no amount of few-shot examples convincingly fixes. The support agent needs to output structured JSON with 30 fields and the top model does it 96% of the time, but a fine-tuned smaller model does it 99.8% and costs 1/10 as much.

At that wall, hosted fine-tuning becomes a legitimate applied lever. Not the first lever — a good prompt, a well-chosen model ([mod-108](../mod-108-model-routing-and-provider-selection/README.md)), and a well-designed retrieval story ([mod-103](../mod-103-rag-integration-in-product/README.md)) fix most problems more cheaply and with faster iteration. But when those levers are exhausted and the remaining gap is a *shape* problem (style, format, small distillation, narrow-domain specialisation), a hosted fine-tuning API can be the right applied answer.

The word that repays attention is *hosted*. The applied engineer's job here is to use OpenAI's fine-tuning API, AWS Bedrock's custom-model API, Google Vertex tuning, or Anthropic's tuned-model surface where available — the surfaces where you upload training pairs and get back a model ID you can call from your existing SDK. The *methodology* of fine-tuning — dataset curation depth, SFT vs PEFT vs DPO vs RLHF selection, hyperparameter sweeps, catastrophic-forgetting audits, alignment tax — is owned in the AICG track by the `fine-tuning-engineer` role. This chapter names the applied surface and the boundary; it does not teach you to derive LoRA.

## Four use cases hosted fine-tuning is *good* at

Fine-tuning is not a general upgrade path. It's a targeted lever for a small set of shapes:

- **Style / voice / tone.** The base model outputs correct answers in a voice that is subtly off-brand — too casual, too formal, too US-centric, doesn't use your product's vocabulary. A small SFT run with 200–500 examples of the desired voice on the same prompts gets you meaningfully closer than prompt engineering alone, especially when the base model has been RLHF'd toward a *different* voice.
- **Structured output reliability.** The base model outputs the right JSON 95% of the time but slips into markdown when the input is long. A fine-tuned model on 500–2000 examples of `(input → exact-schema JSON)` pairs pushes the reliability into the 99%+ range and lets you drop the belt-and-suspenders JSON-repair step from your surface.
- **Small-model distillation.** The top-tier model (`gpt-4o`, `claude-sonnet-4`, `gemini-2.5-pro`) does the task at, say, 92% quality and $12/1K outcomes. A fine-tuned smaller model (`gpt-4o-mini`, `claude-haiku-4`, `gemini-2.5-flash`) can hit 88% quality at $1.50/1K outcomes when trained on outputs from the top-tier model. The trade is 4 points of quality for an 8× cost reduction; whether that trade is worth it is a CPSO decision from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).
- **Narrow-domain specialisation with a stable vocabulary.** Legal templates, medical coding, semiconductor part numbers — domains where the vocabulary is fixed, the correct-answer distribution is narrow, and the base model's general knowledge is *underutilised* by the specific task. Fine-tuning here is a signal-to-noise argument: the base model is trying to be good at everything, and a narrow SFT tightens the distribution.

## Four use cases hosted fine-tuning is *bad* at

Equally important — the shapes where fine-tuning is the wrong applied answer, and where treating it as the answer wastes weeks:

- **Adding new factual knowledge.** Fine-tuning is *not* a substitute for retrieval. Baking facts into weights is expensive, hard to update, and prone to hallucination when the model is queried on adjacent facts it wasn't trained on. Facts live in a RAG index ([mod-103](../mod-103-rag-integration-in-product/README.md)); fine-tuning affects behaviour, not knowledge.
- **Fixing a hallucination problem.** The base model confidently states wrong things because it doesn't know the answer. Fine-tuning on "please don't hallucinate" examples usually *increases* hallucination confidence — the model learns to sound sure. The fix is retrieval, tool-use, or a "say I don't know" prompt discipline.
- **Chasing a moving target model.** You fine-tune `gpt-4o-2024-08-06`. Six weeks later, OpenAI ships `gpt-4o-2024-11-20` with a step-change in default quality. Your fine-tuned checkpoint is now underperforming the base model. You have to re-fine-tune on the new base — with cost, timeline, and eval re-run — every time. For rapidly-improving frontier models, this trap is real; a fine-tune costs multiple weeks of applied engineer time per generation.
- **Anything you don't have 200+ good examples of.** SFT-with-100-examples produces a model that has been mildly nudged in the direction of your examples and unpredictably degraded on adjacent behaviours. The floor for a defensible fine-tune is usually **500 high-quality examples**, and the floor for something that beats prompt-engineering in a controlled comparison is more often **1000–5000**. If you can't collect the dataset, you can't fine-tune usefully.

The one-line rule: **fine-tune to change behaviour on tasks you already have; don't fine-tune to make the model smarter.**

## The applied surface — what the four hosted providers actually offer

The hosted fine-tuning surface has coalesced into roughly the same shape across the major providers. What each offers, at the applied surface level:

### OpenAI

OpenAI's fine-tuning API supports **supervised fine-tuning (SFT)** and **direct preference optimization (DPO)** on their smaller models (currently the `gpt-4o-mini` and `gpt-4.1-mini` families, plus older-generation base models); larger models are gated behind program applications.<!-- needs-research: verify the exact current list of fine-tunable OpenAI models and the DPO availability at authoring time — this list moves. --> The workflow: upload a JSONL of messages arrays, POST to `/v1/fine_tuning/jobs`, poll until the job returns a `fine_tuned_model` ID, then call the ID like any other model. Pricing: a training charge (per token of training data times epochs) and an inference charge (usually 1.5–2× the base model's per-token rate). The fine-tuned model is your account's; nobody else can call it.

### Anthropic

Anthropic's tuned-model surface is available primarily via **Amazon Bedrock's custom-model program for Claude** (fine-tune Claude on Bedrock with your data, get a private endpoint). Direct-from-Anthropic fine-tuning is more restricted at time of writing — it exists but is program-gated rather than a self-service API surface.<!-- needs-research: pin Anthropic direct-fine-tuning availability at authoring time — this is in flux. --> The Bedrock path is the working applied lever for teams already on Bedrock: you upload your training data to S3, kick off a fine-tuning job, and get a custom-model ARN you can call from `bedrock-runtime`.

### AWS Bedrock (custom models)

Bedrock supports fine-tuning across several base-model families (Titan, Llama, some Claude variants), with the usual **SFT** shape (input / output pairs) plus **continued pre-training** for domain adaptation on unlabeled text. The fine-tuned model lands as a *provisioned throughput* endpoint — you buy capacity ahead of time, which is a different cost shape from OpenAI's per-token surface. For high-QPS workloads that would otherwise pay per-token to a hosted API, provisioned throughput on a fine-tuned Bedrock model can be dramatically cheaper; for low-QPS workloads, the provisioned-throughput minimum is prohibitive.

### Google Vertex AI

Vertex offers **supervised tuning** on Gemini family models (Flash-tier for the applied path, Pro-tier and Ultra-tier gated) and **RLHF / reward-model tuning** for teams with human-preference datasets. The workflow: upload training data to Cloud Storage or BigQuery, create a tuning job, deploy the tuned model to an endpoint. Vertex tuning is well-integrated with Vertex's monitoring and Model Registry — the fine-tuned checkpoints get an artefact ID that flows into Vertex ML Metadata.

### Common shape across all four

Regardless of provider, the applied engineer's checklist for a fine-tuning job is the same:

1. **A frozen eval set from mod-106** — the number you will judge the fine-tune against. Do NOT change the eval set to make the fine-tune look good.
2. **A training set** — JSONL of examples in the provider's expected format (usually messages arrays or completions). Cleaned, deduplicated against the eval set, and audited for PII.
3. **A hold-out validation set** — a slice of your training-shape data the provider uses to compute the training-loss curve. Not the eval set (which stays unpolluted).
4. **A commit** in your prompt registry that pins the fine-tuned model ID as the target for a specific `app.prompt.id` value.
5. **A canary rollout plan** ([chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md)) with the same four gates (quality, safety, cost, latency) as any other prompt or model change.

## The 500-example floor and why it matters

The 500-example floor is not a limit imposed by the provider; it's an empirical floor for producing a fine-tune that beats prompt-engineering. The reason is signal-to-noise: fine-tuning nudges the weights, and 50 examples nudge them by an amount indistinguishable from random parameter drift on any eval set with reasonable variance.

The applied engineer's dataset-curation loop:

- **Source examples from real user traffic**, not synthetic ones. Real inputs carry the distribution the model will see in production; synthetic inputs carry the distribution of whoever generated them (usually the applied engineer). The mod-110 feedback-triage pipeline ([mod-110 chapter 03](../mod-110-observability-feedback-and-incidents/03-user-feedback-triage-pipeline.md)) is the source-of-truth for this — thumbs-up turns become "the target behaviour," thumbs-down turns become "avoid."
- **Have a human expert relabel the outputs**, not just accept the model's outputs as targets. If the base model is imperfect and you fine-tune on its outputs, you cap the fine-tuned model's quality at the base model's — you cannot exceed the labels. For distillation this is fine (the goal is to *match* the top model at lower cost); for quality improvement it is not.
- **Balance the dataset**. If 90% of your training examples are the "happy path" scenario, the fine-tune will get worse at the 10% of edge cases. Sample intentionally across scenarios; the mod-106 taxonomy of scenarios is the vocabulary.
- **Check for eval-set leakage**. Every training example must be checked against the eval set (exact match, near-duplicate, and same-source scan). Leakage inflates eval scores by 5–15 points and produces a fine-tune that looks great in eval and regresses in production.
- **Audit for PII** at dataset construction time. Real user traffic includes real PII. If your DPA with the fine-tuning provider does not permit uploading PII, you must scrub before upload — even the training data. This is the same discipline as [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/README.md).

## The evaluation regime — same as any other model change

A fine-tuned model is a new variant. It goes through **the exact same rollout regime as a new prompt or a new base model**:

1. **Offline eval** against the frozen mod-106 eval set. Composite score + per-scenario floor.
2. **Shadow launch** on 5–20% of turns, comparing fine-tuned output to base-model output via the mod-106 judge. Any regression on a scenario blocks promotion.
3. **Canary at 1% → 5% → 25% → 100%** with the four gate families from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md). All gates apply — a fine-tune is not exempt because it "should be better."
4. **The rollback path from [chapter 04](04-hot-rollback-of-a-bad-prompt.md)** is wired: `support_agent.model_id` flag flips back to the base model in seconds.
5. **A monthly re-evaluation cadence** — the fine-tune's quality can drift as the provider retires or upgrades the base model, or as your user distribution shifts. Re-run offline eval monthly; re-fine-tune when the drift crosses a pre-registered threshold.

The fine-tune does not skip the canary because it "was rigorously trained." Every applied engineer who has ever run one has stories of the fine-tune that aced offline eval and blew up in canary because the training set didn't cover a real-user scenario. Ship it the same way you'd ship any other variant.

## Cost model — the two-line arithmetic

The economic decision for a fine-tune is a two-line comparison:

```
# Base-model baseline (per outcome):
  cost_base = (base_input_price_per_token * avg_input_tokens
             + base_output_price_per_token * avg_output_tokens)

# Fine-tuned model:
  cost_ft = (ft_input_price_per_token * avg_input_tokens
           + ft_output_price_per_token * avg_output_tokens)
  # Plus amortised training cost:
  amortised_training = training_cost_usd / expected_lifetime_outcomes
  cost_ft_all_in = cost_ft + amortised_training
```

Two shapes emerge:

- **Distillation to a smaller model.** `ft_price_per_token` is much lower (smaller model), `avg_input_tokens` may go down (shorter system prompt because the format is now baked in), and `amortised_training` is small if the fine-tune lasts months. Distillation is usually a net cost win *if* the quality trade is acceptable.
- **Fine-tuning the same-sized model for structured-output reliability.** `ft_price_per_token` is 1.5–2× the base rate (provider surcharge for hosting a private model), so per-token cost goes *up*, but the surcharge is offset if the fine-tune lets you drop retries, JSON-repair passes, or belt-and-suspenders validation steps that were adding 20–30% to `avg_output_tokens` or 15% to error-rate-driven retries. Compute the CPSO with and without the fine-tune, not just the per-token cost.

Do the arithmetic *before* the fine-tune, not after. A fine-tune that improves quality but doubles CPSO is often not worth shipping; a fine-tune that trades 3 points of quality for a 5× cost reduction usually is. The CPSO number from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) is the deciding artefact.

## The `fine-tuning-engineer` boundary — what applied does NOT own

There is a peer role in the AICG track — `fine-tuning-engineer` — that owns the methodology depth of tuning. The applied engineer should bring them in when:

- The **method** is not obviously SFT. If the problem is "we have preference data and want a DPO or RLHF run," that's fine-tuning-engineer territory. Applied does SFT with clean input/output pairs.
- The dataset requires **advanced curation** (multi-round annotation, active learning, adversarial-example generation, red-team-in-the-loop).
- The choice is **PEFT (LoRA / QLoRA) vs full fine-tuning** on an open-weights model deployed by your infra team; this is a joint decision with `ai-infra-performance`.
- You need to fine-tune **an open-weights model on your own infra** rather than through a hosted API. Applied engineering rarely gets this deep; the operational burden is significant and the peer role knows the pitfalls.
- **Catastrophic-forgetting audits** — the fine-tune improved the target behaviour and quietly degraded five adjacent capabilities. Detecting and quantifying this is peer-role work.
- **Alignment / safety re-tuning** — a fine-tune erased some of the base model's RLHF safety behaviour and you now need to argue about whether that's acceptable.

The applied engineer's stance: **you ship a well-scoped hosted fine-tune where the applied-scale problem is a shape problem, and you route the complexity to the peer role when the problem crosses the shape / method boundary.**

## Anti-patterns

- **Fine-tuning to add knowledge.** Facts live in retrieval. Fine-tuning bakes stale facts into weights and gets them wrong on adjacent queries. Use RAG.
- **Fine-tuning to fix hallucination.** The model learns to sound more confident, not to know more. Use retrieval, tool-use, and an explicit "say I don't know" prompt.
- **Fine-tuning without a frozen eval set.** You "evaluate" the fine-tune on the training set. It looks great. Production regresses. The eval set from mod-106 exists specifically for this discipline.
- **Fine-tuning on your model's own outputs at scale**. Quality caps at the base model's; distillation dynamics accumulate errors. The [Shumailov et al., "Model Collapse," Nature 2024] result is the deep version of this pitfall; the applied version is "you cannot exceed your labels."
- **Fine-tuning on frontier models that ship a new snapshot monthly.** Your fine-tune is obsolete before you finish the canary. Fine-tune on the tier below the frontier — the models that get *rev'd* less often, at lower cost.
- **Fine-tuning without a canary.** "It's rigorously trained, it should be fine." No. It runs through the same four-gate canary as every other variant.
- **Uploading unscrubbed user data to the fine-tuning provider.** DPA violation, GDPR exposure, and a data-residency incident all at once. Scrub before you upload; even the training data.
- **No amortised-training-cost line in the cost model.** The fine-tune looked cheap because training cost was ignored. Amortise over expected lifetime — training will need to be re-run when the base model updates or your data drifts.
- **A fine-tune "owned" by no one.** Nobody re-evaluates monthly, nobody re-fine-tunes when it drifts, nobody catches the deprecation of the base model. Every fine-tune has an on-call owner.

## Summary

- **Fine-tuning is a targeted lever, not a general upgrade path.** Reach for it when style, structured-output reliability, small-model distillation, or narrow-domain specialisation are the remaining problem — after prompt, RAG, and model choice have been tried.
- **Do not fine-tune to add knowledge or to fix hallucination.** Those are retrieval / tool-use problems. Fine-tuning bakes in stale facts and increases hallucination confidence.
- The **applied surface** is hosted fine-tuning: OpenAI SFT (with DPO on some tiers), Bedrock custom models (Titan, Llama, Claude on Bedrock), Vertex tuning (Gemini), Anthropic tuned-model program (largely via Bedrock at time of writing). All follow the same shape: upload data, run job, call the model ID.
- The **500-example floor** is empirical: fewer than 500 clean, real-traffic examples usually don't beat well-crafted prompting. The ceiling for a defensible fine-tune is 1000–5000+, curated from real user turns with human expert labels.
- A fine-tuned model is a **new variant**. It runs through the same offline eval → shadow → canary → GA → monitored rollback pipeline as any prompt or model change. Do not skip the canary because the eval scores look great.
- The **cost decision** is a two-line comparison: `cost_base` per outcome vs `cost_ft + amortised_training` per outcome. Do the arithmetic before the fine-tune. Distillation to a smaller model usually wins; same-model fine-tunes for reliability win only if they let you drop offsetting complexity.
- **PII scrubbing applies to training data.** Uploading raw user data to the provider is a DPA / GDPR violation even if the resulting model is private.
- The **peer role boundary**: `fine-tuning-engineer` owns SFT vs DPO vs RLHF selection, PEFT method choice, catastrophic-forgetting audits, alignment re-tuning, and open-weights fine-tuning on your own infra. Applied does hosted SFT with clean pairs; anything deeper routes to the peer role.
- Every fine-tune has an **owner**, a **monthly re-eval**, a **re-fine-tune trigger** on drift, and a **base-model-deprecation watch**.

This chapter closes the module. The shipping discipline — flags, canary, A/B, rollback, and now hosted tuning — is the *how* an applied-AI engineer moves changes into production without breaking users. [mod-112](../mod-112-owning-an-applied-ai-feature-end-to-end/README.md) is the capstone that composes all of it into ownership of a full feature.
