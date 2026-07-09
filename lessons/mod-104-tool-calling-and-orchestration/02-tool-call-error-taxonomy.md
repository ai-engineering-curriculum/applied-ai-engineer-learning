# The Tool-Call Error Taxonomy

## Why one retry strategy doesn't fit

At the model surface, tool-call failures all look the same shape: a `tool_use` block goes out, a `tool_result` block comes back with something that isn't a clean success. The reasons those failures happened, however, come from very different places — and they demand very different responses. A blanket retry policy is the fastest way to turn a transient hiccup into a bill for redundant frontier-model tokens, and the fastest way to turn a hallucinated argument into a hallucinated argument that this time also succeeded because you kept trying.

Mod-102 chapter 04 built the classification-first mindset for structured-output failures. This chapter carries that mindset over to tool calls. Three sources of failure, four retry postures, and one non-negotiable rule about how the failure surfaces to the user.

The three sources:

- **Model-side** — the model produced arguments that are wrong: they don't match the schema, they violate business rules, they name entities that don't exist. The tool never ran, or ran and rejected the call.
- **Backend-side transient** — the tool tried to do its job and the underlying service failed in a way that is likely to recover on retry: a network timeout, a 502 from an upstream, a rate limit with `Retry-After`.
- **Backend-side permanent** — the tool tried and the underlying service refused in a way retrying won't fix: the resource doesn't exist, the caller isn't allowed, the operation is illegal in the current state.

Everything else (a bug in the handler, an infrastructure outage, a corrupted upstream contract) collapses back into one of these three under stress. Get these three right and the fourth-quadrant "unknown" failures shrink to a manageable rate the eval loop (mod-106) can pick up.

## Model-side: hallucinated arguments and phantom entities

### The failure shapes

There are five recurring shapes to model-side failures, in decreasing order of how often you'll see them.

1. **Schema violation.** Argument doesn't match the JSON Schema — wrong type, missing required field, enum value the model made up.
2. **Reference miss.** Argument names an ID that doesn't exist: `order_id="ord_a1b2c3d4"` for an order that isn't in the tenant. Sometimes the model synthesises a plausible-looking ID from the tokenization of a similar one it saw earlier in the transcript.
3. **Business-rule violation.** Arguments parse and reference real entities but combine illegally: refund larger than the order total, ship-to address in a country the SKU is restricted from.
4. **Wrong tool for the job.** The model called `create_calendar_event` when the user asked to *reschedule* one — the arguments look fine in isolation but the semantics are wrong.
5. **Argument-shape drift over a long conversation.** After 20 turns of tool use, the model starts filling in fields with values it inferred from earlier tool results rather than from the current turn's context. Common with implicit filters ("participant" repeats from a previous query).

Categories 1 and 2 you can catch structurally. 3 and 4 need business logic or an eval. 5 is a symptom of prompt or context-management drift, not really a per-call bug — file it against the transcript, not the tool.

### Repair, once, with a sanitised message

For 1 and 2, a bounded repair loop is warranted, in the same shape as mod-102 chapter 04. On a schema violation, return a `tool_result` marked as an error with a description of the constraint, not the offending value:

```python
# tool_result content — what the model sees.
{
  "is_error": true,
  "content": (
    "issue_refund: invalid arguments. "
    "amount_cents must be a positive integer, at most 50000. "
    "reason_code must be one of: customer_request, duplicate_charge, "
    "quality_issue, shipping_issue, policy_exception."
  )
}
```

For a reference miss, hand back the same shape with the constraint expressed as "must be an existing order_id for this tenant." Do **not** hand back a list of "nearby" order_ids the model can guess against — that turns a miss into a pattern-match at your customer's expense, and it's a common data leak.

Two rules, hard-earned:

- **Cap at one repair attempt per tool call.** More than one and you are debugging your prompt, not helping the user. Two attempts is the top of the range for a batch pipeline; user-facing surfaces use one and fall back.
- **Repair messages describe constraints, not values.** The `describe_errors` utility from chapter 01 is the source of these strings. Any user-controlled input echoed back into the model is a prompt-injection channel — the mod-102 chapter 04 argument applies here word-for-word.

If the repair also fails, drop to fallback. See "The user-facing surface" below.

### Category 3 and 4: business-rule and wrong-tool errors

For 3, the tool ran, the backend rejected the call, and you got a structured error (see the permanent-failure shape below). Feed it back to the model verbatim (subject to the sanitisation rules) and let it decide whether to try a different tool, adjust arguments, or surface a "can't do that" message to the user.

For 4, this is the hardest to detect at call time. Sometimes the tool succeeds and the user gets an outcome they didn't ask for. Sometimes a business-rule check catches it. The right defence is at the eval layer (mod-106) — the "did the model use the right tool for the intent" question is a first-class metric, not a runtime one.

### Category 5: context and prompt drift

Log the transcript, tag the surface, and file a prompt-registry issue (mod-102 chapter 05). This is not a retry problem. A retry with the same context will produce the same drifted arguments.

## Backend-side transient failures

### What counts as transient

The set is smaller than teams think. Transient failures are ones that *retrying the same call will plausibly succeed*:

- Network timeout, connection reset, DNS blip.
- HTTP 502 / 503 / 504 from an upstream.
- HTTP 429 with a `Retry-After` header.
- Downstream deadlock on a database row, IF the underlying transaction was rolled back (i.e., the call was safe to reissue).
- Provider-side "please retry in a moment" errors from LLM providers or external APIs.

Not transient, even when the error code suggests otherwise:

- HTTP 500 from an upstream that mutated state before failing. Without an idempotency guarantee this is not safe to retry, no matter what the RFC says the code means.
- HTTP 400 / 404 / 409 — these are the permanent failure surface.
- Timeouts on a **read** tool are transient; timeouts on a **write** tool are transient *only if* the tool has an idempotency key (chapter 01). Without one, treat as ambiguous — see "Ambiguous outcomes" below.

### The retry policy

Standard shape:

- Exponential backoff with jitter. Cap at 3–5 attempts for a background pipeline, 2 for a user-facing surface. Each attempt eats latency budget from the request-level budget (chapter 05).
- Honour `Retry-After`. If the header says 30 seconds and you have 2 seconds of budget, do not retry — fall back.
- Circuit-break at the surface layer. If the backend has been failing for the last N seconds, stop retrying and fall back immediately. Retries during an outage are what turn a partial outage into a full one.
- Retry on the same `idempotency_key`. If you reissue with a fresh key you've re-armed a duplicate-write bug.

The retries are **invisible to the model**. The model called `issue_refund` once; it doesn't need to know the runtime tried the network three times. Only the final outcome is fed back as a `tool_result`.

### Instrumenting for tomorrow's debug

Every transient retry is a log line:

- `tool_name`, `idempotency_key` (or hash), attempt number, upstream error code, latency.
- The final outcome (`ok`, `fallback`, `budget_exhausted`).
- The wall-clock consumed by the retry chain.

The aggregate metric that matters is **retry rate per tool**. A tool that starts retrying 5× more often overnight is either your backend or your provider having a bad time; correlating that spike with a deploy time is the first debug step.

## Backend-side permanent failures

Permanent failures are the calls where the backend refused, and retrying with the same arguments will refuse the same way. The response the tool hands back to the model must be **structured**, **stable**, and **actionable**.

### A structured error shape

```python
class ToolError(TypedDict):
    code: str        # stable machine-readable identifier
    message: str     # short, sanitised, user-safe
    field: str | None      # if the error is scoped to one argument
    retryable: bool  # whether the runtime treats this as retryable — usually False here
    hint: str | None       # optional next-step suggestion for the model
```

Codes are namespaced to the tool: `issue_refund.order_not_found`, `issue_refund.amount_exceeds_available`, `issue_refund.already_refunded`. The model reads them; downstream analytics rely on them; runbooks map them to on-call actions. A tool that returns free-text errors is a tool the eval loop can't measure.

Standard codes to establish across the whole tool fleet:

- `not_found` — the referenced entity does not exist.
- `forbidden` — the caller (or the principal on whose behalf the call runs) is not permitted.
- `conflict` — the entity is in a state that disallows this operation (already refunded, already sent, superseded).
- `unprocessable` — arguments were valid at the schema level but violate a business rule.
- `invalid_arguments` — the model failed schema validation (as in chapter 01).
- `rate_limited` — the backend rate-limited the call.
- `service_unavailable` — a transient category, but reserved as the "we retried and still failed" outcome fed back to the model.

Names to the model, codes for the machine. Both are stable.

### What the model does with a permanent failure

Return the structured error as a `tool_result`. The model reads it and gets to choose:

- Fix the arguments and try again (for `unprocessable`, sometimes).
- Try a different tool (for `not_found`, especially — often there's a search step it skipped).
- Surface the failure to the user with an explanation ("I couldn't refund order X — it was already refunded on July 3rd").

The model should not "hide" permanent failures. Prompt design (mod-102 chapter 01) needs to include an instruction: "If a tool returns a permanent error, tell the user what happened and what you'll do next. Do not silently try again with random arguments."

### Never leak internal details in the error message

The message field is user-safe. That means:

- No stack traces.
- No SQL fragments.
- No IDs from other tenants.
- No API keys or auth token fragments.
- No PII other than what the user themselves provided in the request.

A `500 internal_error` from your backend surfaces to the model (and eventually the user) as `"issue_refund: temporarily unavailable"`, not as `"KeyError: 'stripe_charge_id' in refund_service.py line 214"`. Real error text goes in your logs, keyed by `trace_id`. This is one of the specific ways applied-AI surfaces leak internal architecture; owning it here saves a mod-107 chapter later.

## Ambiguous outcomes: the "did it happen?" case

The uncomfortable middle: the tool call started, the network dropped, and you don't know whether the backend applied the mutation. Without an idempotency key you cannot tell and you cannot retry safely. With one, you can reissue the same call — the backend either does nothing (returning the original outcome) or completes what was interrupted.

This is why chapter 01 makes `idempotency_key` a required schema field on side-effecting tools. It converts the ambiguous case into a clean transient. Without idempotency, the runtime has three choices, all bad:

1. Retry blindly. Risk of double-effect.
2. Fail closed. Correct answer, but "we don't know if we charged you" is a poor UX.
3. Query the backend to see whether the effect landed. Requires a read tool that mirrors every write tool, and races with the write.

For third-party tools (MCP servers, chapter 03) you may not control whether idempotency is supported. Treat those tools as ambiguous and design the surface to tolerate that:

- Model prompt: "For `send_slack_message`, if the tool result is `service_unavailable`, tell the user you may or may not have sent the message and ask before retrying."
- Runtime: log the ambiguity as a first-class event so on-call can reconcile.

## The user-facing surface for each failure

The single most important instruction in this chapter: **the user must see something coherent for every failure class**. Silence, generic "an error occurred," or a spinner that never resolves are not acceptable failure modes for an agentic surface.

Match the failure class to a UX:

| Failure class | User sees |
|---|---|
| Schema violation, repaired successfully | Nothing. The repair is invisible. |
| Schema violation, repair failed | An explanation ("I couldn't determine a valid amount for that refund — could you tell me the amount you'd like?") |
| Transient, retried successfully | Nothing. |
| Transient, retried and failed | "The refund service is having trouble right now. I'll try again in a bit, or you can retry from the order page." Log the trace_id. |
| Permanent, model handled it | The model's natural-language explanation. |
| Permanent, model didn't handle it (rare, prompt bug) | A generic fallback and a log entry. |
| Ambiguous outcome | The most careful message: "I attempted the refund but couldn't confirm it went through. Please check the order before I retry." |
| Budget exhausted (chapter 05) | A graceful degradation message, not a stack trace. |

Two rules to write on the whiteboard:

1. **No spinner without a timeout.** A stuck spinner is worse than a wrong answer.
2. **No error without a trace_id.** The user pastes the ID; the runbook resolves the incident.

## A unified tool-call runtime

Bringing it together into one shape:

```python
async def invoke_tool(call: ToolUseBlock, ctx: RequestContext) -> ToolResultBlock:
    tool = REGISTRY.get(call.name)
    if tool is None:
        return ToolResultBlock.error(
            tool_use_id=call.id,
            code="unknown_tool",
            message=f"Unknown tool: {call.name}",
        )

    # 1) Model-side validation (chapter 01).
    try:
        args = tool.args_model.model_validate(call.input)
    except ValidationError as ve:
        return ToolResultBlock.error(
            tool_use_id=call.id,
            code="invalid_arguments",
            message=describe_errors(ve),
        )

    # 2) Authorisation.
    if not tool.principal_can_call(args, ctx):
        return ToolResultBlock.error(
            tool_use_id=call.id,
            code="forbidden",
            message=f"{tool.name}: not permitted for this principal.",
        )

    # 3) Bounded retry on transient upstream failures.
    for attempt in retry_policy(tool, ctx.remaining_budget()):
        try:
            result = await tool.execute(args, ctx, attempt=attempt)
        except TransientUpstream as e:
            log_transient(tool.name, attempt, e, ctx)
            if attempt.is_last:
                return ToolResultBlock.error(
                    tool_use_id=call.id,
                    code="service_unavailable",
                    message=f"{tool.name}: temporarily unavailable. trace_id={ctx.trace_id}",
                )
            continue
        except PermanentBackend as e:
            log_permanent(tool.name, e, ctx)
            return ToolResultBlock.error(
                tool_use_id=call.id,
                code=e.code,
                message=e.user_safe_message(),
            )
        return ToolResultBlock.ok(tool_use_id=call.id, output=result)
```

Read carefully what this does — and equally, what it doesn't do.

- It does not retry `PermanentBackend`. Ever.
- It runs auth **after** schema validation, **before** execution.
- Every failure has a `code`. Every user-visible message is sanitised.
- The retry policy is fed the remaining budget from the request context (chapter 05). If the budget can't afford another attempt, the loop exits early with `service_unavailable`.
- It doesn't handle the *model-side repair* — that lives one layer up, in the agent loop, because the repair is another sampled generation, not another tool invocation.

## Feeding the outcome back into the loop

`ToolResultBlock.error` and `ToolResultBlock.ok` produce provider-shaped tool-result content. The exact wire format depends on the provider — Anthropic's Messages API uses `tool_result` content blocks with `is_error: true` for errors; OpenAI's function-calling protocol embeds the tool output as a `tool` message; the OpenAI Responses API and Agents SDK add their own conventions. The wrapper layer normalises to a single internal shape and translates at the provider edge.

Whatever the wire format, the agent loop (chapter 04) sees:

- **Ok** → append and let the model continue.
- **Error with a repair-eligible code** (`invalid_arguments`, some `unprocessable`) → optionally trigger the one repair attempt, or forward to the model as-is if repair is disabled.
- **Error otherwise** → forward to the model, let it decide next step; if it doesn't handle it well, chapter 05's fallback path takes over.

## Anti-patterns

Five failures at this layer, each responsible for real incidents.

1. **Blanket retry.** A `try / except: retry` around every tool call. Duplicate effects, doubled bills, silent hallucinations "confirmed" by retry.
2. **Retry-with-fresh-idempotency-key.** The runtime generates a new UUID on retry. The backend cheerfully processes both.
3. **Error strings, not error codes.** The model gets free-text errors. Downstream analytics can't cluster them. Two similar errors count as different problems; two different errors count as one.
4. **Un-sanitised error messages.** Backend stack trace or SQL surfaced to the user (or, worse, into the model context where it becomes an injection channel).
5. **No user-facing outcome for a class.** Ambiguous-outcome path: the tool returns nothing, the model returns nothing, the user sees a stopped spinner. The debugging session that follows is expensive precisely because there's no signal to correlate.

## Summary

- Three failure sources: model-side (hallucinated arguments), backend-side transient, backend-side permanent. Each needs a distinct posture: bounded repair, bounded retry, no retry.
- Model-side repair is one attempt, sanitised, describes constraints not values. More than one attempt is a prompt problem, not a runtime problem.
- Transient retries are invisible to the model. They respect `Retry-After`, honour the request budget (chapter 05), and reuse the same idempotency key. Circuit-break during an outage.
- Permanent failures return a structured `ToolError` with a stable code, a sanitised message, and (where relevant) a hint. The model reads codes and decides next steps; internals never leak.
- Ambiguous outcomes exist because idempotency wasn't wired end-to-end. Fix the schema (chapter 01); until then, prefer failing closed with a careful user-facing message.
- Every failure class has a mapped user-facing outcome. No stuck spinners; no bare "error occurred"; every error carries a trace_id.

[Chapter 03](03-mcp-in-a-product-agent.md) turns to the case where the tools aren't yours — they belong to an MCP server another team or vendor publishes — and shows how to wire those in without giving up the guarantees this chapter and chapter 01 depend on.
