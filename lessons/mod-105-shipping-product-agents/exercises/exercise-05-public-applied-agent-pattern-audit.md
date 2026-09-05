# exercise-05: Public Applied Agent Pattern Audit

**Estimated effort:** 3 hours

## Objective

Produce a design audit of one (or two) of the four public applied-agent patterns from [chapter 05](../05-public-applied-agent-patterns.md) — Cursor Composer, GitHub Copilot Workspace, Sierra CX, Harvey — grounded in first-hand product observation (or the closest thing you can get to it: current docs, live demos, published case studies, engineering blog posts, screencasts). The output is a decision-grade document that names each shipped design choice, categorises it as applied-engineer or agentic-engineer territory, extracts what generalises, and rejects what does not — and then re-applies the same lens to the surface you have been working on across the previous four exercises.

The purpose is not to admire other people's products; it is to build the muscle of *reading a shipped product agent as a set of decisions*. Every applied-AI engineer will be asked in a design review "how does Cursor / Copilot / Sierra / Harvey handle this?" — and the useful answer is a design signal with a caveat, not a marketing quote.

## Learning outcomes

By the end you should be able to:

- Read a shipped product-agent surface and enumerate its user-observable design choices without conflating them with the marketing narrative.
- Separate applied-AI-engineer decisions (surface, review artefact, tools, policy, tenancy, HITL, eval) from agentic-AI-engineer decisions (models, planner, retrieval research, benchmark work) for a specific shipped product.
- Steal design moves that generalise (visible autonomy tiers, familiar review artefacts, bounded tool catalogues, external blast-radius boundaries) and reject moves that do not.
- Argue in a design review from a *live* reference ("here is how Copilot Workspace handles this in the current UI; here is where our surface should diverge and why") rather than a hearsay one.
- Update your own product-agent design memo from [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md) with the lessons the audit surfaces.

## Prerequisites

- You have read [chapter 05](../05-public-applied-agent-patterns.md) and the module [README](../README.md).
- You have completed at least [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md); ideally you have artefacts from exercises 02–04 to compare against.
- You have working access to the current live product docs for the pattern(s) you pick. For Cursor and Copilot Workspace, you can install the client; for Sierra and Harvey, you are relying on public docs, case studies, video walkthroughs, and their engineering blogs.
- You have a screenshot / recording tool for the surfaces you can drive yourself. You are going to reference specific screens, not paraphrases of screens.

## Problem statement

You are the applied-AI engineer preparing for the second-round design review of the surface you scoped in [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md). Leadership has asked, in the review kickoff, some version of the question: *"how does Cursor / Copilot / Sierra / Harvey do this?"* You have two options:

1. Show up with an anecdotal answer, get argued with, and leave the review with new homework.
2. Show up with a written audit of the relevant public pattern, referenced to current live docs, that separates the design signal from the marketing gloss and cleanly maps the signal onto your surface's shape.

This exercise is option 2. You will pick **one** primary pattern to audit deeply and (optionally, as a stretch goal) a second one to compare against.

Pick the primary based on your surface's shape (as decided in exercise-01):

- **Shape A (coding co-pilot)** → Cursor Composer *or* GitHub Copilot Workspace as primary. The two make instructively different bets; the stretch goal is to also audit the other.
- **Shape B (CX agent)** → Sierra CX as primary.
- **Shape C (retrieval-with-actions)** → Harvey as primary; Sierra as an optional compare.
- **Shape D (browsing agent)** → No perfect match in the four patterns. Audit whichever browsing surface is *actually shipping to real users* today (OpenAI's Operator, Anthropic's computer-use launches, or an equivalent) using the same lens; treat Sierra as a comparative on hand-off UX.
- **Shape E (sales / support / ops assistant)** → Harvey as primary; if your surface is closer to engineering-internal, audit Cursor / Copilot Workspace on their internal-team-tier deployments.

Your job is to read the primary pattern, name the design choices, categorise them, extract what applies, and produce an updated memo for your own surface. The audit output should be strong enough that a colleague who did not read chapter 05 could still act on it.

## Requirements

### Deliverable 1 — The primary audit (`primary-audit.md`)

A structured document, roughly 1,200–2,000 words, covering the primary pattern you chose. Structure it as follows.

#### 1. Surface at a glance

Two to four paragraphs. Name the product, its current user base as best you can determine from public sources, the surface's entry point in the UI (or CLI, or IDE, or chat channel), and the artefact the user reviews at the end of a turn. If the product has multiple tiers or modes, name them and pick which tier you are auditing — do not blur across tiers.

Include at least one **primary source citation** for every non-obvious claim: a live docs page, an engineering blog post, a recorded product demo, or a published case study. Marketing landing pages are secondary references, not primary. If a claim depends on a paywalled or off-record source, mark the claim with `<!-- needs-research: ... -->` rather than guessing.

#### 2. Design choices enumerated

The core of the audit. Enumerate at least **eight** distinct design choices the product makes at the applied-engineer altitude. For each choice, one paragraph covering:

- **What the surface does.** Concretely, with reference to a screen / doc / demo. "Cursor's Composer opens a diff-preview overlay before applying multi-file edits" beats "Cursor reviews changes before applying."
- **Which affordance from mod-105 it maps to.** Shape, memory layer, one of the six steering affordances (progress / cancel / undo / resume / hand-off / kill-switch), confirmation UX, eval, tenancy / policy, tool catalogue, or an anti-pattern avoided.
- **Applied vs. agentic.** Whose decision is this in the vocabulary of chapter 05? A visible tier picker is applied. The planner that decides "what to do at tier N" is agentic. If it's genuinely shared, say so and explain where the seam lives.
- **What generalises.** Would you steal this for your own surface if the shape matches? Under what condition would you not?

Aim for the choices to span the affordance space; a good audit has a shape choice, a memory choice, at least two steering choices, a tool-catalogue choice, a tenancy / policy choice, and an eval or feedback choice.

#### 3. The three most instructive divergences

For each of the following, name the choice the product made and the choice it *could have made and did not*:

- A shape or autonomy-tier choice.
- A blast-radius / confirmation choice.
- A review-artefact or hand-off choice.

For each, one paragraph on *why* the product plausibly made the choice it did (the trade-off) and one sentence on what evidence would move you toward the alternative.

#### 4. What the audit tells you *not* to steal

At least three design moves in the product that you would not carry into your own surface. Be specific and be honest. Common shapes here: choices that are only workable at the audited company's scale, choices that reflect a legacy constraint your surface does not have, choices that only work under a specific tenancy / distribution model, choices that would drift into shape-fusion in a surface that does not need the fusion.

If you find yourself with fewer than three, you have not audited hard enough; every shipped surface has decisions that are load-bearing for it and wrong for someone else.

#### 5. The applied-vs-agentic split for this product

Two lists, side by side:

- **Applied engineer's decisions** (as visible from outside): 6–10 items. Shape, review artefact, tool catalogue, confirmation UX, tenancy model, eval discipline, hand-off UX, and so on.
- **Agentic engineer's decisions** (as visible from outside): 4–8 items. Model routing, reasoning-loop stability, retrieval quality, planner architecture, benchmark work, model-side research.

The lists are your empirical answer to the "which role owns what" question from chapter 05. If you cannot fill the applied list to at least six items from public sources, you did not read the product carefully enough.

#### 6. Citations

A footnotes-style section listing every source referenced above with an accessed-on date. This is the section that keeps the audit useful six months from now, when half the product's UI has moved.

### Deliverable 2 — Product-agent audit rubric (`rubric.md`)

Distil the audit into a **reusable rubric** — a checklist a colleague could take to any other product-agent surface and apply. The rubric should have 10–15 questions, grouped roughly by:

- Shape and autonomy (what tier, is it user-picked, what's the trust contract).
- Review artefact (what is it, is it native to the user's org, is it reviewable).
- Steering (which of the six affordances are visible from outside; which are hidden or absent).
- Tool catalogue (bounded? legible from names? undoable?).
- Confirmation and blast radius (what triggers a modal, what doesn't, what's the boundary the model is not the boundary of).
- Tenancy and policy (single-tenant, multi-tenant, admin controls).
- Feedback and eval signal (what feedback is visible; what signal is likely gating releases).

The rubric is a tool you can bring to any future review. It should read like the [chapter 03](../03-human-in-the-loop-and-steering-ux.md) "steering scorecard" — questions with concrete answers, no vagueness.

### Deliverable 3 — Update to your own surface's memo (`memo-update.md`)

Return to the `shape-decision-memo.md` you wrote in [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md). Produce a *diff* (either as annotated markdown or as a real `git diff` file) that:

- Adds one design move stolen from the audited pattern, with a citation.
- Rejects one design move you had been considering because the audit showed the alternative works.
- Names one open question the audit raised that your original memo did not answer, and either resolves it or flags it for the second review.

Total updates should be modest — the audit's job is not to rewrite the memo, but to make it defensible against the "how does X do this?" question.

### Deliverable 4 — The 90-second design-review answer (`elevator.md`)

Under 250 words. The written form of the answer you would give in the design review when leadership asks "how does Cursor / Copilot / Sierra / Harvey do this?" Structure it as: *what they do* (one sentence), *the design choice underneath it* (one sentence), *what we should copy* (one sentence), *what we should not copy and why* (one sentence), *what our surface should do differently and why* (one sentence). This is the answer you rehearse before walking into the review.

## Starter guidance

- **Drive the product where you can.** For Cursor and Copilot Workspace, install and use the surface for at least one real task — a small refactor, a documentation update, a debug. Note screens, tier names, and modal texts as you go. Second-hand claims degrade fast; first-hand ones do not.
- **Sierra and Harvey are audited from public artefacts.** Read the current docs pages, the engineering-blog case studies, and the founders' / eng leaders' talks. Watch product demos in full, not the marketing supercut. The τ-bench paper is a Sierra source; Harvey's engineering blog has published on model routing.
- **Screenshots or it didn't ship.** Every claim in section 2 that describes a UI ought to reference a specific screen (a screenshot in your deliverable, or a link to a public demo timestamp). Words like "the surface makes it easy to…" are marketing; screens are audits.
- **Products change monthly.** Date every claim. When a UI element you cite has been renamed or removed by review time, an accessed-on date protects the audit's credibility.
- **Do not reverse-engineer the internal architecture.** The applied-engineer audit is about what a user sees and can act on, not about what the model is doing under the hood. If you find yourself speculating about the planner, you are drifting into the agentic-engineer's territory.
- **Do not conflate the four products.** Cursor's Composer is not Copilot Workspace with different branding. Sierra is not "Harvey for CX." Each shape has its own contract; the audit stays honest by naming the differences.
- **Do not use benchmark scores as design signal.** A high SWE-bench Verified score for Cursor does not tell you Cursor's Composer UX is right for you. Chapter 04's rule ("public benchmark is signal, not target") applies at the audit layer too.
- **Do not guess at metrics.** If you do not have a citable number, do not quote a number. `<!-- needs-research -->` is a legitimate output; a fabricated "75% acceptance rate" is not.

## Acceptance criteria

- [ ] The primary audit covers one of the four chapter-05 patterns (or a defensible substitute for Shape D) and states which tier of the product is being audited.
- [ ] At least eight design choices are enumerated, each with a mod-105 affordance mapping and an applied-vs-agentic categorisation.
- [ ] The three-divergences section names each choice, its plausible trade-off, and the evidence that would flip it.
- [ ] At least three design moves are named as *not* worth stealing, with concrete reasons tied to your surface's shape.
- [ ] The applied and agentic decision lists each have at least six and four items respectively.
- [ ] Every non-obvious claim has a primary-source citation with an accessed-on date, or a `<!-- needs-research: … -->` tag.
- [ ] The reusable rubric has 10–15 questions and is legibly grouped by category.
- [ ] The memo update names one design move stolen, one rejected, and one open question the audit surfaced.
- [ ] The 90-second elevator answer is under 250 words and follows the five-sentence structure.
- [ ] No section quotes a benchmark score as a product-quality signal.
- [ ] No screen or feature is described from imagination; every UI claim is anchored to a live source.

## Stretch goals

Pick any two:

- **Cross-pattern comparison.** Audit a second pattern with the same rubric and produce a `comparison.md` that lines the two up side-by-side on the same 10–15 questions. The comparative is where cross-pattern moves (the six from chapter 05) become visible in your own hand.
- **The anti-pattern hunt.** For each of the five cross-pattern anti-patterns in chapter 05 (autonomy dial with no tiers, novel review UI, unbounded tool catalogue, un-audited policy layer, benchmark-topping headline), find and document one surface (public or historical) that shipped it. This is the "learn from other people's post-mortems" library the field is small enough to need.
- **A product tour recording.** Record a 5–10 minute screencast walking a colleague through the audited surface, pausing at each of the eight design choices. Video is what onboards the rest of the team; the audit doc is the reference.
- **The interview version.** Turn the audit into an interview-ready answer for "walk me through how you would build an agent like Cursor / Copilot / Sierra / Harvey." Two pages, structured as: my shape choice, my top three design choices, the two decisions I would flip on their surface, and the eval discipline I would inherit. Interview signal for the role this track prepares you for.
- **A field trip.** Attend a public product demo, developer conference talk, or user-group session for the audited product and update the audit with quotes and observations from the session. Every product-agent design review benefits from someone who has *heard the team talk about it*, not just read the docs.
- **A "next release" prediction.** From the audit, name one design gap you can see and one design move you predict the team will ship in the next 6 months. When you are right (or wrong) you learn to read shipped products more precisely.
- **The competitor's audit.** Pick a smaller / newer product that positions itself as "our version of the audited pattern." Audit it against the same rubric. The gap between the reference pattern and the competitor is the applied engineer's playbook rendered as market difference.

## Deliverable

Submit as `mod-105/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `primary-audit.md` (the ~1,200–2,000 word audit).
- `rubric.md` (the reusable design-review rubric).
- `memo-update.md` (the diff against your exercise-01 memo).
- `elevator.md` (the 90-second design-review answer).
- A short `README.md` naming the pattern audited, the shape of your own surface, and the primary sources you relied on.
- Any stretch-goal artefacts named alongside (`comparison.md`, screencast link, interview-ready `interview.md`, etc.).
