# exercise-02: Prompt-Cache-Driven System Prompt Design

**Estimated effort:** 3 hours

## Objective

Redesign an existing (or scaffold) system prompt for a real LLM feature so that provider prompt caching becomes a first-class property, then measure the cache-hit ratio landing and write the cache-posture doc. The deliverable is not "add cache_control markers" — it is a **prompt-version doc with a cache-shape table**, a written release cadence, and a chart showing the ratio climbing after deploy.

## Learning outcomes

By the end you should be able to:

- Identify the stable-to-volatile ordering of a prompt payload and place cache breakpoints correctly for at least one provider (OpenAI automatic, Anthropic `cache_control`, Bedrock, or Gemini `context_cache`).
- Distinguish cache-preserving edits (append-only after the last breakpoint) from cache-invalidating edits (byte changes above a breakpoint).
- Read the cache-hit fields returned by the provider (`cached_tokens`, `cache_read_input_tokens`, etc.) and compute the cache-hit ratio.
- Write a prompt-version doc that names the cache shape, TTL, release cadence, hit-ratio target, and alert threshold.

## Prerequisites

- You have read [chapter 02](../02-prompt-caching-as-a-design-constraint.md).
- A working LLM feature with a system prompt of at least a few hundred tokens (retrieved context, tool schemas, or few-shot examples all count), OR the scaffold in the paired solutions repo.
- Access to at least one provider that exposes cache-hit fields in the API response (OpenAI, Anthropic, Bedrock, or Gemini). Verify the current fields against the provider's docs before starting.

## Problem statement

Your surface's current system prompt was written in a hurry. It contains:

- The persona and rules.
- A timestamp templated in for "grounding."
- The tool schemas.
- A block of static few-shot examples.
- The user's ID templated in for "personalisation."
- The conversation history.
- The current user turn.

The provider caches nothing because the timestamp and user-ID templating destroy the byte-identical prefix. You need to fix the prompt shape and prove the cache landed.

Three hours. Deliverable is the redesign, the measurement, and the cache-posture doc.

## Requirements

### Deliverable 1 — Cache-shape audit of the current prompt

Under `docs/cache-audit-before.md`, a table of the current prompt's sections in the order they appear in the request payload:

| Section | Byte offset | Volatile? | Reason |
|---------|------------|-----------|--------|

Every "volatile: yes" row is a cache killer. Identify at least two before you touch the code.

### Deliverable 2 — Prompt redesign

Refactor the prompt so:

- All stable content (persona, rules, tool schemas, static few-shot, long-lived retrieved context) is at the top of the payload, in a fixed byte-identical order.
- All volatile content (timestamps, per-user IDs, per-turn state, current user message) is at the bottom.
- If your provider supports explicit breakpoints (Anthropic, Bedrock), place breakpoints at the ends of the stable blocks — up to the provider's limit.
- If your provider is automatic-prefix (OpenAI, Gemini automatic), the ordering *is* the design; document where the "everything above this line caches" boundary lands.
- Anything that was in the system prompt purely for logging (timestamps, request IDs) is removed from the prompt and moved onto the observability span instead.

Commit the redesign as a separate change on top of a copy of the current prompt so the diff is legible.

### Deliverable 3 — Measurement

Run a repeatable traffic sample against the redesigned prompt:

- First 20–30 requests to warm the cache (a fresh deploy).
- 100+ requests after warm-up, at a rate high enough to keep the cache warm (within the provider's TTL).
- Log, per request, the `cached_input_tokens` (or provider-equivalent) and the `input_tokens`.

Compute:

- Cache-hit ratio (p50 over the warm window): `cached / input`.
- Cache-write ratio: for providers that expose it (Anthropic `cache_creation_input_tokens`), `writes / (reads + writes)`.
- TTFT delta between cache-hit and cache-miss requests (using the instrumentation from exercise-01).

Do the same run against the *original* prompt (or a control that skips your redesign) so you have a before / after.

### Deliverable 4 — Cache-posture doc

Under `docs/cache-posture.md`, the artefact from [chapter 02](../02-prompt-caching-as-a-design-constraint.md):

```markdown
# Cache posture — <feature> — v<n> — <date>

## Content
<pointer to the redesigned prompt source>

## Cache shape
| Section              | Byte offset | Breakpoint? | Volatile? |
| ...                  |             |             |           |

## Providers wired
- <provider>: <breakpoints or automatic>; TTL <value>

## Metrics (measured in this exercise)
- Cache-hit ratio (p50): <before> → <after>
- Cache-write ratio: <value>
- TTFT delta (cache-miss - cache-hit) p95: <value>
- Effective input-cost delta: <value>

## Release cadence
<how often the prompt will be edited; how "cache-preserving edits" work here>

## Alert
<threshold and window>
```

### Deliverable 5 — Cache-preserving-edit example

Add one *cache-preserving* edit to the redesigned prompt (e.g. appending a new few-shot example *after* the last stable breakpoint). Show, in the doc, why this edit preserves the cache and what the expected hit-ratio dip is. If your provider is automatic-prefix, describe the equivalent append-only edit.

## Starter guidance

- **Start with the payload as bytes**, not as your mental model of the prompt. Dump the exact JSON the SDK sends and look at the ordering.
- **Sort tool-schema JSON keys deterministically.** Non-deterministic key order (from `set` iteration or environment-variable dumps) is a subtle cache killer.
- **Do not delete the timestamps and user IDs — move them.** They usually belong on the observability span (chapter 01) or, if the model actually needs them, in the user-turn area at the bottom of the payload.
- **Traffic pattern matters.** If your feature is 0.1 QPS in the sample, the cache TTL expires between requests and nothing caches. Either raise the sample rate or use a provider TTL extension (Anthropic 1-hour beta, Gemini explicit).
- **Cost-check the write side.** On Anthropic, cache_creation is more expensive per token than the base input rate. Confirm that reads / writes ≥ some ratio (usually ≥ 5) before declaring victory.

## Acceptance criteria

- [ ] `docs/cache-audit-before.md` names at least two cache killers in the current prompt.
- [ ] Redesigned prompt commits are legible as a diff and the redesigned payload passes the byte-identical-prefix check across at least two consecutive requests.
- [ ] Measurement run yields a cache-hit ratio (p50 over the warm window) meaningfully higher than the control (typically 3× or more; provider-dependent).
- [ ] `docs/cache-posture.md` includes the cache-shape table, provider wiring, measured metrics, release cadence, and alert threshold.
- [ ] The doc includes one cache-preserving-edit worked example.
- [ ] The doc explicitly names the provider version / model version the numbers were measured against.

## Stretch goals

Pick any one:

- **Cross-provider port.** Run the same redesigned prompt against a second provider whose cache mechanic differs (e.g. OpenAI automatic + Anthropic explicit) and compare hit ratios and cost deltas.
- **Rolling breakpoint on chat history.** For a multi-turn surface, implement a rolling cache breakpoint placed just before the current user turn. Measure the incremental savings as conversations grow.
- **Cache-hit ratio dashboard.** Wire a per-version cache-hit ratio chart into your team's observability stack. Include a release-marker overlay so future releases show up as ratio dips-and-recoveries.
- **Cache-write-cost floor.** Derive the minimum traffic rate at which caching pays off (given the provider's write-vs-read pricing and TTL). Contrast against your feature's current QPS.

## Deliverable

Submit as `mod-109/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/cache-audit-before.md`, `docs/cache-posture.md`.
- The redesigned prompt source and the diff-vs-original.
- The measurement script and the raw run log.
- A short `README.md` naming the feature and pointing at the posture doc.
