# Hot Rollback of a Bad Prompt

## The scenario this chapter is written for

It's 14:12. A prompt-registry PR merged an hour ago that tightened the system prompt for the customer-support summariser. At 14:07, a support engineer pings the team channel: "hey, is something weird with summaries today? Half of them start with 'I cannot help with that' now." The dashboard in [mod-110 chapter 05](../mod-110-observability-feedback-and-incidents/05-per-feature-status-page-and-on-call-runbook.md) confirms it: refusal rate on `support_agent_v15` has climbed from 0.8% to 24% since the merge. The on-call engineer's clock starts. How long does it take to get every user off `v15` and back onto `v14`?

If the answer is "the length of a deploy cycle" — twenty minutes on a good day, ninety on a bad one because the CI pipeline is running someone else's slow test — the answer is too long. Real users will spend an hour hitting the bad prompt. In a high-stakes surface (support, medical, legal, sales-facing), an hour is a customer-trust bill you cannot pay back.

The applied engineer's job in this chapter is to bring the "how long" down to **five minutes at the outside, thirty seconds if the drill has been rehearsed**, without shipping code and without paging a platform team. This is a *shipping discipline*: it lives in the code you write when the feature launches, not in the runbook you write after the incident.

The four pieces of that discipline: **a prompt registry with append-only version semantics**, **a rollback SLA that names the target time**, **blast-radius containment via cohort / kill-switch flags**, and **a rehearsed rollback drill**. This chapter names each in the order the on-call needs them.

## Prompt registry semantics — append-only, versioned, cheap to read

A "prompt registry" in this module is a versioned key–value store of prompt templates. Every entry is `(prompt_id, version, template_text, metadata)`. The version is a **content-addressed** SHA (or a monotonic integer if you prefer human-readable IDs); the template text is immutable once written. The registry is a table in your primary datastore, a git-backed YAML directory, or a dedicated service (LangSmith, PromptLayer, Humanloop, Braintrust, or an in-house one) — the *shape* matters more than the vendor.

Four invariants make a registry rollback-ready:

- **Append-only.** You never edit an existing version in place. `support_answer_v14` is what it was on the day it was published, forever. Edits are a *new version* (`v15`). This is what makes rollback a **pointer flip**, not a content restore.
- **Versioned by content, not by wall-clock time.** The version identifier is derived from the template's hash, or is a monotonically incrementing integer. `latest` is an alias, not a version; feature code never resolves against `latest`. This is the same reason mod-108 tells you to pin dated model IDs — a pointer to "the latest thing" is a rollback trap the day someone else ships.
- **Every span carries `app.prompt.id` and `app.prompt.template_hash`.** The trace substrate from [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md) is how you *prove after the fact* that the rollback landed. Without those attributes on every span, the rollback is invisible in the trace and the post-mortem is a fistfight.
- **The registry is cached at the app process.** A rollback that requires a live network call to a config service adds seconds to the incident. Registry entries are loaded at process start, refreshed on flag-layer signal (SSE from LaunchDarkly, OpenFeature evaluator), and read from local memory on the hot path.

The interaction with the flag substrate from [chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md) is what makes this rollback path exist at all. The flag returns a **version identifier**; the code looks that identifier up in the registry; the registry returns the immutable template. Rollback is a flag flip from `v15` back to `v14`, and because `v14` is still in the registry (append-only), the flip resolves instantly.

## The rollback SLA — a specific number, on the wall

The team owes users a written **rollback SLA**: "when we detect a bad prompt in production, we have variant X off traffic within Y minutes." The SLA has three components:

- **Time-to-detection (TTD).** How long between the bad variant reaching a user and the on-call being paged. Depends on the burn-rate alerts from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md) and [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md). Target: 5–15 minutes for critical surfaces.
- **Time-to-decision (TTDe).** How long between the page and the on-call deciding to roll back. Depends on the runbook clarity and how well the alert names the failing metric. Target: 2–5 minutes.
- **Time-to-effect (TTE).** How long between the decision and the flag flip landing at every user's process. Depends on the flag SDK's cache-refresh cadence and the load balancer's session affinity. Target: 30–90 seconds.

The overall rollback SLA is `TTD + TTDe + TTE`, and a working number for a mature applied-AI feature is **10–20 minutes end-to-end for slow-burn regressions**, **≤ 5 minutes for hard-gate breaches** (auto-rollback path from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md) fires on the alert, not on a human decision).

Publish the SLA. Track TTD / TTDe / TTE per incident. If any component is missing its target, that's the target for the next quarter's investment. The number that hurts the most is almost always TTE — teams cut TTD (better alerts) and TTDe (better runbook) and then discover they built a rollback path that takes 15 minutes to propagate because the flag SDK cache never checked in.

## Blast-radius containment — the flag hierarchy

Rollback is easier when the "bad" thing is quarantined. The flag hierarchy this module assumes for every LLM feature:

```
support_agent.enabled           (kill switch — feature on/off entirely)
├── support_agent.cohort         (percentage rollout — 0..100)
│   └── support_agent.prompt_id  (multivariate — which prompt version)
├── support_agent.model_id       (multivariate — which model)
└── support_agent.max_tokens     (config — inference parameter)
```

The rollback path depends on *which layer* the incident is on:

- **Bad prompt.** Flip `support_agent.prompt_id` back to the last-known-good version. The feature stays live for everyone; only the template changes. TTE is a single flag flip.
- **Bad prompt affecting only a cohort.** Drop `support_agent.cohort` from 25% back to 5% (or 0%). Users outside the cohort are unaffected.
- **Bad model.** Flip `support_agent.model_id` back to the previous model. Requires that the previous model is still available on the provider (dated snapshots are deprecated on a schedule — see mod-108).
- **Bad feature-shape (a regression the flag alone cannot fix).** Flip `support_agent.enabled` to `false`. The feature returns the graceful-degrade surface until a code deploy fixes it.

The hierarchy exists so that the smallest containment lever gets pulled first. Flipping the kill switch when a prompt rollback would have sufficed is over-response — every user of the feature sees the degraded experience for the duration. A team that only has the kill switch as its rollback lever will use it, and every rollback becomes a full-surface outage.

**"Fail closed" defaults everywhere.** If the flag SDK cannot reach the control plane and its cache expires, the `default_value` for the prompt/model flag is the **last-known-good** version — never the new variant, never `None`. This is the [chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md) invariant, and it is the reason a control-plane outage does not become a rollout of the un-vetted variant.

## The rollback drill — rehearse the path monthly

A rollback path that has never been exercised in a rehearsal is a rollback path that will fail on its first real firing. The mechanics that fail in real incidents are almost never the flag flip itself — they're the ergonomic bits: the on-call doesn't have LaunchDarkly credentials, the runbook link is broken, the flag-name in the runbook is stale because someone renamed it two months ago, the acknowledged pager alert doesn't include the rollback command as a copy-pastable line.

The drill:

- **Cadence**: monthly, or after any change to the rollback path (new flag, renamed flag, new SDK version, new provider).
- **Format**: on-call gets a synthetic page from a chaos-injection script. The metric it targets is a low-volume canary cohort on a *test* feature. On-call follows the runbook: acknowledge, open the flag console, flip the version, verify the metric returns to baseline. Every step is timed.
- **Result artefact**: a `rollback-drill-YYYY-MM.md` note with TTD / TTDe / TTE recorded, the runbook step that took longest, and one action item for the following month. Every drill produces one improvement.

The drill also exercises the *social* path — that the on-call knows they are empowered to roll back without paging a manager, that the product manager has been briefed on when rollback is autonomous, and that the incident channel exists and is where people go. All of those fail in real incidents; the drill is when you find out.

## The rollback runbook — one page, one command

The runbook the on-call reads at 3 AM is one page. It has three sections:

```markdown
# Rollback Runbook — support_agent
Owner: <name>. Last drilled: 2026-08-01. TTE target: 60 s.

## Step 1 — Identify the bad variant
Open <trace-dashboard-URL>. Filter: `app.feature = "support_agent" AND app.experiment.arm != "control"`
in the last 15 minutes. If the failing arm is a prompt version, go to Step 2A. If it's a model version, Step 2B.

## Step 2A — Roll back the prompt
In LaunchDarkly / OpenFeature UI:
  Flag: `support_agent.prompt_id`
  Set value: `support_answer_v14`  (last-known-good; check the pinned commit in `docs/lkg.md`)
  Save and confirm.

CLI alternative (if the UI is down or you are in the middle of the night):
  $ ld-cli flag update --project support --env prod \
        --flag support_agent.prompt_id --value support_answer_v14

## Step 2B — Roll back the model
Same shape, flag `support_agent.model_id`, value from `docs/lkg.md`.

## Step 3 — Verify
Wait 60s. In the same dashboard, confirm `refusal_rate` and `cost_usd`
return to their pre-incident baselines. If not, escalate to `support_agent.enabled = false`.

## Step 4 — Notify
Post in #inc-support-agent: "Rolled back to <version> at <UTC time>. Investigating cause."
```

**The runbook lives next to the code, is version-controlled, and is linked from every alert.** It is not in a wiki that only the two most senior engineers can find. It is not in a Slack pinned message that scrolled off six months ago. The paging system's alert text includes a direct link to it.

## Blast-radius containment in action — a worked scenario

The prompt bug from the scenario at the top:

- **14:00** — `support_answer_v15` merged into the prompt registry, `support_agent.prompt_id` flipped to `v15` at 5% cohort.
- **14:07** — burn-rate alert on refusal rate fires (fast burn, 3× threshold on `support_agent.refusal_rate` delta). On-call paged. TTD ≈ 7 min.
- **14:09** — on-call opens the runbook, confirms the failing arm is `v15`, decides to roll back. TTDe ≈ 2 min.
- **14:09:30** — on-call flips `support_agent.prompt_id` from `v15` back to `v14`. Flag SDK propagates via SSE.
- **14:10:15** — trace dashboard shows every new span carrying `app.prompt.id = "v14"`. TTE ≈ 45 sec.
- **14:11** — refusal rate returns to baseline.
- **14:15** — on-call posts incident summary, `support_agent.cohort` for `v15` is set to 0% (so `v15` is not just off-cohort but fully unreachable), and the ticket for root-cause analysis is opened.

The full rollback: **11 minutes end-to-end, ~45 seconds to actually restore user experience once the decision was made.** No deploy, no on-call for the platform team, no wake-ups outside the feature owner.

The prompt registry entry for `v15` is *not* deleted. Append-only means the bad version is still there, marked with a `deprecated_reason: "refusal-rate regression 2026-08-05"` field, so the retro can reproduce the exact template. Deleting the bad version means the next reviewer of the retro thinks the incident happened on an unknown template.

## Anti-patterns

- **The prompt lives in the deploy artefact.** Rollback is a deploy. The clock is a deploy cycle. This is the failure mode this entire chapter is written against — the fix is to move the prompt into a runtime-resolved registry.
- **The registry is not append-only.** Someone "cleaned up" the bad version and now the retro has to work from git history alone. Retros need the exact template that was live; a registry that supports delete is a registry that will lose the evidence.
- **`default_value` is the new variant.** A control-plane outage flips 100% of traffic to the un-vetted variant. The rollback is *the fresh deploy of a fix* — twenty minutes minimum, and by definition happens when the flag layer is unavailable.
- **No `app.prompt.id` on the span.** Rollback lands, but there is no proof in the trace. The retro devolves into "we think it landed at 14:10 based on when the alert cleared." Emit the attribute on every LLM span.
- **The rollback runbook is stale.** The flag was renamed, the dashboard URL moved, the LKG version in the runbook is two versions old. Discovered at 3 AM. The monthly drill exists specifically to catch this.
- **The kill switch is the only rollback lever.** Every rollback becomes a full feature outage. The flag hierarchy exists so the smallest lever gets pulled first.
- **The rollback SLA is not published.** No one can hold themselves accountable to a number no one knows. Publish TTD / TTDe / TTE targets and track per-incident.
- **Rollback drills that only exercise the flag flip.** The drill doesn't test the paging path, the credential access, the escalation channel. All of those fail in the real incident. Rehearse the whole loop.

## Summary

- The rollback path is a **runtime pointer flip**, not a deploy. The pointer is a **flag** ([chapter 01](01-feature-flags-as-the-llm-shipping-substrate.md)), the target is an entry in an **append-only prompt registry**, and the flip propagates via the flag SDK's cache in seconds.
- The **prompt registry** stores every version immutably, keyed by a content hash (or monotonic integer). `latest` is not a version; feature code never resolves against it. Deprecated versions are annotated, not deleted.
- Every LLM span carries `app.prompt.id` and `app.prompt.template_hash` — that's the [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md) contract — so a rollback is provable in the trace.
- The **rollback SLA** has three components: **TTD** (detection, from burn-rate alerts), **TTDe** (decision, from a clear runbook), **TTE** (effect, from the flag SDK cache-refresh). Publish a target; track per-incident; invest in whichever number is missing.
- **Blast radius** is contained by a flag hierarchy: prompt / model / cohort / kill-switch. Pull the smallest lever that will fix the incident; do not default to the kill switch.
- **Fail-closed defaults** everywhere: the flag SDK's `default_value` for the prompt / model / cohort flags is the last-known-good, never the new variant. Control-plane outages do not become inadvertent rollouts.
- **Rehearse the rollback monthly.** The mechanics that fail in real incidents are the ergonomics — credentials, dashboard URLs, stale flag names, missing paging routing. The drill is where you find them, not the incident.
- The rollback runbook is **one page, links a direct copy-pastable command, lives next to the code**, and is versioned. The pager alert includes a link to it.

[Chapter 05](05-hosted-fine-tuning-as-an-applied-lever.md) turns from "how to ship and unship a prompt" to "when tuning a model beats prompting one" — hosted fine-tuning as an applied lever for style, format, and small distillation, and the boundary with the peer role that owns methodology depth.
