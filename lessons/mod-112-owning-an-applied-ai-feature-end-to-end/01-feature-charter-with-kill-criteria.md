# Translating a Roadmap Item into a Feature Charter

## Why the charter exists before the code does

A roadmap line item — *"add an AI assistant to the pricing page"* — is not a spec. It is a wish. Somewhere between "the PM said yes" and "the flag flips to 5%," someone has to decide what user problem the feature solves, which number tells us it worked, what the feature will *not* do, what class of harm is in scope, what the monthly bill can be, and what would make us pull the flag. If the applied-AI engineer does not force that decision to happen at the *start* of the build, it happens implicitly at the first incident — and by then the choices are ratified by whatever is already shipped, not chosen.

The one-page feature-architecture doc from [mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md) is the design-time counterpart to this artefact. The **feature charter** is what the roadmap item becomes *before* the shape is picked. It is a shorter, higher-altitude document: it names the *what* and *why* so the *how* (which the one-pager captures) has a scope to fit inside. Both live in the repo, both are versioned, and both are load-bearing at governance and incident review.

This chapter is the design-substrate chapter of mod-112 — the artefact every subsequent chapter of this module traces back to. Every eval, every incident fixture, every build-vs-buy choice, every model-card row, every delegation contract, and every governance mapping cites lines from the charter. If the charter is thin, the downstream artefacts are thin. If it is precise, they inherit that precision.

## The six charter fields

A working charter fits on one page and answers six questions.

1. **User problem** — whose task, in what surface, is worse without this feature? One sentence, named user, named surface, named cost of *not* shipping.
2. **Success metric** — the *one* number that tells us the feature worked, with the definition of "worked" spelled out. Includes evaluator, window, and target.
3. **Non-goals** — what the feature will *not* attempt in this build. Bounds the scope so review can converge; protects future incident triage from "well, it kind of does X."
4. **Safety scope** — which harm classes are in-scope for this feature to defend against, which are handed off to a peer role, and which are declared out-of-scope with a rationale.
5. **Cost budget** — the monthly cost ceiling the feature is allowed to consume, in a currency the FinOps stakeholder recognises, and the cost-per-successful-outcome (CPSO) target the ops team will monitor against.
6. **Kill criteria** — the pre-registered thresholds at which the flag flips off, the rollout pauses, or the feature is deprecated. Concrete, measurable, and agreed *before* the incident, not after.

Six fields, one page. Each one is short enough that a reviewer can read the whole thing in three minutes. Each one is specific enough that a disagreement in review lands on a concrete line, not on vibes.

## Field 1 — The user problem

The user problem is the hardest field to write and the one worst-served by generic language. "Users want an AI assistant" is not a user problem. "Users need better search" is not a user problem. Both of those describe a *solution shape* while pretending to describe a problem.

A working user-problem statement names four things:

- **The user** — the specific role, seat, or persona. "Tier-1 support agents handling refund requests," not "our customers."
- **The task** — the concrete work being done today. "Drafting a reply to a refund request in the helpdesk composer."
- **The current cost** — the measurable pain today: time, error rate, cognitive load, hand-off latency. "Currently averages 7 minutes per reply; the median agent handles 42 refund requests per shift."
- **What "solved" would look like** — the outcome the user cares about. Not the model's outcome, the *user's* outcome. "Reduces median reply time to under 3 minutes without a rise in reopen rate."

The last bullet is where most drafts flinch. If the applied engineer cannot describe what solved looks like *for the user*, the feature is being built for the org's roadmap, not for the user, and the success metric in field 2 will be a proxy for a proxy.

The reference-book source here is *Working Backwards: Insights, Stories, and Secrets from Inside Amazon* (Bryar & Carr, 2021) and Amazon's PR/FAQ discipline — the same "who is the user, what is the pain, what does solved look like" pass that Amazon runs before any code. LLM features do not get a pass on this; the user still has a task, and the AI is still a tool inside it.

## Field 2 — The success metric

The mod-106 substrate (offline eval, judge-based online eval, feedback triage) exists to *measure* things. Field 2 is where the charter names *which* thing. One metric is enough. Two is a policy in disguise ("we want to be good at X while also improving Y"). Three is a wish list.

The success metric must satisfy four properties:

- **User-facing.** "Judge score on 500 offline fixtures" is a leading indicator, not a success metric — the user has never seen the judge. "Median reply-composition time" is a success metric because a user experiences it. Pair the two if you want; the *success* metric is the user-facing one.
- **Attributable.** The metric moves because of this feature, not because of a marketing push, a seasonality shift, or a competitor's outage. Attribution comes from the experimentation substrate ([mod-111 chapter 03](../mod-111-shipping-experimenting-and-rollbacks/03-ab-experiment-interface-to-the-experimentation-platform.md)); the charter names the metric that A/B will measure.
- **Computable weekly.** If you cannot compute the metric on a rolling 7-day window from the telemetry substrate ([mod-110](../mod-110-observability-feedback-and-incidents/README.md)), it is aspirational. Aspirational metrics do not survive review.
- **Bounded.** A specific number, a specific direction, a specific window. "Reduce median reply time from 7 minutes to under 4 minutes on a rolling 14-day window."

The success metric is what will be argued about at the roadmap review two quarters from now. Write it so that argument has a data source.

Two shapes to avoid. **"Adoption"** is not a success metric; usage without outcome is a vanity graph and every AI feature can drive it by being placed in the primary UI slot. **"Satisfaction"** measured as thumbs-up rate on the feature itself is a mirror-metric — users click thumbs up on shipped features because they are polite. Measure the *task* outcome, not the feature's meta-affordance.

## Field 3 — Non-goals

The non-goals field is protective. It names the things that will *not* be attempted so the review can converge on the things that will, and so an incident three months from now cannot escalate the feature's scope by accident. When a support agent asks "why didn't the assistant offer a refund for me?" and the answer is "the charter's non-goal §3 says drafts are not sent automatically," the review conversation is short.

Working shape:

```markdown
## Non-goals (v0)

- Will not auto-send drafts to customers. Every reply is agent-reviewed.
- Will not process refunds above $500 without human approval.
- Will not answer questions about competitor pricing.
- Will not surface content from the internal legal / HR knowledge bases.
- Will not translate outbound replies to a language other than the ticket's.
- Will not be embedded in mobile agent workflows in v0 (post-v1 candidate).
```

Six negatives, each concrete. Non-goals are versioned with the rest of the charter: a v1 that expands scope adds a line to a `## Non-goals (v1)` block and explicitly *moves* the promoted item from the v0 list. The provenance matters at incident review.

Two anti-patterns show up on every review. **"Won't be perfect"** is not a non-goal; it is a truism. **"Won't do harm"** is a policy statement, not a scope bound — safety scope goes in field 4. Non-goals are shape and behaviour, not virtue.

## Field 4 — Safety scope

The safety scope names which classes of harm the feature is responsible for defending against, at the *surface* level. The deeper work — harm modelling at breadth, red-team programme design, adversarial dataset construction — belongs to `ai-risk-engineer` and is delegated in [chapter 05](05-delegation-contracts-with-peer-specialists.md). The charter's job is to name the surface's posture.

The three lists:

- **In-scope harms** — categories the feature actively defends against at layer 1. Names the guardrail from [mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md) (input moderation, output classifier, PII scrubbing, tool-catalogue allow-list) and the specific policy sections they enforce.
- **Handed-off harms** — categories where the surface *detects and escalates* rather than defending in-line. The escalation contract points at a peer role (`ai-risk-engineer` for red-team-derived shapes, `application-security-engineer` for auth-boundary shapes, `privacy-engineer` for residency shapes).
- **Out-of-scope harms** — categories the feature does *not* defend against, with an explicit rationale. Out-of-scope is not the same as "we don't care"; it is "this feature does not have a plausible surface for this class of harm and defending against it here would be theatre."

Example:

```markdown
## Safety scope (v0)

### In-scope
- Prompt injection via ticket content — layer-1 input classifier (mod-107 ch01).
- PII in outbound drafts — outbound scrub (mod-107 ch03), policy §PII-1.
- Refund-amount tool call above $500 — confirmation gate (mod-107 ch04), policy §REF-3.
- Refusal calibration (must-never medical / legal advice) — mod-107 ch05, policy §S-1.

### Handed off
- Adversarial jailbreak corpus construction — `ai-risk-engineer` (delegation §D-2).
- Model-side refusal fine-tuning — `ai-safety-engineer` (delegation §D-3).
- Provider DPA / cross-border transfer analysis — `privacy-engineer` (delegation §D-5).

### Out of scope (v0), rationale
- CSAM / illegal-content generation — provider moderation is the primary defence; this
  surface handles refund conversations and has no image-generation affordance. Revisit
  if a v1 adds attachment analysis.
- Election / political-influence content — not a plausible surface for a refund composer.
```

The rationale is what makes an "out of scope" defensible at governance review. "We don't do that" fails the review; "we don't do that *because…*" passes it and is auditable.

## Field 5 — Cost budget

The cost budget field expresses the same discipline as the CPSO in [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md), but at the charter altitude: what is the ceiling on this feature's monthly cost, and what is the CPSO target the ops team will monitor against?

Two numbers:

- **Monthly cost ceiling.** The all-in ballpark the FinOps stakeholder signed on. Includes model inference, retrieval, judge inference, and any batch / eval cost. Not an SLO — it is a *procurement* number the stakeholder committed to. "$18k / month at 100% rollout." Anything above it is a re-approval, not a routine expansion.
- **CPSO target.** The cost-per-successful-outcome the ops team monitors against, derived from the model of the success metric and the current pricing. "≤ $0.04 per successfully-drafted reply." This is a runtime SLO; [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) is the substrate that measures it.

Both numbers include the assumed pricing version and the assumed traffic model. When either shifts (price hike per [mod-110 chapter 04 playbook 4](../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md), or a traffic-model doubling), the charter is re-signed, not silently exceeded.

## Field 6 — Kill criteria

Kill criteria are the pre-registered conditions under which the feature flag flips off, the rollout pauses, or the feature is deprecated. They are the artefact that lets an on-call engineer at 3 AM take a reversible action without inventing the policy under pressure.

Two properties are load-bearing.

- **Concrete.** Every kill criterion is a *number*, a *window*, and a *measurement source*. "Groundedness drops below 0.85 on the 7-day rolling window, evaluated by judge-v3" is a kill criterion. "Quality is bad" is not.
- **Agreed before, not after.** The criteria are ratified in the charter sign-off. At incident time, the on-call reads them and acts; they do not negotiate. If the criterion turns out to be too aggressive or too loose after the incident, the *next* version of the charter tunes it — the incident itself does not re-negotiate the policy.

Working shape — five to seven bullets, each satisfying both properties:

```markdown
## Kill criteria (v0)

- Groundedness < 0.85 on 7-day rolling window (judge-v3, mod-106 ch04) → auto-pause
  the rollout; page on-call; require re-baseline before resume.
- Over-refusal rate > 5% on the obvious-yes eval panel (mod-107 ch05) → same.
- CPSO > $0.08 for 3 consecutive days (mod-109 ch04) → pause expansion; PM+FinOps
  re-approve or feature stays at current cohort %.
- Any P0 safety incident tagged `policy.must_never` → feature flag flips off within
  15 minutes; no restart until the root-cause fixture is in the eval-gated CI
  (mod-106 ch03).
- P0 provider outage of the primary model with no shipped fallback → kill switch
  routes to static fallback (mod-107 ch04); do not auto-degrade to unknown model.
- Reopen-rate on refund tickets rises > 20% vs the 30-day pre-launch baseline in
  the treated cohort → pause the experiment (mod-111 ch03); investigate before
  promotion.
- Feature-level thumbs-down rate > 2× baseline for 48h → alert; if trend does not
  reverse in 24h more, deprecate cohort expansion.
```

Two shapes to avoid. **A single kill criterion** ("we pull it if quality is bad") is the same as no criterion — the on-call still has to decide. **Kill criteria that require the CEO's approval to fire** are not kill criteria; they are a wish. The point of pre-registration is to *pre-authorise* the reversible action.

Kill criteria connect directly to the guardrail and rollout substrates: the flag flip is the [mod-111 chapter 04](../mod-111-shipping-experimenting-and-rollbacks/04-hot-rollback-of-a-bad-prompt.md) hot-rollback path, the eval-based trigger reads from [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md), the cost-based trigger reads from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md), and the safety-incident trigger reads from [mod-110 chapter 04](../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md). The charter is where they are named; the substrate is where they fire.

## The one-page shape end-to-end

Putting the fields together — a minimally-viable charter file, `docs/features/support-refund-composer/charter.md`:

```markdown
# support-refund-composer — Feature Charter v0.3

**Owner:** applied-ai-engineer:jferguson
**PM:** pm:mchang
**Governance reviewer:** ai-governance-analyst:oluwe
**Signed:** 2026-07-14   **Next review:** 2026-10-14 (or on any kill-criterion fire)

## User problem
Tier-1 support agents handling refund requests currently spend a median 7 minutes
composing a reply (helpdesk telemetry, 2026-Q2). We want the median under 4 minutes
without raising the 7-day reopen rate on refund tickets.

## Success metric
Primary: median reply-composition time for tagged refund tickets, over a rolling
14-day window. Target: ≤ 4 min at 100% rollout, from the 7 min baseline.
Guardrail (must not degrade): 7-day reopen rate on refund tickets ≤ baseline + 5%.

## Non-goals (v0)
[as above]

## Safety scope (v0)
[as above]

## Cost budget (v0)
- Monthly cost ceiling at 100% rollout: $18k / month (FinOps sign-off 2026-07-10).
- CPSO target: ≤ $0.04 per successfully-drafted-and-sent reply.
- Pricing baseline: gpt-4o-2024-08-06 + text-embedding-3-large, us-east-1, 2026-07 pricing.

## Kill criteria (v0)
[as above]
```

The file is versioned. The next version bumps a semver, records a change log, and re-triggers the three sign-offs. A charter without a version and a next-review date is a Google Doc, and Google Docs do not survive re-orgs, engineering hand-offs, or incident review.

## The charter's relationship to the one-page architecture doc

Two artefacts, two altitudes, both needed.

- The **charter** is the *decision to build*. Six fields. Signed by PM, engineer, and governance owner. Answers *what and why*.
- The **one-page architecture doc** (from [mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md)) is the *decision to build it this way*. Names the shape, the SLOs, the provider, the safety envelope, the rollout plan. Answers *how*.

The charter comes first and is stable; the one-pager comes next and iterates as the build proceeds. The charter's fields flow into specific sections of the one-pager: user problem → Section 2, success metric → Section 3 SLO row, safety scope → Section 6, cost budget → Section 3 cost-SLO row, kill criteria → Section 8.

The two together are the design substrate. Every subsequent chapter of this module treats them as read-only inputs.

## Anti-patterns

- **Charter written after the build starts.** The charter is a design artefact. When it lags the code, its fields are reverse-engineered from what shipped, which means the review can't reject anything. If the roadmap says "start the build," the charter is the first PR, not a docs task.
- **Success metric = adoption or thumbs-up.** Adoption is a vanity graph and thumbs-up is a mirror. The success metric measures the *user's task outcome*.
- **Non-goals that are truisms.** "Won't be perfect" is not a non-goal. Non-goals are behaviours the feature will not attempt so the review can converge.
- **Safety scope with no rationale on the out-of-scope items.** "Not our problem" fails governance review. "Not a plausible surface for this class of harm because [rationale]" passes it.
- **Cost budget without a pricing baseline.** A ceiling without the assumed pricing version is un-monitorable — you can't tell whether the budget was breached because volume rose or because prices did. The pricing version is on the file.
- **Kill criteria that need a meeting to fire.** The point of pre-registration is that the on-call at 3 AM can act without one.
- **A charter you cannot find at incident time.** Version-controlled in the repo, path-stable, and linked from the runbook. Not in Notion.
- **A charter that never re-signs.** A quarterly re-review, or an incident-triggered re-review, keeps the fields honest as the world changes around the feature.

## Summary

- The **feature charter** turns a roadmap wish into a signed decision to build. Six fields, one page: user problem, success metric, non-goals, safety scope, cost budget, kill criteria.
- **User problem** names the user, the task, the current cost, and what "solved" looks like — *for the user*, not for the model.
- **Success metric** is *one* user-facing, attributable, weekly-computable, bounded number. Not adoption. Not thumbs-up.
- **Non-goals** are concrete behaviours the feature will not attempt. Six negatives, each specific. Protects future incident scope.
- **Safety scope** names in-scope harms (defended in-line), handed-off harms (escalated to a peer role), and out-of-scope harms (with a rationale).
- **Cost budget** is a monthly ceiling in a FinOps-recognised currency plus a CPSO target. The pricing baseline is on the file.
- **Kill criteria** are pre-registered thresholds — a number, a window, a source — that the on-call fires without negotiation. Five to seven bullets is the working shape.
- The charter is signed by PM, engineer, and governance owner. Version-controlled, path-stable, and re-signed on a cadence or on any kill-criterion fire.
- The charter and the [mod-101 chapter 04](../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md) one-pager are two altitudes: the charter answers *what and why*, the one-pager answers *how*. Every subsequent chapter of this module treats both as read-only inputs.

[Chapter 02](02-incident-to-fixture-loop.md) picks up the incident-to-fixture loop — how a real production incident becomes a permanent regression fixture, an alert, and a runbook update, so the same failure cannot recur silently.
