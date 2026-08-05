# Per-Feature Status Page and On-Call Runbook

## Two artefacts, two audiences

Everything the four previous chapters set up — the span vocabulary, the sampled trace stream, the feedback pipeline, the six playbooks — is upstream work. The load-bearing artefacts a shipped LLM feature actually needs, at the end of the chain, are two documents:

- A **per-feature status page** — the user-facing (or at least stakeholder-facing) page that says "is this feature working right now, is it degraded, is it broken." Audience: users, PMs, sales, support, executives.
- A **per-feature on-call runbook** — the internal document the rotation engineer opens at 3 a.m. when the alert fires. Audience: whoever is holding the pager, who probably did not write the feature.

They serve different audiences and cover different things, but they read from the same underlying signals. This chapter names both, the shape they take, and — crucially — the **interfaces to the platform / SRE team** (owned by the `ai-infra-mlops` peer role) that both plug into. An LLM feature that owns its own alerting substrate, its own status page infrastructure, and its own pager rotation is a feature that has re-invented the SRE wheel from scratch. Do not do that.

## The per-feature status page

### What goes on it

A per-feature status page is *not* a fleet-wide "all systems operational" board. Those exist; consume them, do not replicate them. A per-feature status page answers three questions for the specific feature it covers:

1. **Is the feature available right now?** Green / yellow / red, per environment (prod, EU-prod, staging-facing-partners).
2. **What is the recent history?** Incidents in the last 7 / 30 / 90 days, with a one-line summary each.
3. **What is the current known-degraded surface if any?** The banner that says "voice mode is currently returning fallback text due to an upstream provider incident."

That is the whole scope. It does *not* try to be a dashboard, an SLO report, a metrics viewer, or a debugging surface. A status page that tries to be all of those things is a status page nobody reads.

### The green / yellow / red rule

The three-state model has to be *observable* — a metric flips it, not a human. Otherwise the page decays into a Slack message someone forgot to update. For an LLM feature, the rules that compose the state:

- **Green**: SLO burn-rate under 1× on the primary user-experience SLI (usually "correct-and-on-time response," combining structural validity from [mod-106](../mod-106-feature-side-evaluation/README.md) and latency from [mod-109](../mod-109-cost-latency-slos-for-llm-features/README.md)). No active incident.
- **Yellow**: burn-rate above 1× but below the incident threshold, *or* a known-degraded posture (kill switch tripped for a sub-surface, feature running on the fallback provider). Not paging, but visible.
- **Red**: active incident *or* SLO fast-burn window firing (from the burn-rate mechanics in [mod-109 chapter 05](../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md)). Users are affected.

The page's colour comes directly from these signals; the human on-call cannot override to green just because they think it looks fine. They *can* narrate on yellow / red — the human-written banner text is separate from the colour.

### The shape

```markdown
# <feature name> — status

Current status: 🟢 GREEN   (as of 2026-03-15 14:22:03 UTC)

<empty when green>
<banner text when yellow / red — human-written, refreshed by the on-call>

## Recent incidents

| When              | Severity | Duration | Summary                                                      |
| ----------------- | -------- | -------- | ------------------------------------------------------------ |
| 2026-03-10 09:12  | Sev 2    | 43 min   | Prompt push regression; rolled back template v14 → v13.      |
| 2026-02-28 14:00  | Sev 3    | 3 h 20m  | Anthropic Bedrock region us-east-1 degraded; failed over to Vertex. |
| ...                                                                                                       |

## What this feature depends on

- Upstream provider: <primary> (fallback: <secondary>)
- Retrieval index: <name>, refresh cadence <cadence>
- Guardrails: <layer names, versions>
- Owning team: <team>, rotation: <link to rotation>
- SLO: <link>. Trace budget: <link>.

## Runbook (internal only)

<link — not exposed publicly>
```

The status page's URL is stable, referenceable, and shared across the organisation. Sales links to it in customer conversations. Support points customers at it during incidents. The PM checks it before a launch meeting. When it is not green, the banner text is what everyone reads first.

### Where the status page lives

Two paths are conventional:

- **Internal-only**: on the same substrate as the rest of the org's status pages, gated to employees. Typical for early-stage features, features not yet exposed to external customers, or organisations without a public-status culture. The stakeholder audience is internal.
- **Public**: on the org's public status page (Statuspage.io, Instatus, or the org's own). Typical once the feature has external customers and an SLA in a contract. The banner text is now a customer-communication artefact — has to be written by someone who understands the customer's perspective.

Either way, the substrate — the page-hosting tool, the incident-declaration flow, the retention of the incidents table — is owned by the `ai-infra-mlops` / SRE team, not the applied-AI engineer. See the interfaces section below.

## The on-call runbook

The runbook is the internal document the on-call opens when a page comes in. Its audience is whoever is holding the pager right now — often not the person who wrote the feature, sometimes not on the same team, sometimes at 3 a.m. It must be reachable and readable without prior context.

The rule: **the runbook is written for the least-familiar engineer who could ever be on-call for this feature**. If it assumes "you know what the retrieval corpus is," it is written for the author, not for the on-call.

### The template

```markdown
# <feature name> — on-call runbook

Owner team: <team>
Rotation: <link to the pager rotation>
Escalation: <team lead>, <VP eng>
Repo: <link>
Status page: <link>
Dashboards: <link>

## 60-second summary

<one paragraph — what does the feature do, what does the user see, what upstreams does it depend on>

## Alerts you might have been paged for

Each alert here is a live link to the alert config in the alerting substrate.
Each alert names a specific playbook below.

| Alert                                              | Threshold                          | Playbook                                    |
| -------------------------------------------------- | ---------------------------------- | ------------------------------------------- |
| `<feature>.slo.burn_rate.fast`                     | 14.4× / 1h                         | Slow SLO — see Playbook A                   |
| `<feature>.slo.burn_rate.slow`                     | 6× / 6h                            | Slow SLO — see Playbook A                   |
| `<feature>.thumbs_down.rate.spike`                 | > 3× baseline for 15 min           | Bad prompt push or quality regression — Playbook B |
| `<feature>.provider.error_rate.high`               | > 2% for 5 min                     | Provider outage — Playbook C                |
| `<feature>.judge.calibration.drift`                | agreement < baseline for 24h       | Judge drift — Playbook D                    |
| `<feature>.guardrail.block_rate.spike`             | > 3× baseline for 15 min           | Guardrail FP spike — Playbook E             |
| `<feature>.trace_budget.burn_rate.fast`            | > 30% of monthly budget in 24h     | Trace budget — Playbook F                   |

## Playbook A — Slow SLO / burn rate

<the mitigation steps from [chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md), specialised to this feature: which fallback, which surface can be degraded, which kill switch>

## Playbook B — Thumbs-down rate spike

<specialised from chapter 04 Playbook 1 (bad prompt push) or Playbook 5 (silent quality regression)>

## Playbook C — Provider outage

<specialised from chapter 04 Playbook 2 — for THIS feature's primary provider, fallback provider, and residency constraints>

## ... etc for each playbook ...

## Kill switches

- **Full feature-off**: `<one command or one config change>`. Effect: users see <the graceful-degrade message>.
- **Fallback-provider-only**: `<command>`. Effect: primary provider is skipped; latency worsens by ~<X>%.
- **Retrieval-off**: `<command>`. Effect: feature answers without retrieval context; expect quality drop on <these use cases>.

Each kill switch has been triggered in staging within the last 90 days. Date of last exercise: <date>.

## Frequently useful queries

- Trace UI: <saved query for "recent errors on this feature">
- Trace UI: <saved query for "traces with app.feedback.signal = thumbs_down in the last 4 hours">
- Cost dashboard: <saved link>
- Feedback triage queue: <saved link>

## Common false alarms

<known-good behaviours the alert sometimes fires on incorrectly — spare the next on-call from re-diagnosing them>
```

### Two properties that separate real runbooks from decorative ones

**Every alert in the table links to a playbook, and every playbook has a first step that is one command.** Not "investigate the SLO dashboard" — a specific link to a specific saved query. Not "consider rolling back the prompt" — the exact command that rolls the prompt back, with the argument names filled in. At 3 a.m. the on-call reads for keywords and clicks; the runbook that requires them to reason is a runbook that produces longer incidents.

**Kill switches are tested regularly.** A kill switch that has not been exercised in six months is broken. The date-of-last-exercise field forces the discipline. When it goes stale, that itself is a task on the team's board.

### The runbook and the playbooks

The playbooks in [chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md) are generic — they apply to any LLM feature. The runbook in this chapter is *specific* — it takes each playbook and specialises it: this feature's primary provider, this feature's fallback, this feature's prompt-versioning tool, this feature's guardrail names. Playbooks live in the module's shared knowledge base; runbooks live in the feature's repository next to the code.

The rule: **playbooks change slowly; runbooks change with the feature.** When the feature swaps a provider, the runbook's Playbook C section is updated; the generic chapter-04 playbook is not touched.

## Interfaces to the platform / SRE team (`ai-infra-mlops`)

The applied-AI engineer does not run their own alerting substrate, their own pager rotation, their own status-page hosting, or their own incident-declaration flow. Those are owned by the platform / SRE / MLOps team at almost every organisation of any size — in this curriculum's org chart, that peer role is `ai-infra-mlops`.

Naming the specific interfaces makes the boundary usable.

### 1. Alerting substrate

The applied engineer defines the *alert rules* (thresholds, windows, severities); the platform team owns the *alerting engine* (Prometheus / Alertmanager, Datadog Monitors, Grafana Alerting, Chronosphere, whatever the org uses). The interface:

- The applied engineer writes alerts as code in the feature's repository, using the org's alerting DSL (Terraform, Jsonnet, YAML — vendor-specific).
- The alert file lives under `alerting/` in the feature's repository and is deployed by the platform team's CI.
- The platform team owns the ingest / storage / evaluation runtime; the applied engineer does not write to it, tune it, or scale it.

The output of the interface: alert rules land in the same alerting substrate as every other team's alerts. When something fires, it fires into the same paging system, with the same escalation, with the same on-call model. LLM features are not paged out-of-band; the LLM alert rings the same phone.

### 2. Pager rotation

The applied engineer defines *who* is on-call (their team, their schedule, their escalation); the platform team runs the *paging tool* (PagerDuty, Opsgenie, Grafana OnCall, VictorOps, Rootly). The interface:

- The applied engineer configures the rotation via the paging tool's API, referencing the alert rules from Interface 1.
- The applied engineer does not choose the paging tool or manage its billing.

### 3. Status-page hosting

The applied engineer owns the *content* of the per-feature status page; the platform team owns the *hosting infrastructure* (Statuspage.io, Instatus, an in-house build). The interface:

- The applied engineer's status-page shape is generated from a template checked into the feature's repository.
- The publish flow (rendering the template, refreshing the dashboards it embeds, updating the colour) is a CI pipeline owned by the platform team, running on their substrate.
- Incident declarations flow through the platform team's incident-declaration tool; the applied engineer declares the incident, the platform team's tool routes the notification to the status page's public banner.

### 4. Incident-management substrate

Every serious incident is declared in the org's incident-management tool (Rootly, incident.io, FireHydrant, PagerDuty Incidents, an internal build). The interface:

- The applied engineer's runbook names when to declare an incident (usually: a Sev-3-or-higher alert fired, or a customer-visible degradation is confirmed).
- The declaration itself is a command the on-call runs via the incident tool; the tool creates the Slack channel, pages the responders, records the timeline, writes the postmortem draft.
- The postmortem process — timeline, contributing factors, action items — is the platform / SRE team's process. The applied engineer participates, fills in the LLM-specific fields, closes the action items.

### 5. Metrics and traces backend

Covered in [chapters 01](01-otel-genai-and-openinference-span-vocabulary.md) and [02](02-sampling-pii-scrubbing-and-trace-budgets.md). The applied engineer's code emits spans and metrics; the platform team runs the backend (Grafana / Prometheus / Tempo, Datadog, Honeycomb, whatever). The interface is the OTel exporter configuration, deployed alongside the alerting rules.

### 6. What the applied engineer does NOT do

- Choose the alerting / paging / status-page vendor.
- Run the observability backend's storage tier.
- Author the org's general incident-response process (severity definitions, communication templates, executive-escalation policy).
- Run the pager rotation for their team (they configure and staff it; the paging tool runs it).

When the applied engineer's runbook interacts with any of these, it *links* to the platform team's page, it does not duplicate.

## Composition: the whole loop

Read together, the five chapters of this module compose one closed loop:

- [Chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) — every LLM call, retrieval, tool call, and guardrail emits a span with a fixed attribute vocabulary.
- [Chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) — the span stream is head-sampled and tail-sampled to fit a written budget, scrubbed for PII at the export boundary.
- [Chapter 03](03-user-feedback-triage-pipeline.md) — user thumbs / structured category / free text is joined to the trace, deduped, triaged, and — after expert review — added to the eval dataset from [mod-106](../mod-106-feature-side-evaluation/README.md).
- [Chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md) — the six recurring failure modes each have a playbook, detecting from the spans and feedback signals, mitigating by rolling back a versioned artefact.
- [Chapter 05](05-per-feature-status-page-and-on-call-runbook.md) — the whole thing composes into a per-feature status page for stakeholders and an on-call runbook for the rotation, both plugged into the platform team's substrate rather than re-inventing it.

The single through-line: **an LLM feature at applied-AI scale is not a research artefact; it is a shipped system with the same on-call, observability, and incident discipline as any other production service — plus the LLM-specific vocabulary of failure the general SRE playbook does not cover.**

## Anti-patterns

- **Status page that requires a human to flip the colour.** Decays to permanently green. Colour must be derived from an observable metric.
- **Runbook that assumes the reader wrote the feature.** The on-call did not. Write for the least-familiar engineer.
- **Runbook that lives in a Google Doc.** Not searchable, not versioned, not linkable from the alert. Put it in the repo next to the code.
- **Kill switches that have never been triggered.** They do not work. The date-of-last-exercise is the discipline.
- **Reinventing the alerting substrate for LLM alerts.** LLM alerts should ring the same pager as every other alert, through the same tool. The platform team's substrate is the substrate.
- **Status page and runbook in the same document.** Different audiences, different content, different update cadence. Split them.
- **Incident declared in Slack only, not in the incident tool.** The timeline is lost, the postmortem is incomplete, action items evaporate. Declare through the platform team's tool.
- **Runbook that lists all six generic chapter-04 playbooks verbatim.** They belong in the shared knowledge base. The runbook links to them and adds the feature-specific specialisation.

## Summary

- Two artefacts close the module: a **per-feature status page** for stakeholders (green / yellow / red, recent incidents, dependencies, one URL) and a **per-feature on-call runbook** for the rotation (alert-to-playbook table, kill switches, saved queries, common false alarms).
- The status page's colour is **derived** from observable metrics — SLO burn rate, active incidents, known-degraded posture — never toggled by a human.
- The runbook is written for the **least-familiar engineer** who could be on-call. Every alert links to a specific playbook; every playbook's first step is one command, not a paragraph.
- **Kill switches are tested regularly.** A kill switch not exercised in the last 90 days is presumed broken. The runbook's date-of-last-exercise field is the tripwire.
- The applied-AI engineer owns the **content** — alert rules, runbook text, status-page template, kill-switch definitions — and the platform / SRE team (`ai-infra-mlops`) owns the **substrate**: alerting engine, paging tool, status-page hosting, incident-management tool, observability backend.
- The five chapters of this module compose one closed loop: instrument (01) → sample and scrub (02) → capture feedback (03) → playbook the incidents (04) → publish the status page and runbook (05). The loop is what turns a working demo into a shipped LLM feature.
