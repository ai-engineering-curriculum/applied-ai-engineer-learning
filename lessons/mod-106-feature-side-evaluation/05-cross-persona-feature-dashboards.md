# Cross-Persona Feature Dashboards

## Different readers, different questions, one substrate

The eval pipeline built across [chapters 01](01-offline-eval-suites-a-pr-can-run.md)–[04](04-sampled-online-eval-loop.md) produces a lot of numbers. If the numbers only live in the eval runner's default UI, the applied engineer is the only person who ever reads them. That is a failure of the dashboard, not of the people who did not read it.

This chapter is about designing dashboards *for the three humans who actually need to look at your feature's quality every week*: the PM who is deciding what to build next, the on-call engineer who has to answer a page in the middle of the night, and the safety reviewer whose sign-off gates a rollout. Each reads for a different question, on a different cadence, with different tolerance for detail. One dashboard per persona, one underlying substrate.

The peer role split from earlier chapters still applies. The applied engineer is *not* building the statistical-rigour surface — the bootstrap CI overlays, the sequential-testing views, the judge-vs-human calibration heatmaps. Those are the `ai-eval-engineer` / `model-evaluation-engineer` peer roles' surfaces, and they read a different dashboard entirely. The applied engineer's dashboards answer product and operations questions; the peer roles' dashboards answer measurement-methodology questions. Both exist. Neither replaces the other.

## Who reads what, and when

Before any chart is designed, name the reader.

| Persona            | Reads on the cadence of                    | Reads to decide                                                                              | Tolerance for detail |
| ------------------ | ------------------------------------------ | -------------------------------------------------------------------------------------------- | -------------------- |
| **PM**             | Weekly + before every launch decision      | *What to build next; whether to launch; whether a change moved a user-facing metric.*        | Low; chart-first     |
| **On-call eng**    | On page + at shift handoff                 | *Whether the surface is currently degraded; what changed; how to roll back.*                 | High; drillable      |
| **Safety reviewer**| Weekly + before every safety-relevant rollout | *Whether known harms are being caught; whether new harms are emerging; whether to sign off.* | High; artefact-oriented |

Three views, one substrate. The substrate is the observability store from chapter 03 (Braintrust, Langfuse, LangSmith, Phoenix, Weave, plus the raw traces). The views are three separately-designed dashboards, each in the tool the persona already uses (Notion / Linear / a BI tool for the PM; PagerDuty + a runbook + the trace UI for the on-call; a compliance-tracker / doc tool for the safety reviewer). Do not force everyone into the eval runner's default UI. Meet each persona where they already look.

## The PM dashboard

The PM's question is "*is the feature getting better on things users care about, and how does that compare to what we shipped?*" The PM is not going to click through 50 fixtures. The PM wants five charts that a five-minute weekly skim can absorb.

**Charts, in the order they should appear:**

1. **Headline quality trend.** Median rubric score (or the composite the PM has agreed to) over the last 90 days, with annotations for every prompt / model / retriever release. The step-changes should line up with the annotations. If they don't, that's a conversation.
2. **Per-persona quality trend.** The same metric, split by the personas the product ships to (free / paid / enterprise; new / returning; language cohort; verticals). A drop hidden in one segment is invisible on the aggregate; this chart is what surfaces it.
3. **Feedback rate.** Thumbs-down / thumbs-up rate over time. This is the user's own vote. When it diverges from the quality metric, one of them is lying — usually the quality metric, and usually because the rubric no longer reflects what users care about.
4. **Volume and adoption.** Weekly active users of the feature, requests per user, and (for surfaces where it matters) task completion rate. Quality with no adoption is a launch failure; adoption with no quality is a churn incident forming.
5. **Cost-per-successful-outcome.** Cost per completed task, not cost per API call. The PM's cost question is unit economics, not token spend.

**What NOT to put on the PM dashboard.** Judge disagreement rates. Position-bias measurements. Retriever recall at k. p99 latency in milliseconds. Per-tool-call token counts. Every one of those is important; none of them belongs on the PM dashboard, because the PM cannot act on any of them. They belong on the on-call dashboard.

**Cadence and framing.** The PM dashboard is reviewed at the weekly product review. It is also the artefact attached to a launch decision. A launch memo that cites "the PM dashboard shows +8% quality on the enterprise cohort over the last four weeks, no per-persona regression, cost-per-successful-outcome flat" is a much stronger memo than "we ran the eval and it looks good."

## The on-call engineer dashboard

The on-call's question is "*something's on fire — what, where, and how do I stop it?*" The on-call is looking at this dashboard at 2am, after a page fired. They need drillability, not aesthetic.

**Charts and panels, top to bottom:**

1. **Current SLO status.** The three or four SLOs the surface commits to (mod-109 owns the SLO framing; this dashboard shows the current burn). Green / yellow / red at a glance.
2. **The last 24 hours of the four dimensions.** Quality, safety incident rate, cost-per-request, latency p50/p95/p99. Overlaid with deploys, feature-flag flips, and provider incidents (from your provider status page).
3. **Segment breakdown of the active alert.** If the alert fired on "quality drop on non-English cohort," this panel shows *that segment*. Not the aggregate. The dashboard obeys the alert.
4. **The most-recently-failed traces.** A live-updating list of the last 20 traces whose quality or safety score fell below threshold, each linked to the trace UI. The on-call opens two of them and forms a hypothesis about the failure mode.
5. **The rollback panel.** The last five deploys / prompt changes / config flips, with the eval-CI comment from that PR (chapter 03), a one-click "revert" link where automation supports it, and the runbook entry pinned. When the on-call decides to roll back, this panel is the tool.

**The runbook links to this dashboard, not the other way around.** The on-call's page includes the URL. The URL loads the dashboard already scoped to the surface and the alert. The on-call should not spend the first three minutes of an incident finding the right dashboard.

**What NOT to put on the on-call dashboard.** Long-term trend charts. Cohort segmentation for its own sake (only include the segment the alert names). Fixture-level drilldowns (that's the safety reviewer's material). PM-facing narrative annotations. The on-call is looking for signal to act on, not context to reflect on.

## The safety reviewer dashboard

The safety reviewer's question is "*are the known harms still contained; are new ones emerging; can I sign off on this rollout?*" The safety reviewer works at a slower cadence than the on-call but a deeper cadence than the PM.

**Charts and panels:**

1. **Safety-spot-check pass rate over time.** The chapter 01 mode-5 fixtures, per category (prompt injection, sensitive-info disclosure, refusal-to-refuse, jailbreak drift). A drop in any category is the story.
2. **New-incident queue.** Any online-loop trace tagged `safety` — hijack, injection success, unsafe generation — from the last 7 days. Each row: timestamp, category, snippet, trace link, current triage state (new / triaged / promoted-to-fixture / fixed / accepted-risk).
3. **Fixture-coverage table.** For each safety category, the count of fixtures in the offline suite, the last time each was rotated, and the pass rate. Categories with low coverage are the reviewer's next request for the applied engineer.
4. **Guardrail activation stats.** For each guardrail (mod-107), how often it fired, on which inputs, and the outcome. A guardrail with zero activations may be dead code; a guardrail with a suddenly higher activation rate is either working overtime or misfiring.
5. **Deferred-to-peer decisions.** A short list of the currently-open questions the applied engineer has explicitly deferred to the `ai-safety-engineer` / `ai-eval-engineer` peer roles. This is not a to-do; it is a *transparency* item — the safety reviewer sees what is not being decided at the applied-engineer altitude.

**Artefact-oriented, not chart-only.** The safety reviewer wants to see the actual failing prompts and the actual model outputs, not summaries. Every category count is a link to the concrete traces. The reviewer signs off on evidence, not on aggregates.

## The peer-role dashboards this chapter does NOT build

The applied engineer's dashboards stop at the *product* altitude. Four dashboards belong to the peer eval-engineering / safety-engineering roles and should be built (or borrowed) *by them*, not by the applied engineer:

1. **The judge-vs-human calibration heatmap.** Per rubric dimension, agreement between the LLM judge and the human graders, over time. This is the peer role's core methodology surface. The applied engineer *reads* it (to know whether to trust their own scores) but does not maintain it.
2. **Bootstrap-CI and sequential-test overlays** on top of every quality metric. The applied engineer looks at the median; the peer role looks at the confidence bound. When a launch decision hinges on statistical significance, the applied engineer defers to the peer role's read.
3. **Judge-model behavioural characterisation.** Position-bias / length-bias / self-preference measurements per judge model, over time. The peer role publishes these; the applied engineer reads and pins.
4. **The comprehensive safety-benchmark surface.** Full runs of the current public safety benchmarks against the surface's model configuration. The peer role runs these; the applied engineer receives the report.

**Why the split matters.** Every one of the four peer surfaces takes real engineering effort and specialised knowledge. Duplicating them at the applied-engineer altitude — because "we should really do our own calibration" — is how a small applied team burns a month on a bad-quality version of the peer role's already-built work. The right move is a documented dependency: the peer role's dashboards are linked, their assumptions are noted, their contact rotation is named.

## Governance the dashboards enforce

A dashboard's second job (after being read) is to *make policy real*. Three governance shapes worth naming:

**Launch-decision checklist.** A launch that passes the PM dashboard's headline chart but *has not been reviewed against the safety reviewer's dashboard for the current sprint* is not a launch. The launch memo template includes both links; a missing safety-review link is a soft block. This is the "safety reviewer can sign off" contract, made mechanical.

**Post-incident retro linkage.** Every safety incident retro (mod-110's territory) references the on-call dashboard state at the moment of the incident. If the dashboard didn't surface the incident, the retro's action items include a dashboard change — usually a new alert or a new segment.

**Quarterly review of what's on the dashboards.** Once a quarter, each dashboard is reviewed for *what's still there that shouldn't be* (a chart nobody looks at) and *what's missing that should be* (a segment the last quarter's incidents surfaced). Dashboards rot; the review keeps them honest.

## Anti-patterns

Five worth naming.

1. **The one-dashboard-for-everyone.** A shared dashboard with 40 charts. Each persona ignores 30 of them. Nobody notices when the two charts they care about drift.
2. **The judge-noise dashboard.** Per-trace scores plotted directly. The noise floor is high; readers stop trusting the dashboard because "the numbers jump around every day." Aggregate to rolling medians before charting.
3. **The uncalibrated composite score.** A single "quality" number derived by averaging six rubric dimensions with unstated weights. Nobody knows what would move it, so nobody trusts a move. Chart the dimensions; chart the aggregate only when it has a shared, written meaning.
4. **The dashboard nobody owns.** Charts point at data that broke six weeks ago. Nobody's job to fix it. Every dashboard has an owner; the owner's name is on the header.
5. **The peer-role territory grab.** The applied engineer builds a shabby judge-vs-human calibration surface because "the peer team is busy." The output is worse than the peer role's, distracts the applied team from their real work, and creates two contradictory sources of truth. Defer, link, name the dependency.

## Summary

- The eval pipeline's numbers only matter when they land on a dashboard the reader actually reads. Design one per persona, in the tool that persona already uses.
- The PM dashboard is chart-first, five panels, cadence weekly. It answers "is the feature getting better on what users care about, and how did our changes move it?"
- The on-call dashboard is drillable, obeys the alert, and links to the rollback tool. It answers "what's on fire, where, and how do I stop it?" at 2am.
- The safety-reviewer dashboard is artefact-oriented and category-broken-out. It answers "are known harms still contained, are new ones emerging, can I sign off?"
- Four dashboards explicitly belong to the peer eval / safety-engineering roles — judge-vs-human calibration, bootstrap-CI overlays, judge behavioural characterisation, comprehensive safety benchmarks. Link them; don't rebuild them.
- Dashboards carry governance: launch checklists, incident-retro linkage, quarterly reviews of what's on them.
- Five anti-patterns to avoid: the one-dashboard-for-everyone, the judge-noise dashboard, the uncalibrated composite score, the un-owned dashboard, the peer-role territory grab.

The five chapters of this module together answer the applied engineer's version of the question *"is this feature good?"*: a pre-registered offline suite that a PR must pass, trace snapshots that keep the suite representative, an eval-gated CI pipeline that turns the suite into a merge gate, a sampled online loop that catches drift before an incident, and a set of dashboards that let the humans downstream act on what the loop finds. The methodology behind the numbers — calibration, bootstrap, sequential testing — belongs to the peer role. The applied engineer's job is to make the numbers *land*.
