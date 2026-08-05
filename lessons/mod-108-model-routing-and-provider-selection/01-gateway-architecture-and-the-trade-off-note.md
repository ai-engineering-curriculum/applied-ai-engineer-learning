# Gateway Architecture and the Trade-Off Note

## Why a gateway at all

The first LLM feature a team ships almost always calls a provider SDK directly. `openai.chat.completions.create(...)` in one file, an API key in a `.env`, and the flow is on the screen the same afternoon. Nothing wrong with this — the point of a feature is to exist. Trouble starts when the second and third features appear. Now there are three copies of the retry loop, three copies of the token-counting logic, three shapes of "which model do we use when the primary is down," three teams each rediscovering that Anthropic's tool-use shape is not OpenAI's, three sets of billing dashboards that do not agree with finance's, three places where a model rollout has to be updated. The team has grown a de-facto gateway with none of the properties of a gateway.

A **gateway** is a single, deliberate seam between your product code and every model provider. Everything above it speaks one shape; everything below it speaks whatever the provider speaks. Once the seam exists, four otherwise-hard things become routine:

- **Swap-in / swap-out of providers and models** without touching feature code. The router (chapter 02) is a config edit, not a diff.
- **Cross-cutting policies** — retries, timeouts, rate-limits, cost caps, prompt-caching, PII scrubbing, moderation, structured-output coercion — that would otherwise be re-implemented per feature live once, at the seam.
- **Uniform telemetry**. Every request has the same span attributes regardless of provider: model, tokens, cost, cache-hit, retry count, provider outage flag. Dashboards work.
- **Governance**. Which teams may call which models, which tenants have which quotas, which regions may route where — enforced in one place instead of hoped for in a Slack thread.

There is no single right gateway. There are four canonical shapes, each optimising for a different property, and the applied-AI engineer's job is to *pick one deliberately and write the trade-off down*. That written trade-off is the load-bearing artefact of this chapter.

## Four canonical shapes

### Shape 1 — LiteLLM (self-hosted OSS proxy)

LiteLLM is an open-source proxy and Python SDK that normalises 100+ provider APIs into an OpenAI-shape interface — one call, `litellm.completion(model="anthropic/claude-...", ...)`, works the same as `model="openai/gpt-..."` or `model="bedrock/..."`.<!-- needs-research: confirm current LiteLLM-supported provider count and whether "proxy" and "SDK" are still the two shipping modes. --> Run it as a library in-process, or deploy the LiteLLM proxy as a long-running HTTP server your services call.

**What it optimises for.** Control and portability. You own the box, the config, the version, and the observability. Nothing about your traffic goes to a third-party control plane.

**Strengths.** OSS, no per-request vendor tax on top of provider cost, full customisation of the proxy config (routing rules, fallback groups, per-tenant keys, cost budgets), easy to fold your own middleware into the request path, and it plugs straight into OpenAI-shaped client code — a codebase already calling OpenAI's Python client can point at the proxy with a base-URL swap.

**Limits.** You are running a service. Availability, upgrades, scaling, and the on-call for the proxy itself are yours. Feature parity across every provider adapter drifts — a niche parameter of one provider may lag in LiteLLM by weeks. The dashboards you get are whatever you wire up (typically OpenTelemetry to your existing observability stack — mod-110 territory).

**When it fits.** Mid-to-large teams that already run services, need cross-provider routing, and have a strong preference for OSS or on-prem control. Also fits when regulatory posture forbids sending prompts to a third-party gateway control plane.

### Shape 2 — OpenRouter (managed marketplace)

OpenRouter is a hosted API gateway with a single OpenAI-shape endpoint that fronts hundreds of models across dozens of providers. One API key, one bill, per-request routing by name (e.g. `openai/gpt-4o`, `anthropic/claude-3.5-sonnet`, `meta-llama/llama-3.1-405b-instruct`).<!-- needs-research: confirm current OpenRouter model catalogue size and the exact model-slug conventions. -->

**What it optimises for.** Dev velocity and breadth of catalogue. Getting a new model onto your surface for an A/B is a config edit — the account, billing, and integration all already exist.

**Strengths.** Access to a very wide model set (including open-weights hosted on multiple inference providers, with OpenRouter selecting the cheapest / fastest at request time), one bill, one API key, no proxy to run. Fast experimentation surface for "does model X do better than Y on this prompt."

**Limits.** You are sending every request through a third-party control plane — a hard block for many regulated contexts. Availability is theirs (with all the tail-latency and outage stories that go with any hosted service in the request path). Unit economics have a marketplace markup (typically small but non-zero). You do not get residency guarantees the way a hyperscaler-native gateway gives them, and per-tenant policy has to be built on top of OpenRouter's primitives.

**When it fits.** Small teams, prototypes, breadth-of-catalogue needs, and any product where sending prompts through a hosted gateway is compatible with the data-handling posture. Often paired with a self-hosted gateway later — OpenRouter for the experimentation surface, LiteLLM or a hyperscaler for the production surface.

### Shape 3 — Portkey (managed gateway with guardrails / cache / fallback as first-class)

Portkey is a hosted (also OSS-deployable) AI gateway that centres its product on **policy** — guardrails, semantic caching, fallbacks, load-balancing, retries, virtual keys per tenant — all configured through a control plane, not code.<!-- needs-research: confirm Portkey's current deployment shapes (managed vs self-hosted) and the current feature envelope. --> It sits between LiteLLM (roll-your-own policy) and OpenRouter (do it yourself entirely).

**What it optimises for.** Policy density with low engineering overhead. You get a config surface for "route to A first, fall back to B on timeout, C on rate-limit, cache identical prompts for 24 hours, block outputs matching regex X" without writing the plumbing.

**Strengths.** Rich policy primitives, semantic and exact caching, first-class guardrail hooks, virtual-key issuance per team / per tenant, cost and latency observability out of the box, deployable managed *or* self-hosted (which reads on the trade-off dimensions differently — a managed deploy is more like OpenRouter, a self-hosted deploy is more like LiteLLM).

**Limits.** If self-hosted, you own another service. If managed, you inherit the third-party-in-request-path constraint. The policy DSL is another thing to learn; teams that already have a strong middleware layer may find it a duplicate.

**When it fits.** Teams that want cache / fallback / guardrail policy without building it, and either accept the managed posture or are ready to run the self-hosted deploy.

### Shape 4 — Hyperscaler-native (Bedrock, Vertex AI, Azure AI Foundry)

If your workload already lives in AWS, GCP, or Azure, the cloud vendor sells you a first-party gateway into models. **Amazon Bedrock** offers a unified API over Anthropic, Cohere, AI21, Meta, Mistral, Stability, Amazon Titan / Nova, and others, plus Bedrock Guardrails, Bedrock Knowledge Bases, Bedrock Agents. **Google Vertex AI** offers a unified API over Gemini, Anthropic (Claude on Vertex), Meta, Mistral, and open-weights on Model Garden. **Azure AI Foundry** (the successor to Azure OpenAI + Azure AI Studio) fronts OpenAI models plus a growing catalogue of partner models with the same Azure IAM, networking, and residency envelope.<!-- needs-research: confirm current provider catalogue on each hyperscaler; confirm Azure AI Foundry is the current umbrella name and the Azure OpenAI naming trajectory. -->

**What it optimises for.** Blast-radius, residency, and IAM alignment with the rest of your infrastructure. Prompts stay inside the same VPC, region, and account boundary as your database and your code. No new vendor onboarding, no new bill, no new SSO trust relationship.

**Strengths.** Regional data residency guarantees baked into the platform. IAM, VPC endpoints, PrivateLink, and audit logging you already trust. Committed-use / provisioned-throughput pricing that can be cheaper than direct-provider pricing at scale. First-party guardrails and safety primitives (Bedrock Guardrails, Vertex safety filters, Azure AI Content Safety).

**Limits.** Model availability lags — the newest model from any provider typically appears on the provider's own API weeks or months before it lands on the hyperscaler.<!-- needs-research: confirm current model-availability lag pattern; this is qualitatively true but has narrowed over time. --> Cross-hyperscaler multi-cloud requires a *second* gateway on top. The "unified" API is not portable — Bedrock Anthropic and Vertex Anthropic take different request shapes than the Anthropic-native API in some fields.

**When it fits.** Regulated industries, enterprise deployments already deep in one cloud, workloads where residency / IAM / audit trump model-of-the-month, and any surface where "the prompt must not leave our account" is a hard constraint.

## The trade-off dimensions

Every architectural pick lives on the same six axes. The trade-off note ranks the four shapes on each — for *your* surface, not in the abstract.

1. **Control** — how much of the routing, retry, cache, and policy behaviour you own and can change without a vendor's release cycle. LiteLLM ≈ self-hosted Portkey > hyperscaler ≈ managed Portkey > OpenRouter.
2. **Blast radius / residency** — where prompts and completions physically go and under whose IAM. Hyperscaler-native > LiteLLM (self-hosted) > self-hosted Portkey > managed Portkey ≈ OpenRouter.
3. **Model breadth** — how quickly a new model becomes routable. OpenRouter > LiteLLM ≈ Portkey (both track upstream fast) > hyperscaler-native.
4. **Feature velocity** — how quickly *your* team can add a policy (a new cache key, a new fallback tier, a new tenant quota). Portkey (managed) ≈ OpenRouter (limited) > LiteLLM (config-and-code) > hyperscaler-native (constrained by vendor primitives).
5. **Unit economics at scale** — cost of the gateway itself relative to the provider cost. LiteLLM ≈ hyperscaler-native (no marketplace tax) > self-hosted Portkey > managed gateways.
6. **Operational load** — who is on-call for the gateway. Managed gateways ≪ self-hosted gateways. Hyperscaler is in-between (you own the config, they own the availability).

None of the shapes wins across all six axes. That is what makes this a trade-off, and why the answer is *for this surface, given these constraints, we picked shape X*. A team routing prompts for a HIPAA-covered inpatient chart tool answers this differently from a team building an internal writing assistant that has to A/B five frontier models a week.

## Compositions you will see in the wild

Real deployments frequently compose two shapes, not one. Three common compositions:

- **Hyperscaler for production, OpenRouter (or provider-direct) for experimentation.** Production traffic stays inside Bedrock / Vertex / Foundry for residency; the experimentation surface (offline evals, A/Bs, prompt-tuning scripts) uses OpenRouter for breadth. The gateway seam sits at the feature code — features call one router, and the router picks the production path or the experimentation path per environment.
- **LiteLLM in front of a hyperscaler.** LiteLLM handles the policy layer (retries, fallback, cost caps, cross-provider abstractions) and calls Bedrock / Vertex as its own "provider." Gives you code-owned policy without giving up hyperscaler residency for the models that live there. The trade-off is that you are now running two things.
- **Portkey for the guardrail / cache / fallback layer in front of LiteLLM.** Portkey does the policy DSL, LiteLLM does the provider abstraction. Rarer, and worth doing only when the policy layer's complexity genuinely justifies the second hop.

The compositions are legitimate. What is *not* legitimate is composing them by accident — a feature that goes through LiteLLM, then OpenRouter, then a hyperscaler mirror, because three engineers each added a shim without deleting the previous one. The trade-off note names the composition.

## The trade-off note as an artefact

The single non-negotiable deliverable of "we picked a gateway" is a one-page written note. Its shape:

```markdown
# Gateway architecture — <feature or surface> — <date>

## Decision
We are shipping <shape or composition> for <feature>.

## Constraints
- Data residency: <e.g. all prompts must stay in eu-west-1; hyperscaler-native mandatory for this reason>
- Model breadth requirement: <e.g. we need Anthropic + OpenAI + one open-weight tier>
- Team size / on-call posture: <e.g. one on-call engineer; managed strongly preferred>
- Existing infra: <e.g. all workloads in AWS; Bedrock is the path of least resistance>
- Cost floor: <e.g. current spend $X/mo; gateway markup > 3% is disqualifying>

## Trade-off matrix
| Dimension           | LiteLLM | OpenRouter | Portkey | Hyperscaler |
| ------------------- | ------- | ---------- | ------- | ----------- |
| Control             | High    | Low        | Med     | Med         |
| Residency           | High    | Low        | Med     | High        |
| Model breadth       | Med     | High       | Med     | Med-Low     |
| Feature velocity    | Med     | Med        | High    | Low         |
| Unit economics      | High    | Med        | Med     | High        |
| Operational load    | Med     | Low        | Low-Med | Low         |

## Why not the alternatives
- <one sentence per shape we did not pick>

## Revisit trigger
- <the condition under which we would re-open this decision — e.g. multi-cloud requirement lands, or model-of-the-month becomes a business requirement>
```

Two properties. First, the matrix is filled in *for your surface* — a "High" in the residency row for hyperscaler is only "High" if your surface actually needs residency. Second, the **revisit trigger** is not decoration. Gateway decisions have a shelf life measured in quarters, not years; every large team has a fossil gateway pick made when the model catalogue was a third of its current size. Name the condition that would flip the decision, so the next re-open is a scheduled event and not a forced migration under an incident.

## Anti-patterns

- **The two-gateway accident.** Feature A calls LiteLLM. Feature B, added six months later, calls OpenRouter directly because the launch engineer did not know LiteLLM existed. Now there are two billing surfaces, two retry policies, two residency stories. The cheapest fix is not to unify overnight; it is to name the seam in the trade-off note and stop new features from picking a third.
- **The gateway that never actually abstracts.** The proxy is deployed. Every feature still special-cases per provider ("if the model starts with `claude-`, wrap tool_calls in the Anthropic shape"). The gateway is decoration; the code has learned every provider's quirks anyway. Fix: move the shape-normalisation into the gateway middleware. If the gateway does not save you from knowing whose model you are calling, it is not doing its job.
- **The gateway that hides cost.** All traffic goes through the gateway, but no per-tenant / per-feature cost attribution exists downstream, so finance sees one lump and can never answer "which feature is driving spend." Chapter 04 of mod-109 develops the accounting layer; the gateway is where it lives.
- **The gateway locked to one provider's shape.** LiteLLM and OpenRouter both use the OpenAI shape, and it is easy to write feature code that only works because of it — `response.choices[0].message.tool_calls` everywhere. When a genuine shape divergence appears (Gemini's parts array, Anthropic's separate content blocks), the "abstraction" leaks. Either commit to the OpenAI shape as your ubiquitous language *and* document the leaks, or write a real DTO layer.
- **The trade-off note that does not exist.** The team picked a gateway in the founder's DM to the CTO. Two years later no one remembers why. Every debate ("should we move to Portkey?") re-litigates from scratch. The one-page note is the artefact that ends this loop.

## Summary

- A gateway is a single, deliberate seam between product code and providers. It exists to make swap, policy, telemetry, and governance a config edit instead of a rewrite.
- Four canonical shapes: **LiteLLM** (self-hosted OSS, high control), **OpenRouter** (managed marketplace, high breadth), **Portkey** (policy-first, managed or self-hosted), **hyperscaler-native** (Bedrock / Vertex / Foundry — high residency and IAM alignment).
- No shape wins across all six trade-off axes: control, blast radius / residency, model breadth, feature velocity, unit economics, operational load.
- Compositions (hyperscaler for prod + OpenRouter for experimentation; LiteLLM in front of Bedrock) are legitimate — but only if named. Accidental compositions are a source of every incident on this list.
- The load-bearing artefact is a **one-page trade-off note** with the matrix filled in for *your* surface and a written revisit trigger. Every gateway debate a year from now starts by editing that note, not re-deriving it.

[Chapter 02](02-classifier-gated-routing-and-fallback.md) turns from "which gateway" to "what does the router that sits behind the gateway actually do" — classifier-gated routing, quality gates, and provider-outage fallback.
