# exercise-02: Classifier-Gated Routing and Fallback

**Estimated effort:** 3 hours

## Objective

Build a working router that composes all three of chapter 02's axes for one product surface: a **classifier-gated** difficulty route (cheap tier for easy requests, frontier for hard ones), an escalation-on-**quality-gate-fail** path, and a **per-provider circuit-breaker** fallback with a routing-decision log. Ship it against a fixed test workload and produce the escalation-rate / cost / fallback-fire metrics from chapter 02 that make the router honest.

## Learning outcomes

By the end you should be able to:

- Write down a **classifier schema** (categories, seed examples, per-category target route) before writing the classifier.
- Encode the route table so **model names live in exactly one place** and features never mention providers.
- Wire a **quality gate** with a `max_escalations` cap and log the gate result on every request.
- Implement a **per-provider circuit breaker** with a hard per-call timeout, a half-open probe, and a routing-decision log with the full attempts list.
- Produce a **routing report** that names, per route and per provider: escalation rate, cost per successful outcome, fallback fire rate, and category drift.

## Prerequisites

- You have read [chapter 02](../02-classifier-gated-routing-and-fallback.md).
- Access to at least two providers (any two of OpenAI, Anthropic, Gemini, Bedrock, Vertex, Foundry, OpenRouter, LiteLLM in front of the above). If only one provider is available, run a self-hosted OSS model (vLLM with an 8B open-weights model) as the second provider.
- A gateway pick from exercise-01 (or a stand-in — LiteLLM in-process is a reasonable default for the exercise).
- A test workload: at least 100 real-or-shaped user requests representative of the surface. If none from a day-job surface is available, use the scaffold workload in the paired solutions repo.

## Problem statement

Your team's current pattern is "call the frontier model for everything." The bill is running 4x what it needs to; a recent provider outage took the feature down because there was no fallback; when the model returns malformed JSON the caller crashes instead of retrying. In three hours, close the loop for **one route** on **one surface** with all three routing axes wired end-to-end.

## Requirements

### Deliverable 1 — The classifier schema

Under `routing/classifier-schema.yaml`, a schema following the shape in [chapter 02](../02-classifier-gated-routing-and-fallback.md):

- **At least four categories** covering the observed distribution of your surface (chit-chat, factual-short, reasoning-multi-step, creative-long-form, tool-heavy, out-of-scope — pick what fits).
- **≥ 3 seed examples per category**, drawn from the test workload.
- **Per-category target route** (`cheap`, `mid`, `frontier`, or `refuse`).

### Deliverable 2 — The route table

Under `routing/routes.yaml`, a route table naming, per route:

- `primary` — dated model ID (e.g. `openai/gpt-4o-2024-08-06`, not `openai/gpt-4o-latest`).
- `fallbacks` — at least one alternative per route, on a different provider.
- `max_p95_ms` and `max_cost_per_call_usd`.

Model names must appear **only** in this file. Grep for a provider name in your feature code — it should return zero hits.

### Deliverable 3 — The classifier implementation

Under `routing/classifier.py`:

- A small classifier — a distilled encoder, a small LLM in JSON-mode against your schema, or an embedding-nearest-seed rules layer. **Not** the frontier model.
- Latency budget: p95 under 200ms.
- Returns a `Classification(category, confidence, model_used, latency_ms)`.

### Deliverable 4 — The quality gate

Under `routing/gates.py`:

- Implement at least the **structured-output validation** gate — if the route was expected to return JSON conforming to a schema, parse and validate.
- Add one additional gate of your choice (**judge-model check** or **self-report confidence with a calibrated threshold**).
- Config file `routing/quality-gates.yaml` names, per gate: `applies_to`, `check`, `threshold` (if applicable), `on_fail.action`, `on_fail.to`, `max_escalations`.

### Deliverable 5 — The router

Under `routing/router.py`:

- Composes classifier → route pick → provider call with circuit breaker → quality gate → escalate-or-return, per the state machine in [chapter 02](../02-classifier-gated-routing-and-fallback.md).
- **Per-provider circuit breaker** with a rolling failure window, an `open_duration`, and a half-open probe.
- **Hard per-call timeout** derived from the route's `max_p95_ms` (not the provider's default).
- **Routing-decision log** emitted per request in the shape shown in chapter 02 (attempts list preserved, escalation and fallback distinct fields).

### Deliverable 6 — The routing report

Under `bin/routing-report`:

- Runs the router against the test workload.
- Emits `reports/YYYY-MM-DD.yaml` and `reports/YYYY-MM-DD.md` with:
  - Per-route **escalation rate**.
  - Per-route **cost per successful outcome** (sum of primary + escalations + gates, divided by successful returns).
  - Per-provider **fallback fire rate** and **circuit-breaker open events**.
  - **Category distribution** and a note on any drift vs. a baseline (if a baseline exists).
- The report format is machine-readable so a CI job can diff it week-over-week.

### Deliverable 7 — Two proofs

Under `proofs/`:

- `escalation-worked.md` — pick one request where the primary route's quality gate failed and the router correctly escalated. Show the routing-decision log for the request, the failed gate output, and the successful escalated response. Prove the `max_escalations` cap prevented a runaway.
- `fallback-worked.md` — force a provider failure (kill an API key, use a mock provider that raises, or hit a rate-limit). Show the routing-decision log for a request during the outage. Prove the circuit breaker opened, the fallback served the request, and the breaker half-opened after `open_duration`.

## Starter guidance

- **Start with the static heuristic gates (Shape 1a).** Wire route selection from prompt length and surface metadata before adding a classifier. The classifier layer is worth adding only after the static shape is working and you can quantify its gap.
- **The classifier must not be the frontier.** If you use an LLM for classification, use a `-mini`, `-haiku`, or `-flash` tier — or a distilled encoder. A classifier that costs 30% of the primary call has already lost the argument.
- **Escalation cap is a hard number.** `max_escalations: 1` is the default. Only raise if a specific route needs two hops and the cost math still works.
- **Circuit-breaker per provider, not per model.** A provider region 5xx takes every model on that provider down; the breaker granularity that fits is the provider, not the model ID.
- **Cache gate results.** If you re-run the workload against the same fixtures, cache `(input_hash, model_id, gate_config_hash)` so re-runs are cheap.
- **Set the timeout to `max_p95_ms * 2`, not `max_p95_ms`.** A hard timeout equal to the SLO fires on every marginally-slow request; a timeout at 2× SLO fires on the actual outages you care about.

## Acceptance criteria

- [ ] `routing/classifier-schema.yaml` has ≥ 4 categories, ≥ 3 seed examples each, and a per-category target route.
- [ ] `routing/routes.yaml` names dated model IDs, per-route SLOs, and ≥ 1 fallback per route on a different provider.
- [ ] `grep -r "openai\|anthropic\|gemini" src/features/` returns zero hits.
- [ ] `routing/classifier.py` has p95 latency < 200ms measured on the test workload.
- [ ] `routing/gates.py` implements structural validation + one additional gate; `routing/quality-gates.yaml` names their config.
- [ ] `routing/router.py` implements the chapter-02 state machine with per-provider circuit breaker, hard per-call timeout, and full routing-decision log.
- [ ] `bin/routing-report` emits per-route escalation rate, cost per successful outcome, fallback fire rate, and category distribution.
- [ ] `proofs/escalation-worked.md` shows a real escalation with the routing-decision log and the `max_escalations` cap enforced.
- [ ] `proofs/fallback-worked.md` shows the breaker opening, the fallback serving, and the breaker half-opening.
- [ ] The report format is machine-readable (yaml or JSON) so it can be diffed in CI.

## Stretch goals

Pick any two:

- **Cost SLO enforcement.** Add a hard `max_cost_per_call_usd` check at the route level. When a request's total spend (primary + escalations + gates) exceeds the cap, log the breach as a first-class failure — not a silent absorption. This is the pipeline-side version of mod-109 chapter 04's per-feature cost accounting.
- **Bandit routing spike.** Replace the static classifier for one route with a learned policy trained on the last week's `(features, chosen_route, judged_outcome, cost)` logs. Compare the bandit's escalation rate and cost to the static classifier's. Ship only if the delta justifies the machinery.
- **Multi-provider synthetic outage drill.** Extend `proofs/fallback-worked.md` into an *automated drill* that runs weekly: kills the primary provider (via config or a mock), replays the last hour of traffic, verifies every request succeeded via the fallback, and pages if any did not. This is the router's version of a game-day.
- **Per-tenant quotas.** Add per-tenant caps on `frontier` calls per hour. When exceeded, the router refuses (or forces `cheap`) rather than serving the request. Log the quota decision distinct from the routing decision.
- **Streaming quality gate.** Extend one gate to work on a streaming response — moderate on a rolling buffer, retract-on-fail. This mirrors mod-107 chapter 01's retract-on-flag pattern.

## Deliverable

Submit as `mod-108/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `routing/` with the schema, route table, classifier, gates, router, and quality-gate config.
- `bin/routing-report` and `reports/` with at least one baseline report against the test workload.
- `proofs/escalation-worked.md` and `proofs/fallback-worked.md`.
- A short `README.md` naming the surface, the routes, and how to run the report.
- Any stretch-goal artefacts named alongside.
