# exercise-01: Agent Shape Decision for a Real Product Surface

**Estimated effort:** 3 hours

## Objective

Turn the shape decision from [chapter 01](../01-agent-shape-for-a-real-product-surface.md) into a written case for one *specific* product surface — one you either work on, are pitching, or are asked about in an interview. The output is a decision memo that a PM, an engineering manager, and a security reviewer can all agree with. The exercise is deliberately about writing and defending a decision, not shipping code — a bad shape choice made confidently is one of the more expensive product-agent failures, and the drill is to make the *right* choice legibly.

## Learning outcomes

By the end you should be able to:

- Name a real product surface and place it in one of the five shapes (or defend a fused-shape composition) with reasons a non-technical reader can follow.
- Walk the six-question decision procedure from chapter 01 without collapsing into "it depends."
- Enumerate the downstream demands the chosen shape places on memory, HITL, eval, cost, and rollout.
- Separate the applied-AI engineer's remit for this surface from the agentic-AI engineer peer role's remit.
- Push back on a wrong-shape ask in a design review with a written alternative, not just "no."

## Prerequisites

- You have read [chapter 01](../01-agent-shape-for-a-real-product-surface.md) and skimmed the module [README](../README.md).
- You have picked a *real* surface. Options, in order of preference:
  - A product-agent surface at your current company (even if it's a proposal, not a shipped feature).
  - A surface you would build if you joined a specific target company (name the company; use their public product surface).
  - A hypothetical surface if neither of the above applies. If you go hypothetical, pick something you can defend from users' perspective — no "an AI assistant for everyone."
- You do **not** need to have written a line of code for the surface yet. This exercise is upstream of code.

## Problem statement

You are the applied-AI engineer being asked to ship an agent surface. The ask lands as one of the following flavours (pick one to make the exercise concrete, or invent an equally specific ask you've actually heard):

- *"Add an AI assistant to our product."* Product leadership wants an entry to the market. There is no concrete shape yet; the ask arrives as a blank cheque, which is the most dangerous flavour.
- *"Our competitor shipped X — we need our version by Q3."* The shape is being implicitly copied from someone else's surface. Your job is to decide whether the copy makes sense for your users.
- *"Users are asking us to automate Y."* A specific user-side problem is on the table. Your job is to decide whether an agent (and which shape) is the right solution, or whether a simpler feature is.
- *"We already have an agent, but it's not landing."* The surface exists but the shape is wrong. Your job is to diagnose the shape mismatch and propose a re-shape.

In all four flavours, the deliverable is the same: **a decision memo that names the shape, defends it, enumerates the downstream cost, and sets up the next design review.** The audience is a PM, an EM, and a security reviewer. All three must be able to read it and agree.

## Requirements

### Deliverable 1 — The shape decision memo

A single markdown file, `shape-decision-memo.md`, structured as follows.

#### Section 1: The surface, in one paragraph

Name the product, the target user, the entry point (where the agent lives in the UI), and what job the surface is doing. Two sentences on the *user's* job before the agent existed; two sentences on what the agent does to that job. If you cannot state the user's job before the agent, the ask is not ready to ship.

#### Section 2: The six-question walkthrough

Answer each of the six questions from chapter 01 explicitly. One paragraph per question. The answers must be specific — "developers" is not a Q1 answer; "our SRE team on the payments platform" is. If two answers pull in different directions, name the tension.

#### Section 3: The shape verdict

State the shape (A, B, C, D, or E, or a fused shape stated as two shapes with two contracts). One paragraph on why this shape wins the six-question test. One paragraph on the *specific alternate shape you considered and rejected*, with the reason for the rejection. If you did not consider an alternate, you did not do the exercise.

#### Section 4: Downstream demands

For each of the following, one short paragraph on what the chosen shape demands:

- **Memory** (session boundary, per-user layer, workspace layer, retention window, deletion contract) — the questions from [chapter 02](../02-memory-system-with-retention-and-pii-policy.md).
- **Steering UX** (progress, cancel, undo, resume, hand-off, kill-switch) — from [chapter 03](../03-human-in-the-loop-and-steering-ux.md). Which of the six affordances are load-bearing for *this* shape?
- **Eval** (which public benchmark shape templates apply; what would go in the internal suite) — from [chapter 04](../04-eval-suites-from-benchmark-shapes.md).
- **Cost model** (per-turn LLM cost, tool-call cost, retrieval cost, human-in-the-loop cost). Order-of-magnitude estimates are fine; be honest about which cost dominates.
- **Rollout** (feature flag posture, canary shape, kill-switch scopes, what your first 1% of traffic looks like).

#### Section 5: The applied-vs-agentic split for this surface

One paragraph naming what *you* (the applied engineer) own for this shape and what the agentic-AI engineer peer role owns. Be specific: name the retrieval quality, the planner architecture, the model routing, the reasoning-loop stability. If you cannot see the split, walk chapter 01's applied-vs-agentic section again and try naming three concrete artefacts on each side.

#### Section 6: What could kill this in review

Three bullets: the three arguments a skeptic could raise that would kill the surface in the design review. For each, one sentence on how you would respond and one sentence on what evidence you would bring. This section is graded on honesty; if your three bullets are all straw-men, the exercise failed.

### Deliverable 2 — The one-slide pitch

An additional file, `one-slide.md`, containing exactly what would fit on one slide in an all-hands: the shape, the shape's headline design constraint, the top three risks, and the single metric that would tell you the surface is working. Keep it under 200 words. Real product decisions get sold in one slide before they get sold in a memo.

### Deliverable 3 — The "wrong shape" counter-memo

Write, in a separate section (or file `counter-memo.md`), a one-page argument for the *other* shape you rejected in Section 3, from the perspective of someone who genuinely disagrees with your decision. This is not a straw-man. It is a real argument — the strongest version of the counter-position. The applied engineer who cannot argue the other side confidently is the applied engineer who loses the design review to a strongly-argued wrong shape.

## Starter guidance

- **Do not pick "an AI assistant for our product."** If the ask arrives that way, the first move is to sharpen the ask. Pick a specific job the assistant is helping with, and pick the shape for that job.
- **Do the six-question walk in one sitting.** If you break to research between Q3 and Q4 you will drift. Do them in an hour; refine after.
- **Read the corresponding case study.** If you land on Shape A, read [chapter 05](../05-public-applied-agent-patterns.md)'s Cursor / Copilot Workspace sections. On Shape B, read the Sierra section. On Shape C or E, read Harvey. The lens for arguments-in-review is often "how would Product X have argued this?"
- **Write for the security reviewer.** The rest of the audience will forgive vagueness; the security reviewer will not. If a shape decision has blast-radius implications, name them.
- **Do not defend a shape by pointing at a benchmark score.** The chapter 04 rule applies at the shape-decision layer too. A model's public benchmark performance is not a shape signal.
- **If you find yourself writing "the model can figure it out" anywhere in the memo, delete the sentence.** The applied engineer's memo is about the surface, not the model's capacities.

## Acceptance criteria

- [ ] The surface is specific, named, and its user's pre-agent job is stated in the first paragraph.
- [ ] Every one of the six chapter-01 questions is answered with a concrete, non-hedged sentence.
- [ ] The shape verdict names one of A / B / C / D / E, or a fused shape stated as two shapes with two contracts, and includes an alternate shape considered-and-rejected.
- [ ] The downstream-demands section touches memory, steering UX, eval, cost, and rollout, each with one concrete demand — not a generic checklist.
- [ ] The applied-vs-agentic split names three concrete artefacts on each side.
- [ ] The "what could kill this" section has three honest bullets, each with a response and a piece of evidence.
- [ ] The one-slide pitch is under 200 words and contains a single metric.
- [ ] The counter-memo argues the rejected shape from its strongest position, not as a straw-man.
- [ ] No section defends a shape by citing a public benchmark score.
- [ ] The memo does not contain the phrase "AI assistant" as a shape.

## Stretch goals

Pick any two:

- **Peer review it.** Send the memo to a colleague (or a fellow learner) and ask them to argue against it. Rewrite the memo to address their strongest objection.
- **Draw the first UI screen.** A rough wireframe of the surface's primary interaction. The shape decision should be visible in the wireframe — a Shape A workspace surface should look nothing like a Shape B chat surface. If they look alike, one of them is wrong.
- **Enumerate the first 10 real tasks the eval suite would carry.** Chapter 04 shape template applied to your surface. Names, initial state, tool catalogue, success criterion for each. You will use this in [exercise-04](exercise-04-product-agent-eval-suite-derived-from-benchmark-shapes.md).
- **Interview a real user.** If the surface is at your company, talk to a real user of the pre-agent workflow. Ask them what they wish they could delegate. Update Section 1 of the memo with a quote and a citation.
- **Write the "why we killed this" post.** A hypothetical retro, six months from now, in which the surface was shut down. What went wrong? Which of the design decisions in the memo were the cause? This is the honest cousin of Section 6.

## Deliverable

Submit as `mod-105/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `shape-decision-memo.md` (Sections 1–6).
- `one-slide.md` (the pitch).
- `counter-memo.md` (the argued alternative).
- Any stretch artefacts named above.
- A short `README.md` linking the files together and naming the target surface / company.
