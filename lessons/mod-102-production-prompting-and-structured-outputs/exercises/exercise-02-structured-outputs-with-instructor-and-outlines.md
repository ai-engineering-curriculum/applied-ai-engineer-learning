# exercise-02: Structured Outputs With Instructor And Outlines

**Estimated effort:** 3 hours

## Objective

Build a schema-safe extraction pipeline for a real-shaped document (vendor-contract clause extraction) against two backends — one managed provider using Instructor + Pydantic + Structured Outputs (or the closest equivalent) and one self-hosted model using Outlines + Pydantic. Confirm that both backends produce the same downstream shape, and that grammar-constrained decoding truly eliminates schema-shape failures.

## Learning outcomes

By the end you should be able to:

- Model an extraction task as a Pydantic schema with per-field constraints, cross-field semantic validators, and a first-class "refusal" outcome.
- Wire Instructor to a managed provider (OpenAI or Anthropic) with `strict` / constrained-decoding mode where the provider supports it.
- Wire Outlines to a self-hosted model (vLLM, TGI, or SGLang) with a JSON-schema constraint derived from the same Pydantic model.
- Show empirically that schema-shape failures on the strict / grammar-constrained paths are zero, and that Instructor's retry loop is *not* the mechanism doing the work when strict mode is on.
- Layer semantic validation on top so a valid-shape wrong-content extraction is caught by a `@model_validator`, not by trusting the prompt.

## Prerequisites

- You've read [chapter 02](../02-schema-safe-structured-outputs.md).
- You have API access to at least one managed provider *and* a self-hosted inference server running a modern instruction-tuned open model (Llama 3.x instruct, Qwen 2.5 instruct, or DeepSeek — pick a size your machine can run). If you can't run a self-hosted server, use a hosted vLLM endpoint or a colab-shaped notebook.
- Familiarity with Pydantic v2 (`model_validator`, `field_validator`, `Field(pattern=...)`, `Literal` unions).

## Problem statement

You are the applied-AI engineer on a legal-ops team. Your input is a vendor contract as plain text (the PDF → text step happens upstream). Your output must be a validated JSON object matching the schema below, ready for insertion into the vendor-management system.

### The schema

Model these fields:

- `vendor_name` — string, non-empty.
- `effective_date` — ISO-8601 date (`YYYY-MM-DD`).
- `contract_type` — one of `msa`, `sow`, `nda`, `subscription`, `other`.
- `initial_term_months` — non-negative integer, may be `null` if not specified.
- `auto_renewal` — boolean.
- `termination_notice_days` — non-negative integer if `auto_renewal` is `true`; must be `null` if `auto_renewal` is `false`. (Cross-field constraint.)
- `governing_law` — string; must be one of a known set of US states + `england_and_wales` + `other` (define the enum).
- `total_contract_value_usd` — non-negative number, may be `null` if not specified.
- `line_items` — array of objects, each with `description` (string), `quantity` (non-negative integer), and `unit_price_usd` (non-negative number). If `total_contract_value_usd` is specified and any line items are present, `total_contract_value_usd` must equal `sum(quantity * unit_price_usd)` within a $1.00 tolerance. (Cross-field constraint.)
- `extraction_confidence` — float in `[0.0, 1.0]`.

Add a **refusal outcome**: a distinct Pydantic model `RefusedExtraction` with a `reason` field taking values from a known enum (`not_a_contract`, `unreadable`, `partial_document`, `budget_exhausted`, `semantic_check_failed`, `other`) and an optional `detail` field.

The extraction function returns `VendorContract | RefusedExtraction`.

## Requirements

### Part A — Managed provider path (Instructor + Pydantic)

Wire Instructor around your chosen managed provider (OpenAI `strict: true` or Anthropic tool-use). Requirements:

- The Pydantic model above is the single source of truth for the schema. Do not hand-write JSON Schema.
- Set `max_retries` explicitly to a small number (0 or 1 for a strict-mode surface, up to 2 for a non-strict fallback).
- Log per-request: attempts made, cached-input tokens (from chapter 01), total tokens spent, wall-clock, outcome (success vs refusal reason).
- Include a semantic `@model_validator` that enforces both cross-field constraints listed in the schema. When it fails, the pipeline returns a `RefusedExtraction(reason="semantic_check_failed", detail=...)` — not a raised exception.

### Part B — Self-hosted path (Outlines + Pydantic)

Wire Outlines against a self-hosted vLLM or SGLang server running an open model. Requirements:

- Derive the JSON-schema constraint from the same Pydantic model. Do not write two schemas.
- The sampler-level grammar guarantees the shape. Confirm this by asserting that `attempts_made == 1` on every request in Part D below.
- Semantic validation is still applied on top (the grammar cannot enforce cross-field constraints).

### Part C — Test fixtures

Create three small fixture sets under `fixtures/`:

- `fixtures/happy/` — 5 clean contracts that the model should extract successfully. You can synthesise these; they need to include one of each `contract_type` and one auto-renewal case with a non-null `termination_notice_days`.
- `fixtures/edge/` — 5 tricky contracts: missing dates in one, non-USD amounts in another, a `line_items` list whose total does not match `total_contract_value_usd`, an ambiguous governing-law phrase, and a contract with no explicit term length.
- `fixtures/refusal/` — 3 documents that are *not* contracts (a marketing email, a resume, an image caption). Expected outcome: `RefusedExtraction(reason="not_a_contract")`.

Do not use real vendor contracts. Synthetic or public-domain templates only.

### Part D — Confirm the two paths converge

Write a runner that:

1. Loads each fixture.
2. Runs it through Part A's backend and Part B's backend.
3. Records: outcome type, field values, attempts made, wall-clock, tokens.
4. Compares the two outcomes and asserts:
   - For happy fixtures: both backends return `VendorContract` and agree on critical fields (`vendor_name`, `contract_type`, `auto_renewal`, `governing_law`). Non-critical fields (exact `total_contract_value_usd` on ambiguous inputs) may differ; report the disagreement, do not fail on it.
   - For edge fixtures: at least one of the cross-field semantic checks fires on the fixture that's designed to trigger it. Both backends surface it as `RefusedExtraction(reason="semantic_check_failed")`.
   - For refusal fixtures: both backends return `RefusedExtraction` with a non-`other` reason.

## Starter guidance

- **Do not use two schemas.** The same Pydantic model must feed both Instructor and Outlines. If you find yourself writing a separate `json_schema.json` for one path, restructure.
- **Confidence is not a substitute for validation.** Instruct the model to return `extraction_confidence` — a self-reported number — but do not gate promotion on it alone. Use it as a signal for a downstream human-review queue, not as a validator.
- **Refusal is a valid product outcome, not an exception.** A `RefusedExtraction` is not an error. Do not raise; return it. Downstream code handles it.
- **The cross-field validators belong in the Pydantic model**, not in the prompt. Chapter 04's rule: schema validation catches the shape, `@model_validator` catches the meaning.
- **Structured Outputs subset.** OpenAI's strict mode does not accept every JSON Schema feature. Test your Pydantic → JSON Schema output with a small script before wiring; iterate on the schema (e.g. represent the conditional `termination_notice_days` constraint as a semantic validator instead of a JSON Schema `if/then/else`).
- **Outlines with vLLM.** vLLM has native support for guided decoding via Outlines-style regex / JSON schema. Check the current vLLM docs for `guided_json` / `guided_decoding_backend`; the API has evolved.
- **Deterministic runs.** For this exercise, set temperature to 0 (or the closest supported deterministic setting on your model) so you can compare backends without seed noise. In production this changes — but for the exercise, deterministic makes the diff readable.
- **Do not use real contracts.** Synthetic data only. Real vendor contracts are confidential and often subject to NDA; a solutions submission should not include them.

## Acceptance criteria

- [ ] A single Pydantic model (`VendorContract`) is the source of truth for the schema.
- [ ] Part A path wires Instructor with `max_retries` set explicitly. On strict-mode requests, attempts made == 1 on every happy fixture.
- [ ] Part B path wires Outlines with a grammar derived from the Pydantic model. Attempts made == 1 on every happy fixture (grammar guarantees this).
- [ ] `RefusedExtraction` is a first-class typed outcome, not a raised exception.
- [ ] The cross-field constraint on `termination_notice_days` vs `auto_renewal` is enforced by a `@model_validator`, not by prompt instruction.
- [ ] The cross-field constraint on `total_contract_value_usd` vs `line_items` is enforced by a `@model_validator`, with the $1.00 tolerance from the spec.
- [ ] The runner (Part D) compares both backends across all fixtures and prints a summary table.
- [ ] The 3 refusal fixtures both return `RefusedExtraction(reason="not_a_contract")` on both backends.
- [ ] Per-request logs include: attempts, tokens, wall-clock, outcome type. Retries on strict / grammar paths are zero.
- [ ] No real vendor-contract text in the submission.

## Stretch goals

Pick any two:

- **Provenance and citation.** Extend the schema so every field carries a `source_span` — the character offsets in the original document that support the value. Show which backend handles this cleanly and which requires prompt engineering.
- **Confidence calibration.** Take 20 happy-fixture runs, sort by `extraction_confidence`, and plot confidence vs. per-field correctness. Report whether the model's self-reported confidence is calibrated (higher confidence → higher correctness).
- **Non-strict fallback.** Add a third path: JSON mode without strict enforcement (OpenAI `json_object` without `json_schema`, or a Gemini path without structured output). Show how many attempts Instructor's retry loop makes on each fixture, and how many fail even at max_retries=2.
- **Semantic-check tightening.** Add one more semantic check (e.g. "governing law referenced in the text must appear as a substring of the extracted `governing_law` normalised form"). Show a fixture where the model happily returns a schema-valid but semantically wrong value, and the check catches it.
- **Cross-model portability.** Run Part A against two different managed provider models (e.g. an OpenAI Structured Outputs model and an Anthropic tool-use model). Report where they disagree on the edge fixtures.

## Deliverable

Submit as `mod-102/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `schema.py` — the Pydantic model, semantic validators, `RefusedExtraction` type.
- `managed.py` — Part A backend (Instructor + provider).
- `selfhosted.py` — Part B backend (Outlines + vLLM / SGLang).
- `runner.py` — Part D comparison runner.
- `fixtures/` — the three fixture sets, synthetic only.
- `results.md` — the runner's summary tables and any disagreement analysis.
- `report.md` — a short (≤ 1 page) discussion of what the "attempts made" numbers proved about which mechanism is doing the work.
