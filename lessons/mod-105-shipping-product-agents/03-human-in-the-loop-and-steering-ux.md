# Human-in-the-Loop, Kill Switches, and Steering UX

## Why steering is a product feature, not a debug hatch

The mental model that ships bad agents is "the agent does the work; the human is a fallback for when it fails." The mental model that ships good ones is the inverse: **the human is the operator; the agent is the operand.** Every affordance you build for progress, cancel, undo, resume, and hand-off is a control the operator uses to steer the operand toward the outcome they want.

This is a product decision as much as a safety one. Users who cannot see what the agent is doing distrust it and abandon it. Users who cannot stop it distrust it and never start a real task with it. Users who cannot undo its output rate-limit themselves down to a task size where the cost of a bad outcome is trivial — which is exactly the task size where the agent's value is trivial. Steering UX is what unlocks the ambitious tasks; without it, you have a demo, not a product.

This chapter is about the six affordances that make a product agent trustworthy: **progress, cancel, undo, resume, hand-off, and kill-switch.** The first five are per-task UX. The kill-switch is the operational break-glass — the mechanism that lets the operator (the applied engineer, the on-call, the incident commander) stop the whole surface for everyone. All six are the applied-AI engineer's job; the peer role handles the agent runtime that makes them observable, but the surface that exposes them is yours.

## The six affordances, at product-agent altitude

Below is the working set. Each is defined by what the user (or operator) must be able to do at *any* moment during an in-flight agent task.

### 1. Progress — "what is the agent doing right now?"

The user must always be able to answer this question without asking. That means:

- A live view of the current step, in the user's language, not the model's. `"Searching your inbox for 'invoice'…"` beats `"tool_call: search_email(query='invoice')"`. The step summary is *for the user*; if you leak the raw tool call, you're making the user do translation work.
- A trace they can expand for detail. The details view shows the tool calls, their arguments (redacted), and their responses (redacted). Two-tier UX — summary by default, detail on click — is the shape every ambitious agent surface converges on.
- Time as a first-class element. Elapsed seconds, an ETA if you can compute one, and a "this is taking longer than usual" affordance when the current step crosses a p90 latency threshold.

The failure mode is the spinner-with-no-detail. A five-second spinner is fine; a two-minute spinner is a broken product. If the agent is doing real work over minutes, the progress view is the product.

### 2. Cancel — "stop what you're doing"

The user must be able to cancel *at any moment*, without a confirmation modal that costs them another turn. That has three implications the applied engineer has to hold.

- **Cancel is a first-class signal in the runtime.** LangGraph's interrupt, the OpenAI Agents SDK's cancel token, the Claude Agent SDK's stop event, or your own equivalent — the runtime must propagate cancellation into every in-flight LLM call, tool call, and background task within a bounded time. "Cancel takes 30 seconds because we wait for the current LLM stream to finish" is a design bug you fix, not a limitation you accept.
- **Cancel is *not* undo.** Cancel stops future work; undo reverses past work. The two are different buttons with different semantics. Cancelling after a `send_email` tool call does *not* unsend the email; that's what §3 is for.
- **Cancel has a legible aftermath.** After cancel, the user sees what did happen ("I searched your inbox and found 12 messages; I did not draft the reply.") and what to do next ("Resume this task, edit the goal, or discard the plan.").

The failure mode is cancel-as-window-close: the user hits the X, the agent keeps working in the background, tools finish, side effects land. Every product-agent post-mortem has a story that starts this way.

### 3. Undo — "reverse what you did"

Undo is only possible for tools with a compensating action; the schema-side story is [mod-104 chapter 01](../mod-104-tool-calling-and-orchestration/01-tool-schema-and-side-effect-boundaries.md). The UX side is:

- **Every visible write in the trace has an undo button, wired to the compensating action.** `send_message` → `retract_message`. `create_task` → `delete_task`. `update_field` → `revert_field(prior_value)`. If a tool has no compensating action, the trace says so explicitly ("this action cannot be undone") and the write itself is gated behind confirmation.
- **The undo is bounded in time.** Some compensating actions have a window (a chat platform lets you retract a message for 30 seconds; a payment processor lets you void an intent for a business day; a git commit is undoable for as long as the reflog holds it). The UI shows the window and dims the button when it closes.
- **Batched writes have a batch undo.** If the agent made ten edits to a document, one undo reverts them as a unit. Ten separate undos is a UX failure — the user's mental model of "the agent did *a thing*," not ten things.

The failure mode is the never-undone write. A tool without an undo path, called by an agent without confirmation, produces an outcome the user has to accept. Do not ship those without explicit design review.

### 4. Resume — "pick up where we left off"

Resume is what makes agents on multi-minute tasks tolerable. The user starts a task, walks away, comes back, and finds either a completed result or an in-progress task they can watch or steer. Three requirements:

- **Persistent state.** The agent's session state is checkpointed to durable storage between steps (see [chapter 02](02-memory-system-with-retention-and-pii-policy.md) on the session layer). "Resume" is "reload the checkpoint and continue."
- **Legible pause points.** The user can pause the agent (soft cancel) and the runtime picks up cleanly. Pause is not "kill." Pause is "hold at the next safe checkpoint."
- **Cross-device.** The user can start on desktop and check progress on mobile. If your agent's state lives in a browser tab, you don't have resume.

The failure mode is the "I closed the tab" tragedy: the agent's work is lost because the runtime lived in the browser. If you're building anything above a coding-inline (Shape A tier 1) surface from [chapter 01](01-agent-shape-for-a-real-product-surface.md), your agent lives on the server.

### 5. Hand-off — "let a human take this"

Every surface with a real user has a moment when the agent should stop and let a human take over. Two archetypes.

- **Agent → human hand-off.** The agent decides (or the policy layer decides) that the case is out of scope, and transfers to a human. In a CX agent (Shape B), this is the escalation to the support rep. In a workspace agent (Shape C or E), this is "I can't do this; here's what I found, take it from here." The essential UX detail: **the human receives the agent's context**, not a cold start. The rep sees the customer's message history, the agent's tool calls, the retrieved facts, and the reason for hand-off, in one pane. Anything less is worse than no hand-off — it makes the customer re-explain everything.
- **User → agent hand-off.** The user starts a task manually, then hands it to the agent. "I've drafted a reply; take it from here." The essential detail: the agent inherits the user's *intent* explicitly, not by inference. A prompt (`"Continue from where I left off; the goal is to schedule a follow-up next week"`) is the right shape.

Both directions share a rule: **the receiver sees the full context, and the sender knows the receiver got it.** Anything else is a lost-in-translation UX.

### 6. Kill-switch — "stop the whole surface"

The kill-switch is the operator's, not the user's. Different button, different loop, different consumer. It exists for cases the per-task cancel cannot solve:

- A model regression ships and the whole agent surface starts hallucinating. The on-call needs to turn it off in seconds, not release-cycle minutes.
- A backend tool starts returning garbage data. The agent, blissfully unaware, propagates the garbage into user-visible writes. The on-call disables the tool immediately.
- A prompt-injection payload is loose on your surface (a user's document is exfiltrating). The security team disables the read tool that surfaces the payload.

Three requirements:

- **Multiple scopes.** Global (kill the whole surface). Per-tenant (kill this workspace). Per-tool (disable this tool for everyone). Per-user (disable for this individual). Each scope has its own switch; each is auditable.
- **Legible to the user.** When the kill-switch fires, the surface shows the user a real message ("This feature is temporarily unavailable while we investigate an issue.") — not a stack trace, not a silent failure, not a hallucinated reply.
- **Fast.** The switch flips a feature flag or a config value that the runtime respects on the next request. The runtime does not poll a git-committed config file that ships on the next release. A kill-switch that takes a deploy is not a kill-switch.

The failure mode is the "we don't have a kill-switch" incident. The team has to yank a Kubernetes deployment or deregister an API key to stop the bleed. That is not a kill-switch; that is scorched earth. Ship a real one on day one.

## A shape for the runtime signals

The five per-task affordances plus the kill-switch turn into six runtime signals. Below is a provider-neutral shape.

```python
# runtime/steering.py — the six signals a product-agent runtime must expose.
from enum import Enum
from typing import Callable, Awaitable, Any
from pydantic import BaseModel

class TurnStatus(str, Enum):
    running     = "running"
    awaiting_user = "awaiting_user"
    completed   = "completed"
    cancelled   = "cancelled"
    failed      = "failed"
    paused      = "paused"

class StepEvent(BaseModel):
    turn_id:    str
    step_index: int
    kind:       str        # "search", "propose", "tool_call", "reply", ...
    summary:    str        # the user-language summary
    started_at: float
    finished_at: float | None = None
    detail:     dict[str, Any] # redacted args + result

class SteeringHandle:
    """Handle returned to the surface for an in-flight turn."""
    async def cancel(self, reason: str) -> None: ...
    async def pause(self) -> None: ...
    async def resume(self) -> None: ...
    async def approve(self, step_id: str) -> None: ...
    async def reject(self, step_id: str, note: str | None = None) -> None: ...
    async def status(self) -> TurnStatus: ...
    def subscribe(self, on_event: Callable[[StepEvent], Awaitable[None]]) -> None: ...

class KillSwitch:
    """Operator-side; not exposed to user surfaces."""
    async def disable_surface(self, reason: str) -> None: ...
    async def disable_tool(self, tool_name: str, reason: str) -> None: ...
    async def disable_for_tenant(self, tenant_id: str, reason: str) -> None: ...
    async def restore(self, key: str) -> None: ...
```

Three properties to notice.

- **`SteeringHandle` is the surface's contract with the runtime.** Every UI affordance (Cancel button, Undo button, Approve modal) is a call on this handle. Every runtime event that changes state (a tool starts, a tool finishes, the model asks for user input) is a `StepEvent`. If your runtime does not expose these, your surface cannot ship the affordances.
- **`approve` / `reject` are first-class.** Human-in-the-loop is not a bolted-on prompt; it is a runtime signal. The runtime pauses at approval steps and only proceeds when `approve` fires. This is exactly the shape LangGraph's `interrupt` primitive provides — not a coincidence.
- **`KillSwitch` is a separate object with its own auth boundary.** No user-facing surface ever holds a reference to it. The operator UI, the on-call tooling, and the incident runbook are the only consumers.

The runtime layer that produces these signals is the peer role's territory. The surface that consumes them is yours. Get the contract right and the two teams can iterate independently.

## The confirmation UX for high-blast-radius writes

Cancel, undo, and resume assume the write has already happened. Confirmation is the guard *before* the write. It matters most for tools that meet any of three conditions:

- The action has no compensating undo (a sent email, a public post, a physical fulfilment).
- The action's cost is material and asymmetric (a large refund, a payment, a shared calendar invite).
- The action touches an audience beyond the user (a message to a customer, a Slack post to a shared channel).

Confirmation UX has three failure modes to avoid.

**1. Modal fatigue.** If every tool call opens a modal, users click through them without reading, and the modal is worse than no modal (it launders responsibility). Confirmation is reserved for the criteria above. For low-blast-radius writes, an inline "propose → click to accept" bar is faster and more legible than a modal.

**2. Legibility failure.** "The agent wants to send an email. Approve?" is not a confirmation. "The agent wants to send this specific email to `boss@company.com` with this subject and this body." is. The confirmation surface shows *exactly* what will be sent, in the form the recipient will see it. This is where dry-run tool outputs (see [mod-104 chapter 01](../mod-104-tool-calling-and-orchestration/01-tool-schema-and-side-effect-boundaries.md)) become UX artefacts, not just internal ones.

**3. Silent bypass.** A confirmation UX that the model can synthesise a "yes" for, or that the runtime skips when a feature flag is set, is not a confirmation. The `user_confirmed: Literal[True]` sentinel from mod-104's exercise 1 is the schema-side of this rule; the UX side is that the sentinel is set only by an explicit user gesture, and the runtime enforces it.

Combined, the three rules produce a confirmation UX that users don't resent: rare, specific, and honest. If your users learn to click through confirmations, you have too many, and you have neither safety nor UX.

## The kill-switch runbook (before you need it)

Kill-switches only work if the operator knows how to use them. Write the runbook before you ship the surface. A workable outline:

- **Symptoms → switch.** A table mapping observable symptoms ("acceptance rate has dropped 20% in the last hour", "tool `send_message` error rate exceeds 5%", "user reports agent is 'saying things it shouldn't'") to the switch that stops the bleed. On-call should not have to think; they should be able to look up.
- **Blast radius per switch.** Global switch: everyone affected. Tool switch: which surfaces still work? Tenant switch: who is unaffected?
- **Communication template.** A pre-written user-facing message for each switch. When the surface goes dark, the message that appears in the UI should not be an afterthought.
- **The re-enable checklist.** What must be true before you flip the switch back? A hotfix deployed? A tool retested? A prompt regression rerun? An incident review scheduled? Do not make on-call improvise the re-enable criteria at 3am.

The runbook is a document; the switches are code; both are versioned. When a switch fires in an incident, the runbook and the switch are both updated in the post-mortem.

## The applied engineer's steering scorecard

For every product-agent surface you ship, walk through the seven questions below in the design review. If you cannot answer them all *before* code freeze, you are not ready to ship.

1. **Progress:** what does the user see while the agent works? Draw the screen.
2. **Cancel:** what happens when the user hits cancel *now*? Walk through the tool call currently in flight.
3. **Undo:** for each tool with a side effect, name the compensating action. If there is none, name the confirmation gate.
4. **Resume:** if the user closes the tab, what state persists? Where does it live? How do they resume?
5. **Hand-off:** when the agent gives up (or the policy says "hand off"), what does the human receive? Draw the screen.
6. **Kill-switch:** what switches exist, who can flip them, and how fast does the surface honour the flip?
7. **Confirmation:** which tools require explicit user approval? What does the approval UI show?

Each question corresponds to one of the affordances above. Each has an answer *in code*, not "we'll figure it out." A "no" on any question is a blocker, not a note.

## Anti-patterns

Seven failures at the steering layer, all seen in real launches.

1. **The spinner-only surface.** No progress detail, no cancel, no trace. Users abandon before the model responds. The runtime probably does emit events; the surface just doesn't show them.
2. **Cancel-that-doesn't-cancel.** The button dismisses the UI but the agent keeps working in the background. The next side effect the user sees came from a task they thought they cancelled.
3. **Cancel-as-undo.** Cancelling after a `send_message` unsends nothing, but the UI's language implies it did. The user finds out from the recipient.
4. **The unreviewable batch write.** The agent edits 47 fields in the CRM. There is no batch undo, only a per-field revert. The user restores half of them and gives up.
5. **The cold-transfer hand-off.** The agent gives up. The user is dumped into a new "human" chat with no context. The support rep opens the ticket to a blank page.
6. **Confirmation modal fatigue.** Every tool call is a modal. Users click through them all. Safety theatre, not safety.
7. **The kill-switch that requires a deploy.** In the incident, the on-call has to `git push` to disable a broken tool. By the time the deploy lands, the tool has run against 40,000 more requests.

## Summary

- Steering UX (progress, cancel, undo, resume, hand-off, kill-switch) is a product feature, not a debug hatch. It is what makes users trust an agent enough to give it real tasks.
- Progress is two-tier (summary + expandable detail) in the user's language, always visible while work is in flight. Elapsed time and ETA are first-class.
- Cancel is a first-class runtime signal that propagates into every in-flight LLM and tool call within a bounded time. Cancel is not undo; both are separate affordances with separate semantics.
- Undo is only possible where the tool has a compensating action; the schema decision from mod-104 chapter 01 is the pre-condition, but the UI wiring is where the user actually sees it. Batched writes need a batched undo.
- Resume requires durable session state on the server, legible pause points, and cross-device continuity. If your agent's state dies with the browser tab, you don't have resume.
- Hand-off (agent→human and user→agent) is defined by the receiver seeing the sender's full context. Cold transfer is worse than no hand-off.
- The kill-switch is the operator's break-glass. Multiple scopes (global / tenant / tool / user), fast propagation, legible user-facing messages, and a runbook that predates the incident. A kill-switch that requires a deploy is not a kill-switch.
- Confirmation UX for high-blast-radius writes is rare, specific, and honest — driven by dry-run tool outputs and enforced by the runtime's approval signal, not by prompt discipline alone.
- Walk the seven-question steering scorecard in every design review. Every answer is in code before code freeze.

[Chapter 04](04-eval-suites-from-benchmark-shapes.md) picks up the question of how you know any of this actually works: what to steal from the public agent benchmarks (`τ-bench`, SWE-bench Verified, WebArena, AgentDojo) as shape templates for internal eval, and how to keep the product-agent eval focused on real product tasks rather than benchmark maximisation.
