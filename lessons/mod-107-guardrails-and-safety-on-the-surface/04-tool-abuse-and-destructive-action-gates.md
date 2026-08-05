# Tool Abuse Guards and Destructive-Action Gates

## Excessive agency is the incident category

Chapters 01 and 02 built defences at the *content* boundary — what the model reads and what it writes. This chapter builds the defence at the *action* boundary — what the model *does*. It maps directly to OWASP's `LLM06: Excessive Agency` in the current Top 10 for LLM Applications, and to the entirety of OWASP's Agentic AI Top 10, which is a 2025-vintage companion taxonomy specifically about the failure modes agents introduce that pure chat surfaces do not have.<!-- needs-research: pin the current OWASP Top 10 for LLM Applications version (2025 revision was announced) and the current state of the OWASP Agentic AI Top 10 — the project name and taxonomy have been evolving. -->

The incident shape is consistent enough to name in one sentence: *an agent with more tools, more permissions, or more autonomy than the task requires took an action the operator did not intend*. Sometimes the trigger is an injection (chapter 02). Sometimes it is a bug in the agent's own reasoning. Sometimes it is a mis-scoped OAuth grant. The defence is the same either way: the tool catalogue, the tool's arguments, the tool's rate, and the tool's confirmation gate are all bounded to what the surface actually needs.

Four guards, all owned by the applied engineer:

1. **Per-feature tool allow-lists.** The catalogue exposed to the model is the intersection of the platform's tools and *this feature's* need.
2. **Per-tool, per-tenant caps.** Cost, rate, and volume bounds that fail fast when they trip, not a soft warning after the invoice arrives.
3. **Destructive-action confirmation gates.** Any tool whose effect is irreversible or high-cost pauses for user (or human-operator) confirmation before executing.
4. **Auth boundary discipline.** The tool's own auth is not the model's auth. The credentials the tool acts under are scoped, rotated, and audited independently of the model's scratch memory.

None of these are new patterns — they are the applied engineer's minimum viable version of the security perimeter the `ai-security-engineer` peer role builds in depth.

## Guard 1 — Per-feature tool allow-lists

The single largest determinant of an agent's blast radius is the tool catalogue it can see. A model with `search_docs` and `render_reply` has one blast radius. A model with `execute_shell`, `send_email`, and `update_customer_record` has a very different one. Injection defences (chapter 02) matter more or less depending on which catalogue is loaded.

The discipline: **every feature declares its tool catalogue explicitly**, and the runtime enforces it. Not "the agent framework has these tools registered globally"; not "the tool exists in the shared MCP server so any feature can call it." The intersection is enforced at request-construction time.

A concrete shape:

```python
# tools/registry.py — sketch.
FEATURE_CATALOGUES = {
    "support-agent-v3": {
        "search_orders",
        "search_kb",
        "issue_refund",
        "render_reply",
    },
    "code-review-bot-v1": {
        "search_pr_diff",
        "run_linters",
        "post_review_comment",
    },
    # NB: `execute_shell` appears in no feature's catalogue.
}

def catalogue_for(feature_id: str, tenant_id: str) -> set[str]:
    base = FEATURE_CATALOGUES[feature_id]
    tenant_denies = TENANT_DENYLIST.get(tenant_id, set())
    return base - tenant_denies
```

Three properties. First, the catalogue is a **hard-coded feature declaration**, not a runtime lookup — a rogue config change should not silently widen it. Second, the tenant layer can *narrow* the catalogue (a customer that has not agreed to the refund policy does not see `issue_refund`) but cannot widen it. Third, adding a new tool to a catalogue is a code change that goes through code review, and the review includes an update to the injection eval fixtures (chapter 02).

The runtime constructs the model request with *only* the catalogued tools. The MCP server or agent framework must respect this — if the framework happens to expose tools by default (some do), the framework configuration is part of the enforcement, not a bypass.

**A common failure to avoid.** A catalogue named "shared_utils" grows to hold every utility any feature ever needed. Two years later every feature includes `shared_utils`. The allow-list has become a global list with a different name. Flat catalogues per feature, explicit imports, no wildcards.

## Guard 2 — Per-tool, per-tenant caps

Even a well-scoped catalogue does not stop an agent that runs a legitimate tool a million times in an hour. Three caps, all necessary, all per-tool-per-tenant:

- **Rate limit.** N calls per (tenant, tool) per window. A single agent invocation that would call `send_email` 500 times inside a loop trips the limit at 10; the remaining 490 fail with a typed error the agent can surface. Rate limits protect the downstream integration (`send_email` provider quotas) *and* provide the smoking gun for an incident review.
- **Cost cap.** Per-tenant, per-tool, per-window spend ceiling. A `run_query` tool that hits an expensive analytics warehouse gets a dollar ceiling; an agent that decides to page through 100k rows hits the ceiling long before the bill does. mod-104 chapter 05 develops the per-request budget shape; this chapter's contribution is the per-tenant aggregation.
- **Volume cap.** Per-argument-shape volume limits. `issue_refund` capped at $500/day per tenant; `bulk_delete_records` capped at 100 records per call; `send_email` capped at N recipients per call. Argument-shape caps prevent one legitimate call from being the incident.

The three caps compose. A single agent turn can be capped by rate (10 per minute), by cost (\$5 per hour), and by argument volume (100 records per call). The tightest binding cap is the one that fires; each cap logs which binding it was and why.

A concrete shape:

```yaml
# tools/caps/support-agent-v3.yaml
issue_refund:
  rate:
    per_tenant: {calls: 20, window: 1h}
    per_user:   {calls: 3,  window: 1h}
  cost:
    per_tenant: {amount_cents: 50000, window: 24h}
  volume:
    per_call: {amount_cents_max: 100000}
send_email:
  rate:
    per_tenant: {calls: 500, window: 1h}
    per_user:   {calls: 10,  window: 1h}
  volume:
    per_call: {recipients_max: 5, attachments_max: 0}
```

Two properties. Each cap has a *unit* (`calls`, `amount_cents`, `recipients`) and a *window* (`1h`, `24h`, `per_call`), so the semantics are legible. Each cap has a per-tenant *and* a per-user version — a single hostile user should not be able to burn the whole tenant's budget on their own.

On breach: **fail typed**. The tool returns a `capped` error the model can reason about ("I've hit the daily refund cap for this tenant; escalating to human review"), not a silent time-out or a stack trace. The event fires an alert; the incident review names which cap tripped and whether the cap was too tight (raise it with a review) or too loose (it should have tripped sooner).

## Guard 3 — Destructive-action confirmation gates

Some tools are irreversible or high-cost enough that no cap justifies letting the model call them without a human in the loop. mod-105 chapter 03 developed the human-in-the-loop UX pattern in detail; this chapter names the *policy* that decides which tool calls interrupt.

The classification is per-tool:

- **Autonomous.** Read-only tools with negligible cost. Search a knowledge base, look up a customer's public profile, render a reply *draft*. Called without interruption.
- **Notify.** Actions that mutate state but are trivially reversible and low-cost. Post a comment on a PR, save a draft, tag an item. Called without a pre-confirmation but visibly announced in the UI so the user sees them happen and can undo.
- **Confirm-per-call.** Actions that are hard to reverse, spend money, contact third parties, or touch regulated data. Refund a payment, send a customer-facing email, delete records, book a flight, execute code in a shared environment. The agent proposes; the human confirms.
- **Confirm-with-cool-off.** The most consequential class. Confirmation now, plus a delay (60 seconds, a minute, an hour) before execution during which the user can cancel. For truly destructive actions — bulk deletes, cross-tenant operations, one-way migrations — the cool-off is what catches the "wait, no" reaction that a naked confirm dialog does not.

Two properties keep confirmation gates honest:

- **Confirmation is per-call by default.** A `remember_my_choice: yes` checkbox that spans a session, a tenant, or forever is a well-documented incident pattern. Any expansion of scope is an explicit policy decision with a review, and expires within a bounded window.
- **The confirmation surface renders the *actual* arguments.** "Send this email? Recipient: `attacker@example.com`. Body: 'Please transfer…'" — the user sees what the tool will do, not a generic "Send email?" prompt. Injection defence depends on the human being able to read what they are approving.

A worked shape:

```python
# tools/confirmation.py — sketch.
class ConfirmationPolicy(str, Enum):
    AUTONOMOUS       = "autonomous"
    NOTIFY           = "notify"
    CONFIRM_PER_CALL = "confirm_per_call"
    CONFIRM_COOLOFF  = "confirm_cooloff"

TOOL_POLICIES = {
    "search_orders":   ConfirmationPolicy.AUTONOMOUS,
    "render_reply":    ConfirmationPolicy.NOTIFY,
    "issue_refund":    ConfirmationPolicy.CONFIRM_PER_CALL,
    "bulk_delete":     ConfirmationPolicy.CONFIRM_COOLOFF,
}

async def dispatch(tool_call: ToolCall, ctx: RequestCtx):
    policy = TOOL_POLICIES[tool_call.name]
    if policy is ConfirmationPolicy.AUTONOMOUS:
        return await tool_call.invoke()
    if policy is ConfirmationPolicy.NOTIFY:
        ctx.ui.announce(tool_call)
        return await tool_call.invoke()
    if policy is ConfirmationPolicy.CONFIRM_PER_CALL:
        approval = await ctx.ui.confirm(tool_call, render=tool_call.rendered_args)
        return await tool_call.invoke() if approval else ToolResult.cancelled()
    if policy is ConfirmationPolicy.CONFIRM_COOLOFF:
        approval = await ctx.ui.confirm_with_cooloff(tool_call, cooloff_seconds=60)
        return await tool_call.invoke() if approval else ToolResult.cancelled()
```

Three properties. Policies are named per-tool, not encoded inside the tool's implementation — a policy update is a data change, not a code change. The UI is a first-class dependency (the classification is meaningless without a surface that can actually pause and ask). The cancelled path returns a typed `ToolResult.cancelled()` that the agent can reason about ("the user cancelled the refund; ask why or offer alternatives") rather than a silent skip.

## Guard 4 — Auth boundary: the tool's credentials are not the model's

The failure mode this guard defends against: an injection convinces the agent to call `run_query` with a query the model constructs. The tool executes the query with a database credential that has broader scope than the current user's session. The result is data the user was not supposed to see, or a write the user was not supposed to make.

Four disciplines:

- **Least-privilege service credentials.** Each tool's own credentials (its database role, its API key on the third-party service, its OAuth token) are scoped to the minimum operations the tool needs — SELECT on specific tables, not on `*`; `send_email` on a per-tenant sender domain, not global; `write_file` under a per-tenant directory, not the filesystem root.
- **Per-request identity propagation.** The current user's identity flows into every tool call as a first-class argument, and the tool enforces authorisation against that identity — not against the model's session, not against the process's own credentials. A tool that fetches "user's orders" takes `user_id` from an authenticated context, not from a model-supplied argument that an injection could rewrite.
- **Rotate and revoke.** Tool credentials rotate on a schedule (weeks, not years) and revoke on incident. A leaked tool credential should be dead within an hour of detection, not the next scheduled rotation.
- **Audit every action.** Tool invocations write to an append-only audit log with (tenant, user, tool, arguments, result-status, timestamp, trace-id). The log is separate from the operational trace so a compromise of the trace store does not obscure the audit trail.

The applied engineer does not build the identity substrate — that is `platform-engineer` and `security-engineer` territory. The applied engineer *depends on it correctly*: the surface's tools call the identity layer, not their own convenience credentials.

## Composing the guards: the request lifecycle

A tool-using request that survives all four guards looks like this:

1. **Feature dispatch.** Request arrives with `feature_id` and `tenant_id`. The catalogue is loaded (Guard 1).
2. **Model call.** The prompt is constructed with the catalogue's tools. Chapter 03's scrubbing has already fired on inputs and prompt.
3. **Tool proposal.** Model proposes `tool_call`. The runtime checks:
   - Is the tool in the current catalogue? (Guard 1)
   - Do the arguments validate? (mod-104 chapter 01)
   - Are the caps still under budget for this (tenant, user, tool)? (Guard 2)
4. **Confirmation gate, if applicable.** (Guard 3.) On decline, the agent gets a typed cancellation; on confirm, execution proceeds.
5. **Execution.** The tool invokes with its own scoped credentials and the propagated user identity. (Guard 4.)
6. **Audit + trace.** The invocation writes to the audit log; the operational trace records the scrubbed version.
7. **Result to the model.** Chapter 02's tool-result quarantine wraps the output; chapter 01's moderation fires on the response.

Every step has a fail-closed default. A missing catalogue entry blocks; a cap breach blocks; a declined confirmation blocks; a scoped-credential failure blocks. The composition is what makes an injection or a bug survivable rather than catastrophic.

## The kill switch

The last-resort control: a **kill switch** the on-call operator can throw to disable an entire tool, a feature, or an entire agent surface within seconds. mod-111 develops the rollout / rollback discipline in general; the tool-abuse-specific instantiation:

- Every tool is registered with an `enabled: bool` flag whose default is `true` and whose false state fails all invocations with a typed `disabled_by_operator` error.
- Flag changes are dashboard-visible, audit-logged, and cross-region propagated in seconds — not "wait for the next deploy."
- The kill switch is documented in the runbook, tested quarterly, and named as *the* first-response action when an incident is suspected. Investigating without the kill switch thrown is often malpractice; kill first, investigate second.

## Anti-patterns

- **The global tool registry.** All tools registered once, filtered nowhere. A new tool ships to every feature by default. Injection eval fixtures never re-run because the catalogue "hasn't changed" — except it has, silently.
- **Caps on the total, not per-tenant.** A daily cost cap of $1,000 across all tenants means one hostile tenant burns the budget and starves every legitimate user.
- **The confirmation gate the model can bypass.** A "confirm" argument the model itself supplies: `issue_refund(order_id=..., confirmed=true)`. The confirmation must come from the *human*, not from the model's tool call.
- **Persistent auto-approve.** "You approved refunds under $50 last week — I'll do this one automatically." Persistence must be bounded (this session, this argument shape) and reviewable in the UI at any moment.
- **Tool credentials in the prompt.** The model sees the database URL because "the tool needs it and it's easier to pass it through." Now the credential is in the prompt cache, the trace, and any future exfiltration channel.
- **Auth-as-argument.** `fetch_user_orders(user_id: str)` takes an arbitrary `user_id`. An injection asks for another user's orders. The tool obliges because it never checked the caller's identity against the argument. Take identity from the authenticated context.
- **The kill switch that requires a deploy.** By the time you notice the incident, the deploy pipeline is 20 minutes; the incident is over — with a worse outcome — before the flag flips. Kill switches must propagate in seconds.
- **Cap breaches that page nobody.** The cap fired, the request failed, no alert. Every cap breach is a signal — either the cap is wrong, or the traffic is anomalous — and belongs in the alert pipeline (mod-110).

## Summary

- Excessive agency is the incident category: an agent with more tools, permissions, or autonomy than the task requires took an action the operator did not intend. OWASP LLM Top 10's `LLM06` and the OWASP Agentic AI Top 10 name it directly.
- Four guards, composed: **per-feature tool allow-lists** (the catalogue is an explicit intersection, not a global list), **per-tool per-tenant caps** (rate, cost, argument-shape volume — all typed, all failing fast), **destructive-action confirmation gates** (autonomous / notify / confirm-per-call / confirm-with-cool-off classification, per-call by default, rendered arguments), and **auth boundary discipline** (least-privilege tool credentials, identity propagated per-request, rotation, audit log).
- The request lifecycle composes the four with fail-closed defaults at each step. An injection or a bug that survives content moderation is bounded by the catalogue, the caps, the confirmation, and the credentials.
- A **kill switch** that propagates in seconds is the last-resort control. Test it. Throw it before investigating.
- Every anti-pattern in this chapter has a real shipped incident behind it; the guards are the discipline that makes those incidents survivable.

[Chapter 05](05-refusal-calibration-and-owasp-mapping.md) closes the module by turning to the meta-discipline that ties every chapter together: writing a **safety policy**, calibrating refusal vs. over-refusal against that policy, and mapping every finding — from moderation FPs to injection eval failures to tool-abuse incidents — to the current OWASP LLM Top 10 and OWASP Agentic AI Top 10 taxonomies, with an owner assigned to each.
