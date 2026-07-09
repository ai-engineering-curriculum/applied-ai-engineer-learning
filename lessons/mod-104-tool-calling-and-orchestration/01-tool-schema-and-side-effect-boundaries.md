# Tool Schema and the Side-Effect Boundary

## Why the tool interface is the tool

A tool call, at the wire level, is a JSON blob whose keys the model chose. Everything the model can and cannot do with a tool is set by the schema you publish and the guards you wrap around the handler. If the schema is loose, no amount of prompting will keep the model from filing a `refund_amount = 1e9` request or a `delete_user(user_id="admin OR 1=1")`. If the schema is tight but the handler treats every call as a fresh, side-effecting action, a retry that the model or the runtime issued will double-charge a customer.

The mental model to hold in your head is this: **the tool schema is a public API against a non-deterministic caller.** The caller is fast, tireless, and will happily invent arguments that look plausible in-distribution but violate every constraint you didn't write down. Design the schema the way you would design an API for the least-trusted client on the internet — because that's what it is.

This chapter covers the three artefacts an applied-AI engineer owns on the tool side of a product agent: the schema (what the model may send), the validator (what the runtime accepts before invoking the handler), and the side-effect boundary (what the handler will actually do to shared state). The other four chapters build on this one — the error taxonomy assumes you have a validator to reject on, the orchestration chapter assumes tools have deterministic keys, the budget chapter assumes the schema declares a cost.

## The four things a tool schema must state

At minimum, every tool declaration a production agent ships needs to say four things, in this order of importance.

1. **What the tool is for**, in plain English, in one or two sentences the model will read as the natural-language description. This is the load-bearing prompt for tool selection. Get it wrong and the model calls the wrong tool.
2. **What arguments are valid**, as a strict JSON Schema (or the provider's equivalent). "Strict" means `additionalProperties: false`, `required` set for every non-optional field, enums and patterns wherever the domain is finite.
3. **What the tool returns**, so the model can plan what to do with the result. Some providers accept this as part of the schema; where they don't, put it in the description ("Returns `{ "status": "created" | "duplicate", "id": string }`").
4. **What side effects the tool has**, encoded in the tool's *name*, its *description*, and its *idempotency contract* — not just in the handler's implementation. The model must know from the name whether it's about to buy something.

### A canonical shape, provider-agnostic

```python
# tools/refunds.py — a side-effecting tool declared strictly.
from typing import Literal
from pydantic import BaseModel, Field, StringConstraints
from typing import Annotated

class IssueRefundArgs(BaseModel):
    model_config = {"extra": "forbid"}  # additionalProperties: false at generation

    order_id: Annotated[str, StringConstraints(pattern=r"^ord_[A-Za-z0-9]{16}$")]
    amount_cents: int = Field(ge=1, le=500_00)  # domain cap, not a runtime cap
    currency: Literal["USD", "EUR", "GBP"]
    reason_code: Literal[
        "customer_request", "duplicate_charge", "quality_issue",
        "shipping_issue", "policy_exception",
    ]
    idempotency_key: Annotated[str, StringConstraints(min_length=8, max_length=64)]
    note: str | None = Field(default=None, max_length=280)

ISSUE_REFUND_TOOL = {
    "name": "issue_refund",
    "description": (
        "Issue a partial or full refund on a completed order. THIS CHARGES MONEY. "
        "Use only after confirming the amount and reason with the user. "
        "Returns { status: 'issued' | 'duplicate_key' | 'declined', refund_id: string, "
        "declined_reason: string | null }. "
        "The same idempotency_key will never issue a second refund; on duplicate the "
        "existing refund_id is returned with status 'duplicate_key'."
    ),
    "input_schema": IssueRefundArgs.model_json_schema(),
}
```

Read that carefully. Every choice is defensive.

- `order_id` has a regex, not just `str`. A model that hallucinates `"order 47"` fails at validation, not inside your billing service.
- `amount_cents` is a bounded integer. `500_00` (500 dollars) is a **domain cap** — the largest refund this tool is *authorised* to issue. Larger refunds go through a human queue. This is not the per-user quota (that's `chapter 05`); it's a schema-level statement of what this tool can even represent.
- `currency` is an enum. The model cannot invent `"BTC"`.
- `reason_code` is an enum. If it were a free-text `reason`, downstream analytics would rot within a week.
- `idempotency_key` is required, not optional. See "Idempotency is a schema field" below.
- `additionalProperties: false` (via Pydantic's `extra: "forbid"`) closes the door on the model inventing a `"metadata"` bag full of prompt-injected instructions.
- The description names the side effect in the first sentence and shouts THIS CHARGES MONEY. Models read tool descriptions when planning; a low-stakes-looking name attached to a high-stakes tool is the setup for a bad demo.

None of this is exotic. All of it is the JSON Schema spec applied with intent.

### JSON Schema is the wire format; Pydantic / Zod / etc. is the source

Anthropic, OpenAI, and Google Gemini all accept JSON Schema (with provider-specific dialect quirks) for tool arguments. Anthropic Messages: `tools[i].input_schema`. OpenAI Chat Completions and Responses: `tools[i].function.parameters`. Gemini: `tools[i].functionDeclarations[i].parameters`.

Writing that JSON Schema by hand is a mistake — it drifts from the runtime validator and it gets copy-pasted incorrectly. Pick a source-of-truth type (Pydantic in Python, Zod or `@sinclair/typebox` in TypeScript, `pydantic-ts` if you're crossing languages) and generate the schema from the type. The same type is what your handler receives after validation.

Two watch-outs:

- **Strict mode is provider-specific.** OpenAI's Structured Outputs requires opting into `strict: true` on function tools, at which point the schema is enforced at the sampler; without it, arguments are best-effort. Anthropic's tool use validates against the schema but does not decode against a grammar by default. If you want guarantee-by-construction that the argument JSON parses and matches, use the provider's strict / grammar-constrained mode where available and check the current provider documentation for supported subsets.
- **Not every JSON Schema keyword is supported everywhere.** Regex (`pattern`), `oneOf` with complex branches, tuple typing, `$ref` — support varies. Keep your schema in the intersection of what your providers accept. When a keyword is unsupported, move the check to the validator, not the model.

## Argument validation: the belt around the sampler

Even with strict decoding, a validator wraps every handler. Treat provider-side schema enforcement as a hint, not a guarantee — provider bugs happen, `strict: true` gets accidentally dropped in a code path, non-strict fallbacks exist for prompt caching or streaming quirks. The pattern:

```python
async def call_issue_refund(raw_args: dict, ctx: RequestContext) -> ToolResult:
    try:
        args = IssueRefundArgs.model_validate(raw_args)
    except ValidationError as ve:
        return ToolResult.model_error(
            code="invalid_arguments",
            message=describe_errors(ve),  # constraint-based, PII-scrubbed
        )

    # Post-schema, pre-side-effect authorisation.
    if not ctx.principal.can_refund(order_id=args.order_id, amount_cents=args.amount_cents):
        return ToolResult.permission_error(
            code="not_authorised",
            message="This principal is not permitted to refund this order.",
        )

    return await execute_refund(args, ctx)
```

Three principles in that shape are non-negotiable.

**1. The validator returns a typed error, not an exception.** A raised exception on invalid args often terminates the whole request. A typed `ToolResult` gets fed back to the model as a `tool_result` block with `is_error: true` semantics, the model reads the failure, and one bounded repair attempt has a chance to fix it (chapter 02). Exceptions are for genuinely unexpected system failures.

**2. Authorisation runs after schema validation, before the side effect.** Two separate checks in two separate places. Skipping the second check because "the model wouldn't have called it if it wasn't allowed" is the classic reason a red-team prompt manages to refund an order the caller doesn't own.

**3. Every error message is sanitised.** The message the model reads should describe the constraint ("amount_cents must be a positive integer at most 50000") not the offending value ("your value 1e9 is too large"). Echoing the offending value back is the injection channel described in mod-102 chapter 04 — an adversarial input can smuggle instructions through the error path.

### `describe_errors` — a shared utility, not per-tool code

Every tool in the surface uses the same error-description function. It maps Pydantic error paths (`("amount_cents",)`, `("idempotency_key",)`) to human-readable constraint statements. The mapping lives in one file and is unit-tested; the tool handlers do not build error strings inline.

## Side-effect boundaries: idempotency, dry-run, undo

The most damaging class of production incident isn't the tool that raised an exception. It's the tool that succeeded twice.

### Idempotency is a schema field

Any tool that changes shared state — creates, updates, deletes, sends, charges — must take an `idempotency_key` as a required argument, and the backend must guarantee that the same key produces the same effect at most once. This is not a nice-to-have; it is what allows the orchestration layer (chapter 04) to safely retry a network hiccup, and what allows the model to be told "your last tool call may have gone through — try again" without doubling the effect.

The pattern is well-established outside of AI. Stripe's API has documented idempotency-key semantics for exactly this reason: "we save the resulting status code and body of the first request … Subsequent requests with the same key return the same result." Every side-effecting tool your agent calls should provide that guarantee, either because the underlying service does (Stripe, Twilio, some Salesforce endpoints, some AWS SDKs) or because you're wrapping the service in a small persistence layer that does.

Who generates the key? Two acceptable answers:

- **The agent host generates it deterministically from the tool call's canonical form.** This lets the model resubmit the exact same call and get the same result. Downside: if the model tweaks the arguments on retry ("try again with amount rounded to whole dollars"), the key changes and you get a second refund.
- **The model generates the key, but the schema requires it and the description explains the semantics.** This puts the burden on the model to pick a stable key across a repair loop. Combined with a strict schema check (`idempotency_key` must be present, must be ≥ 8 chars), this works well when the model is a capable frontier model.

The safer default for high-blast-radius tools is the first pattern with a manual override — the host injects a canonical key unless the model provides one, and the model is prompted to only override the key when it explicitly wants a *new* effect.

### Dry-run mode as a schema variant

For destructive tools, providing a companion `preview_x` tool that runs the query without committing lets the model self-check ("show me what I'm about to delete") before the real call. This is not always affordable — a dry-run refund isn't meaningful because the effect is discrete — but a dry-run "bulk update customer records" is a lifesaver when a model wants to touch a thousand rows.

Two shapes:

- **A separate tool.** `preview_bulk_update` and `commit_bulk_update` with different names. The model sees them both. This is verbose but explicit.
- **A `dry_run: bool` on the same tool.** Compact, but the model may forget to set it. If you pick this shape, default `dry_run: true` in the schema and make committing require an explicit `dry_run: false` argument.

Whichever shape you pick, the dry-run response must include everything the model needs to decide — for a delete, the count of rows and a sample; for a message send, the rendered content; for a schema migration, the diff. A dry-run response the model can't reason about is a dry-run that gets skipped.

### Undo, compensating actions, and the "safe to fail" default

For tools where dry-run isn't structurally available, the second-best defence is an **undo path** that the agent can invoke if the outcome was wrong. In practice this means:

- Every write tool that has an undo has a paired `undo_x` tool with the same idempotency guarantees.
- Every side-effect log entry stores a "compensating action" descriptor (e.g., for a refund, the payment intent to void; for a message send, the message ID to delete).
- The orchestration layer (chapter 04) knows how to build a compensating-action sequence when a downstream step fails after upstream side effects committed.

Tools that don't have a natural undo — sending a customer email, publishing to a public channel, triggering a physical fulfilment — must be gated behind a stronger confirmation path. In a co-pilot surface, that usually means the tool is not called by the agent directly; the agent proposes it, and the user clicks a button that calls it.

The rule to state in a design review: **a tool the agent can call unaided is a tool with either dry-run, undo, or a bounded, well-tested blast radius.** Anything else is a tool the user has to confirm before it runs.

## Naming as a first-class concern

Tool selection is a prompt problem. The model chooses among the tools you've declared using their name and description. Names carry semantic load — `send_email` reads as "yeah do it," `draft_email_for_user_confirmation` reads as "draft, don't send." A model reading a mixed list will lean on the verbs.

Names to prefer:
- `search_*`, `list_*`, `read_*`, `get_*` — read-only, safe to call speculatively.
- `preview_*`, `dry_run_*`, `simulate_*` — read-only, with the semantics of showing what an action would do.
- `draft_*`, `propose_*` — produces a candidate that a user or another tool will confirm.

Names to avoid unless the tool really is that dangerous:
- `execute_*`, `run_*`, `send_*`, `create_*`, `delete_*`, `charge_*` — reserve for tools where the model doing it is the intent.

The pairing matters more than any individual choice. If you have a `search_orders` and a `refund_order`, the model naturally slots into a search-then-refund plan. If your read tool is called `get_order_by_id` and your write tool is `process_order`, the model may treat `process_order` as another read.

## Tool scope: fewer, larger tools vs many small ones

There is a persistent question in agent design: should you expose one `search_calendars(query, filters)` tool, or a fleet of `list_calendars`, `list_events`, `find_event_by_title`, `find_events_by_participant`, etc.? Both shapes ship in production; the trade-off is real.

**One big tool** wins on:
- Prompt token budget — the model sees one schema, not twelve.
- Discoverability — the model doesn't have to know the right verb; the arguments describe the intent.
- Adding a filter is a schema change, not a new tool declaration.

**Many small tools** wins on:
- The name teaches the semantics. `list_events_for_participant` reads like a natural plan step; `search_calendars(filters={"participant": "..."})` requires knowing the filter key.
- Argument validation is tighter. Each tool has exactly the fields it needs; you don't need conditional required-if-X rules across a big union.
- Errors are localised. A hallucinated filter key hits one tool's validator; a hallucinated filter key on a mega-tool may quietly pass the schema and fail deep in the handler.

The pragmatic answer in most product agents: **one search tool per resource type, plus dedicated write tools with narrow, explicit shapes.** So one `search_calendar_events(query, filters)` (many things you can search for, one search shape), and separate `create_calendar_event`, `update_calendar_event`, `cancel_calendar_event` with distinct schemas. Read-side breadth is cheap; write-side breadth compounds risk.

## Versioning and evolution

Tools change. A new required field, a renamed argument, a stricter enum — each is a breaking change to a caller (the model) that you can't run a "please update your integration" campaign for.

Two disciplines keep this sustainable.

**1. Additive changes are default.** A new optional argument is safe. A new enum value on a `Literal` field is safe. Marking a previously-optional field required is not safe — old context in a running conversation may not include it. Renaming a field is not safe. When you have to make a breaking change, publish the new tool under a new name (`issue_refund_v2`), keep the old one for a deprecation window, and log per-tool usage so you can measure when it's safe to remove.

**2. Every tool declaration is versioned as an artefact.** The version string is not exposed to the model — the model doesn't need it — but it's logged with every call. When the eval loop (mod-106) shows a regression on tool-use tasks, you can trivially correlate it with a schema bump the same way mod-102 correlates prompt version bumps.

## Test the schema, not the handler

Standard advice for services is to test the handler. For a tool exposed to a model, that's necessary but not sufficient. The schema itself is a prompt; test it against the model.

Two test types:

- **Schema selection tests.** Given a set of tools and a user query, does the model pick the intended tool? Fixtures live next to the tool declarations. A regression here is often a description-string problem, not a handler problem.
- **Argument-fidelity tests.** Given a user query and the chosen tool, are the extracted arguments correct? Fixtures pair the query with an expected argument dict. A regression here is often a schema-hint problem — the `description` on a field is doing more work than you thought.

Both live in the same eval harness as the mod-102 output-schema tests. The distinction is which layer's failure is being tested. If the handler is fine but the model can't hit the schema, no amount of handler robustness helps.

## Anti-patterns

Six failures at this layer, all seen in real product incidents.

1. **Free-text arguments where the domain is finite.** `status: str` instead of `status: Literal["open", "closed", "pending"]`. The model invents plausible-looking values (`"resolved"`, `"in_progress"`) and downstream code silently treats them as unknown.
2. **`additionalProperties: true` by default.** Whatever the model invents in extra keys is accepted, ignored by the handler, and echoed in logs. Sometimes those extras are prompt-injected instructions the model repeated from user input.
3. **Optional idempotency.** "We'll add idempotency later." The retry loop lands before "later" and the incident is on the runbook by Friday.
4. **Auth checked in the tool description, not in code.** "Only call this tool when the user has the `refund_admin` role" in the description. The model reads it, complies most of the time, and doesn't the once that matters.
5. **The mega-write-tool.** A single `manage_users(action, ...)` with a dispatch on `action`. Schema validation is impossibly conditional, error messages are useless, and each new action is a new attack surface on the same door.
6. **Names that hide side effects.** `finalize_document` deletes the drafts. `sync_customer` sends a marketing email. If the name doesn't tell the model what it's about to do, the model won't ask the user.

## Summary

- The tool schema is a public API against a non-deterministic caller. Design for the least-trusted client on the internet.
- Every tool declares: what it's for, valid arguments (strict JSON Schema), what it returns, and its side-effect profile. The name and description do a large fraction of the tool-selection work.
- Generate the schema from a typed source (Pydantic, Zod). Use provider strict / grammar-constrained decoding where available. Always wrap the handler in a validator that returns typed errors, not exceptions.
- Side-effecting tools require idempotency keys as schema fields. High-blast-radius tools require dry-run, undo, or user confirmation. A tool the agent calls unaided is a tool with one of those three properties.
- Prefer breadth on read-side tools (one search tool per resource) and specificity on write-side tools (dedicated create/update/delete). Names disclose side effects; verbs matter.
- Test the schema as a prompt (selection tests, argument-fidelity tests), not just the handler. Version tool declarations; additive changes only, or a new tool name under a deprecation window.
- Six anti-patterns to memorise: free-text-instead-of-enum, `additionalProperties: true`, optional idempotency, auth-in-description, the mega-write-tool, and names that hide side effects.

[Chapter 02](02-tool-call-error-taxonomy.md) turns the "validator returns a typed error" line above into the full taxonomy of tool-call failures — model-side, backend-transient, backend-permanent — and the retry / repair / user-facing surface for each.
