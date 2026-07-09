# exercise-01: Prompt Cache Aware System Prompt

**Estimated effort:** 3 hours

## Objective

Take an unstructured, all-in-one-block system prompt for a real-shaped product surface and refactor it into a cache-friendly, four-block layout that survives typical live edits without silent invalidation. Verify the refactor against at least one live provider by measuring the reported cached-token counts before and after.

## Learning outcomes

By the end you should be able to:

- Identify the four prompt blocks (stable role/policy, stable structural artefacts, conversation-scoped context, per-request tail) inside a real prompt.
- Move per-request variability out of the cache-key region without changing the model's observed behaviour.
- Choose and place Anthropic `cache_control` breakpoints (or the equivalent surface on OpenAI / Gemini / Bedrock).
- Read cached-token counts out of the provider response and compute a cache-hit rate.
- Name at least three anti-patterns from [chapter 01](../01-cache-aware-prompt-design.md) that the original prompt exhibits.

## Prerequisites

- You've read [chapter 01](../01-cache-aware-prompt-design.md).
- You have an API key (or a shared team key) for at least one of: OpenAI, Anthropic, Google Vertex AI Gemini, AWS Bedrock. Anthropic is easiest to demo explicit breakpoints; OpenAI is easiest to demo automatic prefix caching.
- You are comfortable calling the provider SDK from Python or Node.

## Problem statement

You are handed the system prompt below, currently in production on a co-pilot reply-drafting surface for a helpdesk product. The support team has been asking for "small copy tweaks" every couple of days. The finance team has flagged that per-request cost has doubled over the last quarter for no obvious reason. Both symptoms point at the same problem: the prompt is not cache-friendly and every edit silently invalidates the cached prefix.

### The prompt (as it is today)

```
You are a helpful assistant for {tenant_name} in the {tenant_industry} industry.
Today is {current_date_time_utc}. The current agent's name is {agent_first_name}
({agent_email}) and their tier is {agent_tier}. Trace id: {trace_id}.

Your job is to draft replies to customer emails. Follow these rules:

1. Match the tone of {tenant_name}'s brand voice (friendly, formal, or casual —
   defer to the tenant's own preference: {tenant_tone}).
2. Never promise refunds or credits without checking the policy passages below.
3. If you are unsure, say so — do not invent policy.
4. Cite the passage id (e.g. [P-137]) after any factual claim about our policy.

Here are the tools you may call:

{tools_json_dumped_by_python_default_repr}

Here are 6 example drafts from our team (randomly selected each run for
diversity):

{six_random_examples_from_the_pool_of_50}

Retrieved policy passages for THIS customer's email:

{retrieved_passages}

Previous replies on THIS ticket:

{prior_replies_on_ticket}

The customer email you must draft a reply to:

{customer_email}
```

Every field wrapped in `{…}` is templated at request time by the current renderer.

## Requirements

Produce three deliverables in a single markdown file plus supporting code:

### Deliverable 1 — Written diagnosis (≤ 1 page)

- Label every `{…}` field with its **change cadence**: per-request, per-conversation, per-tenant, per-deploy, per-hour, per-day.
- Call out at least three anti-patterns from [chapter 01](../01-cache-aware-prompt-design.md) present in the prompt above. Name the anti-pattern and the specific line that triggers it.
- Explain in one paragraph why the cost per successful outcome would drift upward over a quarter given the current shape.

### Deliverable 2 — Refactored prompt

Refactor the prompt into the four-block layout from chapter 01. Requirements:

- **Block 1 (stable role/policy)** contains no template variables. If the current prompt inlines a per-tenant tone rule, move it (see block 3).
- **Block 2 (stable structural artefacts)** contains tools and few-shot examples. The tool JSON must be canonicalised (sorted keys, stable whitespace, deterministic argument ordering). The few-shot pack must be in a fixed order, not randomised per request.
- **Block 3 (conversation-scoped context)** contains the retrieved passages and any per-tenant policy overlay.
- **Block 4 (per-request tail)** contains the customer email, prior replies on this ticket, and any timestamp / trace-id.
- On Anthropic, mark cache breakpoints after blocks 1, 2, and 3. On OpenAI, describe in a comment where the automatic prefix cache will end and why.
- On any surface: nothing per-request may appear above the last cache boundary.

### Deliverable 3 — Empirical verification

Write a small script that:

1. Sends a warm-up request through the *refactored* prompt with a representative user turn.
2. Sends 10 subsequent requests with different user turns but the same fixed blocks.
3. For each response, reads the cached-input-token count and the uncached-input-token count from the provider response body. Log them.
4. Prints a summary table: mean cached-token count, mean uncached-token count, cached-token ratio.
5. Repeats the whole thing on the *original* prompt for comparison.

You may mock the retrieval and the tool schema (use fixed strings) to keep the exercise self-contained.

## Starter guidance

- **Read the provider's response schema first.** Every provider surfaces cached-input tokens somewhere (Anthropic: `usage.cache_read_input_tokens` and `usage.cache_creation_input_tokens`; OpenAI: `usage.prompt_tokens_details.cached_tokens`). Confirm the exact field name from current docs before writing the parser — the fields have moved.
- **Instrument the render, not just the send.** A useful debug tool is to serialise the rendered prompt to a stable string and print the SHA-256 hash of the "cache-key" region (blocks 1–3). If the hash changes between requests when you didn't expect it to, that's your bug.
- **Canonicalise tool JSON with a helper**, not with `json.dumps(sort_keys=True)` sprinkled ad-hoc. `sort_keys=True` fixes key ordering but doesn't fix array ordering, nested dicts inside arrays, or float formatting. Write a small `canonicalise(tools) -> str` that deep-sorts keys, forbids unstable float printouts, and always emits UTF-8 without BOM.
- **Handle the tenant-tone rule carefully.** It genuinely varies by tenant. The two acceptable homes for it are (a) block 3, as a small "tenant tone overlay" that follows the same ordering discipline, or (b) block 1 with a hard-coded neutral policy and the per-tenant variation applied downstream (edit the drafted reply's tone in a small second call). Either is defensible; pick one and justify.
- **On the random few-shot pack.** The chapter names this as an anti-pattern. If the team wants a randomised-examples experiment for diversity, propose a shape that runs the experiment offline (choose the best fixed order for the next quarter) rather than randomising at request time.
- **Verify with a small n before scaling.** 10 requests per arm is enough to see the effect and stay cheap. Do not run a 1000-request bake-off for this exercise.

## Acceptance criteria

- [ ] Every `{…}` field in the original prompt is labelled with its change cadence.
- [ ] At least three named anti-patterns from chapter 01 are identified in the original prompt, with the specific line for each.
- [ ] The refactored prompt is organised into blocks 1–4 in the correct order.
- [ ] No per-request template variable appears in block 1 or 2. (The rendered prompt bytes for blocks 1 and 2 must be identical across all 10 verification requests.)
- [ ] Tool JSON is serialised through a canonicalising serializer, not `json.dumps` with default arguments.
- [ ] Anthropic version marks `cache_control` breakpoints in the correct positions (or OpenAI version documents the automatic-prefix boundary).
- [ ] The verification script prints a cached-token ratio for both prompts and the refactored one is higher.
- [ ] The report includes the SHA-256 hash of the block-1-through-block-3 region for at least three consecutive requests and shows they are identical on the refactored prompt.
- [ ] The proposed home for the tenant-tone rule (block 3 overlay vs post-processing pass) is stated with a one-line justification.

## Stretch goals

Pick any two:

- **Multi-provider portability.** Run the same verification against a second provider (if you started on Anthropic, run OpenAI too). Report the differences in cache-hit ratio and named-vs-implicit cache mechanics.
- **Time-decay probe.** Send a request with the refactored prompt, wait varying idle intervals (10 s, 60 s, 5 min, 15 min, 30 min), and measure the cache-hit ratio at each. Report the observed TTL and compare to the provider's documented value.
- **Adversarial rendering.** Introduce a change to the retrieved-passages format (add a trailing space, change a line ending, reorder passages) and show which changes break the cache-key region for blocks 1–3 (they should not — block 3 is conversation-scoped and *may* rechange between users, but per-passage-order shifts inside block 3 should still be caught).
- **Deploy-time diff.** Write a CI check that serialises the rendered prompt (with fixed dummy variables) at every commit and diffs the block-1-2 hash against the previous commit. A change flags the PR as "prompt-cache-invalidating."
- **Cache-hit SLO.** Propose a cache-hit ratio SLO for this surface — target, window, population, evaluator, owner — and justify the target.

## Deliverable

Submit as `mod-102/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `report.md` — deliverables 1 and the written section of deliverable 3.
- `prompts/original.md` and `prompts/refactored.md` — the two prompts (deliverable 2).
- `verify.py` (or `.ts`) — the verification script (deliverable 3).
- `results.md` — the printed summary tables from the script run.
