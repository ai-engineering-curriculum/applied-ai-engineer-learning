# exercise-01: Feature Charter With Kill Criteria

**Estimated effort:** 2 hours

## Objective

Author a one-page **feature charter** for a real (or realistic) applied-AI feature you are building, considering, or maintaining. Deliver the six charter fields — user problem, success metric, non-goals, safety scope, cost budget, kill criteria — in a form that is signed by all three reviewer roles (PM, engineer, governance owner). The deliverable is the *artefact*, not the feature; the exercise is about proving you can turn a roadmap wish into a governance-review-ready decision to build.

## Learning outcomes

By the end you should be able to:

- Distinguish a **user problem** from a *solution shape* and write one that names the user, task, current cost, and what "solved" looks like *for the user*.
- Choose a **single success metric** that is user-facing, attributable, weekly-computable, and bounded — not adoption, not thumbs-up.
- Write **non-goals** that are concrete behaviours the feature will not attempt, not truisms.
- Split **safety scope** into in-scope harms (defended in-line), handed-off harms (escalated to a peer role), and out-of-scope harms (with rationale).
- Turn a **cost ceiling** into a monthly-cost budget and a CPSO target that reference a specific pricing baseline.
- Author **kill criteria** as pre-registered thresholds — a number, a window, a source — that a 3 AM on-call can fire without negotiation.
- Route the charter through a PM sign-off, an engineer sign-off, and a governance-owner sign-off — and record the sign-off in the artefact itself.

## Prerequisites

- You have read [chapter 01](../01-feature-charter-with-kill-criteria.md).
- Familiarity with [mod-101 chapter 04](../../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md) (the one-pager), [mod-109 chapter 04](../../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) (CPSO), and [mod-107 chapter 05](../../mod-107-guardrails-and-safety-on-the-surface/05-refusal-calibration-and-owasp-mapping.md) (safety policy) — the charter cites them.
- A feature idea you can write about honestly. If you don't have one, pick a public analogue (an IDE assistant, a support-agent copilot, a doc-QA feature, a sales-email drafter) and write the charter *as if* you were shipping it.
- A colleague or mentor willing to play PM, another to play governance owner, if you can arrange it. If not, self-review with the charter's field-by-field criteria as a stand-in.

## Problem statement

Your PM slid a roadmap line onto your plate: **"Add an AI assistant to <surface X>."** The kickoff is next week. If you show up with just a design doc, the review will end with more questions than sign-offs, the safety envelope will be renegotiated at the first incident, and the flag will ship without pre-registered kill criteria.

Fix that. Before the kickoff, produce a signed charter that pins the six fields with enough precision that the review converges. The signed charter is the artefact you bring to the kickoff, not the design doc.

## Requirements

### Deliverable 1 — The charter document

Under `docs/features/<feature-name>/charter-v0.1.md`:

A single Markdown file with a header block, six sections, and a signature block, in the shape from [chapter 01](../01-feature-charter-with-kill-criteria.md). Fits on one printed page (~500-800 words). Every section is populated; no `TBD` placeholders.

Header block includes: feature name, owner, PM, governance reviewer, sign-off date, next-review date (a specific calendar date, at most 90 days out).

The six sections:

1. **User problem** — one paragraph that names *the user role*, *the task*, *the current cost of not shipping* (with a number, even if approximated), and *what "solved" looks like for the user*.
2. **Success metric** — one user-facing, attributable, weekly-computable, bounded number. Includes: definition, evaluator, target, and window. Optionally: a guardrail metric ("must not degrade") — but only one primary.
3. **Non-goals (v0)** — six specific behaviours the feature will *not* attempt. Each one line. No truisms.
4. **Safety scope (v0)** — three lists: in-scope (with the guardrail from [mod-107](../../mod-107-guardrails-and-safety-on-the-surface/README.md) that defends against each), handed-off (with the peer role and the delegation reference), out-of-scope (with a rationale per item).
5. **Cost budget (v0)** — the monthly cost ceiling in currency ("$X k/month"), the CPSO target ("$Y per successful outcome"), the assumed pricing baseline (model IDs with dates + the pricing date), and the assumed traffic model in one sentence.
6. **Kill criteria (v0)** — five to seven bullets. Each bullet: the metric, the threshold, the window, the source (a mod-1xx substrate), and the action ("pause rollout / flip flag off / route to fallback").

### Deliverable 2 — The signature ritual

Sign-off happens in-artefact, not out of band. In the signature block at the bottom of the charter:

- **PM sign-off** on Section 2 (success metric), Section 3 (non-goals), and the top-line target in Section 5 (cost budget).
- **Engineer sign-off** on Section 5 (cost budget mechanics) and Section 6 (kill criteria — every kill criterion is measurable from a substrate the engineer owns).
- **Governance owner sign-off** on Section 4 (safety scope) and Section 6 (kill criteria — every safety-related criterion is pre-registered).

Each signature is a line with a name, a role, and a date. If you have colleagues willing to play the roles, real signatures. If not, self-review from each role's perspective and *write down what you would change if you were them*. Attach the self-review notes as `docs/features/<feature-name>/charter-review-notes.md`.

### Deliverable 3 — The charter-vs-anti-pattern check

At the bottom of the charter (or in a companion file), a short section titled **"Anti-pattern check"** that walks through each of the anti-patterns listed at the end of [chapter 01](../01-feature-charter-with-kill-criteria.md) and asserts that this charter does not exhibit it:

```markdown
## Anti-pattern check

- Success metric = adoption or thumbs-up? No — success metric is median-task-completion-time.
- Non-goals are truisms? No — every non-goal names a concrete behaviour with a boundary.
- Safety-scope out-of-scope entries have rationales? Yes — each rationale in Section 4.
- Cost budget has a pricing baseline? Yes — Section 5 pins pricing to 2026-07 rates.
- Kill criteria require a meeting to fire? No — every criterion is a number-window-source that an
  on-call can act on without negotiation.
- Charter is version-controlled and re-signable? Yes — v0.1, next-review 2026-11-14.
```

## Starter guidance

- **Draft the user problem first, then re-write it.** First drafts always contain a solution shape ("build an AI assistant that…"). The second draft names *the user's task* and *the current cost of doing it*. If you cannot describe the current cost in a number, interview one user (or one internal user-proxy) before writing.
- **Pick the success metric from telemetry that already exists** if you can. If the metric requires new instrumentation to compute, add "instrumentation of X" to the charter as a pre-launch dependency in Section 5 (traffic model paragraph) and be explicit that you cannot measure it yet.
- **Non-goals are protective, not aspirational.** The right way to draft them is to imagine the incident that would happen if the goal *were* implicit, and write the negative that protects against it. "Won't auto-send drafts" protects against the accidental-send incident. "Won't answer legal questions" protects against a category of harm.
- **Safety scope handed-off items must name a peer role.** "Handled by security" is not enough; "handled by `ai-risk-engineer` per delegation contract §D-2 (see [chapter 05](../05-delegation-contracts-with-peer-specialists.md))" is. The reviewer needs to see the hand-off has a counter-party.
- **Cost budget needs a traffic model.** "$18k/month" without "at 100% rollout, 200k requests/day, average 3.2k input tokens + 800 output tokens per request" is un-monitorable — you cannot tell whether a budget breach is a price hike or a traffic surge.
- **Kill criteria that read "if quality is bad" fail review.** Rewrite every criterion as *metric ≥/≤ threshold on window*. If a kill criterion cannot be phrased that way, either the metric doesn't exist (instrument it before you sign) or the criterion is actually a policy (rewrite as one).
- **Sign-off is not "I read it."** Sign-off is "I sign that my section is correct." If a reviewer cannot sign, ask them to name — in writing — what would change their mind, and address it before re-circulating.

## Acceptance criteria

- [ ] `docs/features/<feature-name>/charter-v0.1.md` exists as a single Markdown file, ≤ 800 words, with a header block, six sections, and a signature block. No `TBD`.
- [ ] **Section 1 (user problem)** names the user role, the task, the current cost with a number, and what "solved" looks like *for the user*. No solution shape.
- [ ] **Section 2 (success metric)** is *one* number, user-facing, attributable, weekly-computable, bounded. Includes definition, evaluator, target, window. Not adoption. Not thumbs-up.
- [ ] **Section 3 (non-goals)** lists 5–7 concrete behaviours the feature will not attempt. Each one line. No truisms.
- [ ] **Section 4 (safety scope)** has three lists (in-scope, handed-off, out-of-scope). In-scope items name the mod-107 guardrail. Handed-off items name a peer role. Out-of-scope items have rationales.
- [ ] **Section 5 (cost budget)** names the monthly ceiling, the CPSO target, the pricing baseline (with model IDs and pricing date), and the traffic model.
- [ ] **Section 6 (kill criteria)** has 5–7 bullets, each of shape *metric — threshold — window — source — action*. Every one is measurable from an existing (or explicitly-planned) substrate.
- [ ] Signature block contains PM, engineer, governance-owner lines with names, roles, and dates. If self-reviewed, the review notes are attached as a companion file.
- [ ] Anti-pattern check section (in the charter or a companion) walks through each anti-pattern from [chapter 01](../01-feature-charter-with-kill-criteria.md) and asserts the charter does not exhibit it.

## Stretch goals

Pick any one:

- **Charter v0.2.** Two weeks after v0.1 signs, imagine one thing changed (an incident, a scope expansion request, a pricing shift, a peer-role hand-off broke). Author v0.2 with the diff called out — a short "changes from v0.1" section at the top. Prove that the charter is a living artefact.
- **The pre-mortem sibling.** For each of the six recurring failure modes in [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md), write a one-line hypothesis about *how* the feature would fail in that mode and *which kill criterion* would catch it. If a mode is not caught by any current criterion, add a criterion.
- **The two-altitudes exercise.** Take the charter and draft the one-pager from [mod-101 chapter 04](../../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md) alongside it. Show — with an explicit cross-reference table — how every charter field flows into a specific one-pager section.
- **The reviewer's guide.** Write a one-page "how to review this charter" companion for a new PM / governance owner joining the team. Names the four questions each reviewer should ask and where in the document to find each answer.

## Deliverable

Submit as `mod-112/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/features/<feature-name>/charter-v0.1.md` — the signed charter.
- `docs/features/<feature-name>/charter-review-notes.md` — self-review notes or the raw feedback from real reviewers, whichever applies.
- A short `README.md` naming the feature, the surface, and how you selected it (real feature, realistic scenario, or public analogue).
