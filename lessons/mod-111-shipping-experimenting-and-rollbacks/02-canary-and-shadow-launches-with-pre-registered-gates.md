# Canary and Shadow Launches with Pre-Registered Gates

## Why the promotion decision is written before the traffic starts

Every LLM rollout in a team without written promotion criteria eventually plays out the same way. The new prompt goes live to 5% of users on a Tuesday afternoon. Someone eyeballs a dashboard for an hour, does not see anything obviously terrible, and bumps to 25%. By Friday it's at 100%. Two weeks later a support-side quality report shows the new prompt regressed the answer-usefulness score for tenants in the SMB segment — the segment that's under-represented in dashboards because it's under-represented in traffic — and rollback is now negotiating with a product manager who has already told stakeholders the launch went well.

The pattern the applied engineer plants against this is **pre-registered promotion criteria**: before the flag ever flips a single percent, you write down the specific numbers a canary must beat to move to the next stage, the specific numbers a shadow must beat to be promoted at all, and the specific numbers that trigger automatic rollback. The word that matters is *pre-registered*: the numbers are frozen before the observation window opens. This is the same discipline mod-106 imposes on evals ([mod-106 chapter 01](../mod-106-feature-side-evaluation/README.md)) and mod-109 imposes on SLOs ([mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md)) — the pattern is deeper than the surface. It exists because humans looking at time-series charts systematically fool themselves.

This chapter names the two rollout shapes (canary and shadow), their per-cohort gates on quality / safety / cost / latency, the sample-size logic that keeps a canary from being promoted on noise, and the fail-closed rollback wiring.

## Canary vs shadow — what each answers

The two shapes are complementary. Applied engineers routinely conflate them; the difference is load-bearing.

**Canary** — the new variant serves *real user traffic for a subset of users*. Users see the new prompt, the new model, the new agent. Quality, safety, cost, latency, and user-feedback signals all reflect real-world usage. What you learn: how the new variant behaves in the wild. What you spend: exposure to the fraction of users in the cohort, i.e., real product risk if the variant is bad. Sizing is a rollout curve — 1%, 5%, 25%, 100%.

**Shadow** — the new variant is run *in parallel with the current variant, offline from the user's response path*. The user always sees the current variant's answer; the shadow's answer is logged, judged by evals ([mod-106 chapter 04](../mod-106-feature-side-evaluation/README.md)), and compared. What you learn: offline metrics on production-distributed inputs. What you spend: 2× the inference cost of the covered turns, no exposure to real user risk. Sizing is a fraction of turns to shadow (often 5–20%), not a rollout curve.

The right sequencing for a substantial change (new prompt, new model, new chain topology): **shadow first, canary second, general availability third.** Shadow catches gross regressions cheaply and safely; the canary confirms the shadow's picture translates to real user behaviour (including UI, latency-perception, and feedback signals shadow cannot see); GA is the flag flip once the canary's gates pass.

For a small change (a prompt tweak within an existing variant family, a temperature adjustment), shadow is often overkill and a direct canary is fine. The judgment is proportional to blast radius.

## The four gate families, and what "pre-registered" means for each

Every canary or shadow promotion decision runs against four gate families. All four have to pass; any one failure blocks promotion.

### 1. Quality gates

The evals from [mod-106](../mod-106-feature-side-evaluation/README.md) are the substrate. The pre-registered numbers are:

- **Offline eval delta.** Run the frozen offline eval set from mod-106 exercise-01 against both variants. Pre-register: *"Variant B must not regress the composite score by more than 2 points, and must not regress any per-scenario score by more than 5 points."* The scenario floor prevents the "average is fine but the tail-safety scenario collapsed" failure.
- **Shadow judge delta.** For shadow launches, run the LLM-judge from [mod-106 chapter 04](../mod-106-feature-side-evaluation/README.md) on N shadowed turns comparing A vs B. Pre-register the minimum sample size, the win-rate threshold (typically B wins ≥ 55% of comparable turns after excluding ties), and a **judge drift check** — if the judge's ratings on the frozen calibration set have drifted since the offline eval was last run, invalidate the shadow signal until the judge is re-calibrated.
- **User-feedback delta (canary only).** Thumbs-down rate on the canary must not exceed the control's rate by more than X% (usually 0.5–1 percentage point absolute). Requires the feedback triage pipeline from [mod-110 chapter 03](../mod-110-observability-feedback-and-incidents/03-user-feedback-triage-pipeline.md) tagging turns by variant.

### 2. Safety gates

Guardrails ([mod-107](../mod-107-guardrails-and-safety-on-the-surface/README.md)) produce their own metrics. Pre-register:

- **Refusal-rate delta.** Variant B's refusal rate (guardrail block + polite decline) must not exceed the control by more than X% (typically 1 percentage point). Over-refusal is a real quality failure that quality evals often miss.
- **False-positive spike.** Variant B's moderation-block rate on turns that would be considered safe by human review (sampled from the feedback pipeline) must not exceed the control by more than X%. This catches the "new prompt phrases things in a way the moderator classifier over-blocks" failure.
- **Prompt-injection surface change.** If the new variant changes the tool surface or the retrieval-context handling, run the mod-107 injection eval set and require no new failures. Silent regressions here are worst-case.

### 3. Cost gates

The CPSO discipline from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) is the substrate. Pre-register:

- **CPSO delta.** Variant B's cost per successful outcome must not exceed the control by more than X% (a business decision — 10% is a common threshold for a quality-motivated change; 0% or negative for a cost-motivated change).
- **Absolute cost ceiling for the cohort.** The canary's daily spend must stay under $Y. This is the guard against "the new prompt has a runaway retrieval loop" that no CPSO ratio catches until the outcome definition finishes computing.

### 4. Latency gates

The per-surface latency spec from [mod-109 chapter 01](../mod-109-cost-latency-slos-for-llm-features/01-app-altitude-latency-ttft-tpot-and-per-surface-specs.md) is the substrate. Pre-register:

- **TTFT p95 delta.** Variant B's TTFT p95 must not exceed the control by more than X ms, and must remain under the surface spec's "yellow" threshold in absolute terms.
- **Total-latency p95 delta.** Same for the end-to-end p95, particularly for agent variants where the new chain may add tool-call round-trips.
- **Retry rate.** Variant B's retry rate must not exceed the control's by more than X%. A new prompt that trips the JSON-schema validator more often shows up here first.

Write these numbers into a `docs/canary-plan.md` (or `docs/shadow-plan.md`) *before the flag flip*. Commit it. Do not edit it during the observation window. If a gate fails, the response is either roll back or (with product sign-off) file a *plan revision* that names why the gate was wrong; you never silently move the number to make it pass.

## Sample-size math — the number of turns a canary needs

The single most common canary failure mode is *promoting on noise*. A 5% canary that ran for an hour and looks fine has 5% of an hour's traffic in its numerator. If your feature does 10 turns per second at peak, that is 1800 turns. If your success rate is 90%, the standard error on that rate at 1800 samples is about ±0.7 points. A "1-point drop" is inside the noise floor.

The applied engineer does not need to become a statistician; they need to *avoid the obvious mistake*. The working heuristic:

- For each metric with a gate, name the **smallest difference the gate cares about**. Call it Δ.
- The canary needs enough samples per variant to reliably detect Δ. A rough rule for a rate-shaped metric (thumbs-down rate, refusal rate, success rate) at α=0.05, power=0.80: `n ≈ 16 × p × (1 - p) / Δ²`, per variant. For p ≈ 0.10 and Δ = 0.01, that's ~1440 per variant, so ~2880 canary + control samples in the canary cohort. If your canary is 5% of traffic and you want two hours of exposure to reach that number, your feature needs ~200 turns per second — which is a lot. Most features either sit at the canary stage longer than an afternoon, or the canary is a larger fraction of traffic than 5%.
- For metrics that are ratios of small numbers (retry rate, guardrail-block rate) at low base rates (< 1%), the sample-size requirement is much higher and canary is often not the right shape to catch them — shadow with a much larger denominator is.

This is the *interface* to the deeper statistical work owned by `senior-ml-engineer` / `model-evaluation-engineer` — variance-reduction (CUPED), sequential-testing corrections, sample-ratio-mismatch monitoring. [Chapter 03](03-ab-experiment-interface-to-the-experimentation-platform.md) names where that boundary sits. The applied engineer's job at *this* stage is to write down Δ and n in the plan, and to not promote until n samples are actually in the log.

## Shadow-launch mechanics — the double-emission code shape

Shadow serves the current variant *and* runs the new variant offline. The code shape:

```python
def handle_turn(user_input, ctx):
    control_variant = "support_answer_v14"
    shadow_variant = client.get_string_value("support_agent.shadow_variant",
                                             default_value="",
                                             evaluation_context=ctx)
    shadow_enabled = client.get_boolean_value("support_agent.shadow_enabled",
                                              default_value=False,
                                              evaluation_context=ctx)

    # Real answer — the user sees this.
    control_response = call_variant(control_variant, user_input, ctx)
    respond_to_user(control_response)

    if shadow_enabled and shadow_variant and shadow_variant != control_variant:
        # Fire the shadow off the critical path.
        asyncio.create_task(run_shadow(shadow_variant, user_input, ctx, control_response))
```

Two properties this shape must have:

- **The shadow never blocks the user response.** The shadow is fired async, from an executor with a bounded queue that drops on overflow. A shadow that back-pressures onto the request path is worse than no shadow.
- **The shadow emits its own trace span, marked as a shadow.** `app.shadow.of = "support_answer_v14"`, `app.shadow.variant = "support_answer_v15"`. This is what the shadow-comparison analysis joins on later. Do not co-emit shadow spans as if they were real user turns; downstream analytics will double-count.

Cost of shadow: 2× the inference cost of the covered fraction of turns. Budget it in the CPSO ledger under a dedicated `app.feature = "support_agent.shadow"` tag. If the shadow runs for a week at 20% coverage on a feature that spends $2k/day, the shadow costs an extra $80/day. That is often worth it; the *decision* is a budget decision, not an infra decision.

**PII posture for shadow.** The shadow is still an LLM call with the user's data. It has the same DPA and residency requirements as the real call ([mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/README.md), [mod-108 chapter 04](../mod-108-model-routing-and-provider-selection/README.md)). If the shadow variant lives on a provider your DPA does not cover, the shadow is *not* free — it is a compliance change that needs sign-off.

## Canary rollout curve and the promotion decision

The industry-normal canary curve for LLM features is:

- **1%** — 4–24 hours. Confirms the flag flip did not break the world (JSON validator errors, budget explosion, silent 100% cohort due to a `default_value` mis-wire).
- **5%** — 24–72 hours. First stage where the pre-registered quality / safety / cost / latency gates get real numbers. Most canary decisions happen here.
- **25%** — 72 hours to 1 week. Confirms the 5% result holds at 5× volume with more segment coverage.
- **100%** — flip once the 25% has been stable for the pre-registered soak period.

At each stage, an **automated promotion decision** runs against the gates. Manual approval is fine (and often required), but the *decision inputs* — did every gate pass, on how much data, with what effect size — are pre-computed and delivered to the person who clicks the button. A promotion checklist that reads "does the graph look OK to you?" is the failure mode this chapter exists to prevent.

Common promotion-decision artefact:

```markdown
## Canary Promotion — support_agent_v15 at 5% → 25%
Observation window: 2026-08-05 12:00 UTC → 2026-08-07 12:00 UTC (48h)
Samples: canary 34,220 turns; control 649,880 turns.

Gate                        Threshold           Observed    Status
Offline eval (composite)    ≥ v14 - 2 pts      +1.1 pts    PASS
Offline eval (worst scene)  ≥ v14 - 5 pts      -3.0 pts    PASS
Thumbs-down rate delta      ≤ +0.5 pp          +0.2 pp     PASS
Refusal rate delta          ≤ +1.0 pp          +0.3 pp     PASS
Moderation FP delta         ≤ +0.5 pp          -0.1 pp     PASS
CPSO delta                  ≤ +10%             +7.4%       PASS
TTFT p95 delta              ≤ +150 ms          +90 ms      PASS
Retry rate delta            ≤ +0.5 pp          +0.2 pp     PASS

Decision: PROMOTE to 25%. Next observation window closes 2026-08-14 12:00 UTC.
```

That artefact goes in the launch tracking doc, and every subsequent promotion re-runs the same block. When the launch retrospective happens after GA, the sequence of these blocks *is* the retrospective's evidence log.

## Automated rollback wiring — the burn-rate hook

Canaries need an automatic rollback trigger, not just a promotion gate. The wiring is a **canary-burn alert**: if any of the gate metrics move faster than a threshold rate (a spike in refusals, a doubling of retry rate, a runaway CPSO), the flag layer's SDK is instructed to flip the multivariate flag back to the control variant *without a human*.

The pattern is the multi-window multi-burn-rate alert from [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md), adapted:

- **Fast burn**: any hard-gate metric (moderation FP delta, guardrail-block delta, absolute CPSO ceiling) crosses 3× its threshold on the canary cohort over 5 min → auto-rollback via a flag-SDK API call, page the on-call.
- **Slow burn**: any soft-gate metric (CPSO delta, TTFT delta, refusal delta) crosses 1.5× its threshold over 30 min → page the on-call to make the call manually (auto-rollback is skipped because the metric is noisier and false positives are more costly).

The auto-rollback path is exercised on a schedule (weekly or monthly), not just when it's needed. A rollback wiring that has never fired in a rehearsal is a rollback wiring that will fail on its first real firing.

## Anti-patterns

- **The promotion criteria are written during the observation window.** The team sees a metric they didn't originally think about, decides it's the important one, and moves the goalposts. This is p-hacking with extra steps. Freeze the plan doc before flipping the flag.
- **Canary run for one hour and promoted.** Not enough samples to detect the smallest thing the gate cares about. The sample-size math is not optional.
- **Shadow on the critical path.** Latency of every user turn is now `max(control, shadow)`. Fire shadow async, drop on overflow.
- **Shadow variant on a provider the DPA does not cover.** Silent compliance regression that outlives the launch.
- **No `app.shadow.*` attributes on shadow spans.** Downstream analytics double-count shadow turns as real user turns; the accounting ledger blows up.
- **Manual promotion without a checklist.** The launch retro has no evidence trail. When something goes wrong later, no one can reconstruct what was known at each promotion.
- **No automated rollback.** The runbook item that says "flip the flag" takes 20 minutes because the on-call has to walk through the LaunchDarkly UI at 3 AM. Automate the flip for the hard gates.
- **Percentages instead of samples in gate spec.** "TTFT p95 must not regress" is a wish; "TTFT p95 must not regress by more than 150 ms on ≥ 30k turns" is a gate. Include both the effect size and the sample size.

## Summary

- **Shadow first, canary second, GA third.** Shadow catches gross regressions cheaply and safely; canary confirms the shadow picture translates to real user behaviour and cost.
- Every rollout runs against **four gate families**: quality (offline eval + judge + user-feedback delta), safety (refusal / FP / injection deltas), cost (CPSO delta and absolute ceiling), latency (TTFT / total / retry deltas). All four must pass to promote; any one to roll back.
- The gate numbers are **pre-registered** in a plan doc *before* the flag flip. The doc is frozen for the observation window. You never move the numbers to make a launch pass.
- Sample-size the observation window using the smallest Δ each gate cares about. The heuristic `n ≈ 16 × p × (1 - p) / Δ²` per variant is enough to avoid the "promoted on noise" failure. Depth (CUPED, sequential testing) is owned by peer roles — see [chapter 03](03-ab-experiment-interface-to-the-experimentation-platform.md).
- Shadow launches double-emit: the user sees the control; the shadow runs async, marks its span with `app.shadow.of` / `app.shadow.variant`, and is joined offline. Shadow adds 2× inference cost on the covered fraction and inherits the same DPA / residency posture as the primary call.
- Canary rollouts follow a **1% → 5% → 25% → 100%** curve with a **promotion-decision artefact** per stage — the sequence of these artefacts is the launch's evidence log at retro time.
- Every canary has an **automated rollback** on the hard gates, using the multi-window multi-burn pattern. The rollback path is rehearsed on a schedule, not first exercised in the incident.

[Chapter 03](03-ab-experiment-interface-to-the-experimentation-platform.md) turns from "how do I decide whether the new variant is at least *not worse*" to "how do I know whether it's *actually better*" — A/B experiments, the interface to the experimentation platform, and the boundary with the peer roles that own the deep methodology.
