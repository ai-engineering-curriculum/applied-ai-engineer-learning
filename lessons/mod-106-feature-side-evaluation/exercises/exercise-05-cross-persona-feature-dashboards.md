# exercise-05: Cross-Persona Feature Dashboards

**Estimated effort:** 2 hours

## Objective

Design and ship **three dashboards** — one for a PM, one for an on-call engineer, one for a safety reviewer — on the eval substrate you built across [exercises 01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md)–[04](exercise-04-sampled-online-eval-loop-with-judge-drift-monitor.md). Each dashboard answers *this persona's* question at *this persona's* cadence in *this persona's* preferred tool. Additionally, produce a one-page peer-role handoff document naming which dashboards the applied engineer explicitly does *not* build.

## Learning outcomes

By the end you should be able to:

- Map a stakeholder question to a small set of charts, at the tolerance-for-detail that stakeholder actually operates at.
- Build a PM-facing dashboard whose five panels a five-minute weekly skim can absorb.
- Build an on-call-facing dashboard that obeys the alert and links to the rollback tool.
- Build a safety-reviewer dashboard that is artefact-oriented (each row links to concrete traces), not chart-only.
- Explicitly document the four peer-role dashboards the applied engineer defers to — judge-vs-human calibration, bootstrap CIs, judge behavioural characterisation, comprehensive safety benchmarks.

## Prerequisites

- You have completed [exercise-04](exercise-04-sampled-online-eval-loop-with-judge-drift-monitor.md) (online loop writing per-dimension scores back to the observability store).
- You have access to a BI / dashboarding tool the personas already use — Grafana, Metabase, Looker, a Notion-embedded dashboard, or the eval runner's native dashboard surface (Braintrust, Langfuse, LangSmith, Phoenix, Weave).
- You have written access to (or can mock) the alerting / runbook / launch-memo systems the dashboards integrate with.

## Problem statement

The eval loop produces useful numbers. The PM cannot find them; the on-call opens the wrong dashboard at 2am; the safety reviewer keeps asking for the same one-off report every week. Your task is to build the three views that make the numbers *land* — each in the tool the reader already uses, each at the right altitude, each governance-linked to a decision.

You have two hours. This is a design-and-build exercise, not a novel-eval-work exercise; the underlying data is already in the store.

## Requirements

### Deliverable 1 — The PM dashboard

Under `dashboards/pm/`:

- A dashboard with **five panels** in the order from [chapter 05](../05-cross-persona-feature-dashboards.md): headline quality trend, per-persona quality trend, feedback rate, volume/adoption, cost-per-successful-outcome.
- 90-day window on trend panels; annotations for each prompt / model / retriever release visible on the timeline.
- No composite score without a written definition. No per-tool-call token counts. No p99 latency. Anything the PM cannot act on is out.
- A one-paragraph header naming what to look at first ("headline quality by persona; if the enterprise-cohort line drops, escalate").
- Link the dashboard from the weekly product-review agenda template you'd propose (`dashboards/pm/embed.md`).

### Deliverable 2 — The on-call dashboard

Under `dashboards/oncall/`:

- A dashboard that opens *already scoped* to the surface (query params in the URL). The on-call's page contains this URL.
- Panels: current SLO status (green/yellow/red at a glance), last-24h of the four dimensions with deploy / flag-flip / provider-incident annotations, segment-scoped breakdown of the current alert, live-updating list of the last 20 failing traces (each linked), rollback panel with the last five deploys and their eval-CI comments.
- The rollback panel links to the runbook (`dashboards/oncall/runbook.md`) — which itself links back to this dashboard.
- Prove drillability: from a failing trace's tile, one click reaches the raw trace view in the observability store.

### Deliverable 3 — The safety-reviewer dashboard

Under `dashboards/safety/`:

- Panels: safety-spot-check pass rate over time by category, new-incident queue (last 7 days, artefact-oriented), fixture-coverage table (per category: count, last rotation, pass rate), guardrail activation stats, and a "deferred to peer role" list (transparency, not to-do).
- Every count is a *link*, not a summary. Clicking "12 injection incidents this week" opens the 12 traces.
- A `dashboards/safety/signoff-template.md` that the safety reviewer fills in before every rollout, referencing the dashboard by URL, listing which categories they reviewed and what evidence they accepted.

### Deliverable 4 — Peer-role handoff document

Under `docs/peer-handoffs.md`, one page:

- The four dashboards the applied engineer explicitly does not build: judge-vs-human calibration heatmap, bootstrap-CI + sequential-test overlays, judge-model behavioural characterisation, comprehensive safety-benchmark surface.
- For each: which peer role owns it (`ai-eval-engineer`, `model-evaluation-engineer`, `ai-safety-engineer`), where to find it (URL or a note that it does not exist yet), and how the applied engineer's dashboards *read from* it.
- A named contact rotation (a role, not a person, so it survives turnover).
- A short paragraph on the "don't duplicate" discipline — why the applied engineer building a shabby judge-vs-human calibration surface is worse than not having one.

### Deliverable 5 — Governance wiring

Under `docs/governance.md`, one page:

- **Launch-decision checklist.** A launch memo template that requires links to the PM dashboard's current state *and* the safety-reviewer signoff.
- **Post-incident retro linkage.** The retro template's "how did the dashboards do?" section, with action items that create dashboard changes when the dashboards failed to surface the incident.
- **Quarterly dashboard review.** A calendar-invite template (or a documented cadence) for the quarterly review — what got added, what got removed, who owns it.

## Starter guidance

- **Build the on-call dashboard first.** It's the highest-stakes reader and the most concrete brief. Once you can name what the on-call sees at 2am, the other two are relatively easier.
- **Reuse the observability store's native features.** Every named runner ships some dashboarding surface. Use it before you build a Grafana instance. The best dashboard is the one your team is already logged into.
- **Make every dashboard have an owner.** A dashboard with no name on the header rots. The header is `Owner: applied-eng-oncall (rotation)`, not `Owner: TBD`.
- **Aggregate before you chart.** Per-trace judged scores are noisy. Chart rolling medians (7-day, 24h) or cohort medians, not raw per-trace values.
- **Meet each persona in their tool.** Do not force the PM into Grafana. Do not force the safety reviewer into Braintrust. Embed / cross-link into the tool that persona already uses.
- **Don't chase completeness.** A dashboard with 40 charts is a dashboard nobody reads. Five panels, each load-bearing, each linked to a decision.

## Acceptance criteria

- [ ] `dashboards/pm/` contains a dashboard (or a config that reproducibly builds one) with the five specified panels and a header naming what to look at first.
- [ ] `dashboards/oncall/` contains a dashboard scoped to the surface, obeys the alert, and links to the rollback tool.
- [ ] `dashboards/safety/` contains an artefact-oriented dashboard: every count is a link to the concrete traces.
- [ ] `docs/peer-handoffs.md` names the four peer-role dashboards, their owners, and the "don't duplicate" discipline.
- [ ] `docs/governance.md` names the launch checklist, post-incident retro linkage, and quarterly review cadence.
- [ ] Every dashboard has a named owner (a rotation or a role, not an individual).
- [ ] No dashboard reports a composite "quality" score without a written definition.
- [ ] The on-call dashboard's URL is embedded in the alert body (from [exercise-04](exercise-04-sampled-online-eval-loop-with-judge-drift-monitor.md))'s alerting rules and in the runbook.
- [ ] The safety-reviewer dashboard's signoff template is referenced in the launch checklist.
- [ ] No chart plots raw per-trace judge scores without an aggregation.

## Stretch goals

Pick any two:

- **A dashboard-usage audit.** Log which panels were viewed over the last quarter. Panels with zero views for two quarters get retired at the next quarterly review. Automates the "dashboards rot" problem.
- **A launch-memo generator.** A CLI that, given a proposed launch date and a surface name, fetches the PM dashboard's current state, the safety reviewer's last signoff date, and any open alerts, and produces a launch memo skeleton with green/yellow/red assessments the applied engineer edits and sends.
- **A shared-with-peer-role annotation layer.** Cross-annotate the PM and safety dashboards with events from the peer roles' work (a re-calibrated judge; a new benchmark result; a discovered bias category). Sudden score movements that line up with peer-role events are legible; ones that don't get triaged.
- **A "did the dashboard fail?" retro appendix.** For the next incident that hits this surface, produce the standard retro appendix: what did the on-call dashboard show at the alert time; what did it miss; what should have shown up. Add the appendix to the retro template.
- **A safety-reviewer weekly digest.** A generated Markdown summary of the safety dashboard emailed weekly to the reviewer (via a repo-hosted script, not a live SaaS). Includes the new-incident queue, fixture-coverage deltas, and any pin expirations coming up.

## Deliverable

Submit as `mod-106/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `dashboards/pm/` — dashboard config (JSON / YAML export from the tool used), `embed.md`, an ownership note in the header.
- `dashboards/oncall/` — dashboard config, `runbook.md`, the alert-body template that embeds the URL.
- `dashboards/safety/` — dashboard config, `signoff-template.md`.
- `docs/peer-handoffs.md`, `docs/governance.md`.
- A short `README.md` naming the tool used for each dashboard, the ownership rotation, and how a reader would land on each dashboard in real use.
- Any stretch-goal artefacts named alongside.
