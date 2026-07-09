# exercise-04: ISO/IEC 25059 Quality Dimensions Audit

**Estimated effort:** 2 hours

## Objective

Take the outputs of exercises 01–03 for one spec — the feature-shape decision, the SLO table, and the provider trade-off note — and audit them against the ISO/IEC 25059 quality-dimensions checklist from [chapter 04](../04-one-page-feature-architecture-doc.md). Produce a completed one-page feature-architecture doc plus a written **audit** of which dimensions are load-bearing, which are defensibly N/A, and which are gaps that block sign-off.

## Learning outcomes

By the end you should be able to:

- Walk the ISO/IEC 25010 classical dimensions and the ISO/IEC 25059 AI-specific dimensions in order and produce a cell-per-dimension note for a real feature.
- Distinguish a defensible "N/A" (dimension does not apply, here's why) from an undefensible one (dimension applies but hasn't been thought about).
- Produce a governance-review-shaped audit that names the gaps and states what would close each gap.
- Pack the full feature-architecture doc into a genuinely-one-page artefact that a PM, an ML engineer, and a governance owner can each sign off on.

## Prerequisites

- You've read [chapter 04](../04-one-page-feature-architecture-doc.md) and [chapter 05](../05-leaderboard-driven-failure-modes.md).
- You've completed exercises 01, 02, and 03 for the same spec, or you're using the worked warranty-Q&A example from chapter 04.

## Problem statement

You are hosting a sign-off review in three days. The governance owner has asked for the ISO/IEC 25059 checklist to be visible in the doc. Before circulating, you audit your own work: does every dimension have a defensible note? Which ones are gaps that would fail the review? What would close them in the three days you have?

## Requirements

Produce two deliverables.

### Deliverable 1 — One-page feature-architecture doc

Following the template in [chapter 04](../04-one-page-feature-architecture-doc.md). Sections 1 through 8, all filled. Section 7 (ISO/IEC 25059 checklist) has a one-cell note for every dimension.

**Hard constraint**: the entire doc must fit on one page of rendered markdown at a normal reader's zoom. If it doesn't, keep compressing until it does. This is the exercise — the discipline of the constraint is the point.

Use the 25010 + 25059 dimension list from [chapter 04](../04-one-page-feature-architecture-doc.md):

- **25010 base**: functional adequacy, performance efficiency, compatibility, interaction capability, reliability, security, maintainability, flexibility, safety.
- **25059 AI-specific**: functional adaptability, user controllability, transparency, robustness, intervenability.

<!-- needs-research: verify the exact 25059:2023 dimension list against the standard document before final sign-off; the dimensions above draw on the 25010 base plus the AI-specific extensions named in the 25059 abstract. -->

Each cell is one to three sentences. If a dimension does not apply to your feature, mark the cell "N/A — <reason>" instead of leaving it blank.

### Deliverable 2 — Written audit (one page)

A separate markdown document that walks each of the 25059 dimensions and, for each, states:

- **Applies?** (yes / partial / N/A with reason)
- **Evidence** — what artefact in your doc (or in an adjacent doc — SLO table, trade-off note, safety policy) satisfies this dimension?
- **Load-bearing?** — is this dimension one that a real user or reviewer would care about if it broke, or is it a checkbox for compliance? (This is not a signal to skip the ones you mark "compliance checkbox" — it's a signal to keep them one-line.)
- **Gap?** — if the cell is thin, what would you add to close the gap? Estimated effort.
- **Blocks sign-off?** — would this specific gap cause the governance owner to withhold signature? Y / N with reason.

The audit is *your* review of your own work before circulation, in the voice of a reviewer.

## Starter guidance

- Start with the AI-specific dimensions (functional adaptability, user controllability, transparency, robustness, intervenability). These are where LLM features are strongest *or* where they fail loudest, and they are typically the freshest thinking for you. Get them right and the classical dimensions fall out easily.
- **Transparency** is the dimension applied-AI engineers most often under-serve. The bar is not just "we say 'AI generated' on the surface" — it is that a user, an auditor, and an on-call engineer can each reconstruct the model version, the prompt hash, the retrieved context, and the sub-processors that touched the payload for a given response.
- **User controllability** collapses to accept / edit / reject for co-pilot shapes; expands to progress / pause / resume / cancel / undo for autonomous-agent shapes; and includes per-tenant kill-switches for any surface.
- **Intervenability** is not the same as user controllability. Intervenability is the operator's lever — hot-rollback of a prompt, per-tenant kill-switch, feature-flag gating, on-call runbook. User controllability is the user's lever.
- **Robustness** covers behaviour under adversarial or distribution-shifted input (prompt injection, unusual document layouts, unexpected languages, image content with embedded instructions). Refer to your safety envelope from Section 6 and mod-107 for the shape.
- If a dimension is genuinely N/A, say so and give the reason in the cell. "N/A — this surface has no autonomous actions" is defensible; a blank cell is not.
- The one-page constraint forces prioritisation. If Section 3 (SLO table) is 12 rows and Section 5 (data flow) is 8 lines, the ISO cell notes will be very short. That is correct.

## Acceptance criteria

- [ ] The one-pager fits on a single rendered page.
- [ ] Every ISO/IEC 25010 + 25059 dimension has either a written note or a defensible "N/A" reason.
- [ ] The written audit walks every 25059 AI-specific dimension.
- [ ] Each audit entry states Applies / Evidence / Load-bearing / Gap / Blocks-sign-off.
- [ ] At least one dimension is flagged as a gap with a specific gap-closing action and effort estimate.
- [ ] The one-pager references the SLO table (exercise-02) and the trade-off note (exercise-03) by link rather than by embed.
- [ ] The safety envelope (Section 6) references a written policy version.
- [ ] The kill criteria (Section 8) include at least one quality-based trigger and at least one cost-based trigger (see [chapter 05](../05-leaderboard-driven-failure-modes.md), unmeasured cost failure mode).
- [ ] A revisit date is set, no later than 90 days from the sign-off date.

## Stretch goals

Pick any two:

- **Governance dry run.** Recruit a peer — ideally from a different function than yours — and ask them to review the one-pager for 15 minutes. Capture what they asked about, what they wanted expanded, and what they would sign. Iterate the doc once.
- **NIST AI RMF crosswalk.** Map the 25059 cells onto NIST AI RMF's Govern / Map / Measure / Manage functions. Which RMF outcomes are covered by which cells? Where are the RMF gaps?
- **EU AI Act crosswalk.** If your feature would touch EU users, map the doc against EU AI Act obligations relevant to your risk classification (transparency notices, human oversight, documentation). This is a rough sketch, not a legal opinion.
- **Model-card slice.** Produce a first draft of a per-feature model-card slice (mod-112 goes deep here) from the one-pager plus the trade-off note. What is redundant, what is missing, and what shape would the slice take?
- **Anti-pattern review.** Walk your own one-pager against the "what a bad one-pager looks like" list at the end of [chapter 04](../04-one-page-feature-architecture-doc.md). For each anti-pattern, say whether your doc has it (be honest) and how you'd fix it if so.

## Deliverable

Two markdown files:

- `one-pager.md` — the one-page feature-architecture doc.
- `audit.md` — the written audit.

Submit to the paired solutions repo (`applied-ai-engineer-solutions`) under `mod-101/exercise-04/`.
