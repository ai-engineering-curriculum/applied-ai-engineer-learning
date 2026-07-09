# Choosing the Feature Shape

## Why the shape decision comes first

A product spec almost never says "build a chat window" — it says "help our support agents draft replies faster," or "let a customer ask a question about our warranty policy and get an answer they can trust," or "extract the 12 fields we care about from a contract PDF." Between that outcome and any model call sits a design decision that constrains everything downstream: **what shape does the feature take on the surface?**

Get the shape wrong and no amount of prompt engineering, RAG plumbing, or model swapping will save you. A chat UI on top of a workflow that needed a one-shot form burns latency, spends tokens on politeness, and hides the actual decision from the user. An autonomous agent on top of a workflow that needed structured extraction takes minutes to do a job that should take seconds, and introduces failure modes (looping, off-policy tool calls, hallucinated arguments) that a form never has.

The shape decision is a scoping decision, not an aesthetic one. It fixes:

- **The control locus** — who's driving? The model, the user, or a workflow author who wrote the graph the model executes inside?
- **The latency budget** — a form field can wait 2 seconds. A chat message can wait 5. A background extraction pipeline can wait 60. An autonomous agent may take minutes, but only if the user has consented to that and gets progress feedback.
- **The evidence surface** — do citations and confidence show, and where? An extraction JSON puts confidences in fields. A retrieval answer puts them next to sentences. A chat message hides them.
- **The failure surface** — what does the user see when the model is wrong? A form shows a wrong value they can edit. A chat shows a plausible lie. An agent burns tokens and time before returning an unusable trace.

## The seven shapes

The catalogue below is not exhaustive — any real product will blend shapes — but these seven are the discrete choices you defend in an architecture review. Learn to name them, and to name what each one fails at.

### 1. Chat

A turn-taking conversation with no external tools, no retrieval, and no committed side effects. The model reads the conversation state and writes the next assistant turn.

- **Signals it fits**: open-ended user intent, exploratory dialogue, brainstorming, tutoring, low-stakes Q&A on the model's parametric knowledge.
- **Signals it does not fit**: any answer that must be grounded in *your* data, any workflow with a definite outcome, any interaction whose success is measured (chat is optimised for *engagement*, which is not the same as task success).
- **Fails at**: freshness (parametric knowledge is stale), auditability (no citations), determinism (no schema), throughput (turn-taking is expensive for high-volume workflows).

### 2. Assistant

Chat + retrieval + light tools, embedded in a product surface (a sidebar in a CRM, a panel in an IDE, a widget in a helpdesk). The user still drives, but the assistant can pull context and take small actions on their behalf.

- **Signals it fits**: the user is inside a product workflow and needs help without leaving it; the actions the model can take are all reversible or confirm-first.
- **Signals it does not fit**: the workflow has a single well-defined output (use one-shot generation or structured extraction); the actions are irreversible without a clear confirm step (use a co-pilot with explicit approval UX).
- **Fails at**: multi-step goals that need planning across many tool calls (assistants degrade into loops), scenarios where the "help" is unwelcome (assistants that pop in unbidden are the enemy of a working product).

### 3. Co-pilot

The model proposes; the user accepts, edits, or rejects. The output is inline in the user's actual work surface (code, prose, spreadsheet, image canvas), and every accepted suggestion is a positive signal you can log.

- **Signals it fits**: the user is producing an artefact and would benefit from suggestions inline; the acceptance signal is legible; the cost of a bad suggestion is a keystroke to reject.
- **Signals it does not fit**: the user does not have the expertise to evaluate the suggestion (they will accept wrong outputs — see [chapter 02](02-slos-from-spec-to-metrics.md) for why groundedness must be measured explicitly); the artefact is discrete enough that a form fits (extract twelve fields, do not "co-pilot" twelve fields).
- **Fails at**: automation-bias failures (users accept plausible-looking wrong output), evaluation (accept-rate alone is a shallow success signal — see [chapter 05](05-leaderboard-driven-failure-modes.md)).

### 4. Autonomous agent

The model is given a goal, a bounded tool inventory, and a budget (steps, tokens, wall-clock, dollars), and drives itself to completion with or without human check-ins. The user consents to the delegation up front and gets a progress + cancel + resume + undo UX.

- **Signals it fits**: the task requires many steps whose ordering is not fixed in advance; each individual step is checkable; the value of hands-off completion outweighs the risk of a wrong trajectory.
- **Signals it does not fit**: the task is one-shot (do not wrap a form in an agent); the tools have irreversible side effects with no undo; the user cannot intervene mid-run; you cannot afford the tail latency of multi-step generation.
- **Fails at**: cost predictability (tail cost is heavy-tailed), latency (multi-minute wall-clock), safety envelope drift (each additional tool call is another chance to be jailbroken or to act on injected instructions — see the surface-side of prompt-injection defence in mod-107).

### 5. One-shot generation

A single prompt in, a single completion out. No conversation state, no retrieval, no tools. Often used as a background worker (batch scoring, drafting, classification) rather than a user-facing surface.

- **Signals it fits**: the input is fully specified up front; the output is either final or trivially edited downstream; you can run it in a batch pipeline where p95 latency doesn't matter.
- **Signals it does not fit**: the user needs to steer mid-generation; the answer must be grounded in your data (add retrieval); the output has a rigid schema (use structured extraction).
- **Fails at**: iterative refinement (users will *want* to steer even if you didn't design for it), high-recall extraction (single-shot generation is not a substitute for schema-constrained extraction).

### 6. Retrieval-with-citations

The model answers a question with sentences that are each backed by a retrieved evidence span, and the evidence is shown next to the answer so the user (and a downstream reviewer) can verify. When the retrieval turns up nothing sufficient, the answer says so explicitly.

- **Signals it fits**: the domain has authoritative documents the user needs to trust; freshness matters; the "no good evidence" case has product meaning (regulatory answers, product-policy answers, in-product help, sales-collateral Q&A).
- **Signals it does not fit**: there is no authoritative corpus (do not fake it with the open web unless that *is* the corpus); the user does not care where the answer came from (they'll just want the answer — but you'll still want the citations for your eval).
- **Fails at**: multi-hop reasoning across many documents (single-hop is honest; multi-hop is where hallucination lives), open-ended generation without a grounding target.

### 7. Structured extraction

A document (or transcript, or ticket, or invoice) goes in; a validated JSON object with defined fields — including confidence scores, provenance spans, and explicit `null`s for missing fields — comes out.

- **Signals it fits**: the output has a known, stable schema; downstream systems consume the fields programmatically; you can score correctness field-by-field on a labelled set.
- **Signals it does not fit**: the schema is not stable (you'll be re-extracting on every change); the fields are open-ended prose (that is generation, not extraction).
- **Fails at**: novel or emergent fields (extraction assumes you know what you're looking for), unstructured user-facing surfaces (do not hand a user a JSON blob).

## The decision, made explicit

A defensible feature-shape decision has three parts:

1. **The shape you chose.** Named from the list above (blends are allowed, but name them: "retrieval-with-citations *behind* a chat surface, with the citation panel always open").
2. **The rejected shapes, and why.** Each of the other six named, with a one-line rejection ("Autonomous agent rejected: the workflow is single-turn and the user is a novice, so multi-step planning adds cost and failure surface without adding value").
3. **The consequence of being wrong.** If you picked "assistant" and it should have been "co-pilot," what will you see? (Users will complain that the sidebar is chatty when they wanted inline suggestions.) This is the signal you'll monitor to know whether to re-shape.

The rejection step is the one people skip. Without it, a review reads as "we chose chat because chat is what LLMs do," which is not a design decision. With it, the review reads as "we chose chat because the user intent is exploratory and open-ended, and specifically rejected structured extraction because there is no stable schema."

## A worked example

**Spec**: "A support agent gets a customer email. We want them to draft a reply faster, using our knowledge base and our previous replies as context."

Walk the shapes:

| Shape | Fits? | Why / why not |
|---|---|---|
| Chat | No | The agent has an artefact (the draft reply). Chat hides the draft in conversation history. |
| Assistant | Partial | Right neighbourhood — sidebar in the helpdesk — but the primary action is producing the *draft*, not answering questions about it. |
| Co-pilot | **Yes** | The draft goes into the reply textarea; the agent accepts, edits, or discards; every send is an acceptance signal. |
| Autonomous agent | No | The reply must go out under the agent's name; delegating send-authority to the model is a policy question, not a design one. Reject unless the org has an explicit auto-reply policy. |
| One-shot generation | Partial | The draft is a one-shot output, but without retrieval it will invent policy. Combine with retrieval. |
| Retrieval-with-citations | Component | Yes as an *input* to the co-pilot: the drafts are grounded in retrieved policy passages and previous replies, with citations visible in the sidebar so the agent can verify before sending. |
| Structured extraction | No | The reply is prose, not a schema. |

**Chosen shape**: co-pilot on the reply textarea, powered by retrieval-with-citations behind the scenes. The citation panel is always visible; the send button is the acceptance signal. **Rejected**: autonomous agent (policy), chat (hides the artefact), pure one-shot (no grounding), assistant (right shape, wrong primary action), pure extraction (wrong output type).

## Summary

- Feature shape is the first architecture decision and constrains every downstream one — provider, prompt, retrieval, eval, guardrail, cost.
- The seven shapes — chat, assistant, co-pilot, autonomous agent, one-shot generation, retrieval-with-citations, structured extraction — are a rehearsable catalogue; you will pick from them under time pressure in real reviews.
- A defensible decision names the chosen shape, names the rejected shapes with reasons, and names the consequence-of-being-wrong signal that will tell you to re-shape.
- Blends are fine; unnamed blends are not. If a shape is really "assistant with an inline co-pilot component," write that down.

Chapter 02 turns the chosen shape into a concrete SLO table — what "success" and "safe" and "fresh" mean, in numbers.
