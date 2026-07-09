# mod-101 — Applied AI Foundations: From Product Spec to LLM Feature Architecture

**Estimated effort:** 10 hours

Translate a product spec into a shippable LLM-feature architecture: pick the right feature shape, map user-visible promises to concrete SLOs, choose the initial provider and tier with a written trade-off note, and pack it all into a one-page decision doc that a PM, an ML engineer, and a governance owner can each sign off on. Finish by naming — and rejecting — the four failure modes that let leaderboard-driven thinking sneak into feature design.

## Learning objectives

- Read a product spec and choose the right LLM-feature shape — chat, assistant, co-pilot, autonomous agent, one-shot generation, retrieval-with-citations, structured extraction — with an explicit rejection of the shapes that do not fit.
- Map user-visible SLOs to concrete measurable targets: task success, groundedness, safety, cost per successful outcome, TTFT / TPOT / p95 latency, freshness.
- Pick the initial provider and model tier for the surface with a written trade-off note (frontier / mid-tier / open-source; managed vs self-hosted; region / residency).
- Author a one-page feature-architecture doc a PM, an ML engineer, and a governance owner can each sign off on — with the ISO/IEC 25059 quality dimensions used as a checklist.
- Recognise the four failure modes of leaderboard-driven feature design and reject them in review (SOTA-worship, benchmark-vs-surface mismatch, one-model lock-in, unmeasured cost).

## Lecture chapters

1. [Choosing the Feature Shape](01-choosing-the-feature-shape.md) — the seven shapes, the decision, the rejection step, a worked support-agent example.
2. [SLOs From Spec to Metrics](02-slos-from-spec-to-metrics.md) — the six SLO families and how to write a well-formed target row.
3. [Provider and Tier Selection](03-provider-and-tier-selection.md) — capability tiers, hosting tiers, region / residency, and the written trade-off note.
4. [The One-Page Feature-Architecture Doc](04-one-page-feature-architecture-doc.md) — the template, the ISO/IEC 25059 checklist, and the three-role sign-off flow.
5. [Leaderboard-Driven Failure Modes](05-leaderboard-driven-failure-modes.md) — SOTA-worship, benchmark-vs-surface mismatch, one-model lock-in, unmeasured cost, and how to reject each in review.

## Exercises

- [exercise-01 — Feature Shape Decision Rubric](exercises/exercise-01-feature-shape-decision-rubric.md)
- [exercise-02 — SLO Mapping from Spec to Metrics](exercises/exercise-02-slo-mapping-from-spec-to-metrics.md)
- [exercise-03 — Provider Tier Trade-off Note](exercises/exercise-03-provider-tier-trade-off-note.md)
- [exercise-04 — ISO/IEC 25059 Quality Dimensions Audit](exercises/exercise-04-iso-25059-quality-dimensions-audit.md)

## Structure

```
mod-101-applied-ai-foundations-and-feature-design/
├── 01-choosing-the-feature-shape.md
├── 02-slos-from-spec-to-metrics.md
├── 03-provider-and-tier-selection.md
├── 04-one-page-feature-architecture-doc.md
├── 05-leaderboard-driven-failure-modes.md
├── exercises/                 per-exercise prompts (solutions live in the paired -solutions repo)
├── labs/                      long-form hands-on labs
├── quizzes/                   knowledge checks
├── resources.md               curated external references
└── README.md                  this file
```

## What comes next in the track

- mod-102 turns the shape + SLO + provider decision into production prompting and structured outputs.
- mod-103 wires the retrieval-with-citations shape (chapter 01) into a real product surface.
- mod-104 and mod-105 land the tool-calling and product-agent shapes.
- mod-106 turns the SLO table (chapter 02) into an eval-gated CI harness.
- mod-107 turns the safety envelope into a layered guardrail stack.
- mod-108 turns the trade-off note (chapter 03) and portability accounting into a real router bake-off.
- mod-109 turns the cost-per-successful-outcome SLO into per-feature and per-tenant accounting.
- mod-110 turns the kill criteria into on-call runbooks and incident-to-fixture loops.
- mod-111 turns the rollout plan into feature-flag-driven canary and shadow launches.
- mod-112 turns the one-pager (chapter 04) into an end-to-end feature charter.
