# exercise-03: Self-Host vs Managed Decision with a vLLM Baseline

**Estimated effort:** 3 hours

## Objective

For one route on your route table (from exercise-02), produce the **self-host-vs-managed decision doc** from chapter 03 with numbers from a real vLLM (or TGI / SGLang) baseline run — not folklore. The exercise deliberately blurs the peer-role boundary once, so the applied engineer sees the whole picture: you commission the baseline (or run it yourself if no infra peer is available), *then* fill in the decision doc as the applied engineer's artefact.

The point of this exercise is *not* to become a serving-perf expert. It is to feel the shape of the numbers a real baseline produces and to write a decision doc that is defensible against them.

## Learning outcomes

By the end you should be able to:

- Characterise a route's traffic profile (peak / average QPS, prompt / response token distributions, latency tolerance) in numbers, not adjectives.
- Read a vLLM (or TGI / SGLang) throughput / latency sweep and identify the saturation point.
- Compute a break-even *band* (not point) between managed cost per 1k output tokens and self-hosted cost per 1k output tokens at your traffic profile.
- Write a **per-route decision doc** with a break-even band, a residency line, a revisit trigger, and named peer-team dependencies.
- Name the correct **hybrid pattern** (managed frontier + self-hosted tail; managed primary + self-hosted burst; managed primary + self-hosted fallback) when the decision is not all-in on one side.

## Prerequisites

- You have read [chapter 03](../03-self-hosted-vs-managed-with-vllm-baseline.md).
- Access to at least one GPU capable of running a mid-size open-weights model (H100, A100, or L40S; if none, use a rented cloud GPU for the duration of the baseline — a couple of hours is enough).
- Familiarity with vLLM at the "run the OpenAI-compatible server" level; TGI or SGLang are acceptable substitutes.
- A route from exercise-02's route table you want to evaluate.
- Managed pricing pages for the equivalent managed model at hand.

## Problem statement

The route you picked from exercise-02 is currently served by a managed provider. Rumours around the team are "we should self-host to save money" and "we can't self-host because latency." Neither is grounded in numbers. In three hours, produce the decision doc that closes the debate — either "here's the number and here's why we should self-host" or "here's the number and here's why managed wins for this route."

## Requirements

### Deliverable 1 — Traffic characterisation

Under `docs/traffic-profile.md`:

- **Peak QPS** and **average QPS** on the route, measured over the last week (or the scaffold workload if using the paired solutions repo).
- **Prompt token distribution** p50 / p95 (input to the model).
- **Response token distribution** p50 / p95 (output from the model).
- **Tolerated latency** — the surface's SLO for this route (TTFT p50 / p95 and TPOT p50).
- Cadence: is the traffic bursty (marketing-driven spikes), diurnal (business hours), or steady?

Numbers, not adjectives. If you cannot measure one directly, sample it from logs and note the sample size.

### Deliverable 2 — vLLM baseline (or peer-team artefact reference)

Under `baselines/`:

- **Option A (preferred)** — run a vLLM (or TGI / SGLang) baseline on an open-weights model that is a plausible substitute for the managed model. Sweep concurrency (e.g. 1, 4, 8, 16, 32, 64) using a benchmark harness (`vllm bench serve`, `text-generation-benchmark`, or the peer team's standard).<!-- needs-research: confirm current benchmarking harness names — vLLM ships a benchmark script; TGI has its own. --> Emit `baselines/vllm-<model>-<date>.yaml` with per-concurrency TTFT p50 / p95, TPOT p50, and throughput (tokens/sec) in the shape shown in [chapter 03](../03-self-hosted-vs-managed-with-vllm-baseline.md).
- **Option B (fallback)** — reference an existing peer-team baseline. Cite the file, the date, and the version of the server used. This is the shape the applied engineer usually consumes — the exercise deliberately makes this legitimate.

Either way, include a **saturation note** — the concurrency at which throughput stops growing meaningfully. That is where your production incidents live.

### Deliverable 3 — Cost math

Under `docs/cost-math.md`:

- **Managed cost per 1k output tokens** — the provider's list price for the equivalent managed model. Cite the pricing page and the date.
- **Self-hosted cost per 1k output tokens at each concurrency** — computed from the GPU hourly cost and the baseline's throughput at that concurrency:
  $$
  \text{cost per 1k output tokens} = \frac{\text{GPU \$/hr} \times \text{replicas}}{\text{throughput (tokens/sec)} \times 3600} \times 1000
  $$
- **Utilisation assumption** — the fraction of the day the replicas are actually serving at the chosen concurrency. Be honest — 40% is often the realistic number, not 100%.
- **Break-even band** — the QPS range where the two paths are within 20% of each other on cost. Show the math.

### Deliverable 4 — The decision doc

Under `docs/decision-<route>.md`, follow the [chapter 03](../03-self-hosted-vs-managed-with-vllm-baseline.md) shape exactly:

- **Decision** — one sentence, managed vs self-hosted vs a named hybrid.
- **Trade-off table** — filled in with the numbers from Deliverables 1–3.
- **Traffic assumptions** — reference `traffic-profile.md`.
- **Break-even band** — reference `cost-math.md`.
- **Compliance / residency** — one line naming the constraint and the peer-role owner (`privacy-engineer` / `data-security-engineer`) even if the sign-off is out of scope for this exercise.
- **Revisit trigger** — at least two *observable* conditions with concrete numbers.
- **Peer-team dependencies** — reference the baseline in Deliverable 2 (peer-owned or self-run) and name the on-call rotation owner if the decision is self-hosted.

### Deliverable 5 — The hybrid consideration

Under `docs/hybrid-consideration.md`, one page:

- Which of the three hybrid patterns from chapter 03 (managed frontier + self-hosted tail; managed primary + self-hosted burst; managed primary + self-hosted fallback) is a plausible alternative to your Deliverable 4 decision.
- One paragraph: what makes it plausible, what makes it *not* the pick today, and what condition would make it the pick tomorrow.

Even if the answer is "none are relevant for this route," write that down with a sentence of justification.

## Starter guidance

- **Do not tune vLLM.** Use the server's defaults. The goal is a baseline representative of what most teams get out of the box, not a lovingly hand-tuned config only you can reproduce.
- **Pick a plausible open-weights substitute.** If the route uses a frontier managed model, do not baseline against an 8B open-weights model and declare victory. Match by task tier — a mid-size open-weights instruct model as the substitute for a mid managed model; a large open-weights model (Llama 3.1 70B, Qwen 2.5 72B, DeepSeek V3, or the current equivalents) if you are matching a stronger managed tier.<!-- needs-research: keep model names current at author time. -->
- **Ground the GPU cost in your actual cloud.** On-demand H100 pricing differs across AWS, GCP, Azure, and specialised providers. Cite the specific SKU and region.
- **Assume ≤ 50% average utilisation.** The math looks best at 100%; production is not 100%. If you can measure it on your existing infra, use the measurement; otherwise 30–50% is the honest default.
- **Break-even bands are more useful than points.** Managed pricing changes; hardware pricing changes. A band that says "self-hosted wins above 15 QPS at this token profile" is more durable than "self-hosted wins at 12.4 QPS."
- **The residency line does not need a sign-off for this exercise.** But it does need a named owner — the trade-off doc has to say who *would* sign it off if you were actually shipping this.

## Acceptance criteria

- [ ] `docs/traffic-profile.md` gives peak / average QPS, prompt and response token distributions (p50 / p95), the surface SLO, and the cadence — in numbers, not adjectives.
- [ ] `baselines/` has either a self-run vLLM baseline yaml with a concurrency sweep and a saturation note, or a citation to a peer-team baseline with equivalent shape.
- [ ] `docs/cost-math.md` shows managed cost per 1k output tokens (with citation and date), self-hosted cost per 1k output tokens at each swept concurrency, an explicit utilisation assumption, and a break-even band.
- [ ] `docs/decision-<route>.md` follows the chapter-03 shape exactly with a break-even band (not a point) and at least two observable revisit triggers.
- [ ] `docs/hybrid-consideration.md` names the most-plausible hybrid pattern and one condition that would make it the pick.
- [ ] Every model / product / hardware name is pinned to a date or SKU that a reviewer can look up.
- [ ] The decision doc names peer-team owners for residency and (if self-hosted) on-call — even if the sign-off is out of scope for the exercise.

## Stretch goals

Pick any one:

- **Quantisation baseline.** Rerun the baseline with an INT8 or INT4 / AWQ / GPTQ quantisation of the same model. Include a quality-parity note (a small eval against a fixture set) — quantisation quality loss is measurable and material.<!-- needs-research: currently viable quantisation formats and their support across vLLM/TGI/SGLang. -->
- **Two-server bake-off.** Repeat the baseline on a second server (vLLM vs TGI, or vLLM vs SGLang) at the same concurrency sweep. Note the shape of the difference. This is the material `ai-infra-performance` uses; the applied engineer skimming it is a defensible investment.
- **Managed provider zero-retention path.** If the managed provider offers a zero-retention / no-training enterprise tier at a different price, extend the cost math with it. The residency argument sometimes evaporates once this tier is priced in.<!-- needs-research: confirm current zero-retention pricing / availability on major providers. -->
- **Regional cost sweep.** Repeat the GPU cost math across three regions (e.g. us-east-1, eu-west-1, ap-southeast-1). Regional pricing differences are frequently the deciding factor for a marginal route.
- **Sensitivity analysis.** For the deciding QPS in your break-even band, show how the pick changes as (a) managed pricing drops 20%, (b) GPU pricing drops 20%, (c) target utilisation moves from 30% to 60%. Three lines of math; useful for the revisit-trigger conversation.

## Deliverable

Submit as `mod-108/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/traffic-profile.md`.
- `baselines/` with the self-run yaml or the peer-baseline citation.
- `docs/cost-math.md`.
- `docs/decision-<route>.md`.
- `docs/hybrid-consideration.md`.
- Any stretch-goal artefacts named alongside.
- A short `README.md` naming the route and the decision.
