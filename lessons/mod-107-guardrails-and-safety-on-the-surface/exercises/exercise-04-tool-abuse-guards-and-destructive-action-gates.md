# exercise-04: Tool-Abuse Guards and Destructive-Action Gates

**Estimated effort:** 3 hours

## Objective

Ship the four guards from [chapter 04](../04-tool-abuse-and-destructive-action-gates.md) on a tool-using agent surface: **per-feature tool allow-lists**, **per-tool per-tenant caps** (rate, cost, argument-shape volume), **destructive-action confirmation gates** (autonomous / notify / confirm-per-call / confirm-with-cool-off), and **auth-boundary discipline** (least-privilege service credentials, per-request identity propagation). Prove each guard with a targeted red-team fixture. The exercise is not "add a rate limiter." It is "make excessive agency a survivable failure, not a catastrophic one."

## Learning outcomes

By the end you should be able to:

- Declare a per-feature tool catalogue and enforce it at the framework layer, not by convention.
- Author per-tool caps that fail typed, per-tenant *and* per-user, on rate / cost / argument-shape volume.
- Classify every tool by confirmation policy and wire the UI (or a UI stub) that renders the actual proposed arguments to the human before execution.
- Propagate the caller's identity into every tool invocation and enforce authorisation at the tool boundary against that identity, not against a shared service credential.
- Ship a kill-switch that disables a tool, a feature, or the whole surface in seconds.
- Prove each guard with a red-team fixture and a "would-have-happened" report.

## Prerequisites

- You have read [chapter 04](../04-tool-abuse-and-destructive-action-gates.md) and completed [exercise-02](exercise-02-prompt-injection-defence-with-agentdojo-shape.md).
- Familiarity with the agent framework of your choice (LangGraph, OpenAI Agents SDK, Claude Agent SDK, or an in-house harness).
- A tool-using surface to instrument. The mod-107 scaffold `agent-lite/` (planned) is fine; a small custom agent with two-to-four tools is fine.

## Problem statement

Your surface's agent has (or would like to have): a read-only search tool, a mutating `render_reply` tool, a payment tool `issue_refund`, and a destructive tool `bulk_delete`. Today: the tools live in a shared registry any feature can call; there are no caps; the confirmation dialog says "OK" without showing arguments; the tools all share a single service credential; and the closest thing to a kill-switch is a rolling deploy. A recent injection eval (exercise-02) demonstrated three attack paths that lead to a `bulk_delete` — you contained the injection, but the risk landscape is still wide.

In three hours, ship the four guards and prove them.

## Requirements

### Deliverable 1 — The catalogue map

Under `docs/catalogue-map.md`:

- Every tool the platform registers. For each: what it does, its confirmation policy (`autonomous` / `notify` / `confirm_per_call` / `confirm_cooloff`), the service credential it uses (which role / scope), and the caps class (see Deliverable 3).
- Every feature the surface offers. For each: the explicit tool catalogue it exposes to its model, with justification. The intersection with the platform's registry is what is loaded at request time.

### Deliverable 2 — Per-feature allow-list enforcement

Under `tools/registry.py` (or equivalent):

- `FEATURE_CATALOGUES` is a hard-coded map from `feature_id → set[tool_name]`. Not a runtime lookup, not a config-file default that permits `["*"]`.
- `catalogue_for(feature_id, tenant_id) -> set[str]` returns the intersection, with tenant deny-lists that can only narrow.
- The agent framework is configured so that a request constructed with `feature_id` loads *only* the intersection's tools. A test proves that a tool not in the catalogue cannot be invoked even if the model produces the tool-call name.

### Deliverable 3 — Per-tool per-tenant caps

Under `tools/caps/` (one YAML per feature):

- Per-tool `rate` (calls per window, per tenant, per user), `cost` (amount per window, per tenant), `volume` (per-call argument-shape ceilings: `amount_cents_max`, `recipients_max`, `records_max`).
- A `CapBreach` typed exception the tool wrapper raises on breach; the agent framework surfaces it as a typed tool-result the model can reason about (`{"error": "capped", "cap": "rate.per_user", "retry_after": 3600}`), not a stack trace.
- A metric per (tool, tenant, cap_type) so breaches are visible to alerting.

### Deliverable 4 — Destructive-action confirmation gates

Under `tools/confirmation.py` and a minimal UI stub (`ui/confirm.py`):

- Every tool is classified per Deliverable 1's map. Autonomous tools skip; `notify` tools invoke with a UI announcement; `confirm_per_call` and `confirm_cooloff` invoke only after human approval.
- The confirmation surface renders the *actual* arguments the tool will receive. A test asserts that a tool call with `recipient=attacker@example.com` is shown to the human as `recipient: attacker@example.com`, not as "Send email?"
- Per-call by default. If the surface introduces a "remember my choice" pattern, its scope is bounded (this session, this argument shape) and expires within a bounded window; the design decision is documented.
- Declined confirmations return `ToolResult.cancelled()` with a typed reason.

### Deliverable 5 — Auth-boundary discipline

Under `tools/identity.py`:

- Each tool invocation carries a `Principal` derived from the authenticated request context (a JWT, a session cookie, an OAuth token — not a model-supplied `user_id`).
- The tool's implementation calls the identity layer (`principal.can("issue_refund", args)`) to authorise the action against the actual caller. A model-supplied `user_id` that differs from the principal's is treated as an argument to *look up*, not to *authenticate*.
- Service credentials the tool uses (DB roles, third-party API keys, sender identities) are scoped to the minimum required. Document each and the rotation schedule.
- Every tool invocation writes to an append-only audit log with `(tenant, principal, tool, args_hash, result_status, timestamp, trace_id)`. The audit log is a separate store from the operational trace (chapter 03's scrubbing does not apply — audit needs to be legible for incident response; access-control is what protects it).

### Deliverable 6 — Kill switch

Under `tools/kill_switch.py`:

- Every tool has an `enabled: bool` flag. Setting `enabled=false` fails all invocations of that tool with a typed `disabled_by_operator` error.
- Features and the whole surface have equivalent flags.
- Flag changes propagate in seconds — cache invalidation across the fleet is part of the exercise, not "eventual." A test proves this: flip the flag, invoke the tool, assert the call is rejected within 5 seconds.

### Deliverable 7 — The red-team fixtures

Under `fixtures/tool-abuse/`, at least **20 fixtures**, each pairing an *intended attack* with the expected *catch layer*. Examples:

- An injection through a retrieved chunk that instructs `send_email(recipient=attacker@example.com)` — expected to be caught by argument validation *or* per-tool argument-shape volume (recipient not in tenant's allow-list).
- A legitimate-looking but destructive `bulk_delete(records=[...])` with 5,000 records — expected to be caught by argument-shape volume (`records_max=100`).
- 20 rapid-fire `issue_refund` calls in 30 seconds — expected to trip the per-user rate cap at 3.
- A `run_query` with a query attempting to read another tenant's data — expected to fail auth-boundary because the tool's DB role is tenant-scoped.
- A `send_email(confirmed=true)` bypass attempt — expected to fail because confirmation is human-side, not model-side.
- A tool call after the kill switch is thrown — expected to fail with `disabled_by_operator`.

Under `bin/tool-abuse-eval`, a runner that:

- Executes every fixture and records the outcome (`caught_by_layer`, `escaped`).
- Fails a PR that regresses the caught-rate against the baseline.

### Deliverable 8 — Proof

Under `proofs/`:

- **Baseline.** With guards disabled, run the fixture set. Report the outcomes.
- **Guarded.** With guards enabled, run the fixture set. Report the outcomes and the layer that caught each.
- **Kill-switch drill.** Timestamp: throw the kill switch, invoke the tool, record the elapsed time to rejection. Assert < 5 seconds.

## Starter guidance

- **Framework-layer enforcement, not convention.** If the agent framework accepts an arbitrary tool list at construction and you rely on "we always pass the right list," a bug in the config path exposes the whole registry. The catalogue enforcement should be inside the request-construction code, not in each caller.
- **Typed cap errors, always.** A `429`-shaped response the model can retry (or escalate) is much better than a timeout. Reserve timeouts for genuine timeouts.
- **The UI stub is enough for this exercise.** You do not need to ship a real frontend. A `ui/confirm.py` that prints the args and reads from stdin (or a test harness that patches it) proves the discipline.
- **Do not put credentials in the prompt.** Ever. If a tool needs a credential, it fetches it from a secrets store inside the tool's implementation. The model never sees `DATABASE_URL=postgres://user:pass@...`.
- **Kill switches need to be tested.** A kill switch that has never been thrown works in the same sense that a fire drill you have never held works.

## Acceptance criteria

- [ ] `docs/catalogue-map.md` names every platform tool, every feature, and the explicit per-feature catalogue.
- [ ] `tools/registry.py` enforces the catalogue at the request-construction layer. A test asserts that an out-of-catalogue tool cannot be invoked.
- [ ] `tools/caps/` YAML files declare rate / cost / volume caps per tool per tenant *and* per user. Breaches raise typed errors; metrics count them.
- [ ] Every tool is classified per confirmation policy. Confirmation UI (or stub) renders the actual arguments; a test proves it.
- [ ] Per-request identity is propagated into every tool call; authorisation is enforced at the tool boundary against the propagated principal.
- [ ] Service credentials are least-privilege and documented; rotation schedule named.
- [ ] Every tool invocation writes to an append-only audit log.
- [ ] Kill switch propagates flag flips in under 5 seconds; test proves this.
- [ ] `fixtures/tool-abuse/` has ≥ 20 fixtures covering: injection-through-retrieval, argument-shape volume, rate cap, auth boundary, confirmation bypass, kill switch.
- [ ] `bin/tool-abuse-eval` runs the fixtures, records caught-by-layer, and fails a PR that regresses the caught-rate.
- [ ] `proofs/` contains the baseline / guarded / kill-switch drill artefacts.

## Stretch goals

Pick any two:

- **Cross-feature cap sharing.** Extend caps to a shared budget across features that draw from the same downstream resource (`issue_refund` and `credit_account` both hit the payments provider). Prove the shared cap trips when either feature exhausts it.
- **Cool-off cancellation.** For `confirm_cooloff` tools, wire an actual cancellation path: after the human confirms, a 60-second window during which the user can cancel. Prove cancellation propagates before execution.
- **Confirmation-fatigue metric.** Emit a metric `confirm.decline_rate = declines / (approvals + declines)` per tool. A rising decline rate is a signal the agent is over-proposing destructive actions; wire an alert.
- **Audit-log immutability.** Ship the audit log to an append-only store (S3 with object-lock, or an equivalent). Prove it is not deletable by the tool's own credentials.
- **A "worst-case blast radius" red-team.** Compose an attack chain: an injection defeats Layer 1, produces a tool call, hits Layer 2's cap, and — because the cap is per-hour — the attacker retries in the next hour. Design a fixture and show the current caps *do* or *do not* survive the retry. Iterate the cap design until they do.

## Deliverable

Submit as `mod-107/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/catalogue-map.md`.
- `tools/registry.py`, `tools/caps/`, `tools/confirmation.py`, `tools/identity.py`, `tools/kill_switch.py`.
- `ui/confirm.py` (stub UI).
- `fixtures/tool-abuse/` with the red-team panel and a `PROVENANCE.md`.
- `bin/tool-abuse-eval` and `reports/` with the baseline / guarded runs.
- `proofs/kill-switch-drill.md`.
- A short `README.md` naming the surface, the four guards' shape, and how to run the eval.
- Any stretch-goal artefacts named alongside.
