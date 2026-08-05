# Prompt Caching as a Design Constraint on System Prompt Shape

## Why cache-friendliness is a design property, not an afterthought

Provider prompt caches — OpenAI's automatic prefix cache, Anthropic's explicit `cache_control` breakpoints, AWS Bedrock's prompt caching for supported models, Google Gemini's context caching — are the single largest cost and latency lever available to an applied-AI engineer without touching the model. On a well-shaped surface, cache hits routinely cut input-token cost by 50–90% and cut TTFT enough to move a latency budget from yellow to green.<!-- needs-research: confirm current published discount percentages for OpenAI (typ. 50% off cached input), Anthropic (cache reads billed at ~10% of base input, cache writes at ~125%), Bedrock (Anthropic/Nova via Bedrock, discount varies), and Gemini context caching (per-hour storage + reduced per-token) — pricing pages change. --> On a poorly-shaped surface, the same providers cache nothing and you pay full freight on every request.

The trap: teams learn about prompt caching after the surface is already shipped, then discover the system prompt is unstable enough (a timestamp, the user's ID, a randomised list order) that no cache ever hits. Retrofitting cache-friendliness is *possible* but it means rewriting the prompt, re-running the eval suite (mod-106), and shipping a version bump — weeks of work to unlock a discount that would have been free if the prompt had been designed for it in the first place.

So: prompt caching is not an optimisation you turn on. It is a **design constraint on the shape of the system prompt** and the **cadence at which you version it**. Every prompt author needs to know where the cacheable prefix ends and the volatile suffix begins, and every prompt release needs to know what it does to the cache hit ratio. This chapter is the discipline for both.

## How the three main caches actually work

The provider-specific mechanics matter, but the underlying model is the same across all four: **cache hits require a byte-identical prefix**. Everything before the first divergent byte is potentially cached; everything from the first divergence onward is not.

### OpenAI — automatic prefix caching

OpenAI's API automatically caches the deterministic prefix of any prompt above a minimum size (currently 1024 tokens; check the provider docs for the current floor).<!-- needs-research: confirm current OpenAI prompt-caching minimum-token threshold and the current cached-input discount rate. --> You do not opt in; you do not mark breakpoints; the platform detects the reusable prefix. Cache lifetime is short (minutes) and best-effort. The billing signature is a `usage.prompt_tokens_details.cached_tokens` field in the response — non-zero means you hit.

**The design implication.** Everything you want cached must be up-front: the system prompt, the tool schema, the few-shot examples, the retrieved context. Anything that varies per request — the user's message, the current timestamp, the tenant ID — goes *last*. If a single volatile token sits between two cacheable chunks, the second chunk cannot cache. Order is everything.

### Anthropic — explicit `cache_control` breakpoints

Anthropic's Messages API takes an explicit `cache_control: {"type": "ephemeral"}` marker on a content block; the API caches the concatenated prefix up through that block. You get up to four cache breakpoints per request; TTL is nominally 5 minutes (extendable to 1 hour with the extended-TTL beta).<!-- needs-research: confirm current Anthropic breakpoint count (4 vs updated), the base 5-minute TTL, the 1-hour extended-TTL header/parameter name, and cache-write / cache-read pricing multipliers. --> Billing shows `cache_creation_input_tokens` (the first request that writes the cache — costs *more* than a normal input token) and `cache_read_input_tokens` (subsequent hits — costs much less).

**The design implication.** You have four levers you can plan. The canonical pattern is one breakpoint at the end of the system prompt, one at the end of tool schemas, one at the end of a large retrieved-context block, and one that "moves" — placed right before the current user turn in a multi-turn conversation to cache the growing history. Missed breakpoints are wasted requests; misordered breakpoints (a volatile block before a stable one) are wasted breakpoints.

### AWS Bedrock — prompt caching for supported models

Bedrock exposes prompt caching for a subset of hosted models (Anthropic, Amazon Nova, and select others) via a `cachePoint` marker in the Converse API, or via provider-native `cache_control` for Anthropic-on-Bedrock.<!-- needs-research: confirm current Bedrock-supported models for prompt caching, the Converse API `cachePoint` shape, and the pricing envelope (typically mirrors the underlying provider). --> The mechanic mirrors Anthropic's explicit-breakpoint model. IAM, region, and residency remain under Bedrock; the cache lives inside your Bedrock account boundary.

**The design implication.** Same shape as Anthropic-direct. If you already use Bedrock for the residency envelope (mod-108 chapter 01), the caching disciplines transfer without rewrite; the API shape is where the small differences are.

### Google Gemini — context caching

Gemini exposes explicit `context_cache` objects: you create a cache with a chunk of content, get a cache handle back, and reference the handle in subsequent requests. Cache TTL is user-controlled; storage is billed per-hour, and per-token reads are billed at a reduced rate.<!-- needs-research: confirm current Gemini context-caching minimum content size, storage price/hour, and per-token reduction. -->

**The design implication.** Because the cache is a first-class object with a lifetime you pick, Gemini caching is more like a "materialised prompt" — worth the setup for genuinely long-lived context (long documents, code repositories, static tool schemas) and *not* worth it for a system prompt you version weekly.

### The common substrate

Regardless of provider:

- **Byte-identical prefixes are cache-hittable.** One inserted character, one reordered dict key, one different whitespace breaks the cache.
- **Cache writes cost more than normal input tokens.** A prompt that thrashes (writes but never reads) is worse than no cache at all.
- **Cache TTLs are short (minutes to an hour) unless you opt into longer TTLs.** Version cadence and traffic pattern both matter — a prompt versioned hourly on a low-traffic feature never hits its own cache.
- **The reported cache-hit fields (`cached_tokens`, `cache_read_input_tokens`) are the source of truth.** Do not trust "we designed it cache-friendly, it should be caching" — read the field on every request.

## The stable-prefix / volatile-suffix rule

The single design principle that determines whether your surface caches or not: **order every prompt from most stable to most volatile.**

The canonical order, top to bottom, for a request payload:

1. **System prompt** — role definition, persona, non-negotiable rules. Changes when *you* release, not when the user acts.
2. **Tool schemas** — the JSON Schema for every tool the model can call. Changes when the tool surface changes.
3. **Static instructions and few-shot examples** — reusable guidance that lives outside the user turn.
4. **Long-lived retrieved context** — e.g. the entire codebase the code agent operates over, the customer's onboarding docs, the product taxonomy. Cache-friendly if it is stable per session or per tenant.
5. **Conversation history** — the previous turns of the current session.
6. **Retrieved context specific to this turn** — chunks from a per-turn RAG (mod-103) that vary each request.
7. **The current user turn** — always last.

Where the cache breakpoint (Anthropic / Bedrock) or the "everything above this line is cache-hittable" logical boundary (OpenAI / Gemini) lives depends on the surface. Two common shapes:

- **Chat surface.** System + tools + few-shot cached; conversation history rolls forward and is cached up through the last completed turn; the new user message is the only fresh input.
- **Long-context agent.** System + tools + few-shot cached; a large retrieved-context block cached; per-turn state (working memory, current tool observations) uncached.

The rule is enforced not by intent but by structure. If you find yourself templating a timestamp or a request ID into the system prompt "for logging," you have destroyed the cache. Log that field on the span (chapter 01), not in the prompt.

## Versioning cadence is a cache-hit-rate decision

Every prompt-release cadence choice is also a cache-hit-rate choice. Redeploying the system prompt daily on a low-QPS feature means most requests miss the cache — the last version's cache expires before the new version's cache is warm enough to hit. The trade-off:

- **Slow cadence (weekly / monthly releases).** Long steady periods of high cache-hit ratio. Slower iteration on prompt quality. Requires the eval suite (mod-106) to catch regressions since you can't hotfix every day.
- **Fast cadence (multiple releases per day).** Every release resets the cache. Cache-hit ratio is dominated by intra-release traffic bursts. Only pays off if traffic within a release window is high enough for the cache to warm up.

A useful heuristic: your prompt-release cadence should be *slower* than your average intra-release cache warm-up time by at least a factor of ten. On a 50-QPS feature with a 5-minute cache TTL, a release cadence of "hourly" gives you ~55 minutes of warm cache per release — fine. On a 0.5-QPS feature with the same 5-minute TTL, the cache almost never has time to warm; either extend the TTL (Anthropic's 1-hour beta, Gemini's explicit lifetimes) or slow the cadence.

Write the cadence down in the prompt-version doc:

```markdown
## Cache posture — <prompt name>

- Cache breakpoints:
  1. End of system prompt
  2. End of tool schemas
  3. End of static few-shot block
  4. Before the current user turn (rolling)
- Provider: <openai | anthropic | bedrock/anthropic | gemini>
- Nominal TTL: <5 min | 1 hour | user-managed 24h>
- Release cadence: <e.g. weekly on Mondays; hotfix only via cache-preserving edits>
- Cache-hit ratio target: <e.g. cache_read_tokens / prompt_tokens ≥ 0.6 (p50 over 24h)>
- Alert:  ratio below 0.4 for 30 minutes on any tenant
```

Two things to notice. First, cadence and TTL are named together — they set each other's ceiling. Second, the hit-ratio target is a real number, watched on a real dashboard, alerted on a real threshold. Without that, the cache is a hope.

## Cache-preserving edits

Once a system prompt is deployed and warming a cache, "just fix this one line" is not free. Any edit that changes bytes before the last cache breakpoint invalidates every downstream cache and pays the write cost again on every subsequent request until the new cache warms.

The discipline:

- **Additive edits go last.** A new example, a new rule, a new tool — append it *after* the last stable breakpoint if that is possible without breaking meaning. If it must go earlier, expect a cache-invalidation cost and schedule it with the same care as a model swap.
- **Whitespace and punctuation edits are not free.** A trailing newline change or a re-ordered list flips the prefix bytes. If the change is cosmetic, decline it.
- **Semantic edits get a version bump and a metrics window.** The release note names the expected cache-hit-ratio dip and the recovery time.

Two illustrative shapes for the edit conversation:

- **Bad.** "Small change to the system prompt: swap 'you are a helpful assistant' for 'you are an expert helpful assistant.' Merging." Cache-hit ratio drops to zero for the next 30 minutes; costs go up 3× until the cache warms; on a low-QPS surface, the cache never warms and the surface is now permanently more expensive.
- **Good.** "Small change to the system prompt: adding a new example to the few-shot block. The block is between breakpoint #2 and breakpoint #3, so the cached prefix ending at breakpoint #2 is preserved and only the tail re-caches. Expected hit-ratio dip: 5 minutes, then back to steady state."

## The cache posture on request

The load-bearing artefact is a prompt-version document (typically colocated with the prompt source) that names the cache shape. A minimal template:

```markdown
# System prompt — <feature> — v<n> — <date>

## Content
<the actual prompt, versioned>

## Cache shape
| Section              | Byte offset | Breakpoint? | Volatile? |
| -------------------- | ----------- | ----------- | --------- |
| System prompt        | 0–N1        | #1          | no        |
| Tool schemas         | N1–N2       | #2          | no        |
| Static few-shot      | N2–N3       | #3          | no        |
| Retrieved context    | N3–N4       | —           | per-tenant, low churn |
| Conversation history | N4–N5       | #4 rolling  | append-only |
| Current user turn    | N5–end      | —           | yes       |

## Providers wired
- Anthropic: `cache_control` set at #1, #2, #3, and rolling #4.
- OpenAI: automatic; order is what carries the cache.
- Bedrock (Anthropic route): same as Anthropic.

## Metrics
- Cache-hit ratio (24h p50) — target ≥ 0.6.
- Cache-write ratio — target ≤ 0.05 (i.e. writes cost < 5% of read-savings).
- TTFT p95 delta between cache-hit and cache-miss requests — track for the surface latency spec (chapter 01).

## Change log
- v3 (2026-07-01): added static few-shot example to block #3. Expected hit-ratio dip 5 min.
- v2 (2026-06-15): moved tool schemas out of the user turn and into block #2. Recovered ~40% input-token cost.
- v1 (2026-06-01): initial.
```

Two properties. First, cache posture is a **first-class part of the prompt release**, not a side project. Second, the metrics are named at release time, not discovered later; if the release note does not say what the cache-hit-ratio target is, no one can tell whether the release was successful.

## Reading the cache metrics

The billing fields expose the cache directly. Sample-code shape (OpenAI SDK, but same idea across providers):

```python
resp = client.chat.completions.create(...)
usage = resp.usage
cached = getattr(usage, "prompt_tokens_details", {}).get("cached_tokens", 0)
non_cached = usage.prompt_tokens - cached
hit_ratio = cached / usage.prompt_tokens if usage.prompt_tokens else 0

span.set_attribute("gen_ai.usage.input_tokens", usage.prompt_tokens)
span.set_attribute("gen_ai.usage.cached_input_tokens", cached)
span.set_attribute("app.cache.hit_ratio", hit_ratio)
```

Anthropic's fields are `cache_creation_input_tokens` (writes) and `cache_read_input_tokens` (reads); log both, because a high creation-to-read ratio means you are paying the write premium without recouping it.

The one dashboard chart that matters: **cache-hit ratio (`cache_read_tokens / prompt_tokens`) plotted per prompt version over time.** A healthy chart shows a version starting at zero, climbing to a plateau within one cache-TTL, and staying there until the next release. An unhealthy chart shows a version that never plateaus (traffic is too low for the cache to warm) or one that plateaus at 0.1 (the prompt shape is not cache-friendly). Both are actionable.

## Anti-patterns

- **Timestamp in the system prompt.** "Current date: 2026-08-05." Kills every downstream cache. Move it to the user turn or into a tool the model can call.
- **JSON dict ordering that is not stable.** Python `dict` ordering is deterministic in modern versions, but tool schemas built from unordered sets, environment-variable dumps, or database iteration are not. Sort keys explicitly.
- **Cache breakpoints in the wrong order.** Placing a breakpoint after a volatile block wastes the breakpoint; the block above it may still cache, but the *breakpoint* buys you nothing extra.
- **No cache-hit ratio on the dashboard.** You cannot fix what you cannot see. Add it as a first-class chart before you optimise anything.
- **Hotfixes that reset the cache.** A one-word tweak "just to be safe" invalidates the prefix. If the fix is not urgent enough to justify the cache-warm-up cost, batch it with the next scheduled release.
- **Cache-write cost ignored.** Cache creation is more expensive than normal input tokens. If the surface's traffic pattern never re-reads within TTL, caching *raises* your bill; either extend TTL, raise the request base, or stop caching that surface.
- **Retrofit-only mindset.** The team ships v1 of a feature with a cache-hostile prompt and plans to "cache it later." Later never comes; every downstream release must now also fight the cache retrofit. Design for cache in v1.

## Summary

- Prompt caching is a **design constraint** on prompt shape and versioning cadence, not an optimisation you flip on.
- All four provider caches (OpenAI automatic, Anthropic `cache_control`, Bedrock `cachePoint`, Gemini `context_cache`) share the same underlying requirement: **byte-identical prefixes**. Order stable content first, volatile content last.
- Cache breakpoints go at the boundaries of the stable blocks: system prompt, tool schemas, static few-shot, long-lived retrieved context, rolling turn history. The current user turn is always last.
- Versioning cadence and cache TTL set each other's ceiling. Fast release cadence on a low-QPS surface = permanently cold cache = wasted cost.
- Cache-preserving edits (append after the last breakpoint) are the norm. Any edit that changes bytes above a breakpoint is a release, not a tweak, and comes with a cache-hit-ratio dip forecast.
- The load-bearing artefact is the **prompt-version doc with a cache-shape table**, including the target cache-hit ratio and the write-ratio ceiling. The dashboard shows the ratios per version over time; the alert fires when they slip.

[Chapter 03](03-batch-apis-and-the-offline-eligibility-rubric.md) moves from the online request path to the offline / background one — how batch APIs cut cost 40–50% on the workloads whose SLA allows it, and how to write the eligibility rubric that decides which workloads qualify.
