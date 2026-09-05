# mod-110-observability-feedback-and-incidents: Observability, User Feedback, and Incident Response for LLM Features

**Estimated effort:** 13 hours

An LLM feature at applied-AI scale is not a research artefact — it is a shipped system with the same on-call, observability, and incident discipline as any other production service, *plus* an LLM-specific failure vocabulary the general SRE playbook does not cover. The module carries a single through-line: **instrument every LLM call, retrieval, tool call, and guardrail as spans with a shared attribute vocabulary; sample and scrub inside a written trace budget; join user feedback to the traces and route it to expert review; playbook the six recurring LLM failure modes; and compose the whole thing into a per-feature status page and on-call runbook that plug into the platform team's substrate, not around it.**

Peer-role boundaries are named up front. `ai-infra-mlops` owns the alerting engine, paging tool, status-page hosting, incident-management substrate, and metrics/traces backend — the applied engineer writes the alert rules, runbook content, and status-page template, then plugs them into the platform team's substrate. `model-evaluation-engineer` owns the offline/online eval regime that the feedback pipeline (chapter 03) feeds and that the judge-drift playbook (chapter 04) monitors. `sre` / `platform` owns the on-call model and the incident-response process the LLM-feature runbook slots into.

## Learning objectives

- Instrument LLM features with **OpenTelemetry Gen-AI + OpenInference** span attributes: prompt / response / model version / tokens / cost / latency / hashed user id / retrieved chunks / tool calls — at the granularity a debugger needs.
- Adopt a **sampling policy** (head sampling, tail sampling on error / quality signal) with **PII scrubbing** at the export boundary, all inside a **written trace budget** tracked as a first-class product line item.
- Wire **user feedback** — thumbs / structured / free text — into a triage pipeline that deduplicates, routes to owners, and drives the eval dataset via **Argilla** or **Label Studio**.
- Design **incident playbooks** for the **six recurring LLM failure modes**: bad prompt push, provider outage, judge drift, price hike, silent quality regression, guardrail false-positive spike.
- Compose a per-feature **status page** (audience: stakeholders) and **on-call runbook** (audience: the rotation) that a stranger can execute cold — with named interfaces to the platform / SRE team owned by `ai-infra-mlops`.

## Chapters

1. [OpenTelemetry Gen-AI + OpenInference: The Span Vocabulary a Debugger Needs](01-otel-genai-and-openinference-span-vocabulary.md) — the unit-of-work rule (one span per thing that can fail on its own); the `gen_ai.*` + `openinference.*` + `app.*` attribute set; the wrapper library that emits it; where auto-instrumentation stops.
2. [Sampling, PII Scrubbing, and the Trace Budget](02-sampling-pii-scrubbing-and-trace-budgets.md) — head vs tail sampling; the PII scrub matrix keyed to [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md); the trace budget as a first-class line item; the export pipeline as a policy boundary.
3. [User Feedback Triage: From Thumbs-Down to Eval Dataset](03-user-feedback-triage-pipeline.md) — three-signal capture (thumb, structured category, scrubbed free text); the join to the trace; dedup and clustering; owner routing with SLAs; **Argilla** / **Label Studio** for expert review; export to the [mod-106](../mod-106-feature-side-evaluation/README.md) eval set.
4. [Incident Playbooks for the Six Recurring LLM Failure Modes](04-incident-playbooks-for-six-llm-failure-modes.md) — the five-part playbook shape (detection, triage, mitigation, root cause, prevention) applied to bad prompt push, provider outage, judge drift, price hike, silent quality regression, and guardrail false-positive spike.
5. [Per-Feature Status Page and On-Call Runbook](05-per-feature-status-page-and-on-call-runbook.md) — the status page whose colour derives from observable metrics; the runbook written for the least-familiar engineer; the six interfaces to the platform / SRE team (`ai-infra-mlops`).

## Exercises

- [exercise-01 — OTel Gen-AI + OpenInference instrumentation for a feature](exercises/exercise-01-otel-genai-instrumentation-for-a-feature.md).
- [exercise-02 — sampling and PII scrubbing policy](exercises/exercise-02-sampling-and-pii-scrubbing-policy.md).
- [exercise-03 — user feedback triage with Argilla](exercises/exercise-03-user-feedback-triage-with-argilla.md).
- [exercise-04 — incident playbooks for the six LLM failure modes](exercises/exercise-04-incident-playbooks-for-six-llm-failure-modes.md).
- [exercise-05 — the per-feature status page and on-call runbook](exercises/exercise-05-on-call-runbook-for-an-llm-feature.md).

## Structure

- `01-…md` … `05-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references — OTel GenAI semconv, OpenInference spec, provider docs, SRE canon, Argilla / Label Studio, incident-management tooling.

## Cross-module context

- The **span attributes** this module standardises (`gen_ai.*`, `openinference.*`, `app.*`) are the same ones **mod-109** uses for the latency and cost SLIs — one wrapper library, both audiences.
- The **PII scrub matrix** in [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) is a specialisation of the general scrubber from [mod-107 chapter 03](../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md); the export pipeline reuses the same code path.
- The **feedback pipeline** in [chapter 03](03-user-feedback-triage-pipeline.md) is the *source* of the eval fixtures that land in [mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md); expert-reviewed thumbs-downs become CI-gated eval items.
- The **judge-drift playbook** ([chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md), Playbook 3) watches the calibration signal that [mod-106 chapter 04](../mod-106-feature-side-evaluation/04-sampled-online-eval-loop.md)'s online eval loop produces.
- The **bad-prompt-push and silent-regression playbooks** rely on the rollout / rollback substrate from **mod-111** — versioned prompts, canaries, one-command rollbacks.
- The **provider-outage playbook** fires the routing substrate defined in **mod-108** — fallback provider, residency-respecting hedge.
- The **price-hike playbook** reads the per-feature and per-tenant CPSO ledger from [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).
- The **guardrail FP-spike playbook** reads the guardrail decision spans from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) and rolls back the guardrail config from **mod-107**.
- End-to-end ownership of a feature with instrumentation, sampled traces, a feedback pipeline, six playbooks, a status page, and a runbook is the shipping half of the capstone in **mod-112**.

## Peer-role handoffs

- **`ai-infra-mlops`** — owns the alerting substrate the applied engineer writes rules for, the paging tool that fires them, the status-page hosting, the incident-management tool, and the metrics/traces backend. Named in detail in [chapter 05](05-per-feature-status-page-and-on-call-runbook.md).
- **`model-evaluation-engineer`** — owns the offline / online eval regime the feedback pipeline (chapter 03) feeds and the judge whose drift is watched by Playbook 3 in [chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md).
- **`sre` / `platform`** — owns the on-call model, severity definitions, communication templates, and executive-escalation policy the LLM-feature runbook slots into.
- **`privacy-engineer`** and **`data-security-engineer`** — own the DPA-scope constraints on what leaves the process boundary; the PII scrub matrix in [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) is the applied-side implementation of what they define.
- **`ai-security-engineer`** — reads the per-provider attribution on the spans to keep the data-flow / DPA story current across providers and regions.

Naming these boundaries out loud is deliberate: the observability / feedback / incident discipline is a *shipping* discipline that plugs into the platform team's substrate. The applied engineer's leverage is the LLM-specific vocabulary — spans, feedback shapes, failure modes — the platform team does not (yet) own.
