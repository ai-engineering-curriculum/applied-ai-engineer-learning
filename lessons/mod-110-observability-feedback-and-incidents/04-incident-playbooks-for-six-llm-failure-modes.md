# Incident Playbooks for the Six Recurring LLM Failure Modes

## Why an LLM feature needs its own incident playbook — not just the SRE one

The SRE incident playbook exists in every organisation of any size. It answers: the API is 500ing, the queue is backed up, the database is down. It does *not* answer: the model quietly shifted tone last Tuesday, the provider's pricing page updated at midnight, the classifier is now blocking 15% of benign requests, the judge that gates our online eval has drifted so gradually that the "quality" chart is a lie.

Those failure modes are unique to LLM features. They do not look like a 500. Some of them do not even *ring alerts* on a general-purpose observability stack. When they happen — and they will, on every feature, over its lifetime — an on-call engineer without a written playbook is inventing the response in real time under pressure, which is when mistakes get made.

This chapter names the six failure modes an LLM feature actually breaks along, one playbook per mode. Each playbook has the same shape: **detection** (what the alert or symptom looks like), **triage** (what to check in the first five minutes), **mitigation** (the fast reversible action), **root-cause** (the deeper investigation), and **prevention** (the change that stops it recurring). The playbooks are the input to [chapter 05](05-per-feature-status-page-and-on-call-runbook.md), which composes them into a feature-specific runbook the rotation actually opens during a page.

The six modes are drawn from operational patterns observable across the field; they are not a research taxonomy and there will be tenth-percentile modes your feature invents on its own. Adopt these six as the baseline and extend.<!-- needs-research: consider citing SREcon or KubeCon talks on LLM incident postmortems if the org has adopted specific external references. -->

## Playbook 1 — Bad prompt push

**Definition.** A prompt-template change reached production and made responses worse. Could be a helpful-sounding edit that lost a critical instruction, a token cap tightened too aggressively, or a "just clarify the tone" change that broke a structured-output constraint. The model is nominally fine; the harness around it changed.

### Detection

Three signals will show it, in order of leading-ness:

- **Structural-validity metric drops** on the online eval loop from [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md). This is the earliest and cleanest indicator when a structured-output template regresses (JSON schema break, missing required field). Lands in minutes.
- **Thumbs-down rate spikes** on the feature. From [chapter 03](03-user-feedback-triage-pipeline.md)'s pipeline. Usually shows up in 30–120 minutes of the push, depending on traffic volume.
- **Support tickets and Slack complaints** — the slowest signal, often hours later. If this is your first indication, both prior detectors are misconfigured or the affected traffic is too small to move the aggregates; fix that as part of prevention.

The most reliable dashboard: **thumbs-down rate segmented by `app.prompt.id` and `app.prompt.template_hash`** (from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md)). A hash that was 3% thumbs-down yesterday and is 12% today under the same traffic is a bad prompt push.

### Triage (first 5 minutes)

1. **Identify the deploy.** `git log docs/prompts/`, or your prompt-versioning system's diff. Which template hash is in production and when did it land? If the answer is not obvious in one query, your prompt versioning discipline needs work — see [mod-109 chapter 02](../mod-109-cost-latency-slos-for-llm-features/02-prompt-caching-as-a-design-constraint.md) and [mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md).
2. **Confirm the correlation.** The spike aligns in time with the deploy window. If it does not, the incident is not a bad prompt push — likely a provider issue (Playbook 2) or a judge drift (Playbook 3).
3. **Sample the traces.** Open five thumbs-down traces from the last hour under the new template hash. Do they share a failure mode? Common failures a bad prompt push produces: refusing where it should not, missing a required output field, ignoring context, changed tone.

### Mitigation (fast, reversible)

**Roll the prompt back to the last-known-good template hash.** This is the reason prompts are versioned artefacts and not config-panel edits — the rollback is one deploy of the shipping mechanism from [mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md) with the previous hash pinned. Time-to-mitigate should be minutes, not hours.

If the change was part of a larger deploy (prompt + code + model swap in one release), and only the prompt is regressed, roll back the prompt independently — the shipping mechanism must support prompt-only rollbacks, or every prompt regression drags a code regression with it.

### Root cause

After mitigation:

- **Diff the two template hashes.** Line by line. Which specific edit produced the regression?
- **Reproduce on the eval set.** Run the new template against the eval fixtures from [mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md). Does the offline eval catch the regression? If yes, the CI gate from [mod-106 chapter 03](../mod-106-feature-side-evaluation/03-eval-gated-ci-end-to-end.md) let it through — the gate needs a threshold review. If no, the eval set is missing the fixture — add it as the fix.

### Prevention

- **Eval-gated CI** ([mod-106 chapter 03](../mod-106-feature-side-evaluation/03-eval-gated-ci-end-to-end.md)) must be on for every prompt change, not just code changes. Prompt-only PRs need the same gate.
- Every prompt PR ships with **a canary rollout**, not a full flip — the release discipline from [mod-111](../mod-111-shipping-experimenting-and-rollbacks/README.md).
- Add the regression fixture to the eval set as part of the incident close-out, so the same shape cannot recur.

## Playbook 2 — Provider outage

**Definition.** The upstream model provider (OpenAI, Anthropic, Bedrock, Vertex, Azure OpenAI) is degraded — elevated 5xx rates, rate-limit floods, region-specific latency, or a full outage. Your feature is now failing for reasons you cannot fix at your altitude.

### Detection

- **Error rate on `gen_ai.system` = <provider>** spikes across every feature that uses it. If it is one feature only, it is not a provider outage — it is a per-feature bug.
- **Provider status page** (OpenAI, Anthropic, AWS Bedrock, Google Cloud) shows an incident. The status page usually lags the actual failure by 10–30 minutes; do not wait for it.
- **`app.retry.attempt_number` distribution** shifts toward higher attempts across the fleet. The routing gateway from [mod-108](../mod-108-model-routing-and-provider-selection/README.md) is retrying more.
- **Provider request IDs** in error spans cluster in a specific region or model ID. Regional outages are more common than full-provider outages.

### Triage (first 5 minutes)

1. **Confirm scope.** Is the error rate elevated on all features using that provider, or only some? Only some = a routing / config / feature-specific issue, not a provider outage.
2. **Check the provider's status page and their status-updates API.** Note the incident's declared scope (region, model, endpoint).
3. **Check the routing gateway ([mod-108](../mod-108-model-routing-and-provider-selection/README.md)).** Is the fallback provider healthy? Is the failover configured for this specific failure shape? A 5xx storm should trigger failover; a slow-response degradation might not, depending on your gateway's tuning.

### Mitigation (fast, reversible)

The mitigation depends on how much your feature's routing plane already gives you:

- **Failover to the secondary provider.** If [mod-108](../mod-108-model-routing-and-provider-selection/README.md) has a compatible secondary configured, the failover should be automatic and the on-call's job is to verify it is working. If manual, the flip is a single config change with a written pre-approval — the failover is not the moment to invent the change window.
- **Reduce traffic on the failing surface.** If failover is not possible (single-provider feature, residency constraint, no compatible alternative model), the graceful-degrade path from [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/04-tool-abuse-and-destructive-action-gates.md) fires — the feature returns a policy message ("this feature is temporarily unavailable, we're on it") instead of a 500. That is a shipped kill-switch, not an ad-hoc code change.
- **Shift retries to a longer backoff.** If the provider is rate-limiting rather than 5xxing, aggressive retries are making it worse. Increase the retry-budget headroom from [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md).

The single most important mitigation posture: **the incident is often not fixable at your altitude, and the on-call's job is to reduce user impact until the provider fixes it.** That is a communications job as much as a technical one — see the status-page discipline in [chapter 05](05-per-feature-status-page-and-on-call-runbook.md).

### Root cause

Almost always: the provider had a bad deploy, a region issue, or a capacity event. The provider's postmortem, when it lands, is the root-cause artefact. Your own postmortem addresses: did the failover work; did the kill-switch trigger correctly; how long between the provider's first failure and our detection.

### Prevention

- **Multi-provider capability on any feature you care about.** [mod-108](../mod-108-model-routing-and-provider-selection/README.md) covers the substrate; the applied engineer's job is to have exercised the failover path in a game day, not just written the config.
- **A shipped kill switch** from [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/04-tool-abuse-and-destructive-action-gates.md). Not a plan; a merged code path that has been triggered in staging within the last quarter.
- **Provider status-page subscription** routed into your alerting stack. When AWS Bedrock declares an incident, your on-call channel should see it in the first minute, not the tenth.

## Playbook 3 — Judge drift

**Definition.** The LLM-as-judge that powers your online eval loop from [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) has drifted. The "quality" number the dashboard shows is no longer measuring what it measured last quarter — either the judge model was silently upgraded by the provider, the judge prompt template shifted, or the judge's calibration against ground-truth has decayed. The scary version: the quality number *rises* while the actual user experience *degrades*, because the judge has become more lenient.

### Detection

- **Judge-vs-ground-truth divergence.** The online eval's periodic check against expert-reviewed ground truth (from [chapter 03](03-user-feedback-triage-pipeline.md)'s expert-review substrate) shows the judge's agreement rate is falling below its baseline. This is the primary signal — if you do not have this check, you cannot detect judge drift.
- **User feedback and judge agree less.** The correlation between thumbs-down rate and judge-flagged-as-bad rate weakens. On a healthy pipeline, the two move together; when they diverge, one of them is wrong and the judge is easier to change than the user.
- **The judge score distribution shifts.** The judge is scoring 92% "good" this month vs 78% last month with the same underlying model and no known improvement. That is a red flag, not a win.
- **Provider announced a judge-model version bump.** If your judge is `gpt-4o` and OpenAI silently rolled the alias — see `gen_ai.response.model` from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) — the judge is a different model now.

### Triage (first 5 minutes)

1. **What version is the judge on?** Read `gen_ai.response.model` on judge spans over the last 24 hours. If it shifted, the drift is likely provider-side.
2. **Read the ground-truth agreement chart.** Where did it break? A step change points at a version bump; a gradual slide points at a corpus drift (the user distribution changed, and the judge is scoring a different population).
3. **Sample disputed items.** Pick 20 items where the judge said "good" and the user said thumbs-down. Read them. If the judge is defensibly right, the user is over-signalling; if the judge is defensibly wrong, the judge is drifting.

### Mitigation (fast, reversible)

- **Pin the judge model.** If the judge was on an alias (`gpt-4o`), pin it to the specific dated version that had a known baseline (`gpt-4o-2024-08-06`) in the judge configuration. This is the exact same alias-swap failure mode as Playbook 2 in miniature; the fix is the same discipline.
- **Fall back to the last-known-good judge prompt.** The judge prompt is a versioned artefact; roll it back the same way as Playbook 1's product prompt.
- **Stop trusting the current quality dashboard.** If the on-call cannot yet tell whether the drift is fixed, put a banner on the dashboard: `judge under investigation — this number is unreliable`. Silent decoration of an unreliable chart is worse than the chart being missing.

### Root cause

- Did the provider roll the judge model? Provider release notes and `gen_ai.response.model` are the sources.
- Did somebody change the judge prompt? Diff the judge prompt hash against the last-known-good version.
- Did the user distribution shift? If new tenants, new locales, or new feature surfaces started using the judge, the population being judged has changed and the judge's baseline calibration is against a different population.

### Prevention

- **Judge model always pinned to a dated version** in the eval configuration. No aliases.
- **Weekly judge-vs-ground-truth calibration** as a scheduled job. When the calibration drops below the baseline threshold, page. See [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) for the substrate.
- **Judge prompt is versioned artefact** with a change-review process; changes require a re-calibration run against the ground-truth set.

## Playbook 4 — Price hike

**Definition.** The provider changed pricing. Overnight, cost per successful outcome (CPSO from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md)) is 25%+ higher. Nothing about your feature changed; the invoice will bear the cost when it lands on the first of the month. The applied engineer's finance stakeholder finds out from the invoice; the applied engineer's job is to find out from the pricing page announcement, before the invoice.

### Detection

- **`app.pricing_version` on spans** doesn't match the version the ledger is computing against. From [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md). If the pricing table has not been updated in the code but the provider's page has moved, the estimated CPSO understates reality.
- **Estimated-vs-actual reconciliation** at month-end diverges more than the historical noise floor. The reconciliation from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md) is what catches this — the estimator says $10k, the invoice says $13k, the delta is a pricing change or a cost bug you didn't ship.
- **Provider blog / release notes / API-change email** — the leading indicator you should be watching for. Every provider announces pricing changes with lead time; a subscription is not optional.

### Triage (first 5 minutes)

1. **Which SKU changed?** Provider announcements usually name the specific model IDs affected. Cross-reference against the model IDs your ledger sees traffic on.
2. **What is the estimated new CPSO?** Update the pricing table in a scratch calculation; multiply by current token volume; report the new monthly estimate to the stakeholder in the finance / cost-owner role from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).
3. **What is the effective date?** Providers give 30–90 days lead time on hikes; you have that window to react.

### Mitigation (medium-speed, some options are reversible)

- **Update the pricing table** in the ledger. This is not an outage mitigation — nothing is broken — but the estimator drifts from reality until it is done. Bump `app.pricing_version`.
- **Route affected traffic to a cheaper alternative** via [mod-108](../mod-108-model-routing-and-provider-selection/README.md), if quality allows and the router already supports it. This is the whole point of a routing layer — the price hike is the incident that makes the router pay for itself.
- **Move eligible traffic to the batch API** ([mod-109 chapter 03](../mod-109-cost-latency-slos-for-llm-features/03-batch-apis-and-the-offline-eligibility-rubric.md)) if the batch discount still holds. A price hike often shifts the eligibility rubric's economics; re-run the rubric on the affected workloads.

### Root cause

The provider raised prices. That's the root cause. Your investigation is: did we know in advance (yes / no — if no, subscribe to the announcement channels); did we have alternatives (yes / no — if no, that is the actionable finding).

### Prevention

- **Subscription to every provider's release / pricing announcement.** Product-team owned; feed into the on-call channel and the FinOps stakeholder's channel.
- **Pricing-version bump requires a CI check** that the ledger's version matches the code's version — the mismatch is a common source of silent bill drift.
- **Multi-provider quality parity kept warm** so a re-route is a config change, not a re-integration.

## Playbook 5 — Silent quality regression

**Definition.** The feature's output quality dropped, but no error rate went up, no thumbs-down spiked (yet — the users have not noticed en masse), no dashboard turned red. Something quietly broke. Left alone it becomes the "we lost trust with users last quarter and don't know when it started" incident.

This is the hardest of the six to detect. It is why the online eval loop from [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) exists.

### Detection

- **Online eval score for the feature drifts down** across a period longer than daily noise, on the same eval judge, on the same population. This is the primary signal.
- **Retrieval quality metrics degrade** — if you have a retrieval-side eval (top-1 score distribution, MRR against a labelled set) — the retrieval corpus went stale, the embedding model changed, the index was rebuilt with a different chunker.
- **Model alias-swap.** `gen_ai.response.model` shows a different string than last week. The provider rolled the alias; the new model is subtly different on your prompts.
- **Thumbs-down category shift.** The category mix from [chapter 03](03-user-feedback-triage-pipeline.md) changed — same overall thumbs-down rate, but now 30% of thumbs-downs are `wrong_answer` when they used to be `too_long`. Something specific broke.

### Triage (first 15 minutes — silent regressions rarely need a 5-minute triage)

1. **When did the eval score start drifting?** Overlay the eval score against the deploy history — for the model provider, for the prompts, for the retrieval index, for the guardrail config. A step change aligns with a change; a slow slide aligns with corpus drift.
2. **Split the score by cohort.** Per prompt version, per model version, per retrieval index version, per tenant, per surface. The regressed cohort is the smoking gun.
3. **Read the traces.** Twenty samples from the regressed cohort. What is the shared failure mode?

### Mitigation (variable speed)

The mitigation is whichever of the four sub-causes it is:

- **Prompt-version regression** → roll the prompt back (Playbook 1's mitigation).
- **Model alias-swap** → pin the model to the last-known-good version (Playbook 3's mitigation applied to the product model, not the judge).
- **Retrieval index drift** → roll the index back to the last-known-good snapshot; investigate the ingest pipeline separately. This may require the ingestion team's involvement.
- **Guardrail config change** → the classifier is now blocking or over-filtering more; see Playbook 6, which is a specific sub-case.

### Root cause

The specific artefact that changed, plus the reason the change did not trip a pre-production gate. Silent quality regressions are almost always **either a change that didn't run through the eval-gated CI, or a change to a system the eval-gated CI does not cover** (a retrieval index rebuild that is not evaluated by the CI, a model alias-swap that is upstream of your gate).

### Prevention

- **Extend the eval-gated CI** to cover every system whose change can affect output quality — model version pin, retrieval index version, guardrail config, prompt.
- **Weekly eval "canary run"** — the online eval from [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) against a fixed subset of gold answers, published as a chart. When the chart moves, page. This is the substrate that turns silent regressions into loud ones.

## Playbook 6 — Guardrail false-positive spike

**Definition.** The moderation / classifier / safety layer from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md) started blocking (or retracting) responses at a rate higher than the FP baseline. Users see refusals where they should not. The feature *looks* fine on the LLM-cost dashboard because inference is still running; it *looks* fine on the error-rate dashboard because guarded blocks are not errors. But your users are getting refused, and the thumbs-down rate on `unhelpful_refusal` is climbing.

### Detection

- **Guardrail block rate spike.** From the `guardrail.decision` attribute in [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md). Alert on `block_rate` per feature per hour, with a threshold based on the baseline from the FP/FN report in [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md).
- **`unhelpful_refusal` category thumbs-downs spike.** From [chapter 03](03-user-feedback-triage-pipeline.md). This one is the user-side confirmation.
- **A specific guardrail category is over-firing.** `guardrail.categories` in the spans clusters heavily on one category (`hate` or `self_harm` most often) — that classifier's threshold is misaligned or the model was updated.

### Triage (first 5 minutes)

1. **Which layer is firing?** Provider moderation or classifier? The `guardrail.name` and `guardrail.model_version` attributes tell you. If both, which came first?
2. **What is the category distribution?** Is one category driving the whole spike? That is a classifier-side issue on that category.
3. **What changed?** The provider's moderation model version, the classifier version, the classifier threshold, the classifier's system prompt. Diff the last 24 hours.

### Mitigation (fast, reversible)

- **Loosen the threshold on the affected category** back to the last-known-good value. The threshold is a versioned artefact; roll it back.
- **Fall back to the previous layer version** if the classifier was updated. `guardrail.model_version` is the pin.
- **Feature-off the offending layer temporarily** if the pipeline supports "provider moderation only" or "classifier only" — the two-layer composition from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md) should be feature-flag-able for exactly this scenario.

The failure mode here has a specific reversibility risk to name: **do not loosen guardrails to reduce over-blocking without confirming false-negatives have not risen.** A loosened classifier passes more benign traffic *and* more truly harmful traffic. If the fix is a rollback to a known-good version, that is safe. If the fix is a new threshold, it must be validated against the FP/FN eval set from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md) before shipping.

### Root cause

- Classifier model was updated (silent version bump on a hosted classifier, or a deploy of Llama Guard / Prompt Guard).
- Classifier threshold was changed.
- Classifier's system prompt was changed.
- **User traffic shifted** — new locale, new phrasing patterns, new use case that resembles the flagged category. This is legitimate over-blocking against a new population; the fix is expanding the FP/FN fixture set from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md) to cover it and re-calibrating.

### Prevention

- **Guardrail versioning** — `guardrail.model_version` on spans, dated version pins in config.
- **FP/FN report is re-run on every guardrail change** (the artefact from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md)). A guardrail threshold PR that does not include a delta report against the FP/FN fixtures gets rejected in review.
- **Block-rate baseline alerts** with an anomaly threshold, per feature per category.

## The recurring shape

Read the six playbooks side by side and a common shape emerges:

- **Every one of them is detectable from telemetry that already exists in [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md)** — `app.prompt.template_hash`, `gen_ai.system`, `gen_ai.response.model`, `guardrail.decision`, `app.pricing_version`, `app.feedback.signal`. If your span attribute set is short of any of these, you have a blind spot for the corresponding playbook.
- **Every one of them has a "roll it back" mitigation** — because prompts, model versions, judge versions, pricing versions, retrieval indexes, and guardrail configs are all versioned artefacts. If any of them is not versioned, the mitigation for its playbook is invented in real time, which is when incidents get worse.
- **Every one of them has a prevention that is "extend the eval-gated CI to cover this thing"** — [mod-106](../mod-106-feature-side-evaluation/README.md) is the substrate that turns most of these from an incident into a caught PR.
- **None of them are "the model got a wrong answer once."** LLM features have a wrong-answer rate greater than zero, always. A single wrong answer is not an incident; a **shift in the rate** is. This distinction is what keeps the on-call from being paged every hour.

## Two things every playbook shares in practice

**A written owner per playbook.** Even at a small team, one person is the "prompt push" owner and someone (possibly different) is the "guardrail FP" owner. When the alert fires, the on-call engages the owner, does not investigate alone.

**A game-day schedule.** Playbooks that have never been rehearsed are aspirational. A quarterly game day where the team simulates a bad prompt push (deploy a known-bad template to staging, run the alert, run the mitigation, time it) is what turns a written playbook into a real one. Google's SRE Workbook and the wider SRE community publish this discipline; the applied engineer's job is to adopt it, not invent it.

## Anti-patterns

- **One-size-fits-all incident response.** The 500-error playbook does not apply to a silent quality regression. The provider-outage playbook does not apply to a bad prompt push. Six different failure modes need six different playbooks.
- **Alert on wrong-answer count instead of wrong-answer rate shift.** The wrong-answer count is never zero. Alerting on it fires constantly and is ignored. Alert on the rate shift.
- **No rollback path for prompts.** Every "bad prompt push" incident becomes a code fix instead of a config revert. Ship the versioned-prompt substrate before you need the rollback.
- **Guardrail changes without an FP/FN delta report.** Every threshold tune is a coin flip against user experience. The eval discipline from [mod-107 chapter 01](../mod-107-guardrails-and-safety-on-the-surface/01-layered-moderation-with-classifier-and-provider.md) is not optional for changes; it is the change process.
- **Judge on an alias.** Provider rolls the alias, the quality chart shifts, nobody knows if the feature or the judge changed. Pin the judge to a dated version.
- **Provider status-page subscription lives in someone's inbox, not the on-call channel.** The 30-minute delay between the provider's outage and the on-call learning about it is unforced.
- **Silent regression playbook that assumes the eval loop exists.** If the online eval loop is not shipped, silent regressions are undetectable; the playbook cannot fire because the alert cannot fire. Ship [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md) first.
- **No game days.** The first time the on-call runs a playbook is during a real incident. The response is slower, the mitigation is buggier, the postmortem is longer. Rehearse.

## Summary

- The six recurring LLM-feature failure modes: **bad prompt push, provider outage, judge drift, price hike, silent quality regression, guardrail false-positive spike.** Each has its own detection signal, triage path, mitigation, and root cause — the general-purpose SRE playbook does not cover them.
- Every playbook detects from telemetry that already exists in [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md). Missing attributes are missing playbooks.
- Every playbook's mitigation is "roll back the versioned artefact" — prompts, models, judges, pricing tables, indexes, and guardrail configs must all be versioned artefacts with a rollback path, or the mitigation is invented under pressure.
- Every playbook's prevention is "extend the eval-gated CI to cover this thing." The eval discipline from [mod-106](../mod-106-feature-side-evaluation/README.md) is the shared substrate that turns most incidents into caught PRs.
- Playbooks are **written**, **owned**, and **rehearsed** in quarterly game days. An unrehearsed playbook is aspirational.

[Chapter 05](05-per-feature-status-page-and-on-call-runbook.md) composes these playbooks — plus the sampling policy, the feedback pipeline, and the observability substrate — into the two artefacts the on-call actually reaches for: a per-feature status page users can see, and a per-feature runbook the rotation can use at 3 a.m.
