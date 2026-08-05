# exercise-03: Build-vs-Buy Matrix — Gateway, Eval Platform, Agent SDK

**Estimated effort:** 3 hours

## Objective

Run the build-vs-buy decision on **all three surfaces** — gateway, eval platform, agent SDK — for one real (or realistic) applied-AI feature, and produce a signed trade-off note per surface. The deliverable is not "we chose LiteLLM"; it is the *signed matrix* that shows every option was compared against the feature's actual constraints, the exit door is planned, and the revisit date is on file. The exercise proves you can make a defensible substrate choice without folklore.

## Learning outcomes

By the end you should be able to:

- Enumerate the current buy options in each of the three categories with enough precision to compare them on the axes that actually move the decision.
- Score each option against a **decision matrix** with 5–7 named axes, and defend the weighting.
- Justify the **build option** honestly — including the head-count commitment it implies — or reject it with a written rationale.
- Design an **exit door** for each choice such that switching to a different option is a one-sprint, not one-quarter, migration.
- Author a **signed trade-off note** in the [mod-101 chapter 03](../../mod-101-applied-ai-foundations-and-feature-design/03-provider-and-tier-selection.md) shape for each surface, with FinOps and privacy sign-offs where relevant.

## Prerequisites

- You have read [chapter 03](../03-build-vs-buy-across-three-surfaces.md).
- Familiarity with [mod-108](../../mod-108-model-routing-and-provider-selection/README.md) (gateways), [mod-106](../../mod-106-feature-side-evaluation/README.md) (eval platforms), and [mod-104](../../mod-104-tool-calling-and-orchestration/README.md) + [mod-105](../../mod-105-shipping-product-agents/README.md) (agent SDKs).
- A concrete feature to run the decision against. Ideal: the feature from [exercise-01](exercise-01-feature-charter-with-kill-criteria.md), with the charter as an input. If you skipped exercise-01, use the same feature you plan to use for [exercise-04](exercise-04-per-feature-model-card-slice.md) — the trade-off notes will be cited there.
- If possible, a colleague or mentor available to play a peer role for the sign-off: FinOps, `privacy-engineer`, or `data-security-engineer`. Self-review acceptable if not.

## Problem statement

Your charter is signed, the roadmap kickoff went well, and now you have to pick the substrates. Three decisions are due this month — the gateway, the eval platform, and the agent SDK. Each is a decision your successor will inherit; each is one that folklore will overwrite in 18 months if the reasoning is not on file.

Write the reasoning down. For each of the three surfaces, produce a signed trade-off note that lists the options considered (both buy and build), scores them on 5–7 axes weighted by the feature's actual constraints, names the chosen option, plans the exit door, and sets the revisit date. Then run the same discipline across the three notes to make sure the choices *compose* — an agent SDK that doesn't emit OTel spans your eval platform reads is a stack that doesn't work end-to-end.

## Requirements

### Deliverable 1 — The gateway trade-off note

Under `docs/features/<feature>/decisions/gateway-tradeoff-v0.1.md`:

A signed trade-off note in the shape from [chapter 03](../03-build-vs-buy-across-three-surfaces.md)'s worked example. Includes:

- **Decision** — one line naming the chosen option and rejecting the build option (or accepting it, with the head-count commitment named).
- **Context** — feature reference (link to charter), provider posture (primary / fallback provider from the charter), residency constraint, traffic model.
- **Options considered** — every option that was seriously evaluated. At minimum: LiteLLM, one hosted commercial gateway (OpenRouter, Portkey), one cloud-native gateway that applies to your provider posture (Bedrock / Vertex / Azure AI Foundry), and the build option.
- **Decision matrix** — a table with one row per option and one column per axis. Axes must include, at minimum: provider coverage, deployment shape, fallback semantics, governance features, observability integration, data-plane residency. Each cell either scores (1–5) or notes ("N/A — provider not covered").
- **Chosen option deep-dive** — a paragraph explaining *why* the chosen option won given the feature's specific constraints. Not marketing-speak; the specific axis (or axes) where it wins for this feature.
- **Exit door** — a specific description of what portability shape keeps the choice reversible. Include the estimated switching cost (in days of engineering work) to the runner-up option.
- **Revisit** — a specific next-review date (calendar date, ≤ 6 months out) and the triggers that would move the review earlier.
- **Signed** — applied-AI engineer, FinOps stakeholder, `privacy-engineer` (if the chosen option inserts a third party into the data plane), with date.

### Deliverable 2 — The eval-platform trade-off note

Under `docs/features/<feature>/decisions/eval-platform-tradeoff-v0.1.md`:

Same shape. Options considered must include: at least two managed offerings (Braintrust, Weave, or a cloud-native equivalent), at least two OSS offerings (Langfuse, Phoenix, Promptfoo, DeepEval), and the build option. Decision matrix axes: fixture format portability, judge substrate, online-eval sampling, CI integration, framework neutrality, cost model.

Special requirement: the exit door section must include a **fixture round-trip test plan** — show that the fixture format you commit to lives in the repo as plain text and can be run against at least one alternative platform without transformation.

Revisit cadence: 12 months (with named triggers).

### Deliverable 3 — The agent-SDK trade-off note

Under `docs/features/<feature>/decisions/agent-sdk-tradeoff-v0.1.md`:

Same shape. Options considered must include: single-provider SDKs from at least two providers (OpenAI Agents SDK, Anthropic Claude Agent SDK), at least one multi-provider framework (LangGraph, Agno, or Semantic Kernel), and the build option. Decision matrix axes: provider neutrality, control-flow model, observability semantics, tool-call semantics, human-in-the-loop primitives, community and roadmap velocity.

Special requirement: the exit door section must describe the **`run_turn` wrapper** interface the feature will use to insulate feature code from the SDK. The wrapper's signature is written out; the migration cost to the runner-up SDK is estimated with the wrapper in place.

Revisit cadence: 6 months for the first year (with named triggers).

### Deliverable 4 — The composition check

Under `docs/features/<feature>/decisions/stack-composition-check-v0.1.md`:

A short (one-page) document that verifies the three chosen options *compose* end-to-end. Includes:

- **Data path** — a one-paragraph description of a request's path through the stack, from user → agent SDK → gateway → model provider → back → judge (via eval platform) → observability.
- **Observability normalisation** — evidence (or a plan) that the agent SDK emits OTel Gen-AI + OpenInference spans (or has a translator) that the eval platform ingests and that the observability substrate stores. No format-mismatch handwaving.
- **Data-plane residency** — a one-paragraph review of every third party in the request path (or in the observability path) that touches user data. Confirms all are inside the feature's residency envelope.
- **Cost composition** — a rough estimate of the three-surface *combined* fixed and per-request cost. If any of the three choices imposes a per-request surcharge (Portkey, OpenRouter margin, managed eval platform per-call), the composed number reflects it.
- **Exit doors compose** — a short note confirming that migrating any *one* of the three does not force migrating the other two.

### Deliverable 5 — The revisit calendar

Under `docs/features/<feature>/decisions/revisit-calendar.md`:

A short calendar-like document listing the three revisit dates in one place, with the trigger conditions for each. The doc is the applied engineer's own reminder mechanism; it also lands in the feature card ([chapter 04](../04-per-feature-model-card-slice.md)) as a link.

## Starter guidance

- **Fill the decision matrix before you write the prose.** The prose in the "chosen option deep-dive" section is derivative; the matrix is the primary evidence. If the matrix ties or is very close, the decision is not defensible — either your axes are wrong (missing the axis that actually differentiates) or the weighting is off.
- **Score honestly, especially on the axes where the incumbent product is weak.** A trade-off note that scores your preferred option 5/5 on every axis is a note nobody trusts. Every option has weaknesses; naming them is what makes the decision credible.
- **Reject the build option in writing.** A note that says "we didn't consider build" is a note that will get relitigated in 6 months when someone asks "should we just build our own?" A short paragraph explaining why build was rejected — usually "no long-term owner committed" — settles it.
- **The exit door test is falsifiable.** "We could switch to X" is not a plan; "we could switch to X in 3 days by changing the base URL and re-verifying the OTel span emission" is. If the estimate is more than a sprint, the exit door is too narrow and one of the choices needs to change.
- **Reach out to FinOps for the gateway and eval-platform sign-offs.** Both categories have real per-request or per-seat costs; the FinOps stakeholder wants to see the numbers before you commit. A note without a FinOps signature invites the "the invoice was a surprise" incident.
- **Reach out to `privacy-engineer` for any hosted / managed choice.** OpenRouter, Portkey managed, Braintrust cloud, Weave cloud, Langfuse cloud — all insert a third party into the data plane. The privacy peer role is who signs on residency; a note without their signature is one that will be rejected at governance review.
- **Compose before you commit.** Deliverable 4 is not decoration. If the agent SDK doesn't emit spans your eval platform reads, or if the gateway's third-party is outside your residency envelope, the three-surface decision doesn't work end-to-end. Catch it now.
- **Do not name a "winner" per category in the abstract.** The winner depends on *this feature's* constraints. A feature that must be multi-cloud has a different gateway winner than a feature that lives in one cloud; a feature with a small eval set has a different eval-platform winner than one with 5,000 fixtures. The matrix's job is to make the constraint-driven answer visible.

## Acceptance criteria

- [ ] Three trade-off notes exist, one per surface, under `docs/features/<feature>/decisions/`.
- [ ] Each note has: decision, context, options considered (buy and build both named, minimum option counts met per surface), decision matrix (with named axes and cell-level scores or rationales), chosen-option deep-dive, exit door with switching-cost estimate, revisit date and triggers, and signatures.
- [ ] The build option is either accepted (with a named long-term owner) or rejected (with a rationale) in each note. No note is silent on build.
- [ ] Every managed / hosted option in the "chosen" row has a **`privacy-engineer` signature** (real or self-reviewed with notes).
- [ ] Every note has a **FinOps signature** (real or self-reviewed with cost estimates on file).
- [ ] Deliverable 4 (composition check) exists and verifies the three choices compose on data path, observability, residency, cost, and exit doors.
- [ ] Deliverable 5 (revisit calendar) lists the three revisit dates in one place with triggers.
- [ ] All revisit dates are concrete calendar dates, not "in 6 months." Gateway ≤ 6 months, agent SDK ≤ 6 months (first year), eval platform ≤ 12 months.
- [ ] Every note is ≤ 2 pages of Markdown. The matrix table counts as one section; long lists of features per option belong in an appendix if needed.

## Stretch goals

Pick any one:

- **The counter-factual note.** For the surface you feel most confident about, write a *second* note as if the chosen option were unavailable — which runner-up wins, and why? Prove the exit door plan by running the decision matrix a second time with the incumbent excluded.
- **The three-surface prototype.** Wire the three chosen options together end-to-end against a single test request. Show the trace flowing through the agent SDK → gateway → provider → judge → observability, and demonstrate the composition check works with real data.
- **The revisit dry-run.** Simulate the 6-month revisit today. Write a "revisit notes" companion for the gateway note as if 6 months had passed and one new option had shipped (e.g., a hypothetical Portkey feature or a new LiteLLM provider). Would the decision change? Why or why not?
- **The multi-feature perspective.** If your org has more than one feature, imagine the three-surface decision at *org altitude* — do all features use the same gateway / eval platform / agent SDK? Where do the choices diverge, and where do shared substrates make sense? Author a short "org substrate strategy" note that names the shared choices and the feature-specific ones.
- **The head-count cost sheet.** For the build option in each of the three notes, write a specific estimate: how many engineers, for how long, would it take to reach parity with the chosen buy option? Compare against the buy option's per-year cost. Sometimes the build option is cheaper than the buy option; more often it is not — but the estimate makes the choice arithmetic, not folklore.

## Deliverable

Submit as `mod-112/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/features/<feature>/decisions/gateway-tradeoff-v0.1.md`
- `docs/features/<feature>/decisions/eval-platform-tradeoff-v0.1.md`
- `docs/features/<feature>/decisions/agent-sdk-tradeoff-v0.1.md`
- `docs/features/<feature>/decisions/stack-composition-check-v0.1.md`
- `docs/features/<feature>/decisions/revisit-calendar.md`
- A short `README.md` naming the feature and any assumptions you made (e.g., provider posture inherited from the exercise-01 charter, residency envelope assumed as US-only).
