# Retry, Repair, and Bounded Fallback

## Why "just retry" is a cost bomb

The naive schema-repair loop is a `while` loop with a validation check. The model returns something; you try to parse it into your Pydantic model; if that fails, you append the validation error to the messages and call again. If it fails again, you call again. You cap it at `max_retries`. Instructor calls this out of the box.

That loop is correct on the happy path and is a **cost, latency, and jailbreak amplifier** on the tail.

Consider the accounting. A retry that succeeds on the second try costs 2× as much (two full billed generations for one delivered result) and takes 2× as long. On a p95-latency SLO that budgets one generation, the retry can consume the entire budget and push the request over. On a per-successful-outcome cost target, a 5% retry rate raises the denominator not by 5% but by the average tokens-per-retry × 5% (retries are usually longer, because the failure message got appended and the model over-explains itself the second time).

Consider the jailbreak surface. Appending the validation error verbatim to the next prompt (`Your last response failed with: {error}`) means an adversarial input can now inject text via the error message. A malicious document that causes a controlled parse error whose message contains adversarial instructions is a known injection vector.

Consider the semantic drift. A model that was going to say "I don't have evidence for that" the first time may, when told "your answer failed validation, try again," instead invent a fake answer that parses. The retry loop can turn a correct refusal into a hallucination.

This chapter builds the retry-and-repair design that avoids all three failure modes: bounded budgets, differentiated retry strategies for different failure classes, sanitised repair prompts, and an explicit fallback so a schema failure has a defined product outcome.

## Classify the failure before you retry

A blanket retry is wrong because the failures have different causes and different repair strategies.

### Class 1 — Transport / infrastructure failure

Network timeout, 5xx from the provider, rate-limit 429, upstream stream aborted mid-generation. These have nothing to do with the schema and everything to do with the network.

- **Retry policy**: exponential backoff with jitter, respecting `Retry-After` on 429. 3–5 retries is defensible; more is a queue-management problem, not a retry problem.
- **Do not** append error messages to the next prompt — the model didn't fail; the network did.
- **Do** honour a circuit breaker: if the provider is having an outage, per-request retries only make it worse. Fail fast to the router (mod-108) and let it try a different provider.

### Class 2 — Truncation

The model hit `max_tokens` and returned an incomplete answer that fails validation because the JSON object was not closed.

- **Retry policy**: increase `max_tokens`. If you're already at model max, split the input.
- **Do not** just retry with the same params — you'll truncate again.
- **Root cause**: your token budget was miscalibrated. Log the truncation, and update the p95-output-tokens estimate for this surface.

### Class 3 — Schema validation failure

The model returned parseable output but it doesn't match the Pydantic model — wrong types, missing required fields, invalid enum values, regex failures.

- **Retry policy**: this is the class where a bounded repair loop is warranted. See "Repair with a sanitised error" below.
- **Cap tightly**: 1 retry for a user-facing surface, up to 2 for a batch pipeline. More than 2 in production is a smell — the model is not converging on your schema and the schema is the problem.

### Class 4 — Semantic validation failure

The output parses and matches the schema but a Pydantic `@model_validator` (or a downstream check) rejects it: cross-field constraint violated, referential integrity failed, confidence below threshold on a critical field.

- **Retry policy**: usually **do not retry**. A semantic failure means the model does not actually know the answer well enough. Retries produce hallucinations. Fall back to the "explicit not-sure" path (see below).
- **Exception**: if the semantic check is temporarily flaky (a downstream lookup failed), retry the check, not the generation.

### Class 5 — Safety / guardrail block

An output-side guardrail (mod-107) refused the response as unsafe.

- **Retry policy**: do not retry with the same prompt. A guardrail block is a policy signal, not a bug.
- **Do**: return the guardrail's refusal message to the user, log the block for review, and let the eval loop (mod-106) decide whether the block was a true positive.

## Budgets before code

Before writing the retry loop, write the budget table. Every retry class has three numbers.

| Class | Max attempts | Time budget | Token budget |
|---|---|---|---|
| Transport | 5 (backoff) | 8 s wall-clock | none (retries reuse the same call size) |
| Truncation | 2 | 15 s wall-clock | first retry at 1.5×, second at 2× |
| Schema validation | 1 | 5 s wall-clock added | repair prompt ≤ 200 tokens |
| Semantic validation | 0 | 0 | 0 |
| Guardrail block | 0 | 0 | 0 |

The exact numbers are surface-specific. Landing them in a table before writing the loop forces the argument early — "how bad is this call if it retries three times?" — instead of discovering it in production.

The **cumulative** budget across all retry classes on a single user turn is the number that matters for user-facing latency SLOs. If your end-to-end p95 is 6 seconds and a single generation is p95 3 seconds, the retry budget is 3 seconds total, not 3 seconds per class.

## Repair with a sanitised error message

For Class 3 (schema validation failure), the repair loop resends the prompt with information about what went wrong so the model can correct itself. The trap is to send the raw validation error.

Bad:

```
Your last response failed validation:
1 validation error for VendorContract
effective_date
  string does not match regex "^\d{4}-\d{2}-\d{2}$"
  input_value='03/07/26', input_type=str
```

That message contains user-controlled input (`03/07/26`) and any regex or schema strings the caller wrote. On a prompt-injection surface (mod-107), any part of the error that echoes upstream input is a channel for adversarial content.

Better:

```
Your last response did not match the required schema. Correct these fields
and return only valid JSON:
- effective_date: must be ISO-8601 date (YYYY-MM-DD)
- termination_notice_days: must be a non-negative integer or null
```

Rules for the repair message:

- **Describe the constraint, not the observed value.** "Must be an ISO-8601 date" is safe. "Your value 'DROP TABLE users;' failed" is not.
- **Enumerate only the failing fields, not the whole schema.** Adding the full schema on retry is a common quiet cost inflator.
- **Do not include the model's previous failing output as-is.** If you must reference it, echo only the field name and the expected type, not the offending value.
- **Cap the repair message length.** 200 tokens is plenty for most schemas.

A field-error → constraint-description map lives in your validation layer, not in the LLM prompt. Build it once per Pydantic model.

## Structured Outputs / grammar constraints eliminate Class 3

Recall from chapter 02: when the provider or serving layer enforces the schema at the sampler level (OpenAI Structured Outputs with `strict: true`, Anthropic's constrained tool-use, Outlines on a self-hosted deployment), Class 3 failures **do not occur by construction**. The retry loop is dead code for those paths.

Where those mechanisms are available, use them. The retry-and-repair loop is only warranted where the schema doesn't fit the constrained-decoding subset (nested unions, arbitrary `oneOf` shapes, patterns Structured Outputs doesn't support), or where you're on a provider path without strict enforcement, or where a semantic validator sits on top and can still fail.

## Self-correction: model-driven repair vs prompt-driven repair

Two different mechanisms sometimes both get called "self-correction." Keep them separate.

- **Prompt-driven repair.** The one described above: on failure, resend with a corrective message and retry. Simple, bounded, well-understood.
- **Model-driven self-correction ("critique-and-revise").** A pattern where the model, in a single generation, first produces a candidate output, then critiques it against the schema and requirements, then produces a corrected final output. Sometimes structured as a chain-of-thought scratchpad; sometimes as two turns.

Model-driven self-correction is not free. It roughly doubles the output length and cost, and empirical results are uneven — it helps on some tasks (math, code) and hurts on others (fact recall, refusal correctness, where the model may talk itself out of a correct "I don't know"). Do not add a critique-and-revise step by default. Reserve it for a specific surface where an eval (mod-106) shows it improves task success without breaking refusal correctness.

## Explicit fallback: what happens when repair fails

Every retry loop ends in one of two outcomes: success or fallback. **Fallback must be a designed product outcome, not a raised exception in the UI.**

Fallback options, in order of preference:

1. **Return "no result" as a first-class response.** For a co-pilot suggestion, this is "no suggestion available." For a structured extraction, this is a `RefusedExtraction` object with a reason. For a chat surface, this is an explicit "I couldn't produce a valid answer — could you rephrase?" (Not a 500.)
2. **Fall back to a different model or tier.** If the router (mod-108) is available, escalate to a stronger model. This is expensive and slow but preserves the outcome.
3. **Fall back to a rule-based extractor.** For structured tasks (contract clauses, invoice line items), a regex / keyword extractor with well-known limits is a legitimate fallback and often catches the low-hanging fruit.
4. **Queue for human review.** For high-value low-volume tasks (contract redlines, medical intake summarisation), a "we couldn't extract these fields, please review" queue is the right product surface.

The choice depends on the shape and the SLO. A batch pipeline can afford (2) and (4); a live streaming surface probably cannot afford (2) at the p95 budget and must fall back to (1) with an affordance to re-ask.

Fallback outcomes **must be logged as first-class events**, not as errors. They feed the eval loop and the "what fraction of requests hit the fallback path" health metric.

## Instrument the loop

Every retry attempt is a data point. Log per-request:

- The failure class that triggered the retry.
- The specific validation error (with values scrubbed for PII).
- Attempt number, cumulative token spend on this request, cumulative wall-clock.
- Final outcome (success on attempt N, or fallback path taken).

Aggregate:

- Retry rate per failure class (top-line signal; a spike is a prompt regression or a model regression).
- Cost-per-successful-outcome (mod-101 chapter 02) computed **with retry costs included**. A retry is not free and must show up in the denominator numerator ratio.
- Distribution of "attempts to success" — a bimodal distribution (mostly-1, sometimes-2) is healthy; a heavy tail past 2 is not.
- Fallback-path invocation rate. This is a product signal; if 10% of requests fall back, the feature is not working.

Two alerts worth writing:

- **Retry-rate step change.** A prompt or model change that spikes retries from 1% to 8% overnight is a regression, whether or not final success rates changed.
- **Cost-per-outcome step change.** A 20% cost bump with no traffic change is almost always retries or a lost cache-hit (chapter 01). Correlate with retry-rate to disambiguate.

## Sample flow: extraction pipeline with bounded repair

```python
def extract(document: str) -> VendorContract | RefusedExtraction:
    start = clock.now()

    # Class 3 handled by Structured Outputs — no retry needed
    raw = call_with_structured_outputs(
        document, response_model=VendorContract, timeout=3.0
    )

    try:
        model = VendorContract.model_validate(raw)
    except ValidationError as ve:
        # Class 3 — sanitised repair, one attempt
        remaining_time = 5.0 - (clock.now() - start)
        if remaining_time < 1.0:
            return RefusedExtraction(reason="budget_exhausted")
        raw = call_with_repair_message(
            document, error_summary=describe_errors(ve),
            timeout=remaining_time,
        )
        try:
            model = VendorContract.model_validate(raw)
        except ValidationError:
            return RefusedExtraction(reason="schema_repair_failed")

    # Class 4 — semantic validation, no retry
    semantic = validate_semantics(model)  # runs @model_validator
    if not semantic.ok:
        return RefusedExtraction(reason=semantic.reason)

    return model
```

Notice: `describe_errors` returns constraint descriptions, not raw error strings; `call_with_repair_message` shares nothing with the original document text beyond the field-name → constraint pairs; the time budget is checked before each new call; every exit is a first-class typed value, not an exception.

## Summary

- The naive "retry on failure" loop is a cost, latency, and jailbreak amplifier. Classify failures first.
- Five failure classes: transport (backoff), truncation (increase max_tokens), schema (bounded sanitised repair), semantic (do not retry — fallback), guardrail (do not retry — surface the refusal).
- Land the budget table before writing the loop: max attempts, time budget, token budget, per class.
- Structured Outputs and grammar-constrained decoding eliminate Class 3 failures by construction. Prefer those.
- Repair messages describe the constraint, not the observed value; do not echo user input back into the prompt.
- Model-driven "critique-and-revise" self-correction is expensive and unevenly beneficial — reserve for specific surfaces after an eval.
- Fallback is a designed product outcome — a typed "no result," a router escalation, a rule-based extractor, or a human-review queue.
- Instrument every attempt. Retry-rate step changes and cost-per-outcome step changes are the two alerts worth writing.

Chapter 05 turns the prompt, the schema, and the retry policy into versioned artefacts — a prompt registry, pinned regression fixtures, and a rollback path that lets you undo a bad prompt as quickly as you undo a bad code deploy.
