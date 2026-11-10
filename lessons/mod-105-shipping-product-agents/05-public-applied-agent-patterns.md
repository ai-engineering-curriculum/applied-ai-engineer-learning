# Public Applied-Agent Patterns

## Why we study shipped agents, not demoware

Every applied-AI engineer will be asked, at some point in a design review, "how does Cursor / Copilot / Sierra / Harvey handle this?" The right response is not "I don't know" and not "let me recite their marketing page." The right response is the *design choice* their surface visibly makes, and what it implies about the trade-off they took. This chapter is that reference — four public applied-agent patterns, each of them shipped at scale, each with design choices worth stealing, avoiding, or arguing with.

The four patterns cover the full space of shapes from [chapter 01](01-agent-shape-for-a-real-product-surface.md):

- **Cursor Composer** — a workspace-tier coding co-pilot (Shape A, higher-autonomy tier).
- **GitHub Copilot Workspace** — a task-decomposition coding co-pilot (Shape A, top tier).
- **Sierra CX agents** — customer-experience agents shipped as a platform, not a single product (Shape B).
- **Harvey** — a legal-work agent for professional users (Shape E, with retrieval-heavy overlap into Shape C).

For each, this chapter names what the surface does, calls out the design choices that stand out, and separates what belongs to the applied-AI engineer (the surface, the eval, the memory, the HITL) from what belongs to the agentic-AI engineer peer role (the underlying engine, the planner / executor loop, the model-side research). The point is not to imitate; the point is to have a clean reference set of *shipped* choices to compare your own decisions against.

Two constraints on the reader before we start:

1. **These products change monthly.** Feature sets, tier names, and UI conventions have shifted materially since each surface first shipped. Cite the current live product docs before you quote a specific behaviour in a design review.
2. **These are public patterns, not internal architectures.** The applied-engineer reading is about the *user-observable* design choice and what it implies. The internal architecture that produced the choice is often not disclosed and is not the applied engineer's remit to reverse-engineer.

## Cursor Composer

**What it is (as a user experience).** Cursor is a fork of VS Code whose current feature surface spans Tab (inline completion), Chat, Composer, Plan Mode, Agent, and Cloud Agents (parallel long-running sessions that evolved from the earlier Background Agents) — see [cursor.com/features](https://cursor.com/features) and [docs.cursor.com](https://docs.cursor.com/). Composer and Agent are the tiers that matter for this chapter — the user opens a session, describes a change ("add a rate limiter to the checkout endpoint"), and the agent searches the workspace, reads files, proposes a diff spanning many files, and applies it under user review.

**Design choices worth naming.**

- **The four-tier autonomy ladder is visible to the user.** The user picks the tier explicitly for each task. Small edits go through tab completion or chat; multi-file changes go through Composer. Cursor does not try to "figure out" the tier from a single input field; the tier is a user-facing choice with its own UI. That legibility is the shape decision from [chapter 01](01-agent-shape-for-a-real-product-surface.md), rendered as product.
- **The diff is the artefact, always.** Even when Composer edits many files, the outcome the user reviews is a diff hunk they can accept or reject. The IDE's diff view is the confirmation UX (see [chapter 03](03-human-in-the-loop-and-steering-ux.md)) — dry-run tooling folded into the primary UI.
- **The agent's tools are IDE-native.** Search across the workspace, read a file, edit a file, run terminal commands (with user consent), read the linter output. The tool catalogue is small, well-named, and legible. When Composer decides to run a shell command, the command is visible before it runs; the user approves it in the same UI.
- **Context is workspace-scoped.** The agent's inputs are the open workspace, its files, the current cursor / selection, and the user's message. There is no cross-workspace long-term memory injected by default; the user's Cursor account is a login, not a persistent context. That is a deliberate memory-layer choice from [chapter 02](02-memory-system-with-retention-and-pii-policy.md).
- **Background agents extend the shape upward.** The most recent product move — background agents that can work on a task while the user does something else, returning a completed diff — is the workspace-tier version of Composer. The design choice worth stealing is that the background agent hands its artefact back to the user in the *same review UI* as a foreground Composer session. New autonomy, same trust contract.

**What's the applied engineer's read.** Cursor's product surface is the applied engineer's territory: the tier ladder, the diff review, the tool catalogue, the workspace scoping, the review UI for background work. The model choice, the retrieval quality over the workspace, and the tool-use loop's stability under long tasks are shared with the peer role.

**What's the peer role's read.** The reasoning-loop research (how many turns to run, how to plan a multi-file edit, when to stop) and the model-selection engineering (which model at which tier) are agentic-engineer work.

**Where you might disagree.** Cursor bets that the developer will read every diff. The moment developers start clicking "accept all" reflexively, the trust contract slips. In a design review for your own coding surface, this is the question to name: *are we shipping a co-pilot the user reviews, or an autonomous agent whose reviews are theatre?*

## GitHub Copilot Workspace

**What it is (as a user experience).** Copilot Workspace is GitHub's task-decomposition surface, launched as a GitHub Next technical preview. The user opens a workspace tied to a repo, files a natural-language task ("add pagination to the users API"), and the agent produces a **plan** (a numbered list of steps), an **implementation** (edits to specific files), and a **summary** (a PR body). The user reviews and edits *the plan* before the implementation runs, and reviews the implementation before it lands as a PR. GitHub has since extended the product line with Copilot's cloud / coding agent, a more autonomous issue-to-PR tier whose mainline docs live at [docs.github.com/copilot/concepts/agents/coding-agent](https://docs.github.com/copilot/concepts/agents/coding-agent/about-coding-agent); the design choices below are the ones the Workspace preview made explicit and that persist in the newer surface.

**Design choices worth naming.**

- **Planning is user-visible and user-editable.** This is the biggest divergence from Cursor Composer. The user sees the plan first, edits it, then triggers the implementation. That's a very explicit HITL affordance: the human steers the agent *before* it writes code, not just after.
- **The PR is the artefact.** The workspace's output lands on the user's repo as a real PR, with a real branch, a real diff, and a real body. The rest of the GitHub review surface (comments, checks, merge policy) takes over from there. Copilot Workspace does not invent a new review UX; it uses the one every GitHub user already knows.
- **The task list is a first-class object.** The workspace UI keeps a persistent task with its history — plan, implementation, iterations, comments. The developer can return to it, revise it, and re-run. Resume UX from [chapter 03](03-human-in-the-loop-and-steering-ux.md), rendered as a durable object rather than a chat.
- **Trust boundaries are explicit.** Copilot Workspace never merges the PR on the user's behalf; it never pushes to `main`; it stops at the review gate. The blast radius is bounded by version control by design.
- **Enterprise data controls sit above the surface.** GitHub Copilot Business and Copilot Enterprise expose admin controls (which repos, which developers, whether the model may see this file) that the applied engineer at the customer's company configures. The end-user surface is the same; the policy layer is different.

**What's the applied engineer's read.** The plan-first UI, the resumable task object, the PR-as-artefact choice, and the enterprise admin surface are all applied-engineer decisions. When you build your own workspace-tier agent, the *first* thing to decide is whether the plan is user-visible and user-editable — the answer changes everything downstream.

**What's the peer role's read.** The task-decomposition quality (does the plan actually decompose the task well?), the search-and-read loop over the repo, and the reasoning about which files to modify are the agentic-engineer's territory. GitHub's own research posts on Copilot Workspace lean heavily on these.

**Where you might disagree.** Plan-first UX has a real cost: the user has to read a plan. On tasks where the user knows exactly what they want and could have written the fix themselves in five minutes, the plan-review step is friction. Cursor's Composer bets that the fix + diff-review is faster; Workspace bets that the plan-review + fix + diff-review is safer. Both are shipping. The applied engineer's job is to know which bet fits their surface, not to pick a universal winner.

## Sierra CX agents

**What it is (as a product).** Sierra is a company that ships customer-experience agents to enterprises; its public customer page ([sierra.ai/customers](https://sierra.ai/customers)) lists deployments including SiriusXM, SoFi, Rocket Mortgage, Airtable, Chime, CarMax, Philippine Airlines, Virgin Media O2, GoFundMe, Paychex, and a growing catalogue of others. The product is not a single agent; it's a **platform** — Sierra provides the runtime, the customer builds an agent inside it for their brand, their tools, and their policies. The agents live on the customer's chat surface (web, in-app, sometimes voice), talk to the customer's users, and act on the customer's backend.

**Design choices worth naming.**

- **The platform ships a policy-first authoring model.** Instead of "write a prompt," Sierra's authoring surface is closer to "write a set of policies and skills." The policy layer is the primary artefact; the model consumes it. This is the shape [chapter 01](01-agent-shape-for-a-real-product-surface.md) named for the CX shape (Shape B) — customer + policy + tools + hand-off — rendered as a product primitive.
- **Sierra published τ-bench.** The company shipped the benchmark that measures the shape they're selling. The design signal from that is a strong claim of the sort [chapter 04](04-eval-suites-from-benchmark-shapes.md) endorses: *evaluate on the shape you ship*. The applied engineer's move — steal the shape from τ-bench, throw away the target — is a direct consequence.
- **Hand-off is a first-class product feature.** Sierra's public case studies emphasise the warm hand-off to a human rep: transcript, retrieved facts, tool-call history, and the reason the agent is handing off, all in one pane. The mod-105 chapter-03 rule ("the receiver sees the full context") is a Sierra product feature.
- **Voice is a peer surface, not a translation of chat.** Sierra ships voice as one of several channels on its Agent OS platform ([sierra.ai/product/channels](https://sierra.ai/product/channels)); the design decisions (interruption handling, silence handling, back-channel acknowledgements) are shape-specific. An applied engineer building a voice agent should not assume "same brain, different pipe" — the shape's UX contract is different.
- **Deployment is bounded by policy at the tenant level.** Sierra's customers configure the agent's scope (which actions it may take, which knowledge it may retrieve, when it must escalate). The runtime enforces the policy; the model does not.

**What's the applied engineer's read.** The policy-first authoring, the hand-off UI, the voice-vs-chat split, and the multi-tenant deployment model are the shape of a real CX product. When you build a CX agent for your own company, you are effectively re-implementing one row of Sierra's platform — do it consciously, or buy it.

**What's the peer role's read.** The tool-use loop under the CX shape, the response-generation research, the evaluation harness's design (τ-bench itself is a peer-role artefact) — these are the reasons Sierra hires deep AI research talent.

**Where you might disagree.** Sierra's bet is that CX is a *shape*, generalisable across many companies. That means the agent-authoring surface can be a product on its own. The counter-bet — that CX is domain-specific enough that each vertical needs its own agent-authoring product — is also a live bet in the market. The applied engineer's job at a company that ships CX is to know which bet applies to their vertical.

## Harvey

**What it is (as a product).** Harvey is a professional-user agent for legal work, sold to law firms and in-house legal teams rather than to individual lawyers. The surface spans drafting, review, research, and firm-specific workflows (contract analysis, litigation prep, transaction diligence, regulatory Q&A). Harvey partnered with OpenAI early and publicly claims deployments across several thousand legal organisations ([harvey.ai](https://www.harvey.ai/)), making it one of the most visible examples of a professional-user (Shape E) agent.

**Design choices worth naming.**

- **The user is a professional.** The UX assumes a lawyer at the surface, not a consumer. The agent's outputs are drafts (memos, clauses, case summaries) that a lawyer reviews and iterates. The trust contract is closer to "senior associate producing work for a partner to review" than to "AI answering questions."
- **Retrieval is the load-bearing capability.** Harvey's public writeups repeatedly emphasise the quality of the retrieval layer over legal corpora (case law, statutes, firm-specific documents). This is Shape C (retrieval-with-actions) at the top of its game — the retrieval quality is what makes the professional user willing to consume the drafts.
- **Firm-specific customisation is a first-class product.** Harvey ships not a single agent but an agent that ingests the *firm's own* documents, precedents, and templates. This is per-tenant memory (Layer 3 in [chapter 02](02-memory-system-with-retention-and-pii-policy.md)) taken as the product's core value.
- **The output is a document the user edits.** Harvey does not send drafts to clients; it does not file with courts; it does not enter contracts. Every act happens in the user's hands. The blast radius is bounded by the user's authority, not the agent's. That is a shape decision that keeps the risk profile manageable.
- **Evaluation is domain-specific and human-in-the-loop.** Legal quality is not a leetcode test. Harvey's publicly-described evaluation model leans on expert reviewers — former practicing lawyers Harvey calls Applied Legal Researchers — who build curated golden-query sets and grade model outputs via side-by-side comparisons ([Scaling AI Evaluation Through Expertise](https://www.harvey.ai/blog/scaling-ai-evaluation-through-expertise)). The applied engineer's chapter-04 discipline (steal the shape template; build an internal suite with domain-specific success criteria) is not optional here — it is the product.

**What's the applied engineer's read.** The professional-user UX, the tenant-per-firm memory model, the "draft the user edits" artefact, and the domain-expert evaluation model are all applied-engineer decisions. In any regulated vertical (legal, medical, financial, compliance), Harvey is the reference for how to keep the shape honest under real professional pressure.

**What's the peer role's read.** The retrieval research (multi-corpus reasoning, jurisdiction-aware retrieval), the model routing (Harvey has publicly described integrating OpenAI, Anthropic via AWS Bedrock, and Google via Vertex, with auto-routing per task — see [Expanding Harvey's Model Offerings](https://www.harvey.ai/blog/expanding-harveys-model-offerings)), and the evaluation harness at the model level are agentic-engineer work.

**Where you might disagree.** Harvey's bet is that the professional-user shape (Shape E) will be a large enough market on its own to sustain a company. The counter-bet — that consumer-facing legal tools (a Shape B agent for consumers seeking legal help) will eat that market from below — is a live conversation in the industry. As an applied engineer in a regulated vertical, you will have this conversation about your own shape.

## Cross-pattern moves worth stealing

Across the four patterns, six design moves recur enough to be worth stealing directly.

1. **The autonomy tier is a visible product choice.** Cursor makes it a tier picker; Copilot Workspace makes it a plan-first review; Sierra makes it a policy scope; Harvey makes it a professional-in-the-loop draft. In none of them does the surface try to "figure out" the tier silently.
2. **The artefact is one the user's org already knows how to review.** A diff for Cursor. A PR for Copilot Workspace. A chat transcript with a hand-off for Sierra. A document draft for Harvey. Do not invent a new review artefact; borrow the one the user already reviews.
3. **The tool catalogue is small, well-named, and legible.** All four surfaces expose a bounded catalogue with names that disclose the side effect (see [mod-104 chapter 01](../mod-104-tool-calling-and-orchestration/01-tool-schema-and-side-effect-boundaries.md)). None of them ship a `do_anything` mega-tool.
4. **The blast radius is bounded by something outside the model.** Version control for Cursor / Copilot. Policy + hand-off for Sierra. Professional authority for Harvey. The model's judgement is *not* the boundary; a system boundary is.
5. **Enterprise / tenant policy is a first-class layer.** Copilot Business / Enterprise, Sierra's tenant deployments, Harvey's firm-specific customisation, and Cursor's team plans all ship enterprise controls above the end-user surface. The applied engineer builds those controls, not the end-user surface, when they ship to a B2B market.
6. **Public evaluation is a signal, not a target.** Sierra ships τ-bench, but Sierra sells CX agents; the benchmark is a public artefact of the shape, not the product. Harvey publishes some evaluation work but sells lawyer-graded drafts. Cursor and GitHub publish benchmark numbers (SWE-bench, others) but sell developer productivity. The applied engineer takes the benchmark seriously as a signal and never as a headline number.

## Cross-pattern anti-patterns to avoid

Five failures visible in shipped surfaces that *tried* to look like the ones above.

1. **The autonomy dial with no tiers.** A slider from "conservative" to "aggressive" instead of explicit tiers. Users have no mental model for the slider; the surface has no way to reason about the tier at eval time.
2. **The novel review UI.** A custom "here's the AI's proposal, click to accept" surface that isn't the user org's normal review flow. The reviewer's discipline (comments, checklists, workflows) doesn't transfer, and the artefact is treated as a demo, not as work.
3. **The unbounded tool catalogue.** Twenty tools, five of them mega-tools, none of them named for their side effect. The model picks the wrong one; the user cannot tell which one it picked; the anti-pattern from mod-104 chapter 01 wins.
4. **The un-audited policy layer.** A "system prompt as policy" that no compliance team has seen, no auditor can verify, and no eval measures. When a customer asks how the agent knows what it's allowed to do, the answer is "in the prompt" and the enterprise sale dies.
5. **The benchmark-topping surface.** A product whose headline number is a public benchmark score. Every incident lands harder because the marketing has trained users to expect the benchmark's ceiling.

## Reading the split: applied engineer vs. agentic-engineer peer

The most important thing to notice across the four patterns is what they share on the applied side and what they share on the peer side.

**Shared applied moves.** All four patterns invest heavily in: shape choice, review artefact, tool catalogue, policy layer, tenant / enterprise controls, hand-off UX (where the shape needs it), and domain-specific evaluation. These are the applied engineer's territory. In none of them is any of these decisions delegated to the model.

**Shared peer moves.** All four patterns depend on: strong underlying models, careful model routing per task, tool-use loop stability, retrieval quality, planning quality, and public benchmark work. These are the agentic-engineer's territory. In none of them can the applied engineer succeed without the peer role's investment.

The consequence for your career choice is simple: **the applied engineer's job is to make a shape work in a product; the agentic-engineer's job is to make an engine work across shapes.** Both roles exist in every one of the four companies studied above. Both are hiring. Neither will replace the other in any of the shipped products — the pattern needs both.

## Summary

- Four public applied-agent patterns span the shape space and are worth studying: Cursor Composer (coding co-pilot, high tier), Copilot Workspace (task-decomposition coding surface), Sierra CX (customer-experience agent platform), Harvey (professional-user agent for legal work).
- Cursor Composer's design signal: visible autonomy tiers, diff-as-artefact, workspace-scoped context, background agents that hand back through the same review UI.
- Copilot Workspace's design signal: plan-first review, PR-as-artefact, resumable task objects, enterprise policy layer above the end-user surface.
- Sierra's design signal: policy-first authoring, first-class warm hand-off, shape-specific voice mode, the company shipping the benchmark (`τ-bench`) that measures its own shape.
- Harvey's design signal: professional-user UX, tenant-per-firm memory, "draft the user edits" as the artefact, evaluation grounded in domain-expert review.
- Six cross-pattern moves to steal: visible autonomy tier, familiar review artefact, small legible tool catalogue, boundary outside the model, enterprise / tenant policy, public benchmark as signal.
- Five anti-patterns to avoid: the unbounded autonomy slider, the novel review UI, the mega-tool catalogue, the un-audited policy layer, the benchmark-topping headline.
- The applied engineer owns shape, review, tools, policy, tenant controls, and domain-specific eval. The agentic-engineer peer owns the models, the reasoning loop, retrieval / planning research, and public benchmarks. Both roles ship every one of these four products.

That closes the module. The exercises turn the five chapters into worked artefacts: a shape decision on a real surface, a memory system with the retention contract, a HITL / kill-switch UX, an internal eval suite adapted from benchmark shapes, and a design audit of one of the four public patterns above. See the module [README](README.md) for the exercise list and the [resources](resources.md) file for citations.
