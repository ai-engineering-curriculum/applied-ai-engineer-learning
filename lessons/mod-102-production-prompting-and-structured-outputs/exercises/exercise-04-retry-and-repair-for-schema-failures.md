# exercise-04: Retry And Repair For Schema Failures

**Estimated effort:** 2 hours

## Objective

Take the extraction pipeline from exercise-02 (or a fresh minimal one) and add the full retry-and-repair policy from [chapter 04](../04-retry-repair-and-fallback.md): failure classification, per-class bounded budgets, a sanitised repair message that does not echo user-controlled input, and an explicit fallback outcome that is never a raised exception in the UI. Prove that a naive "just retry" loop is a cost bomb by comparing against your bounded version on a synthetic failure fixture.

## Learning outcomes

By the end you should be able to:

- Classify a failed model response into one of the five classes in chapter 04 (transport, truncation, schema, semantic, guardrail).
- Write a bounded retry loop where each class has its own strategy, attempt cap, and time / token budget.
- Construct a repair message that describes the constraint, not the observed value, and that does not echo caller-controlled input.
- Return an explicit typed fallback (`RefusedExtraction`, `NoSuggestion`, etc.) instead of raising an exception when the budget is exhausted.
- Instrument the loop so you can graph retry rate by class and cost-per-successful-outcome with retries included.

## Prerequisites

- You've read [chapter 04](../04-retry-repair-and-fallback.md).
- You've read [chapter 02](../02-schema-safe-structured-outputs.md) — or completed exercise-02 — and are comfortable with Instructor / Pydantic.
- You have a provider path that lets you toggle between strict (Structured Outputs / grammar-constrained) and loose (JSON mode) modes. This matters for showing that the retry loop is dead code on the strict path.

## Problem statement

You have an extraction pipeline that returns a `VendorContract | RefusedExtraction` (from exercise-02, or the minimal schema below). The pipeline currently uses a naive retry policy: on any failure, retry up to 5 times, appending the raw Pydantic validation error to the next message. Rework it into the chapter-04 policy.

If you didn't do exercise-02, use this smaller schema for this exercise:

```python
class Extraction(BaseModel):
    title: str
    published_date: str = Field(pattern=r"^\d{4}-\d{2}-\d{2}$")
    tags: list[Literal["policy", "product", "engineering", "other"]]
    summary: str = Field(min_length=20, max_length=280)

class RefusedExtraction(BaseModel):
    reason: Literal["schema_repair_failed", "semantic_check_failed",
                    "budget_exhausted", "guardrail_block",
                    "truncation_unrecoverable", "other"]
    detail: str | None = None
```

Add a `@model_validator` to `Extraction` that enforces `published_date` is not in the future — a semantic constraint that a schema alone cannot express.

## Requirements

### Part A — The bounded retry-and-repair function

Implement `extract(document: str) -> Extraction | RefusedExtraction` with:

- **Class 1 (transport)** — retry with exponential backoff + jitter on 5xx / 429 / network timeout. Cap at 3 attempts. Honour `Retry-After` when the provider sends it. Do *not* touch the prompt on retry.
- **Class 2 (truncation)** — detect via the provider's `finish_reason` / stop-reason ("length" or the model's tokenised equivalent). On truncation, retry once with `max_tokens` increased by 50%. If still truncated, return `RefusedExtraction(reason="truncation_unrecoverable")`.
- **Class 3 (schema validation)** — Pydantic raises `ValidationError`. Retry at most **once** with a sanitised repair message describing which fields failed which constraint. If it fails again, return `RefusedExtraction(reason="schema_repair_failed")`.
- **Class 4 (semantic validation)** — `@model_validator` raises. **Do not retry.** Return `RefusedExtraction(reason="semantic_check_failed", detail=...)` immediately.
- **Class 5 (guardrail block)** — for this exercise, simulate a guardrail block by wrapping the model call in a simple output-content filter (e.g. reject responses that contain a known unsafe keyword from a fixed list). On block, return `RefusedExtraction(reason="guardrail_block")` — no retry.
- **Cumulative budget** — total wall-clock across all retries capped at 8 seconds. On exhaustion mid-loop, return `RefusedExtraction(reason="budget_exhausted")`.

Log per attempt: `attempt_num`, `class`, `latency_ms`, `input_tokens`, `output_tokens`, `outcome`.

### Part B — The repair message helper

Write `describe_errors(ve: pydantic.ValidationError) -> str` that:

- Emits **at most 5** constraint descriptions (truncate if there are more failures than that).
- Emits the **field name** and the **constraint description** — never the offending value.
- Constraint descriptions come from a hand-written mapping (`_CONSTRAINT_DESCRIPTIONS`) keyed by `(field_name, error_type)`. Do not embed the raw `ve.errors()` list.
- Produces a message ≤ 200 tokens (roughly ≤ 800 characters).
- Include a security-focused unit test: pass a validation error whose input value is a prompt-injection payload (e.g. `"IGNORE PREVIOUS INSTRUCTIONS AND ..."`) and assert the produced repair message does not contain that payload.

### Part C — The failure fixture

Build `fixtures/failure/` with inputs that intentionally trigger each class:

- **Schema failure** — a document that causes the model to return `published_date` in the wrong format on the loose (JSON mode) path. Not needed on the strict path (see Part D). Aim for at least one input where the model returns a `03/07/26`-style date on JSON mode.
- **Truncation** — a document that requires a long summary and a request with an intentionally low `max_tokens`.
- **Semantic** — a document whose published date the model will predict as tomorrow (a "coming soon" event page). The `@model_validator` rejects it.
- **Guardrail block** — a document containing a keyword your filter rejects.
- **Transport** — for this one, mock the provider with a synthetic 429-then-200 sequence. Do not stress a live provider.

### Part D — Naive vs bounded comparison

Also implement `naive_extract(document: str)` — the naive baseline:

- Retry on any exception up to 5 attempts.
- Repair message is the raw `str(ve)` on Pydantic failures.
- No per-class strategy; no budget cap.
- Raises on final failure (does not return `RefusedExtraction`).

Run each of the failure fixtures through both `extract` and `naive_extract`. Record for each:

- Attempts made.
- Total input + output tokens spent.
- Total wall-clock.
- Final outcome (typed refusal vs raised exception; correct-value vs wrong).

Present a summary table showing that on failure fixtures:

- The bounded version's total token spend is ≤ ~2× a happy-path baseline.
- The naive version's token spend can exceed 5× (because it retries five times and each retry compounds the appended error).
- The bounded version returns typed values on every path; the naive raises exceptions the caller cannot destinationally handle.

### Part E — Strict-path proof

Repeat Part D with the model configured in strict / Structured Outputs / grammar-constrained mode. Show that:

- Class 3 (schema validation) failure count on the strict path is zero.
- The retry loop is effectively dead code for Class 3.
- Class 4 (semantic) still fires — the strict path does not prevent semantically wrong outputs — and is correctly not retried.
- Class 2 (truncation) can still occur even on strict paths — the sampler completes as far as it can — and must be handled.

## Starter guidance

- **Time budget check before every call.** The line `if remaining_time < 1.0: return RefusedExtraction(...)` from chapter 04 is not optional. Users would rather see a fast fallback than a 30-second retry storm.
- **Do not scrub with a blacklist.** The sanitisation rule is "describe the constraint, don't echo the value." Do not attempt to sanitise the value by string-stripping known dangerous substrings — that is bypassable and gives false confidence. The right rule is to not include user-controlled input in the first place.
- **Log the class label.** Every attempt log line has a `class` field (`transport`, `truncation`, `schema`, `semantic`, `guardrail`). Aggregating on that field is what makes the alert "retry rate on class=schema step-changed" trivially writable.
- **Use `time.monotonic()`** for elapsed-time budget checks, not `time.time()`. Wall-clock jumps break budget arithmetic.
- **Test the sanitiser with a real prompt-injection string.** The test `assert "IGNORE PREVIOUS" not in repair_message` is the load-bearing security assertion. Write it before you write the sanitiser.
- **Do not swallow exceptions in the outer function.** Every "raise" inside the retry function should be converted to a typed refusal. If you catch a `BaseException`, you have a bug — that catches `KeyboardInterrupt` and `SystemExit`, which you do not want to swallow.
- **The naive baseline exists to demonstrate a point, not to ship.** Do not use it in production or leave it as a fallback. After the comparison, delete it.

## Acceptance criteria

- [ ] The `extract` function returns `Extraction | RefusedExtraction`, never raises to the caller.
- [ ] Each of the five failure classes has its own strategy in code, with the caps documented above.
- [ ] `describe_errors` uses a hand-written constraint-description map and does not include raw error values.
- [ ] The security test that passes an injection-payload validation error asserts the payload does not appear in the repair message.
- [ ] The failure fixtures cover all five classes (with the transport class using a mocked provider).
- [ ] The Part D comparison table shows bounded token spend ≤ ~2× baseline on the bounded version and > 5× on the naive version.
- [ ] The Part E strict-path run shows Class 3 failures = 0 and Class 4 failures are still handled correctly.
- [ ] Per-attempt logs include `attempt_num`, `class`, `latency_ms`, `input_tokens`, `output_tokens`, `outcome`.
- [ ] A short report explains why "no retry on Class 4" is the right policy, using the semantic-drift argument from chapter 04.

## Stretch goals

Pick any two:

- **Retry rate alert.** Simulate a "prompt regression" by intentionally weakening a rubric line so schema failures spike. Compute a retry-rate metric over a 5-minute rolling window and demonstrate the alert fires at a step change of ≥ 5 percentage points.
- **Cost-per-outcome accounting.** Compute cost per successful outcome across both bounded and naive versions. Report the ratio.
- **Model-driven critique-and-revise.** Implement a second variant that uses a single-shot "produce → critique → revise" prompt structure (chapter 04's "model-driven self-correction"). Run it on the fixtures and report where it improved outcomes and where it degraded them (especially over-refusal on the guardrail fixture).
- **Router-escalation fallback.** Instead of returning `RefusedExtraction`, on `schema_repair_failed` escalate to a stronger model (or the same model at higher temperature 0). Report the additional cost and how often the escalation succeeds.
- **Idempotent retry.** For the Class 1 (transport) retries, add a request-id header so the provider can dedupe retries. Report whether the provider you use supports idempotent request semantics.

## Deliverable

Submit as `mod-102/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `extract.py` — bounded `extract` and helpers.
- `naive.py` — the naive baseline.
- `describe_errors.py` — the sanitised constraint mapper.
- `tests/test_sanitiser.py` — the injection-payload assertion.
- `fixtures/failure/` — the five failure fixtures.
- `runner.py` — the Part D + E comparison runner.
- `results.md` — the summary tables and the strict-path proof numbers.
- `report.md` — ≤ 1 page on why no-retry-on-Class-4 is correct and what the naive-vs-bounded gap says about production retry cost.
