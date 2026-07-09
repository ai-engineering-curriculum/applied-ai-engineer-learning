# Schema-Safe Structured Outputs

## Why "just parse the JSON" is not a plan

The system downstream of an LLM call — a database write, a UI form, a workflow trigger, another model call — expects a shape. It expects field `vendor_name` to be a string, field `total_amount` to be a number, field `effective_date` to be an ISO-8601 date, and field `line_items` to be an array of objects with a known set of keys. If the model returns something else — a missing key, a stringified number, a date in `03/07/2026` format, an extra chatty preamble ("Here is the JSON you requested:") — the downstream system fails, silently or loudly, and you inherit the bug.

The naive plan is: prompt the model to "return valid JSON," parse the response with `json.loads`, catch exceptions, retry. That works for a demo and fails in production for the same reason that "trust the user's input" fails in a web form: the input distribution has a long tail, and every unhandled tail case is a runtime incident.

The production plan has three layers, in order of preference:

1. **Constrained decoding at the provider or serving layer** — the model literally cannot emit output that violates the schema.
2. **Schema-validated post-processing with bounded repair** — the model returns text; a validator either accepts it or triggers a bounded self-repair loop.
3. **Semantic validation on top of both** — schema validity is not correctness; a field can be structurally valid and semantically wrong.

This chapter walks the four mechanisms you will use to implement those layers, with their failure modes.

## Mechanism 1 — JSON mode (loose)

**JSON mode**, on OpenAI as `response_format={"type": "json_object"}` and on some other providers with similar names, tells the provider *"the response body must be a valid JSON object."* The provider constrains generation so the raw response parses as JSON. That is all it promises.

What it does not promise:

- **The JSON matches your schema.** JSON mode guarantees "parses as JSON," not "has field `vendor_name` of type string." Extra fields, missing fields, wrong types — all are valid JSON.
- **The values are meaningful.** `{"answer": "…"}` is valid JSON with a string field. So is `{}`.
- **The output is not truncated.** If the model hits the max-token limit mid-object, you get invalid JSON. Providers try to avoid this on JSON mode by finishing the object, but the guarantee is imperfect and depends on model / version.

JSON mode is a floor, not a solution. Use it when a downstream consumer merely needs *some* structured object and can validate the fields itself, and always pair it with a Pydantic (or equivalent) validation layer.

## Mechanism 2 — Function / tool calling and Structured Outputs

Function calling (originally OpenAI's `functions` API, now `tools` on OpenAI and Anthropic, and analogously on Gemini and Bedrock) lets you declare a JSON Schema for a tool's arguments and receive a tool-call object with those arguments filled in. The provider validates the arguments against the schema before returning them.

Two important shapes exist:

- **Best-effort schema conformance.** Older tool-call surfaces validate types loosely. `total: 42.5` in a field declared `integer` may pass through; `total: "42.5"` may or may not. You still need Pydantic on the receiving side.
- **Strict / guaranteed schema conformance.** OpenAI's **Structured Outputs** (available via `response_format={"type": "json_schema", "json_schema": {…, "strict": true}}` and on tool calls via `strict: true`) uses constrained decoding on the provider side: the model literally cannot emit a token that would violate the declared JSON Schema. Anthropic's tool-use surface has an equivalent shape and produces schema-conformant arguments in practice.

When Structured Outputs is available and your schema fits its subset (which excludes some JSON Schema features — see the provider docs for the exact list; `additionalProperties: false` is required on every object, and `oneOf` / `anyOf` / patterns / conditional schemas have restrictions), prefer it. When the schema does not fit, either simplify the schema (unions can be modeled as a `type` discriminant field with `oneOf` on the constrained subset) or drop to JSON mode + Pydantic.

<!-- needs-research: pin the exact current list of JSON Schema features supported by OpenAI Structured Outputs and Anthropic tool-use strict mode; both providers publish subsets that change. -->

### Function calling with side effects

An important layering point for later modules: a function call is *only* a message from the model saying "I want to call this tool with these arguments." It is not an execution. Your code decides whether to run the tool, with what identity, and returns the result to the model in the next turn.

That distinction matters here because Structured Outputs on tool calls buys you *schema safety on the arguments* — nothing more. It does not sanitise the semantics. `delete_user(user_id="admin")` is a schema-valid tool call. Authorisation, rate limits, and semantic guards live in your dispatcher, not in the schema. mod-104 owns the tool-execution surface; here we are only concerned with the shape of the arguments.

## Mechanism 3 — Instructor + Pydantic

[Instructor](https://python.useinstructor.com/) is a thin adapter around the major provider SDKs. You pass a Pydantic model as the `response_model`, and Instructor:

1. Serializes the Pydantic model to a JSON Schema.
2. Wires it into the underlying provider's structured-output surface — Structured Outputs on OpenAI, tool-use on Anthropic, function-calling equivalents elsewhere.
3. Parses and validates the response with Pydantic.
4. On validation failure, resends with the Pydantic error message as feedback, up to a configured retry budget.

The value of Instructor is not "it does something you cannot do yourself"; it is that it gives you one code shape across providers, with retry-on-validation-error already wired.

A minimal example:

```python
import instructor
from anthropic import Anthropic
from pydantic import BaseModel, Field
from typing import Literal

class VendorContract(BaseModel):
    vendor_name: str
    effective_date: str = Field(pattern=r"^\d{4}-\d{2}-\d{2}$")
    auto_renewal: bool
    governing_law: str
    termination_notice_days: int | None = Field(ge=0, default=None)
    contract_type: Literal["msa", "sow", "nda", "other"]

client = instructor.from_anthropic(Anthropic())

result = client.messages.create(
    model="claude-<pinned-model-id>",
    max_tokens=1024,
    max_retries=2,
    response_model=VendorContract,
    messages=[
        {"role": "user", "content": contract_text},
    ],
)
```

`result` is a `VendorContract` instance, not a dict and not a string. Pydantic has already validated types, the date pattern, the enum, and the non-negative constraint on `termination_notice_days`. If the model returned something that didn't parse, Instructor retried up to `max_retries` times with the validation error appended.

A production-shaped model does more:

- **Explicit `None` for missing.** Give every field a well-defined "not present in source" value. Do not use empty strings; those confuse downstream code.
- **Confidence and provenance per field.** A production extraction schema returns `VendorContract` as `{value, confidence, source_span}` per field, not just the value. mod-106 uses those confidences for calibration.
- **Refusal is a valid output.** A `RefusedExtraction` variant with a reason field is better than throwing away a request that the model correctly refused (e.g. "this is not a contract").

Instructor's retry loop is not free. Each retry is another billed request and another latency add. Bound `max_retries` at 1 or 2 for user-facing surfaces; go higher only for batch pipelines where wall-clock does not matter. Chapter 04 covers the retry-and-repair design in detail.

## Mechanism 4 — Outlines and grammar-constrained decoding (self-hosted)

[Outlines](https://dottxt-ai.github.io/outlines/) constrains the sampler on a self-hosted model so that the emitted tokens are guaranteed to match a grammar — a JSON Schema, a regex, an EBNF, or a Pydantic model. It works by masking the logits at each decoding step: tokens that would take the output off-grammar have their probability set to zero before sampling.

The consequence: **the model cannot emit invalid output.** Not "will not with high probability." Cannot. If your JSON Schema says `type: "integer"` for a field, no token that would make that field non-integer is sampleable at the position that field must appear.

Where this shines:

- **Self-hosted vLLM, TGI, or SGLang deployments.** Outlines integrates directly with these servers. On a self-hosted Llama or Qwen or DeepSeek deployment, grammar-constrained decoding is table stakes — you almost never want to run schema-free on your own weights when a downstream system depends on the shape.
- **Batch extraction pipelines.** When you're processing thousands of documents through a schema, grammar constraints eliminate the retry loop entirely — zero validation failures by construction.
- **Deep enums and unions.** JSON Schema features that Structured Outputs restricts are typically available in a grammar-based approach.

Where it does not help:

- **Managed provider APIs.** OpenAI, Anthropic, Gemini, and Bedrock do not expose their samplers. Outlines works against models whose sampler you control.
- **Semantic correctness.** A grammar guarantees the *shape*. A `total_amount: 0` is grammatically valid and semantically catastrophic. Semantic validation is still your job.

The rule of thumb: **on managed providers, use Structured Outputs (or the closest equivalent) plus Pydantic. On self-hosted models, use Outlines plus Pydantic.** Both paths converge on the same downstream shape — a validated Pydantic model — which lets a single application layer consume both.

## Semantic validation on top of schema validation

Schema validation catches the shape. Semantic validation catches the meaning. Both are your job, and they belong in the same layer.

Examples of semantic checks that are *not* schema-expressible in practice:

- **Cross-field constraints.** `end_date >= start_date`, `total_amount == sum(line_items.amount)`, `if auto_renewal then termination_notice_days is required`.
- **Domain-membership constraints.** `governing_law in known_us_states` is expressible as an enum, but a *large* enum (say 200 countries × 50 states) is impractical to encode in the schema and better as a post-validator.
- **Referential integrity.** A `vendor_id` that must exist in the vendor table.
- **Confidence thresholds.** If confidence for a critical field is below 0.7, the extraction is "unsure" and should be flagged for review, not silently accepted.

Pydantic's `@model_validator` and `@field_validator` hooks are the right home for these. Do not push them into the model prompt as instructions; the prompt cannot be trusted to enforce them, and the validator can.

## The four-mechanism decision

Which mechanism you reach for depends on the surface, the provider, and the schema.

| Situation | Reach for | Why |
|---|---|---|
| OpenAI / Anthropic / managed API, straightforward schema | Structured Outputs / tool-use (strict) + Pydantic | Provider-side constrained decoding, no retry-on-shape needed |
| Managed API, schema uses features Structured Outputs doesn't support | JSON mode + Pydantic + Instructor | Retry-on-validation-error covers the schema-mismatch tail |
| Self-hosted vLLM / TGI / SGLang | Outlines + Pydantic | Sampler-level grammar constraints, zero shape retries |
| Cross-provider router | Instructor + Pydantic | One code shape; provider details hidden behind the adapter |
| High-volume batch extraction | Self-hosted + Outlines + Pydantic | Grammar makes throughput predictable and cost-per-row known |

Do not mix mechanisms per-request. Pick one path per surface, land the Pydantic model on both, and centralise semantic validators in one place.

## Streaming and structured outputs — a note

Streaming a strict-schema response has a shape mismatch: schemas describe *complete* objects, not partial ones. Structured Outputs on streaming providers emits partial JSON that is not valid mid-stream and only becomes valid at the end. Consumers must not attempt to `json.loads` a mid-stream buffer. Chapter 03 covers the streaming design — partial-output rendering, incremental JSON parsers, and cancellation — in detail.

## Summary

- Four production mechanisms exist for schema-safe outputs: JSON mode (loose), function / tool calling (with or without strict mode), Instructor + Pydantic (validation + bounded retry), Outlines (grammar-constrained decoding on self-hosted models).
- JSON mode guarantees "parses as JSON" and nothing about your schema; use it only paired with Pydantic.
- Structured Outputs / strict tool-use on OpenAI and Anthropic performs constrained decoding on the provider side and eliminates shape-retry when the schema fits the supported subset.
- Instructor gives you one code shape across providers with retry-on-validation-error wired in; bound `max_retries` at 1–2 for user surfaces.
- Outlines constrains the sampler on self-hosted models and *cannot* emit off-grammar output; use it on vLLM / TGI / SGLang deployments.
- Schema validation catches the shape; semantic validation catches the meaning. Both live in the same Pydantic layer, not in the prompt.
- One mechanism per surface. Do not mix.

Chapter 03 takes the schema-safe output and adds streaming, partial-output rendering, and cancellation — end-to-end from the model back to the UI.
