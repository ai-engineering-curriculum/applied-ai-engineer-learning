# exercise-03: Human-in-the-Loop and Kill-Switch UX

**Estimated effort:** 4 hours

## Objective

Build a working steering layer for a product agent — the six affordances (progress, cancel, undo, resume, hand-off, kill-switch) from [chapter 03](../03-human-in-the-loop-and-steering-ux.md) — as a runnable surface. The goal is to prove that the affordances are real *code*, not slide decks: a user really can cancel a running task mid-step, a real undo really reverses a real write, a real kill-switch really stops the surface within seconds. If you cannot demonstrate any of the six with a live keypress, you have not shipped that affordance.

## Learning outcomes

By the end you should be able to:

- Expose a `SteeringHandle` shape (cancel, pause, resume, approve, reject, subscribe) that the UI consumes and the runtime honours.
- Wire cancellation through in-flight LLM calls and tool calls within a bounded time budget, without leaving orphans behind.
- Design the undo path as a compensating-action ledger, not "hope the model doesn't write the wrong thing."
- Build a resumable session that survives a browser close and a server restart.
- Ship a kill-switch with multiple scopes (global, tenant, tool, user), fast propagation, and a user-facing message that reads honestly.
- Write the operator runbook that maps symptoms → switch and the re-enable checklist.

## Prerequisites

- You have read [chapter 03](../03-human-in-the-loop-and-steering-ux.md) and skimmed [chapter 01](../01-agent-shape-for-a-real-product-surface.md) for shape context.
- You have finished [exercise-02](exercise-02-memory-system-with-retention-and-pii-policy.md) or you have equivalent persistent-session infrastructure to reuse.
- You are comfortable with async code in your language of choice (Python `asyncio` + FastAPI, or Node + a WebSocket server), and either a small UI (Streamlit / SvelteKit / plain HTML with SSE) or a CLI-with-live-panel (Textual / Ink) that can render a progress trace and take keypresses.
- You have `pytest` / `Vitest` and can run integration tests that spin up your runtime and drive it programmatically.

## Problem statement

You are the applied-AI engineer on **a marketing-ops agent** for a mid-market SaaS. The surface is embedded in the marketing team's campaign tool. A marketer types "draft a Q3 promo email for the enterprise segment, cross-reference last quarter's conversion data, and schedule the send for next Tuesday morning." The agent may:

- Read the CRM (segment definitions, user counts).
- Read the analytics warehouse (conversion data).
- Draft a promo (structured output — subject, body, CTA).
- **Send the promo to a bounded audience list** (high blast radius — an unsend after send is impossible).
- Schedule the send in the campaign tool for a specific date/time.
- Post a notification to a shared Slack channel that the campaign is scheduled.

Business context you inherit:

- Marketing has already been burned once: an early agent sent a promo to the wrong audience segment and had to public-apology-email 40,000 customers.
- Product wants an "auto-schedule" mode that lets senior marketers skip the confirmation step for low-risk campaigns.
- Legal has drawn a red line: no send-to-audience without explicit user confirmation *of the exact rendered content*.
- Security has drawn a red line: an incident response mechanism that lets on-call disable the surface globally, disable individual tools, or disable a specific tenant, without a deploy.

The MVP scope is the six-affordance surface for the "draft, review, schedule, notify" flow, plus the kill-switch for the on-call.

## Requirements

### Deliverable 1 — The runtime steering signals

Under `runtime/`, the `SteeringHandle` and `KillSwitch` shape from [chapter 03](../03-human-in-the-loop-and-steering-ux.md), implemented over your chosen runtime (LangGraph, OpenAI Agents SDK, Anthropic Claude Agent SDK, or a hand-rolled state machine — pick one and defend the choice in your `README`).

- `SteeringHandle` exposes: `cancel(reason)`, `pause()`, `resume()`, `approve(step_id)`, `reject(step_id, note)`, `status()`, `subscribe(on_event)`.
- Every step of the agent's execution emits a `StepEvent` with `turn_id`, `step_index`, `kind`, human-readable `summary`, `started_at`, `finished_at | null`, and a redacted `detail`.
- **Cancellation propagates within a hard 5-second SLA.** Test it: start a tool call that would take 60 seconds, hit cancel, verify no side effect landed and the runtime settles within 5 seconds. If it doesn't, you have not shipped cancel.
- The runtime pauses at approval steps (send / schedule / notify) and only proceeds when `approve` fires.

### Deliverable 2 — The user-facing surface

Under `surface/`, a minimal UI (or CLI-with-panels) that consumes the runtime signals.

- **Progress panel.** Live, two-tier: summary by default (`"Reading conversion data for the Enterprise segment…"`), expandable to a detail view (redacted tool args + returns). Elapsed time; ETA if computable; a "taking longer than usual" indicator crossing a p90 threshold.
- **Cancel button.** Wired to `SteeringHandle.cancel`. After cancel: a legible aftermath panel — what did happen, what did not, and options to resume, edit, or discard.
- **Undo affordance.** Every visible write in the trace has an undo button wired to a compensating action (draft edits are reversible; scheduled sends have a "cancel scheduled send" button; Slack notifications have a "delete notification" button up to Slack's retraction window). A batch undo reverses a group of writes as a unit.
- **Resume affordance.** The user closes the tab; reopens the campaign tool; sees the in-flight task in a `In progress` list and can re-attach. State is checkpointed to persistent storage (SQLite is fine).
- **Hand-off affordance.** A "hand this to a marketing manager" button that transfers the session (with context: retrieved data, drafts, tool calls, reason for hand-off) to a second user. The receiver sees the full context in the same UI, not a cold start.
- **Confirmation UX for sends.** When the agent proposes `send_promo`, the confirmation surface renders the *exact* email (subject, body, CTA, audience list preview) as the recipient would see it. Approval requires an explicit user gesture; the model cannot synthesise it.

### Deliverable 3 — The kill-switch

Under `kill_switch/`, the operator break-glass mechanism.

- **Scopes.** Global (whole surface), per-tenant, per-tool, per-user.
- **Storage.** Redis, a config service, a feature-flag provider, or a simple in-memory + database-backed toggle store your runtime consults on each request. The switch flip is *not* a deploy.
- **Propagation SLA.** A flipped switch is honoured by the runtime within 10 seconds. Test it: fire a switch in-process, spawn a request, confirm the request is blocked with the correct user-facing message.
- **User-facing message.** When a switch fires, the surface shows a legible message ("This feature is temporarily unavailable while we investigate an issue.") — no stack trace, no silent failure, no hallucinated reply. Templates live in `kill_switch/messages.py`.
- **Auth boundary.** No user-facing surface holds a reference to the switch API. Only an operator CLI / dashboard can flip a switch. Every flip is logged (who / when / why / scope).

### Deliverable 4 — The runbook

`runbook.md`. Written for on-call at 2am. Contents:

- **Symptoms → switch.** A table mapping observable symptoms to the switch that stops the bleed. Include at least: sudden acceptance-rate drop, tool-error-rate spike, prompt-injection report from security, tenant-scoped hallucination cluster.
- **Blast radius per switch.** For each switch, who is affected and who is not.
- **Communication template.** Pre-written user-facing messages for each switch.
- **Re-enable checklist.** What must be true before you flip the switch back. At least: the hotfix / prompt regression / rollback is deployed, the failing eval passes on the current version, the incident review is scheduled.

### Deliverable 5 — Integration tests

Under `tests/`. At minimum:

- **Cancel-during-tool-call.** Start a task; wait for a tool call to begin; cancel; assert the task settles within 5 seconds and no side effect landed (the stubbed backend received no `send_promo` call).
- **Cancel-versus-race.** A test that races cancel against a mid-flight LLM stream. The stream is aborted; no partial output is applied downstream.
- **Undo-of-a-batch.** The agent makes three field edits to a draft; a batch undo reverses all three as a unit.
- **Resume-after-restart.** Serialise the session; kill the process; restart; resume the session; the runtime picks up at the last checkpoint.
- **Hand-off context transfer.** User A hands off to user B; user B sees the same trace, retrieved data, and drafts.
- **Kill-switch-global.** Flip the global switch; assert every subsequent request is blocked with the correct user-facing message; unflip; assert requests flow again.
- **Kill-switch-tool.** Flip the `send_promo` tool switch; assert the runtime blocks that tool but permits others; confirm the user-facing message names the disabled feature honestly.
- **Confirmation-cannot-be-synthesised.** A test that shows the runtime rejects an "approve" event that did not originate from a user gesture (a request without a valid CSRF / gesture token is refused).

## Starter guidance

- **Cancel is the hardest.** Wire cancel first; it will exercise every part of the runtime. If cancel works, resume and undo will fall into place. If cancel doesn't, nothing does.
- **Stub the backend, but don't stub the runtime.** The `send_promo` tool is a stub that records "would have sent." The runtime around it is real — the cancel test proves the tool was *not* called, not that the stub returned early.
- **Do not build a novel UI framework.** Use whatever's in your muscle memory — Streamlit, Gradio, a minimal FastAPI + SSE endpoint with a plain HTML client, or a Textual TUI. The UI is a means to demonstrate the affordances, not the point.
- **Confirmation UX means rendering the artefact.** A confirmation modal that reads "The agent wants to send an email. Confirm?" is not confirmation; it's rubber-stamping. Show the email as the recipient will see it.
- **Feature-flag the auto-schedule.** Product asked for it. Do not ship it in this MVP. Add a `TODO` in the runbook naming the criteria (eval-scored risk score under threshold, user-role check) that would gate it later.
- **Kill-switch is not "delete the deployment."** If your first instinct is "just kill the pod," you're solving a different problem. The switch flips a config, not the platform.
- **Log every switch flip.** In a real incident, "when did we disable this and why" is what saves the post-mortem.

## Acceptance criteria

- [ ] `SteeringHandle` exposes cancel, pause, resume, approve, reject, status, and subscribe; all wired to a real runtime.
- [ ] The user sees a two-tier progress panel with summary + expandable detail, elapsed time, and a slow-step indicator.
- [ ] Cancel propagates into in-flight LLM and tool calls within 5 seconds. A stubbed tool call in flight is aborted; no side effect lands.
- [ ] Every visible write in the trace has an undo button wired to a compensating action, or is explicitly gated behind confirmation for tools without an undo.
- [ ] A batch of edits has a batch undo.
- [ ] Sessions are checkpointed to durable storage; a browser close + reopen (or a process restart) re-attaches the user to the in-flight task.
- [ ] Hand-off transfers the full context (trace, retrieved data, drafts) to the receiver.
- [ ] `send_promo` cannot be executed without an explicit user gesture that produces a valid `user_confirmed` sentinel; the confirmation UI renders the exact rendered email.
- [ ] The kill-switch has global, per-tenant, per-tool, and per-user scopes; a flip is honoured by the runtime within 10 seconds; a legible user-facing message is shown; every flip is logged with who / when / why / scope.
- [ ] The runbook is written for on-call, not for engineers reading design docs. Symptoms → switch is a table; the re-enable checklist is a checklist.
- [ ] Integration tests cover cancel-during-tool-call, cancel-versus-race, batch undo, resume-after-restart, hand-off context transfer, and each kill-switch scope.

## Stretch goals

Pick any two:

- **A "what would have happened" preview.** Before send-confirmation, the confirmation UI shows the audience list preview (first 10 recipients + total count) as read from the CRM at confirmation time. If the CRM changed between draft and confirm, the user sees the change. This is the dry-run tool output from mod-104 chapter 01 turned into a UX artefact.
- **Slack unretract window awareness.** The Slack notification undo button dims when the retract window closes (Slack's message-edit window). A late click shows a legible "this can no longer be undone" message.
- **A cancel-race fuzz test.** A test that fires cancel at random points inside a scripted task 100 times, verifying no side effect landed on any run. Race conditions are what break cancel in production; you find them by fuzzing.
- **Voice-mode adaptation.** The steering handle is the same; the UI is different. Show what the six affordances look like in a voice surface (verbal cancel, "hold on" for pause, "read that back" for progress, "start over" for undo of the last turn). One paragraph per affordance is enough.
- **Multi-tenant kill-switch dashboard.** A small operator UI listing the switches, their states, their audit logs, and the affected surfaces. This is the tool the on-call actually uses in an incident; ship a legible version.
- **A `cancel-and-explain` mode.** After cancel, the agent produces a one-sentence "here's what I would have done next" summary from the checkpointed plan. Users are more forgiving of cancel when the aftermath tells them what they interrupted.

## Deliverable

Submit as `mod-105/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `runtime/` — the SteeringHandle, KillSwitch, and StepEvent implementations.
- `surface/` — the UI (or CLI-with-panels).
- `kill_switch/` — the switch storage, message templates, and operator CLI.
- `tests/` — the integration coverage in Deliverable 5.
- `runbook.md` — the operator-facing runbook.
- A short `README.md` naming the runtime framework you chose and defending the choice in one paragraph.
- Any stretch-goal artefacts named alongside.
