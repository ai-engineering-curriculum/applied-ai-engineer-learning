# Self-Hosted vs Managed: The vLLM / TGI / SGLang Baseline

## The question this chapter answers — and does not

For each route on your route table (chapter 02), the model behind it is either **managed** (called via a provider's hosted API — OpenAI, Anthropic, Gemini, Bedrock, Vertex, Foundry, OpenRouter) or **self-hosted** (running on your infrastructure, typically an OSS inference server like **vLLM**, **TGI**, or **SGLang**, on GPUs you or your hyperscaler own).

The question this chapter answers: **when should the applied-AI engineer argue for self-hosting a specific route, and how should the trade-off be written down so the decision is defensible next quarter?**

The question this chapter does *not* answer: **how do you tune vLLM's KV-cache paging, TGI's continuous batching, or SGLang's RadixAttention for maximum throughput?** That is serving-perf work, and it lives in the `ai-infra-performance` track. The applied engineer *consumes* the throughput and latency numbers those teams publish; the applied engineer does not derive them. Attempting to short-circuit that boundary — deploying vLLM once, running one benchmark, and declaring the project done — is the specific failure mode this chapter is written to prevent.

Two roles, one artefact: the **self-host-vs-managed decision doc**, filled out with numbers from a peer team's baseline instead of numbers you improvised on the afternoon of the decision.

## The five trade-off axes

Every self-host-vs-managed conversation ends up on the same five axes. Rank the route on each — with a number when possible, with a range when the number is unknowable.

### Axis 1 — Unit economics at your traffic profile

Managed pricing is per-token. Self-hosted pricing is per-GPU-hour, and cost-per-token is a function of *your utilisation*. A rough shape:

$$
\text{cost per 1k tokens (self-hosted)} \approx \frac{\text{GPU \$/hr} \times \text{replicas}}{\text{throughput (tokens/sec)} \times 3600} \times 1000
$$

The crossover point where self-hosting beats managed is *high utilisation, at scale, on models the OSS ecosystem serves well*. Two lessons follow.

- **Utilisation is the make-or-break variable.** A vLLM replica idling at 5% average utilisation loses to managed at any traffic level. The applied engineer's contribution is not tuning the server — it is *characterising the traffic* on the route (average QPS, peak QPS, prompt-length distribution, response-length distribution, tolerance to queueing) accurately enough that the infra team can size the replicas realistically.
- **Break-even bands, not break-even points.** Do not tell yourself "self-hosting breaks even at X tokens per second." Tell yourself "at X ± 30% tokens per second, the two are within 20% of each other and the qualitative axes decide." Managed pricing changes; hardware pricing changes; the OSS model catalogue changes. The band survives a quarter; the point does not.

### Axis 2 — Latency (both TTFT and TPOT)

Both are user-visible. **TTFT** (time-to-first-token) is dominated by network path and prefill compute; **TPOT** (time-per-output-token) is dominated by decode throughput. Managed frontier APIs typically publish and hit low TTFT for short prompts, but tail latency (P95, P99) is where self-hosting sometimes wins — if you own the queue, no one else is in it.

Two applied-engineer lessons:

- **Self-hosting only wins tail latency if you overprovision.** A self-hosted replica that runs at 90% utilisation has queue tail latency that dominates any wire-latency saving. If the reason you want to self-host is tail latency, budget for the idle capacity. If you cannot, do not sell the decision on tail latency.
- **TTFT and TPOT trade differently against batching.** Continuous batching (vLLM, TGI, SGLang all do it in some form) improves throughput at some cost to TTFT for the head-of-batch request.<!-- needs-research: this is qualitatively true across servers; specifics vary by version and scheduler. --> Route decisions that require chat-perceptible TTFT need explicit measurement, not folklore.

### Axis 3 — Control (model choice, weights, versioning, quantisation)

Self-hosting is the only path to *model versions you control*. Concrete implications:

- You can pin an exact checkpoint and be certain no upstream update ever changes behaviour under you. This matters most for surfaces where behavioural stability is a compliance requirement, or where every model change requires a formal re-evaluation.
- You can run **fine-tuned or continued-pretrained variants** of open-weights models — LoRA adapters, DPO / ORPO tunes, task-specific fine-tunes — without needing them to be first-party-supported by a provider.
- You can pick your **quantisation**. INT8 / INT4 / FP8 / AWQ / GPTQ / FP4 quantised weights let you run a larger nominal parameter count on a given GPU; the quality tradeoff is measurable and needs its own eval.<!-- needs-research: current quantisation formats and their support across vLLM/TGI/SGLang. -->
- You can serve *speculative decoding* or *prefix-caching* configurations that a managed API does not expose.

Control has a mirror-image cost: everything becomes your job. Model deprecation, security patches for the inference server, GPU driver updates, upstream weight-license changes — all your on-call.

### Axis 4 — Residency and blast-radius posture

For any prompt-carrying data with regulatory constraints (HIPAA, GDPR, PCI, GLBA, DoD IL5, EU AI Act residency requirements, and any private-cloud commitment your enterprise customers have signed), the question "where does the byte physically live during inference" has one answer that satisfies auditors: *inside our own boundary*. Managed APIs have improved here (hyperscaler-native Bedrock / Vertex / Foundry stay inside your account; provider zero-retention offerings exist on OpenAI, Anthropic, and Google's enterprise tiers)<!-- needs-research: confirm current zero-retention / no-training options at each frontier provider. --> — but **owning the box** removes an entire class of trust question.

The applied engineer's contribution here is not to *make* the residency decision (privacy-engineer and data-security-engineer own that) — it is to *represent* the residency constraint in the trade-off doc so the infra decision serves the compliance envelope, and to flag it early so infra is not surprised at ship time.

### Axis 5 — Feature velocity

Managed APIs are the only path to *the current frontier model this week*. Open-weights ecosystem lag is real: the highest-performing open-weights instruct model on any given benchmark is usually a version or two behind the best-performing frontier closed model.<!-- needs-research: characterise this qualitatively rather than pin a specific gap — the ecosystem moves and the gap has been narrowing. --> If your surface *requires* the frontier — because your product's quality bar is only met by it — the answer is managed for that route, at least for now.

Feature velocity also cuts the other way: shipping a new fine-tune of an open-weights model can be days for a self-hosted stack (deploy new weights, reload the server, run evals, cut over) and *impossible* for a managed one that does not offer fine-tuning of the model you care about.

## The vLLM / TGI / SGLang baseline

For self-hosting, three OSS inference servers cover most production deployments. The applied engineer does not need to become a server expert — but does need to know the shape and read the peer team's benchmark output.

- **vLLM** — the most widely-deployed OSS inference server. PagedAttention KV-cache management, continuous batching, wide model-family support, an OpenAI-compatible HTTP server out of the box.<!-- needs-research: confirm vLLM remains the ecosystem leader in adoption and its current feature envelope. -->
- **TGI (Text Generation Inference)** — Hugging Face's inference server. Tensor parallelism, quantisation support, similar OpenAI-shape HTTP endpoint. Good defaults; particularly common inside Hugging Face's own inference products.<!-- needs-research: confirm TGI's current version and any material feature envelope shifts. -->
- **SGLang** — a newer entrant with a domain-specific frontend for structured outputs, prefix caching (RadixAttention), and constrained decoding. Strong for JSON-heavy, tool-heavy, structured-output-heavy workloads.<!-- needs-research: confirm SGLang's current standing and feature set; RadixAttention terminology should be verified. -->

You do not need to pick all three. Pick one, run the peer team's baseline against your traffic shape, and only revisit the pick if a routing metric goes sideways.

The **baseline** the applied engineer *consumes* — the ai-infra-performance team runs it, the applied engineer reads it — has this shape:

```yaml
# baselines/self-hosted/2026-08-01.yaml
target_model: meta-llama/Meta-Llama-3.1-8B-Instruct
server: vllm@0.6.x                  # pinned version
hardware: 1 x H100 80GB (BM instance)
quantisation: fp16                  # baseline; awq/int4 alt baselines below
scheduler: continuous batching
concurrency_sweep:
  - concurrency: 1
    ttft_p50_ms: 45
    ttft_p95_ms: 90
    tpot_p50_ms: 15
    throughput_out_tok_s: 65
  - concurrency: 8
    ttft_p50_ms: 60
    ttft_p95_ms: 150
    tpot_p50_ms: 22
    throughput_out_tok_s: 420
  - concurrency: 32
    ttft_p50_ms: 210
    ttft_p95_ms: 800
    tpot_p50_ms: 40
    throughput_out_tok_s: 950
notes:
  - "Throughput saturates around concurrency 32 on this prompt/response profile."
  - "Above concurrency 16, TTFT p95 blows past the target 500ms for the chat surface."
```

Two properties matter for the applied engineer:

1. **Numbers are illustrative and time-scoped.** Do not use the yaml above as the input to your decision — it is a shape, not a benchmark. Insist on numbers from a run against *your* traffic profile.
2. **The concurrency sweep is the point.** A single "here's the throughput" number is not a baseline. The sweep tells you how the server behaves as you approach its saturation, which is where your production incidents will live.

## The decision doc

The load-bearing deliverable of this chapter is a per-route decision doc — one page, versioned, with a revisit trigger. A shape:

```markdown
# Self-host vs managed — route <name> — <date>

## Decision
For route `<name>`, we will use <managed provider X | self-hosted server Y with model Z>.

## Trade-off table (this route, this traffic profile)
| Axis                    | Managed (<provider>)          | Self-hosted (<server>/<model>)    |
| ----------------------- | ----------------------------- | --------------------------------- |
| Cost per 1k out-tokens  | $<managed>                    | $<self-hosted at projected util>  |
| TTFT p50 / p95          | <ms> / <ms>                   | <ms> / <ms>                       |
| TPOT p50                | <ms>                          | <ms>                              |
| Residency               | <region / posture>            | <region / posture>                |
| Feature velocity        | <new-model latency>           | <new-model latency>               |
| Operational load        | Low                           | <specific>                        |

## Traffic assumptions
- Peak QPS: <n>
- Average QPS: <n>
- Prompt token distribution p50 / p95: <n> / <n>
- Response token distribution p50 / p95: <n> / <n>

## Break-even band
At QPS <a>..<b> with the above token distribution, the two paths are within <n%> cost.
Below <a> QPS, managed wins. Above <b> QPS, self-hosted wins.

## Compliance / residency
<Yes/No the route has a hard residency constraint; who owns confirming it>

## Revisit trigger
- QPS breaches <n> for a rolling 4 weeks (self-host case becomes stronger)
- Peer model released on managed provider with quality metric M above threshold T (managed case becomes stronger)
- OSS model matches or exceeds current managed quality on M

## Peer-team dependencies
- ai-infra-performance owns the throughput/latency baseline; refresh due <date>
- privacy-engineer signs off on the residency line
```

Two things the doc must have and one thing it must not:

- **Must have** — a break-even *band*, not a point.
- **Must have** — a **revisit trigger** with concrete numbers, so re-opening the decision is a scheduled event, not a fire.
- **Must not have** — throughput numbers the applied engineer improvised. Those come from a peer team's baseline. If no baseline exists, the doc's status is "pending" and the decision is not made.

## The hybrid patterns worth naming

Real deployments are rarely all-in on one side. Three hybrid patterns show up:

- **Managed frontier, self-hosted tail.** The `frontier` route is managed (need the model). The `cheap` route is self-hosted on an open-weights 7B / 8B / 13B model where you can hit 10x cheaper at your traffic. Volume-weighted, most of your requests are on the self-hosted route; quality-weighted, most of your hard requests reach the frontier.
- **Managed primary, self-hosted burst-buffer.** The primary is a managed API; a self-hosted vLLM tier is spun up during a marketing spike or a launch, absorbing burst traffic that would otherwise blow through the managed quota. Cost per token on the burst path is higher than steady managed pricing, but the *reliability* is under your control.
- **Managed primary, self-hosted fallback.** Reverse of the previous: managed carries the load, self-hosted stands ready as the fallback tier so a provider outage does not knock the surface offline. Rare because it is expensive to keep warm, but justifiable for revenue-critical surfaces.

The pattern lives on the route table (chapter 02). Naming it in the decision doc keeps hybrid choices from being "we accidentally have both."

## Anti-patterns

- **The vLLM PoC that becomes production.** An engineer stood up vLLM in a demo, showed a graph, and it silently ended up in prod. There is no eval against the target model, no baseline against the target traffic, no on-call runbook, no cost-vs-managed doc. The fix is not to rip it out; it is to slot it into the process — commission the baseline, write the decision doc, sign the residency line, and either promote it or retire it.
- **The self-host case that ignores utilisation.** The math looks great at 100% utilisation. Actual traffic hits 8% average utilisation. Managed would be cheaper by 5x. Ownership of the traffic assumptions in the decision doc is the specific fix.
- **The managed decision made once and never revisited.** The route was picked in 2024 when Model X was frontier. In 2026 Model X is mid-tier and open-weights alternatives at the mid tier are competitive. Nothing forces a re-open. The revisit trigger is the specific fix — it forces the re-open when a named condition fires.
- **The residency claim without an auditor.** The decision doc says "self-hosted for residency" but no privacy engineer signed off on what "residency" actually requires (which region, which retention, which key-management, which audit-log path). The residency line is a sign-off, not a checkbox.
- **The "we'll self-host to avoid the OpenAI dependency" argument, without a quality baseline.** Self-hosting on an open-weights model that scores 20 points lower on the quality metric that gates your feature is not de-risking the dependency — it is trading a vendor risk for a quality risk. Every self-host decision needs a quality-parity eval against the model it replaces (chapter 05's contract-tests are the specific instrument).
- **The single-server-vendor lock-in inside self-hosted.** "vLLM only" or "TGI only" locked into the CI, the fixtures, the deployment tooling. Each server has quirks; a workload that hits a corner case of one may work fine on the other. Keep the OpenAI-shape HTTP interface as the seam, so servers are swappable.

## Peer-role handoffs

- **`ai-infra-performance`** owns the serving-perf depth: continuous-batching schedulers, KV-cache paging, tensor / pipeline / expert parallelism, quantisation quality vs. throughput trade-offs, MLPerf-shape benchmarking. The applied engineer *consumes* their baseline and asks for a re-run when the traffic profile shifts.
- **`ml-platform-engineer`** owns the GPU capacity plan, the reservation strategy (spot / on-demand / reserved / provisioned throughput), the autoscaler behaviour, and the disaster-recovery story for the inference tier.
- **`privacy-engineer`** and **`data-security-engineer`** own the residency envelope. The self-host decision serves the envelope; it does not define it.
- **`sre` / `platform`** owns the on-call rotation for the inference cluster if you self-host. Do not sign the decision doc without their explicit acknowledgement of the on-call load.
- **`model-evaluation-engineer`** owns the deeper eval regime for a model swap (public benchmarks, task-specific probes, calibration). The applied engineer's contract-tests (chapter 05) *feed* their work; they do not replace it.

## Summary

- The self-host-vs-managed decision is *per route*, not per stack. Some routes want managed for feature velocity; some want self-hosted for cost or residency.
- Five axes: **unit economics** (at *your* traffic), **latency** (TTFT and TPOT, tail included), **control** (weights, versions, quantisation, fine-tunes), **residency / blast-radius**, **feature velocity**. Rank the route on each with numbers where possible.
- The applied engineer *consumes* serving-perf baselines from `ai-infra-performance`. Do not improvise throughput numbers. If no baseline exists, the decision is pending.
- The **decision doc** is one page, per route, with a break-even *band* (not point), a **revisit trigger** with concrete numbers, and named peer-team sign-offs (privacy, infra, SRE).
- Hybrid patterns (managed frontier + self-hosted tail; managed primary + self-hosted burst / fallback) are legitimate — the route table (chapter 02) is where they live.

[Chapter 04](04-multi-modal-routing-vision-audio-realtime.md) generalises the router beyond text: vision, audio, image / video generation, and real-time voice — every modality is a routing decision, not a hard-coded model.
