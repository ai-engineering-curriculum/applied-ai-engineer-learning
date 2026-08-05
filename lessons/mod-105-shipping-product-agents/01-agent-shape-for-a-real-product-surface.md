# Agent Shape for a Real Product Surface

## Why "shape" is the load-bearing decision

Before you write a prompt, before you pick a model, before you draw a graph, you pick a **shape**. Shape is the answer to five questions taken together: *where does the user meet the agent, what does the agent take from them, what does it hand back, what is it allowed to touch on their behalf, and who confirms the touches?* Get shape right and the rest of the module (memory, hand-off UX, eval, cost) has somewhere sensible to attach. Get shape wrong and every subsequent decision becomes a workaround.

The classic wrong turn is to skip the shape decision because "we already know it's an agent." That word covers at least five product surfaces that make wildly different design demands. A coding agent that edits your working tree in place is not the same product as a customer-support agent that answers a chat, even if they share a runtime. A retrieval-with-actions assistant embedded in a Notion doc is not the same product as a browsing agent that navigates a public site on your behalf. The applied-AI engineer picks a shape, writes the case for it, and defends it in the design review.

This chapter is that decision. It names five agent shapes that show up in real product-agent work, gives each one a written case (what it's for, what it costs you, what it costs the user, what it demands of the rest of the stack), and closes with a decision procedure you can walk through with a PM in an hour. The remaining chapters — memory ([02](02-memory-system-with-retention-and-pii-policy.md)), steering UX ([03](03-human-in-the-loop-and-steering-ux.md)), eval ([04](04-eval-suites-from-benchmark-shapes.md)), and public patterns ([05](05-public-applied-agent-patterns.md)) — assume you've made this call.

## What "product-embedded" changes about the shape decision

An agent embedded in a product is not the same artefact as a standalone chatbot on a URL, and it is not the same artefact as an autonomous agent that runs unattended in a cron job. Three constraints are load-bearing.

**The user is right there.** They can see what the agent is doing. They can cancel. They can push back within seconds. This is a *feature*, not a limitation — the closer the user, the shorter the trust loop, the more the agent can do. A shape that hides the agent's work from the user (background summarisation with no visible trace) throws away the strongest feedback channel you have.

**The surface has a job.** Users didn't come to your product to talk to an agent; they came to write code, close tickets, book meetings, review a contract, ship a marketing brief. The agent is instrumental. If the shape makes the user do more work than they would have without it, the shape is wrong, and no amount of model capability will fix it.

**The blast radius is real.** Every write the agent performs touches something the user or the company owns — a repo, a customer record, a payment, a calendar, a shared document. The shape decides how many of those writes the agent may perform unaided, and the memory and HITL chapters build on that decision.

Hold those three constraints in mind for every shape below.

## Five shapes that show up in real product-agent work

The list below is not exhaustive — you will meet fusions (a coding co-pilot with retrieval, a support agent with browsing) — but every product-agent design review starts by placing the surface in one of these regions.

### Shape A — Coding co-pilot (inline / chat / composer / workspace)

**What it is.** An agent embedded in a developer's IDE, editor, or code review tool that helps them read, write, and modify code. The internal shape spans a four-step ladder from lowest to highest autonomy:

- **Inline completion** — token / line / block suggestions as the user types. The agent proposes; the user accepts or rejects with a keystroke. Very tight loop, very low autonomy.
- **Chat** — a side-panel conversation, often with the current file or selection as context. The agent explains, drafts, refactors on demand. Higher autonomy per turn, but every artefact lands in a diff the user reviews before applying.
- **Composer / multi-file edit** — the user describes a change; the agent produces a diff spanning many files, often after searching, reading, and calling tools like `grep` and `read_file`. The user reviews the diff as a unit and accepts, discards, or iterates.
- **Workspace / long-running task** — the user files an "issue-like" task ("port this module to TypeScript strict mode", "add a rate limiter to the checkout endpoint"); the agent works over minutes, possibly with a plan the user reviews, then produces a branch or PR the user reviews.

**Written case.** Choose this shape when the artefact is *code*, the user is a developer, the acceptance surface is a diff, and the environment is instrumented (files, tests, build). The user-visible signal is strong (compile errors, test failures, "the code doesn't run") and cheap to consume. The blast radius is bounded by version control — a bad diff is a revert away.

**What it costs you.** Ranking (which suggestion to show first), latency at the inline tier, retrieval over the workspace, agentic loop stability at the composer / workspace tier, cost per accepted line as the north-star business metric.

**What it costs the user.** The user has to *read the diff*. If the composer surfaces are so productive that no one reviews the diffs, you have shipped an autonomous coding agent, not a co-pilot, and the shape has quietly changed under you. Keep an eye on acceptance rate vs. modification rate.

**What it demands of the rest of the stack.** Memory is mostly workspace-scoped ([02](02-memory-system-with-retention-and-pii-policy.md)); HITL is diff-review + revert ([03](03-human-in-the-loop-and-steering-ux.md)); eval is SWE-bench-shaped at the workspace tier ([04](04-eval-suites-from-benchmark-shapes.md)); the shipping model is Cursor / Copilot ([05](05-public-applied-agent-patterns.md)).

**Where it goes wrong.** Confusing the tiers. Shipping a composer that behaves like a workspace agent (edits without a review gate). Shipping a workspace agent that behaves like a composer (returns a diff so large no one can review it). The four tiers have four different UX contracts; pick one and honour it.

### Shape B — Customer-experience agent (support / CX)

**What it is.** An agent that talks to your customer, in your product's chat / email / phone surface, on behalf of your company. It has read access to the customer's account, may have write access to a bounded set of actions (issue a refund, reset a password, exchange an item, open a support ticket), and hands off to a human when the case is out of policy.

**Written case.** Choose this shape when there is a policy-bounded catalogue of actions the customer commonly needs and the volume of routine cases dwarfs the exception cases. The savings case is real: a support agent that resolves half of a company's tier-1 tickets to the customer's satisfaction, with a clean hand-off on the rest, changes the shape of the CX org. Sierra's public deployments (Sonos, Weight Watchers, ADT) and Klarna's public announcement about their in-house agent handling two-thirds of chats are the reference points here.<!-- needs-research: confirm Klarna's current published deployment metrics before quoting a specific number. -->

**What it costs you.** A policy corpus you can point the agent at (not a wiki dump). A tool catalogue that maps to real backend systems. A hand-off path with warm context transfer to a human agent. Instrumentation that measures resolution rate, containment rate, escalation rate, and CSAT — separately, because they trade off.

**What it costs the user (the customer).** Latency (a human is slower than a chatbot but faster than "please hold"). Trust (they must be able to escalate). Legibility (they must be able to tell whether they're talking to a human or an agent — laws differ by jurisdiction, but this is a UX honesty question independent of law).

**What it demands of the rest of the stack.** Retrieval on the policy corpus (see mod-103). Tool calling on the backend actions (see mod-104). Kill-switch and hand-off UX from [chapter 03](03-human-in-the-loop-and-steering-ux.md). Eval that looks like `τ-bench` at the shape level — task-list-plus-tool-catalogue-plus-user-simulator ([chapter 04](04-eval-suites-from-benchmark-shapes.md)).

**Where it goes wrong.** Treating the CX agent like a chatbot — no tool access, no hand-off — and calling it "self-service." Treating the CX agent like an unbounded assistant — write access to everything the customer service rep can touch — and skipping the policy layer. The shape lives in a narrow band between those two failure modes.

### Shape C — Retrieval-with-actions assistant

**What it is.** An agent embedded in an information-dense product (a Notion / Confluence / CRM / EHR / analytics tool) that answers questions grounded in the user's own data and can then act on that data through a small set of write tools. The read side is retrieval-heavy (see mod-103); the write side is a bounded action catalogue (see mod-104). The two together — read *and then* act — is the shape.

**Written case.** Choose this shape when the value is *inside* the user's workspace, when a plain chatbot bolted onto the product is worse than the product's existing search, and when the tools the user would use manually (create a task, update a field, send a follow-up, generate a report) can be shipped as agent-callable actions. Notion AI, Salesforce Einstein, Glean Assistant, and Perplexity's workspace features sit in this region.

**What it costs you.** A serious retrieval layer over the user's workspace: permissions-aware, freshness-aware, source-cited. A bounded action catalogue where every write is undoable or dry-runnable. A confirmation UX on the write path that is legible to a user who does not want to read a modal.

**What it costs the user.** The user has to trust that the retrieval isn't lying (source citations do most of the work) and that the writes are the ones they asked for (preview + one-click undo does most of the rest).

**What it demands of the rest of the stack.** Deep retrieval discipline (mod-103). Tool schemas that make side effects legible from the name (mod-104 [chapter 01](../mod-104-tool-calling-and-orchestration/01-tool-schema-and-side-effect-boundaries.md)). Memory that stays inside the workspace boundary ([chapter 02](02-memory-system-with-retention-and-pii-policy.md)). Undo / revert UX ([chapter 03](03-human-in-the-loop-and-steering-ux.md)).

**Where it goes wrong.** Retrieval is treated as "load the whole workspace into context" and the answers hallucinate. Actions are shipped without dry-run and the first user demo damages a customer's real data. The read half is shipped without the action half — the assistant answers but cannot do anything, and users bounce.

### Shape D — Browsing agent

**What it is.** An agent that navigates the public web on the user's behalf: reads pages, fills forms, clicks buttons, extracts data. The user's job is delegated wholly for the duration of the task. OpenAI's Operator surface, Anthropic's "computer use" surface, and the emerging class of "book a flight / apply to this job" agents live here.

**Written case.** Choose this shape when the task is *browsable* (public site, form-driven, no privileged API), when the user's cost to do it themselves is higher than the agent's cost to fail, and when the user is willing to review the outcome before it commits (basket at checkout, not "we bought it for you"). The shape is the newest of the five and the least settled — see [chapter 05](05-public-applied-agent-patterns.md) for what's actually shipping.

**What it costs you.** A browser environment the agent can drive. Screenshot / DOM handling. Robust page-recovery when the site changes. A cost model that reflects real page fetches and tool calls, not just LLM tokens. Screen recording / event log so the user can audit what the agent did.

**What it costs the user.** Latency (minutes, not seconds). Trust in an opaque process (a screen recording helps, a "the agent did the thing" summary alone does not). Legal exposure if the agent violates a site's terms of service on the user's behalf.

**What it demands of the rest of the stack.** Aggressive HITL ([chapter 03](03-human-in-the-loop-and-steering-ux.md)) — this shape has the highest blast radius per turn of any in the list. WebArena-shaped eval ([chapter 04](04-eval-suites-from-benchmark-shapes.md)). Prompt-injection defence against hostile pages (mod-107), because a browsing agent that reads instructions off a webpage will follow them.

**Where it goes wrong.** Shipping the shape as "we'll figure out safety in v2." Optimising for benchmark scores on WebArena while ignoring the confirmation UX. Treating it as a "solved" shape when the failure rate is still one-in-three at the top end of the current SOTA and the failure modes are visible-to-the-user.

### Shape E — Sales / support / ops assistant

**What it is.** An agent embedded in the tooling of an *internal* user — an SDR, a support rep, an ops analyst, an underwriter, an SRE — who is *not* the end customer. The internal user is the customer of the agent; they trust it, they can override it, they close the loop on quality. Common products: Gong / Clari / Salesforce-for-SDRs, Zendesk / Front / Intercom agent-assist, Datadog's Bits, Cursor for internal codebases.

**Written case.** Choose this shape when a skilled human worker sits at the surface, when their throughput is the metric that matters, and when the *policy* the agent must follow is their policy, not the customer's. The user is more forgiving of latency and more capable of steering — you can ship more autonomy per turn than in a customer-facing shape, at lower risk. Harvey (associates + lawyers) and Sierra's own agent-assist products sit here.<!-- needs-research: confirm Harvey's current publicly-described positioning between "for lawyers to use" and "for law firms to embed" before making stronger claims. -->

**What it costs you.** Domain knowledge in the retrieval corpus. Tool access to the internal system the user is trying to move through. Metrics that reflect the internal user's throughput (calls per hour, tickets per shift, cases per week) not vanity chat metrics.

**What it costs the user.** Learning a new tool. Sometimes: trust that the agent won't be used against them (an agent that scores your calls, or ranks your reps, has a wholly different consent contract).

**What it demands of the rest of the stack.** Retrieval and tool calling similar to Shape C, plus a memory model tuned to the internal user (their preferences, their playbook, their prior cases). HITL is lighter than in Shape B — the internal user is qualified — but auditability is heavier, because internal users are more likely to be regulated actors.

**Where it goes wrong.** Building for the *manager* (dashboards) not the *user* (throughput). Building an agent that saves time for the user but takes so long to learn that six-month time-to-value kills adoption. Copying the customer-facing shape's confirmation UX (heavy) onto the internal shape, where it's condescending and slows down the user.

## A one-hour decision procedure

Six questions, in order. The answers place your surface in one of the five shapes above, or reveal that you have a fused surface that needs to be split.

1. **Whose day are you saving?** A developer (A). A customer of yours (B). A user inside your workspace product (C). A user delegating a public-web task (D). An internal team member of yours (E).
2. **What is the artefact the user receives?** A diff (A). A resolved case (B). An answer with actions applied (C). A summary + evidence of a completed web task (D). A next-best-action recommendation the user takes (E).
3. **How many writes may the agent perform unaided?** Zero, and everything lands in a diff (A). A bounded catalogue of policy-approved writes (B). A bounded catalogue of undoable writes (C). Ideally zero; more realistically, form fills only up to a confirmation step (D). Bounded reads-and-suggests plus a small write catalogue behind explicit user approval (E).
4. **Who owns the policy the agent must obey?** The user (A, C). The company (B). The user, mediated by the company (C, E). The site the agent is browsing (D). The company, enforced against the internal user (E).
5. **What's the eval fixture that would fail the surface today?** A hard bug in a real repo (A → SWE-bench-shaped). An angry customer with an ambiguous refund case (B → τ-bench-shaped). An out-of-date document the assistant confidently misquotes (C). A shopping site that A/B tests its UI weekly (D → WebArena-shaped). A weird ticket the ops user sees once a quarter (E).
6. **What is the failure the user will forgive most quickly?** A rejected suggestion (A). A hand-off to a human (B). "I don't know; here's what I found" with a link (C). A "task incomplete; here's the recording" (D). A wrong recommendation the user chose not to take (E).

If Q1–Q6 all point at the same shape, write the case for that shape, take it to the design review, and move on to [chapter 02](02-memory-system-with-retention-and-pii-policy.md). If they point at two shapes at once, you have a fused surface — most often A+C (a coding assistant with workspace retrieval) or B+E (a customer agent with an agent-assist mode for the human rep behind it). Fused surfaces are fine, but ship them as two shapes with two contracts, not one uncomfortable middle ground.

## The applied-AI engineer's remit, and where the peer track begins

A recurring line across mod-105 is the split between the applied-AI engineer and the agentic-AI engineer peer role. Shape is where the split first shows up.

The **applied engineer** owns the shape decision, the product-facing UX contract for that shape, the tool catalogue exposed to the model, the retrieval corpus, the memory boundaries, the HITL surface, the eval fixtures that mirror real product tasks, the observability of the running surface, and the shipping / rollback discipline. They own the *product agent* as a shipped feature.

The **agentic engineer** on the peer track goes deeper on planner / executor architecture, reasoning and reflection loops, multi-agent coordination, self-play / synthetic-data pipelines for agent training, and benchmark-topping shape research. They own the *agent engine* as an infrastructure primitive that the applied engineer consumes.

At the shape-decision layer, this split is stark: **the applied engineer never picks a shape because "it will benchmark better on WebArena."** They pick a shape because the user's job is best served by that shape, and then they consume the agent-engine capabilities the peer role has built to make the shape work. Chapters [04](04-eval-suites-from-benchmark-shapes.md) and [05](05-public-applied-agent-patterns.md) re-examine this split for eval and for concrete shipped agents.

## Anti-patterns

Five failures at the shape layer, all seen in real product retros.

1. **"It's an agent, so it's the same product as our other agents."** No — a coding agent, a support agent, and a browsing agent share almost nothing at the shape level even if they share a runtime. Each needs its own case.
2. **Shape drift under pressure.** A composer that starts editing files without a review gate because "the diff is small" is a workspace agent. Ship it as one, or hold the gate.
3. **The undifferentiated "AI assistant."** A shape that answers questions, edits documents, sends emails, and browses the web is five shapes glued together. It will do none of them well and every failure will be blamed on "the AI."
4. **Fusing two shapes for engineering convenience.** Shape B (CX agent) and Shape E (agent-assist) look similar in a code diff but have opposite trust contracts — one talks to your customer, the other talks to your employee. Ship them as two products with two policies.
5. **Picking the shape from the model's capabilities, not the user's job.** The frontier model can do X, therefore we should build a product around X. That's a demo, not a shape. Start from the user's job; consume model capabilities to serve it.

## Summary

- Shape is the first product-agent decision: where the user meets the agent, what artefact they get, and how many writes the agent performs unaided. Every other module decision depends on shape.
- Five shapes cover most product-embedded agent surfaces: coding co-pilot (A), CX agent (B), retrieval-with-actions assistant (C), browsing agent (D), sales / support / ops assistant (E). Each has its own written case, its own cost model, and its own downstream demands.
- Product-embedded shapes have three constraints a standalone chatbot doesn't: the user is present and steering, the surface has a real job that predates the agent, and the blast radius is real. Use those constraints to keep the shape honest.
- The one-hour decision procedure (whose day, what artefact, how many writes, who owns policy, what fixture would fail, what failure is forgiven) places most surfaces cleanly. Fused surfaces exist, but ship them as two shapes with two contracts.
- The applied-AI engineer owns the shape decision and the product-facing UX around it. The agentic-AI engineer peer owns the agent-engine primitives the applied engineer consumes. Do not pick a shape because it benchmarks well; pick it because the user's job is best served by it.

[Chapter 02](02-memory-system-with-retention-and-pii-policy.md) picks up where this one left off: given the shape, what does the memory system look like, and how do you reconcile the "the agent remembers me" UX with the "I have the right to delete my data" contract you owe every user?
