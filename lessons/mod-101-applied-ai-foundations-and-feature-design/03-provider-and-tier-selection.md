# Provider and Tier Selection

## Why this decision is *initial*, not permanent

By the time you reach provider and tier choice, you have a feature shape (chapter 01) and an SLO table (chapter 02). Those two artefacts are what you are optimising the provider choice *against*. Without them, "pick a model" is astrology.

Two disciplines govern the choice at this stage:

1. **Pick the initial model for the surface, not the forever model.** Model families move fast enough that a "best model" claim has a shelf life measured in weeks. You are picking the model that goes into the first eval-gated CI run, so the router (mod-108) has something to bake off against.
2. **Write the trade-off note before you build.** The choice must be defensible to a PM (does this hit the SLOs the spec implied?), an ML engineer (do we have anything to compare against, and are we locked in?), and a governance owner (does this cross a residency line, or add a data-processing agreement we can't sign?). The trade-off note is the record of that defence.

## The tier taxonomy

Providers are moving targets; tiers are more stable. You will make the choice, when you make it, against three tiers of capability and two tiers of hosting.

### Capability tiers

- **Frontier** — the top-of-line models from the major labs at the time of the decision. Highest task-success, highest cost per token, longest context, best instruction-following, best tool use. Use when the surface actually needs it (autonomous agents on high-value tasks, extraction over messy documents, safety-critical Q&A) or as the *judge* tier (mod-106).
- **Mid-tier** — the next-cheaper models from the same labs plus their competitors' second tier. Often 5–20× cheaper per token than the frontier, often 2–5× faster, materially worse on hard reasoning and long-context tasks. Use for the working majority of production traffic once the frontier version has proved out the surface.
- **Open-source / open-weights** — Llama, Qwen, Mistral, Gemma, DeepSeek, and their descendants, run via a managed serverless endpoint (Together, Fireworks, Groq, Bedrock's open-weight endpoints, Vertex Model Garden) or self-hosted (vLLM, TGI, SGLang). Use when you need residency guarantees managed frontier providers can't give, when you have workload volume that makes self-hosting cheaper, when you want a fixed dependency (see one-model lock-in in [chapter 05](05-leaderboard-driven-failure-modes.md)), or as a safety-net fallback when frontier providers are down.

Do not confuse "open weights" with "cheaper." A small serverless open-weights model can be more expensive than a mid-tier managed model once you count the routing overhead, the throughput ceiling, and the on-call cost of running the endpoint. Do the arithmetic.

### Hosting tiers

- **Managed** (OpenAI, Anthropic, Google Vertex, AWS Bedrock, Azure OpenAI / Foundry, and the open-weights aggregators) — someone else runs the GPUs. You pay for tokens, sometimes pay for reserved capacity, get an SLA, and inherit their region / residency / data-processing terms.
- **Self-hosted** (vLLM / TGI / SGLang on your own or a rented GPU fleet) — you run the GPUs. You pay for capacity, you own the throughput / latency, and you get to pin the exact model version.

The default for a first-shipped feature is **managed** unless there is a specific reason otherwise. Self-hosting adds an on-call load, an inference-serving depth requirement (owned by `ai-infra-performance`, not here), and a set of failure modes (capacity planning, GPU driver crashes) that will pull your team off product work. Move to self-hosted when the trade-off note justifies it, not before.

## Region and data residency

A managed provider's data-processing terms and its available regions can override every other input. Common constraints:

- **EU / EEA data residency** — some tenants require that inference on their data happen in an EU region, that no training happens on their data, and that any provider sub-processors be disclosed and approved. Most major managed providers offer EU-region endpoints; not every model is available in every region on day one, and features (fine-tuning, batch, prompt caching) may lag.
- **US federal / regulated** — AWS Bedrock and Azure OpenAI GovCloud / Government have specific accreditation footprints; consult your compliance owner rather than a blog post.
- **PHI / financial data** — BAA-style agreements (US healthcare) and financial-services terms are typically negotiated per-provider; do not assume the standard commercial terms cover you.
- **China / other export-controlled regions** — model availability and legal exposure are entangled; talk to legal.

Two heuristics for the trade-off note:

1. **List the tenant categories the feature will serve.** If any of them cross a residency line, the region choice is a hard constraint before you pick the model, not a preference after.
2. **List the sub-processors.** A managed provider is one; the guardrail model may be another; the eval judge (if it's a different provider) is another; the trace warehouse is another. Governance owners will ask for this list.

## Writing the trade-off note

A defensible note is a page. Structure it so a reviewer can scan the top and dig into any row.

### Header

- **Feature**: name of the feature, the surface, and the chosen shape from chapter 01.
- **Decision**: e.g. "Anthropic Claude Sonnet 4.5 via managed API, us-east-1 by default, eu-west-1 for EU tenants."
- **Decision owner**: name.
- **Review date**: for the mandatory revisit.

### The comparison table

Score the top-three candidates on the dimensions that matter for the feature. Don't score the full universe; the eliminated candidates go in a one-line "also considered and rejected" list at the bottom.

| Dimension | Candidate A | Candidate B | Candidate C |
|---|---|---|---|
| Task success on internal eval slice | 0.91 | 0.88 | 0.82 |
| Groundedness on RAG slice | 0.96 | 0.94 | 0.89 |
| TTFT p95 (measured) | 850 ms | 1200 ms | 400 ms |
| End-to-end p95 (measured) | 4.2 s | 5.8 s | 3.1 s |
| Cost per successful outcome | $0.031 | $0.019 | $0.006 |
| Region availability for our tenants | ✅ US, EU | ✅ US, EU | ⚠️ US only |
| Structured-output support | native | native | via constrained-decoding lib |
| Fine-tuning / distillation lever | ❌ | ✅ | ✅ (self-hosted) |
| Prompt-caching semantics | explicit cache breakpoints | ephemeral, opaque | manual, self-managed |
| Vendor concentration risk | high (1 of 3 frontier) | medium | low |

The measured numbers are from your **own** eval slice on your **own** production-shaped inputs. Public leaderboards are a hint at a starting shortlist, not evidence for the choice — see [chapter 05](05-leaderboard-driven-failure-modes.md).

### The written argument

Three to five sentences that name the SLO the choice is optimising, the ones it is trading against, and the constraint that killed the alternatives. Example:

> Chose Candidate A because it is the only option that hits the ≥ 0.95 groundedness SLO on our slice without cost-per-successful-outcome exceeding the $0.04 target, and because it offers the EU-region endpoint that our top-3 enterprise pilot needs. Traded off cost (Candidate B is ~40 % cheaper) and latency (Candidate C is materially faster) — accepted because groundedness is the primary product promise and Candidate A's TTFT is inside the SLO. Rejected Candidate C on region (US-only in the release window). Fine-tuning not available today; will re-evaluate at review date.

### Portability

State how portable the choice is, because a router bake-off (mod-108) is on the roadmap:

- Which parts of the prompt are provider-specific (Anthropic's system-prompt-then-messages shape, OpenAI's system role, Gemini's system instruction, tool-schema dialects)?
- Which structured-output mechanism is used, and how would it be replaced (native JSON mode vs Instructor / Outlines vs Pydantic-validated re-parse)?
- Is prompt caching a functional dependency (you rely on cache hits for TTFT) or an optimisation (you'd survive without it)?
- If the primary provider went down for four hours, what's the fallback and what quality / cost regression would users see?

Portability is not a binary; it is a set of costs. Writing them down means the router bake-off in mod-108 has a starting point instead of an argument.

### The revisit

A trade-off note without a revisit date is a fossil. Add:

- **Trigger revisit if**: task-success drops below X on the weekly eval; cost-per-successful-outcome exceeds Y; provider price change; new frontier release from any of the top-three labs; residency requirement changes.
- **Mandatory revisit**: on `<date + 90 days>` regardless.

## Common shapes of the initial choice

Some patterns recur. Naming them helps you defend the specific choice in the meeting.

- **Frontier as judge, mid-tier in production** — pick a mid-tier model for user-facing latency and cost, use the frontier tier only as an eval judge (mod-106) and as an escape valve when the mid-tier fails a guard. The judge model must be at least one tier above the model it judges to avoid the "same-model-can't-critique-itself" failure.
- **Frontier for the shape, mid-tier as the router destination** — build to frontier while the surface is new so shape decisions aren't blamed on model weakness, then swap in mid-tier destinations behind a router when the eval slice says it's safe. The router (mod-108) is the mechanism.
- **Managed frontier + self-hosted open-weights fallback** — the fallback is not for cost; it's for the outage-response SLO. The self-hosted stack lives in a "warm but idle" state; the fallback path is contract-tested in CI.
- **Open-weights primary for residency** — an EU tenant, a healthcare tenant, a government tenant. The task-success gap vs frontier is the price of the contract.

Every one of these is a defensible starting point *for the right feature*. None of them is the right answer to "which model should we use" without the surface, the SLOs, and the tenants named.

## Summary

- The provider / tier decision is the *initial* one — you are seeding the router bake-off in mod-108, not making a forever choice.
- Score against your own eval slice on your own production-shaped inputs; leaderboards are shortlist hints, not evidence.
- Capability tiers: frontier, mid-tier, open-source / open-weights. Hosting tiers: managed, self-hosted. Default to managed until the trade-off note justifies otherwise.
- Region and data residency can override every other input; list the tenant categories and the sub-processors before you pick the model.
- The written trade-off note has a comparison table, a three-to-five-sentence argument, an explicit portability accounting, and a revisit trigger + date.

Chapter 04 wraps the shape, the SLO table, and the trade-off note into a single one-pager the PM, ML engineer, and governance owner can each sign off on.
