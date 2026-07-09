# exercise-03: Provider Tier Trade-off Note

**Estimated effort:** 2 hours

## Objective

For one of the specs from [exercise-01](exercise-01-feature-shape-decision-rubric.md), write the provider + tier trade-off note that goes into Section 4 of the one-pager (see [chapter 04](../04-one-page-feature-architecture-doc.md)) — a defensible **initial** choice that the router bake-off in mod-108 can build on top of, and that a governance owner can accept without argument.

## Learning outcomes

By the end you should be able to:

- Score three provider candidates against your feature's SLO table on the dimensions that matter (task success on *your* slice, groundedness, TTFT, end-to-end p95, cost, region, structured-output support, portability).
- Write a three-to-five-sentence written argument that names which SLO the choice is optimising, which SLOs are being traded off, and which candidate is rejected on which specific constraint.
- Complete a portability accounting so the router bake-off in mod-108 has an inventory of what's provider-specific.
- Set an explicit revisit trigger and a mandatory revisit date.

## Prerequisites

- You've read [chapter 03](../03-provider-and-tier-selection.md).
- You've completed [exercise-02](exercise-02-slo-mapping-from-spec-to-metrics.md) or an equivalent SLO table for your chosen spec.
- You have access to public documentation for at least three providers you'd consider. If you have access to a small API credit budget across two or three providers, run the shortlisted candidates on a hand-built 20–40 example slice; if you don't, use publicly documented values and mark measured cells clearly as "not measured on our slice."

## Problem statement

You have a spec, a shape, and an SLO table. Produce the provider + tier trade-off note that would go into Section 4 of the one-pager and defend it against three reviewers:

- A PM who wants the SLOs met at the promised cost.
- An ML engineer who wants portability and a real bake-off shape.
- A governance owner who wants residency, sub-processors, and an audit trail.

## Requirements

Produce a single markdown document with the following sections.

### Section 1 — Header

- **Feature name + surface + chosen shape** — one line, from exercise-01.
- **Decision (one line)** — e.g. "Anthropic Claude Sonnet 4.5 via managed API, us-east-1 by default, eu-west-1 for EU tenants, self-hosted Llama 3.3 70B fallback on internal vLLM."
- **Decision owner + reviewers** — names.
- **Review date and next mandatory revisit** — the calendar dates.

### Section 2 — Shortlist

Name your top-three candidates. For each, give one line on why it made the shortlist. Then, in a one-line "also considered and rejected" tail, name every other provider / tier you eliminated *before* comparison and why (residency, licensing, cost floor, no structured-output support, no fallback path, etc.).

### Section 3 — Comparison table

Score the three candidates on at least these dimensions:

| Dimension | Candidate A | Candidate B | Candidate C |
|---|---|---|---|
| Task success on internal eval slice |  |  |  |
| Groundedness on RAG slice (if applicable) |  |  |  |
| TTFT p95 (measured or published) |  |  |  |
| End-to-end p95 (measured or published) |  |  |  |
| Cost per successful outcome (measured or estimated) |  |  |  |
| Region availability for our tenants |  |  |  |
| Structured-output support |  |  |  |
| Prompt-caching semantics |  |  |  |
| Fine-tuning / distillation lever |  |  |  |
| Vendor concentration risk |  |  |  |

Add rows if your feature needs them (multi-modal support, long-context needs, tool-calling fidelity, guardrail integrations, batch API availability, etc.).

Cells that are **measured on your slice** and cells that are **from public documentation** must be visually distinguishable (a suffix, a footnote, an italic, whatever your team uses). Do not blur measured with published.

### Section 4 — Written argument

Three to five sentences. Must contain:

- The primary SLO the choice is optimising.
- The SLOs being traded off, and by how much.
- The specific constraint that killed the rejected candidates.
- One line on what the router bake-off in mod-108 might change.

### Section 5 — Portability accounting

For each of the following, state what the chosen provider does and what the fallback would do:

- Prompt shape (system-role vs system-then-messages vs system-instruction).
- Structured output mechanism (native JSON mode / native tool calling / constrained decoding via Instructor / Outlines / manual repair).
- Prompt caching (explicit breakpoints / ephemeral / manual / none).
- Tool schema dialect.
- Streaming semantics and cancellation.
- Any other feature (long context, prompt / message roles, thinking blocks, image / audio input) whose availability differs between primary and fallback.

State, for each, whether the item is a **functional dependency** (feature would regress a specific SLO if it changed) or an **optimisation** (nice-to-have but the SLO would still be met).

### Section 6 — Residency and sub-processors

- The tenant categories the feature serves.
- The residency requirements each category triggers.
- The regions used for each provider on the primary path and the fallback path.
- The sub-processors the payload touches (guardrail model, judge model, trace warehouse, feature-flag provider, RAG store, etc.), each with a note on data-processing terms.
- The retention and deletion contract in one line.

If any cell is "I don't know," the note is not ready for the governance reviewer.

### Section 7 — Revisit

- **Trigger revisit if:** three to five specific things (SLO breach on a specific metric with a specific threshold, price change, new frontier release from a top-three lab, residency requirement change, provider outage of severity > X).
- **Mandatory revisit on:** a date, no later than 90 days from decision.

## Starter guidance

- The best defence against "we should have used Model X because it's SOTA" is one row in the comparison table with the measured number on your own slice. Even a 20-example hand-built slice is enough to move the argument from vibes to numbers. See [chapter 05](../05-leaderboard-driven-failure-modes.md).
- If you cannot measure, publish the published-number cells and explicitly caveat them — do not launder them as measured.
- For structured-output support, the *right* question is not "does it work?" but "does it work on our schema, with our long-tail edge cases, with a defined retry / repair path when it fails?"
- On cost per successful outcome, if you don't yet have real success telemetry, use a candidate cost model: `(input_tokens * price_in + output_tokens * price_out) / assumed_success_rate`. State the assumed success rate.
- The portability accounting is the single most under-invested section of a real one-pager. Complete it, and mod-108's router bake-off starts from a known-good position instead of a discovery exercise.

## Acceptance criteria

- [ ] Three candidates named, and every other provider rejected in one line each.
- [ ] Comparison table complete, with measured vs published cells visually distinguishable.
- [ ] Written argument names the primary SLO, the trade-offs, and the specific constraints that killed the rejects.
- [ ] Portability accounting complete, with each row labelled as functional dependency or optimisation.
- [ ] Residency + sub-processors section names the tenant categories, regions, sub-processors, and retention contract.
- [ ] Revisit triggers are specific and falsifiable; mandatory revisit date is ≤ 90 days from decision.
- [ ] Length ≤ ~2 pages of markdown. If it's longer, the trade-off note is competing with the one-pager; compress.

## Stretch goals

Pick any two:

- **Router-ready.** Restate your primary and fallback as a two-tier router config (see mod-108) — which requests go to which tier by default, which requests escalate, which requests hedge across both.
- **Cost stress test.** Model what happens to cost per successful outcome if traffic 5×'s in a month and the primary provider announces a 25 % price increase. State the pre-committed lever.
- **Reg / SoC framing.** Rewrite the residency + sub-processors section as it would appear in a SOC 2 or ISO/IEC 42001 evidence pack, referring to actual public sub-processor lists for the providers you chose.
- **On-call view.** Write the three-line summary an on-call engineer would see in the runbook: primary, fallback, and what to do if primary is down for > 4 hours.

## Deliverable

A single markdown file, submitted to the paired solutions repo (`applied-ai-engineer-solutions`) under `mod-101/exercise-03/`. Include a small `slice/` folder if you ran a hand-built eval slice: the 20–40 prompts, the raw outputs from each candidate, and the scoring rubric you used.
