# exercise-05: The Per-Feature Status Page and On-Call Runbook

**Estimated effort:** 3 hours

## Objective

Compose the two load-bearing artefacts from [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md) for the same feature you have carried through exercises [01](exercise-01-otel-genai-instrumentation-for-a-feature.md)–[04](exercise-04-incident-playbooks-for-six-llm-failure-modes.md): a **per-feature status page** (audience: stakeholders — PM, sales, support, executives, sometimes external customers) and a **per-feature on-call runbook** (audience: the rotation engineer holding the pager, who probably did not write the feature). Both plug into the platform / SRE team's substrate rather than reinventing it. The exercise ends when a teammate — from another team, not on your rotation — can open the runbook cold and route a page to mitigation in under fifteen minutes.

## Learning outcomes

By the end you should be able to:

- Compose a status page whose green / yellow / red colour is **derived from observable metrics** (SLO burn rate, active incidents, known-degraded posture), not toggled by a human.
- Write an on-call runbook for the **least-familiar engineer** who could ever be on-call — every alert links to a specific playbook, every playbook's first step is one command, every kill switch has a last-exercised date.
- Name the interfaces to the platform / SRE / MLOps team owned by `ai-infra-mlops` (alerting substrate, pager rotation, status-page hosting, incident-management, metrics/traces backend) so the feature does not re-invent them.
- Prove the runbook works via a **cold-read drill**: a teammate reads only the runbook and its links, and mitigates a simulated page within the target window.

## Prerequisites

- You have read [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md).
- Exercises [01](exercise-01-otel-genai-instrumentation-for-a-feature.md)–[04](exercise-04-incident-playbooks-for-six-llm-failure-modes.md) are complete: the feature is instrumented, sampled and scrubbed, has a feedback pipeline joined to the trace, and has six written playbooks with at least three game-day reports.
- Access to the feature's alerting substrate (Prometheus / Alertmanager, Datadog Monitors, Grafana Alerting, Chronosphere — whichever the org uses) and the paging tool (PagerDuty / Opsgenie / Grafana OnCall / Rootly / VictorOps).
- Access to the org's status-page substrate (Statuspage.io, Instatus, an internal build) or, for an internal-only page, whichever page-hosting substrate the org uses.
- Access to the org's incident-management tool (Rootly, incident.io, FireHydrant, PagerDuty Incidents, or an in-house build).
- The SLO from [mod-109 chapter 05](../../mod-109-cost-latency-slos-for-llm-features/05-retries-hedging-slos-and-burn-rate-alerts.md) is written and has burn-rate alerts wired.
- A teammate — from an adjacent team, ideally not someone who has read your feature's code — willing to spend 30 minutes on the cold-read drill.

## Problem statement

Your feature has an SLO, six written playbooks, an instrumentation stack, and a feedback pipeline. What it does not have is the surface a stakeholder or an on-call opens when something is wrong. If the CEO asks "is the support agent down right now?" the answer today is "let me ask the team." If a page fires at 3 a.m. into someone else's rotation because your team's on-call is unreachable, the receiving engineer has no single URL to open and no idea which of the six playbooks the alert corresponds to.

In three hours: publish the status page whose colour is derived from the SLO burn state, write the runbook a stranger can navigate under duress, and prove both work — the page by wiring it to a live signal, the runbook by watching a teammate execute it cold.

## Requirements

### Deliverable 1 — The per-feature status page template

Under `status/status-page.md.tmpl` (or in the org's status-page substrate — Statuspage.io template, Instatus component, etc.), the shape from [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md):

- A single-line **current status** — 🟢 GREEN / 🟡 YELLOW / 🔴 RED — timestamped, refreshed on a schedule (not by hand).
- A banner region for human-written narration on YELLOW / RED. Empty on GREEN.
- A **recent incidents table** — last 90 days, one line per incident: date, severity, duration, one-sentence summary.
- A **dependencies section** — upstream provider (primary + fallback), retrieval index and refresh cadence, guardrail layers with versions, owning team, rotation link.
- A **links row** — SLO doc, trace budget doc, feature repo, internal runbook (link only; not exposed on a public page).

The template renders per environment (prod, EU-prod, staging-facing-partners if applicable). It does *not* embed dashboards, metrics charts, or debugging surfaces — those live behind the internal runbook link.

### Deliverable 2 — The colour-derivation rules

Under `status/colour-rules.md`, name the exact metrics that flip the colour:

- **Green**: SLO fast-burn window not firing; no active incident declared in the incident tool; no known-degraded posture recorded.
- **Yellow**: SLO burn rate above 1× but below the fast-burn threshold, *or* a kill switch tripped for a sub-surface, *or* running on the fallback provider.
- **Red**: an incident is declared in the incident tool at Sev-2 or higher, *or* the SLO's fast-burn window is firing.

For each state, the exact metric expression (`slo:<feature>:burn_rate_1h > 14.4`, `incident_status{feature=...} = 'declared'`, etc.) and the pipeline that reads it. A human on-call **cannot override to green**; they can only narrate the banner text.

Ship a small script (or a scheduled job — GitHub Action, cron, whatever the org has) that reads the three signals every N minutes and updates the page's colour. Idempotent; safe to run twice.

### Deliverable 3 — The on-call runbook

Under `runbooks/oncall.md` — checked into the feature's repository, linked from every alert:

- A **60-second summary** — one paragraph naming what the feature does, what the user sees, and what upstreams it depends on. Written for someone who has never touched the code.
- A **contacts block** — owner team, rotation link, escalation ladder (team lead → VP eng), repo URL, status-page URL, dashboards URL.
- The **alerts-to-playbooks table** using the shape from [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md). Every row: the alert name (live link to the alert config), the threshold, and the specific playbook name below.
- The **six playbook sections**, each specialised from the generic chapter-04 playbook by [exercise-04](exercise-04-incident-playbooks-for-six-llm-failure-modes.md). Each specialisation names *this* feature's primary provider, its fallback, its prompt-versioning tool, its guardrail names — do not copy the generic chapter text.
- A **kill-switches section** from [exercise-04](exercise-04-incident-playbooks-for-six-llm-failure-modes.md) Deliverable 4. Each switch has: the one-command action, the user-visible effect, and the date it was last exercised in staging. Anything not exercised in 90 days is flagged.
- A **frequently-useful-queries section** — saved trace-UI queries, cost dashboard link, feedback triage queue link, judge-agreement dashboard link. Each is a *link*, not a query string the on-call has to paste.
- A **common false alarms section** — known-good behaviours the alerts sometimes fire on incorrectly. Spare the next on-call from re-diagnosing them.

Two invariants:

- **Every alert row in the table links to a playbook by name.** No orphan alerts. If an alert has no playbook, either write one or delete the alert.
- **Every playbook's first mitigation step is one command or one link.** Not "investigate the SLO dashboard." The specific saved query URL.

### Deliverable 4 — The interfaces-to-platform document

Under `runbooks/platform-interfaces.md`, name the six interfaces from [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md) as they exist at your org (or the scaffolded org in the solutions repo). For each: what the feature team owns, what the platform team owns, and the specific handoff artefact.

- **Alerting substrate** — the applied engineer owns the alert rules (in `alerting/` in the feature repo); the platform team owns the alerting engine. Name the alerting engine at your org and the deployment path from the feature repo to it.
- **Pager rotation** — the applied engineer configures the rotation via the paging tool's API and staffs it; the platform team runs the tool. Name the paging tool and link the rotation.
- **Status-page hosting** — the applied engineer owns the template content; the platform team owns the hosting substrate. Name it and link the publish pipeline.
- **Incident-management substrate** — the applied engineer declares incidents through the platform team's incident tool. Name the tool and link the declaration flow. Name the severity threshold at which the runbook says "declare an incident."
- **Metrics / traces backend** — the applied engineer's code emits spans (per [exercise-01](exercise-01-otel-genai-instrumentation-for-a-feature.md)); the platform team runs the backend. Name the backend.
- **What the applied engineer does NOT do** — vendor selection, backend storage tier, org-wide incident process authoring, running the pager tool itself. Named in one short list.

This document is the applied-AI engineer's contract with the `ai-infra-mlops` peer role. It is *also* how you know what to escalate to them (an alerting-substrate feature request) vs what to build yourself (a new alert rule).

### Deliverable 5 — Wire the runbook to the alerts

For every alert row in the runbook's alerts table, edit the alert config so its annotations include a link to the runbook and to the specific playbook section:

```yaml
# Example — Prometheus / Alertmanager style
- alert: SupportAgentSLOBurnFast
  expr: slo:support_agent_v3:burn_rate_1h > 14.4
  labels:
    severity: page
    feature: support_agent_v3
  annotations:
    summary: "support_agent_v3 SLO burning fast (>14.4x)"
    runbook_url: "https://.../runbooks/oncall.md#playbook-a-slow-slo"
    playbook: "A — Slow SLO"
```

The exact syntax depends on the alerting engine. What matters: the on-call receives the page, taps the runbook link, and lands on the correct playbook section, *not* on the runbook's table of contents.

### Deliverable 6 — The cold-read drill

Pick one of the game-day scenarios from [exercise-04](exercise-04-incident-playbooks-for-six-llm-failure-modes.md), or set up a new one (deploy a regressed prompt to staging; block the primary provider at the network layer; nudge the guardrail threshold; etc.). Then:

1. Have a teammate — **from a different team, who has not read the feature's code** — sit down cold with only the runbook URL and their laptop.
2. Fire the alert into staging so it hits the paging tool the way a real page would.
3. Ask the teammate to open the page, follow the runbook link, execute the correct playbook, and reach mitigation.
4. Time the whole thing from page-received to mitigation-confirmed.
5. Write a report under `runbooks/drills/YYYY-MM-DD-cold-read.md`:

```markdown
# Cold-read drill — <date>

## Scenario
<what was fired, on which environment>

## Reader
<name and their team>. Prior exposure to this feature: <none / meeting / brief chat / ...>.

## Timeline
- HH:MM — page received.
- HH:MM — runbook opened.
- HH:MM — correct playbook identified.
- HH:MM — mitigation command run.
- HH:MM — mitigation confirmed (metric returned to normal or incident closed).

## What worked
<parts of the runbook that were unambiguous and executable>

## What broke
<parts that required guessing, external Slack pings, or Googling>

## Actions
- [ ] <edit to the runbook>
- [ ] <edit to an alert>
- [ ] <edit to a playbook>
```

The drill's target: page-to-mitigation under **15 minutes** for the failure modes with a one-command rollback (bad prompt push, guardrail FP spike), and under **30 minutes** for the ones that require a decision (provider outage — which fallback? at what capacity?). If the drill runs long, the runbook is the bug, not the reader.

## Starter guidance

- **The status page is not a dashboard.** Do not embed metrics. It answers three questions: available? recent history? current known degradation? Nothing else. A status page that tries to be a metrics viewer is one that stakeholders stop reading.
- **The runbook lives next to the code.** Not in a Google Doc, not in the wiki, not in Notion. In the repo, as markdown, versioned with the code. When the feature swaps a provider, the runbook change is in the same PR.
- **The colour must derive, not toggle.** The most common failure of status pages is "someone forgot to flip it back to green." A cron job reading a metric never forgets.
- **The cold-read drill's reader is deliberately unfamiliar.** If you pick a teammate who already knows the feature, the tacit-knowledge gap is invisible and the drill is falsely reassuring. Pick someone outside your rotation.
- **Do not skip the platform-interfaces doc.** It is one page and it is the artefact you cite when the platform team asks "what should we prioritise for the observability roadmap." Teams that ship it get their asks funded; teams that do not, do not.
- **Link the runbook from the alert.** If the on-call has to search for the runbook, the runbook is invisible. The paging tool must render the URL in the notification body.
- **Do not run the cold-read drill in production.** Staging only, on a scenario that cannot page real users. If the drill breaks because staging does not have enough fidelity, that itself is a finding.
- **The kill-switch dates decay.** Add a monthly recurring task (in the org's task tool) to re-exercise any switch older than 90 days. A stale kill switch is a broken kill switch.

## Acceptance criteria

- [ ] `status/status-page.md.tmpl` (or the org-substrate equivalent) renders per environment with the shape from [chapter 05](../05-per-feature-status-page-and-on-call-runbook.md).
- [ ] `status/colour-rules.md` names the exact metric expression for each of green / yellow / red; a scheduled job flips the colour based on those signals.
- [ ] A human cannot override the colour to green while the metric says otherwise (banner text is separate).
- [ ] `runbooks/oncall.md` lives in the feature repo and contains: 60-second summary, contacts, alerts-to-playbooks table, six specialised playbook sections, kill switches with last-exercised dates, saved queries, common false alarms.
- [ ] Every alert row in the runbook's table links to a playbook by name; every playbook's first mitigation step is one command or one link.
- [ ] Every alert config's `runbook_url` (or equivalent) points to the specific playbook section, not the runbook root.
- [ ] `runbooks/platform-interfaces.md` names the six interfaces to `ai-infra-mlops` at the org, with the specific tool for each and what is owned where.
- [ ] `runbooks/drills/YYYY-MM-DD-cold-read.md` shows a real drill: an outside-team reader, a timed page-to-mitigation, a "what worked / what broke" section, and follow-up action items.
- [ ] The drill was run in staging; no real users were paged.
- [ ] The cold-read drill's page-to-mitigation time meets the target (15 min for one-command rollbacks; 30 min for decision-requiring modes) OR the report names the specific runbook gap that caused the miss.

## Stretch goals

Pick any one:

- **Public status page.** Publish the page on the org's external status-page substrate with the banner ready to be translated for customer language. Route the incident-declaration flow to update the public banner automatically. Note whether legal / support signoff is required in your org.
- **Auto-generated recent-incidents table.** Wire the incident-management tool's API to write directly into the recent-incidents table on close. The applied engineer's only job becomes writing the one-sentence summary field; everything else derives.
- **Cross-feature runbook consolidation.** If your team owns two or three LLM features, pull the *shared* sections (provider fallback commands, kill-switch conventions, saved queries) into a `runbooks/shared/` directory that each feature's runbook links into. Reduces drift; keeps the feature-specific specialisation on top.
- **A "runbook staleness" tripwire.** For each playbook and each kill switch, record the last time it was executed (real incident or drill). Alert when any playbook has not been touched in > 6 months or any kill switch has not been exercised in > 90 days. That is how playbooks stop rotting.
- **Runbook accessibility from the paging tool alone.** Configure the paging tool to render the top of the runbook (60-second summary + alerts table) directly in the notification, so the on-call sees the essentials without leaving the paging app. The full runbook is one tap away.

## Deliverable

Submit as `mod-110/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `status/status-page.md.tmpl` (or equivalent) and `status/colour-rules.md`.
- The scheduled job / script that derives the colour.
- `runbooks/oncall.md`.
- `runbooks/platform-interfaces.md`.
- `runbooks/drills/YYYY-MM-DD-cold-read.md`.
- The alert-config diff showing every alert now carries a runbook URL to a specific playbook section.
- A short `README.md` naming the feature, the linked status-page URL, the linked rotation, and the top three findings from the cold-read drill.
