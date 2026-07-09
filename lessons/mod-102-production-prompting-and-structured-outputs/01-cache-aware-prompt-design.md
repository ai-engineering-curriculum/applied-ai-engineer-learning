# Cache-Aware Prompt Design

## Why the prompt is a cache key

The first time you look at a modern provider's per-token pricing, prompt caching looks like a discount you happen to qualify for. It is not. On any surface that resends a large fixed context — a system prompt with policy, a rubric, a few-shot pack, an ingested set of documents, a tool schema — cached input costs a fraction of uncached input and streams a good deal faster to the first token. On OpenAI cached tokens are billed at a discount off the uncached rate; on Anthropic the discount is deeper, in exchange for a small write premium on the first request that seeds the cache; on Gemini and Bedrock the numbers and mechanisms differ again. Consult each provider's current pricing page — the exact ratios move — but the shape is the same: **repeated context is much cheaper if the provider can recognise it, and much more expensive if it silently doesn't.**

That word *silently* is the trap. Providers do not error when caching fails to kick in. Your system prompt just… isn't a cache hit, and you find out next month when the finance team asks why per-request cost doubled after a "small" prompt tweak.

The applied-AI engineer's job is not to *know* the current cached-token discount. It is to design prompts that stay cacheable across the edits a live product accumulates — new features, new tools, new safety rules, new tenants, new experiments — and to build the telemetry that catches a silent invalidation the day it happens, not the next quarter.

## How the four major providers cache

The mechanics differ enough that a one-line rule ("put the fixed stuff first") is not enough. The mental model you need is:

- **What is the cache key?** — the exact byte sequence at the start of the prompt (a prefix hash), or a caller-declared breakpoint.
- **Who decides what gets cached?** — automatic (the provider caches whatever prefix it can) or explicit (the caller marks the boundary).
- **What is the TTL?** — how long the cached prefix survives an idle period before a fresh request has to re-seed it.
- **What is the minimum size?** — below what number of tokens the cache is not used at all.
- **How is caching billed?** — cached-read discount, and any write premium on the seeding request.

### OpenAI — automatic prefix caching

OpenAI's prompt caching is **implicit**: you do not annotate anything. The API caches the longest matching prefix of your input and reuses it on subsequent requests to the same model. The unit of caching is the prompt prefix, and the key is a hash of the byte sequence up to some boundary. Any byte change in the prefix invalidates it from that point on.

Concrete consequences:

- **Order matters absolutely.** Anything that varies per-request — the user turn, dynamic tool results, a per-tenant identifier — must land *after* everything that stays fixed. A tenant name spliced into the system prompt with `f"You are Acme's assistant, {tenant.name}."` is a cache-key change on every request; the tenant name belongs in the user turn or a tool result or a header-shaped tail.
- **Whitespace, order, and encoding matter.** Rewrapping the system prompt from `\n` to `\r\n`, reordering a JSON `tools` array, changing a number's formatting (`3.0` vs `3`), or renaming a tool argument all break the prefix.
- **Minimum prefix size.** Below roughly a thousand tokens of shared prefix, caching does not apply. Very small system prompts do not benefit; padding to reach the threshold is not a productive optimisation — restructure the surface if you're spending that much effort.
- **TTL is short and idle-driven.** A prefix that is not requested again inside its idle window is evicted. Long-tail traffic on niche prompts does not stay cached.

Consult the OpenAI Prompt Caching docs for the current byte-boundary and TTL numbers; they've shifted at least once. <!-- needs-research: pin the current cache prefix minimum (in tokens) and idle TTL from the live OpenAI Prompt Caching docs page and cite the exact section title. -->

### Anthropic — explicit `cache_control` breakpoints

Anthropic's prompt caching is **explicit**. You add `cache_control: {"type": "ephemeral"}` to a message block or a system block to declare "everything up to and including this block is a cache breakpoint." Up to four breakpoints per request are supported. The provider hashes the content up to each marked breakpoint and reuses that hash across requests.

Consequences:

- **You pick the layers.** A typical layering: (1) a stable system prompt (policy + role + rubric), (2) a slower-moving tool schema block, (3) a slower-moving few-shot pack, (4) a stable-per-conversation retrieval context. Each gets a breakpoint. When the user turn changes, layers 1–4 are all cache hits.
- **Ordering is still key.** The same rule applies: the fixed layers come first, in the order you marked them, and any per-request variability is *after* the last breakpoint.
- **A write premium exists.** The first request that seeds a cache breakpoint costs a bit more than an uncached request; the subsequent cache-read cost is much lower. If a breakpoint is only ever hit once, you paid the premium for nothing — cache only what you actually re-use.
- **TTL is short by default with a longer opt-in.** The default ephemeral TTL is a few minutes; a longer TTL is available at a higher write premium. Choose based on inter-request cadence, not "just in case."

<!-- needs-research: confirm the current default and long-TTL numbers on Anthropic's Prompt Caching docs; both have been adjusted since the feature launched. -->

### Google Gemini — context caching (explicit, TTL, name-referenced)

Gemini exposes **context caching** as a separate API: you create a `CachedContent` resource with the content you want to cache (system instructions, large corpora, tools) and a TTL, receive a name, and reference the cache name on subsequent generation requests instead of resending the content.

Consequences:

- **Two-step, not one.** You call `cachedContents.create` once, then include the returned name on generate calls. This is more code than the transparent prefix cache but gives you an explicit handle: you can list, refresh, and delete caches.
- **TTL is a first-class parameter.** You choose it. A cache with a 10-minute TTL that is not renewed goes away; you can also extend it.
- **Minimum size is real.** There is a minimum token count below which content is not cacheable at all. It's larger than you'd guess and worth checking against your actual system-prompt length.
- **Billing splits.** Cached input is billed at a discount but there is a per-hour storage charge tied to the TTL. A long-lived cache for content that is used only occasionally can cost more than not caching at all.

<!-- needs-research: pin the current Gemini context-caching minimum-token count, storage-charge model, and the exact API surface name for the current API generation. -->

### AWS Bedrock — provider-specific, exposed through model-specific fields

Bedrock does not have a single unified caching API. Caching semantics come from the underlying model provider. Anthropic Claude on Bedrock accepts the same `cache_control` breakpoints as Anthropic's own API, subject to model support in the current Bedrock console. Meta Llama, Mistral, Amazon Nova, and Cohere models each have their own caching surface (or none). If you are running a "same code across models" router on Bedrock (mod-108), you must code around the fact that a breakpoint that is honoured on Claude may be silently ignored on another model.

Bedrock also exposes **prompt management** — a separate feature that stores versioned prompts as named resources and returns them at request time. That is a version-control feature (chapter 05), not a caching feature; do not conflate them.

<!-- needs-research: confirm which model families on Bedrock currently support prompt caching, and whether Bedrock's cross-provider caching semantics have converged since launch. -->

### Cross-provider summary

| Provider | Mechanism | Who marks the cache | Minimum size | Write premium | TTL |
|---|---|---|---|---|---|
| OpenAI | Automatic prefix cache | Provider | Low-thousands of tokens | None advertised | Short, idle-driven |
| Anthropic | Explicit `cache_control` | Caller (up to 4 breakpoints) | Model-dependent | Yes | Short default; longer opt-in |
| Google Gemini | `CachedContent` resource | Caller (named handle) | Model-dependent, non-trivial | Storage charge | Caller-chosen TTL |
| AWS Bedrock | Delegated to model | Model-dependent | Model-dependent | Model-dependent | Model-dependent |

The one shared rule: **cachable content goes first, per-request content goes last, and anything in between that changes between requests silently kills the cache from that point onward.**

## Structuring the prompt for cacheability

A production prompt has four blocks. Order them from most stable to most volatile.

### Block 1 — Stable role and policy (system)

The role definition, the tone rules, the refusal policy, the "must always include a citation" rule, the "never emit medical advice" rule. This block should not change per-tenant, per-user, or per-request. If you catch yourself splicing a tenant name in here, stop: put the tenant name in block 3.

Anthropic breakpoint: after this block.

### Block 2 — Stable structural artefacts (tools, rubrics, few-shot)

Tool schemas (JSON Schema for function-calling), rubric definitions, few-shot examples. These change on a code deploy or a rubric revision, not per-request. Keep the JSON deterministically ordered (a canonicalised serializer, not `json.dumps(dict)` which relies on insertion order and Python version). Keep the few-shot examples in a fixed order.

Anthropic breakpoint: after this block.

### Block 3 — Conversation-scoped context

Retrieved documents for this conversation, the current user's profile blob (if fixed for the conversation), a per-tenant policy overlay if you must have one at all. This is the "slow-changing but not global" layer.

Anthropic breakpoint: after this block.

### Block 4 — Per-request tail

The user's current turn, the latest tool result, the current cursor selection, whatever is different this exact call. **Nothing per-request goes above this line.**

A prompt that follows this layering caches on the first three blocks and re-computes only the tail. On OpenAI, the automatic prefix cache captures the same layering for free — up to the first byte that differs.

### Explicit anti-patterns

Each of these has broken a real deployment somewhere:

1. **The chatty timestamp.** A "Today is 2026-07-09, current time 14:32:07 UTC" line spliced into the system prompt. Cache breaks every second. Move to the user turn tail, or round to the day if that is enough for the feature.
2. **Per-tenant policy inlined in the system prompt.** "You are Acme's assistant. Acme's tone is formal." Now every tenant has a different cache. Either use a tenant-agnostic system prompt with tenant policy in block 3, or accept the fan-out and monitor cache-hit rate per tenant.
3. **Randomised few-shot ordering.** "We shuffle the examples to reduce recency bias." Fine as an experiment; disastrous for caching. Fix the order after the experiment concludes.
4. **JSON field-order drift.** Serialising the tool schema through a library that does not sort keys means a Python version bump or a dict-literal reorder invalidates the cache. Use a canonicalising serializer (sorted keys, stable whitespace).
5. **Per-user identifiers in the system prompt for "personalisation."** "You are talking to Sarah, a Gold-tier customer." The user's identity belongs in the user turn or a tool result — not in the cache-key layer.
6. **Content-hash IDs in the prompt as instrumentation.** "Trace ID: 8a3f2b…" injected into the system prompt "for debugging." Every request is a new key.

## Prompt caching and A/B tests

An A/B test on the system prompt is *by definition* a cache-invalidation event on the arm you changed. That is fine, but plan for it:

- Warm the new prompt's cache on a small deterministic set of canary requests before the test opens to real traffic.
- Compare cost per successful outcome across arms with the cache-hit rate included in the readout. A "better" arm that runs at 0 % cache hit is not better on cost.
- Do not run two live A/B tests on the same feature simultaneously — the interaction breaks both arms' cache economics and confounds the analysis.

Chapter 05 covers the versioning + rollback loop that operationalises this.

## Observability: catching silent invalidations

The mechanism that catches silent cache regressions is telemetry, not code review. At minimum:

- **Log the reported cached-token counts on every request.** All four providers include cached-input and uncached-input token counts in the response. Sum both and log both — the ratio is your cache-hit rate.
- **Alert on a drop in the ratio.** A step-function drop overnight is almost always a prompt edit that shifted a byte in the cached prefix. A gradual drop is often TTL-related (traffic slowed enough for entries to expire between requests).
- **Diff the seeded prefix on every deploy.** Serialize the system prompt + tools + few-shot pack to a stable representation and diff it in CI. A diff on any deploy that was not intended to touch the prompt is a caching bug in disguise.
- **Report cache-hit rate per prompt version.** Ties into the versioning table from chapter 05.

An SLO for cache health looks like: *"Cached-token ratio ≥ 0.80 on the retrieval-with-citations surface, trailing 24 h, production traffic, alerting on a drop of ≥ 10 percentage points sustained over 30 minutes."*

## Summary

- Prompt caching is a first-class production concern, not an optimisation. Silent invalidation shows up in cost weeks later.
- OpenAI caches automatic prefixes; Anthropic uses explicit `cache_control` breakpoints; Gemini uses named `CachedContent`; Bedrock delegates to the underlying model.
- The universal rule: fixed content first, per-request content last, no per-request bytes above the last cache layer.
- Layer prompts into four blocks — stable role/policy, stable structural artefacts, conversation-scoped context, per-request tail — with breakpoints in that order.
- Six named anti-patterns break real deployments: chatty timestamps, per-tenant policy inlined, shuffled few-shot, JSON field-order drift, per-user identifiers in the system prompt, trace-IDs injected for debugging.
- Cache hit rate is telemetry, not trust. Log the reported cached-token counts, alert on drops, and diff the serialised prefix in CI.

Chapter 02 turns the cached prefix into an actually-consumable output: JSON mode, function calling, Instructor + Pydantic, Outlines — the mechanics of schema-safe outputs the rest of the product can rely on.
