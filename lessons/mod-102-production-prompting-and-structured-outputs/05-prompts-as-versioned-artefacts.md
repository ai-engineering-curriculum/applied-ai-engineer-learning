# Prompts as Versioned Artefacts

## Why prompts need version control at all

A production prompt is not documentation. It is code. It has an interface (the inputs the caller provides), a contract (the schema it promises to fill), a runtime cost (tokens, cache hits, latency), and failure modes (hallucinations, refusals, format drift). Change a comma and the model behaves differently. Change a rubric line and safety metrics move. Change a few-shot ordering and prompt-cache hit rate collapses (chapter 01).

The industry treats prompts inconsistently. Some teams check them into `.py` files as string constants — versioned by git, but coupled to code deploys. Some teams put them in a database that a PM can edit at runtime — decoupled from deploys, but with no CI, no diff, no code review. Some use a "prompt registry" product (LangSmith Hub, LangFuse, PromptLayer, Weights & Biases Weave, Vellum, Humanloop, Bedrock Prompt Management, Vertex AI Prompt Optimizer) — better tooling, but still needs the practices this chapter names.

The right question is not "which of these do we buy?" — it is: **what does a versioned prompt need to guarantee, and how do we get those guarantees on whatever substrate we picked?** This chapter names the guarantees, shows how to build them, and lands the rollback flow that lets you undo a regressing prompt in minutes, not the next code deploy.

## Five guarantees a prompt-as-artefact must give you

Whether you store prompts in git, in a registry, or in a database, the guarantees below are the hard requirements. Anything less and you cannot ship prompt changes with confidence.

### 1. Identity — every rendered prompt has a version id

Every prompt sent to the model is the render of an identified artefact, not an ad-hoc string. `system_prompt_v3.jinja` at git SHA `4a2b1c…` rendered with tenant=`acme` and locale=`en-US` produces a prompt whose full identity is `(template_id, template_version, render_variables_hash)`. That identity is logged with every completion.

Concrete: the completion log row includes `prompt_id`, `prompt_version`, and a hash of the rendered bytes. When someone asks "which version generated this bad response?" the answer is a git SHA or a registry version id, not "the one that was live yesterday around 3."

### 2. Immutability — a published version is frozen

Once you publish `v3`, you never mutate `v3`. Bug fix, typo, one-comma tweak — all go into `v4`. This is a rule you enforce with tooling, not with discipline; a "you can edit yesterday's version" mode in a registry is a feature you should turn off.

Why: everything downstream — cache hits, eval results, telemetry aggregations, incident forensics — references the version id. Mutating a published version silently invalidates every historical claim tied to it.

### 3. Environment binding — the runtime knows which version is live for which surface

Which prompt version is live in production, which is on the beta canary, which is in staging, and which experiments are running are configuration state, not code state. A prompt version becomes live for a surface via a change to a config artefact (a `prompts.yaml`, a feature-flag rule, a registry environment binding), not by editing the prompt file itself.

Concrete shape:

```yaml
# prompts.yaml
- surface: co_pilot_reply_drafting
  environments:
    prod: v7
    canary: v8
    staging: v9
  experiments:
    exp-2026-07-drafting-tone:
      control: v7
      variant: v8_tone_experiment
      split: 0.10   # 10% of eligible traffic to variant
```

Deploying a new prompt version is a config change — small, reviewable, and reversible in minutes. A rollback is a one-line revert of that config change. The prompt code file is unchanged.

### 4. Regression fixtures — every change runs against a pinned dataset

Every new prompt version runs against a pinned regression suite before it can be promoted. The suite is a dataset — inputs, expected shapes, sometimes labelled outputs — that lives in git next to (or alongside) the prompt.

Structure:

```
prompts/co_pilot_reply_drafting/
├── v7/
│   ├── system.md
│   ├── few_shot.jsonl
│   └── metadata.yaml
├── v8/
│   ├── system.md
│   ├── few_shot.jsonl
│   └── metadata.yaml
└── fixtures/
    ├── golden.jsonl          # 50 canonical inputs + reference outputs
    ├── regression.jsonl      # inputs collected from prior incidents
    ├── adversarial.jsonl     # inputs from prompt-injection probes (mod-107)
    └── freshness.jsonl       # inputs whose correct answer depends on fresh data
```

For a new version to be promotable, it must pass:

- **Golden set**: task-success rubric ≥ current-prod score (never regress on the canonical inputs).
- **Regression set**: no new failures on inputs collected from prior incidents.
- **Adversarial set**: prompt-injection attack-success-rate ≤ current-prod value.
- **Cache health check**: the serialised prompt's stable prefix has not shifted from the prior version (see chapter 01).

Fixture design is a topic mod-106 owns in depth. For this chapter, the point is: **the fixtures live with the prompt, are versioned with it, and gate promotion.**

### 5. Rollback semantics — a bad version can be undone in minutes

A regression in a prompt version must be undoable without a code deploy. If your rollback path requires "revert the git commit, cut a release, promote through canary" you have not built rollback; you have built a slow rollout.

Rollback is a config change: `prod: v8 → prod: v7`. Applied in seconds. Verified in the next minute. No user visible restart.

This is only possible if you have all four prior guarantees. Immutability (v7 is still there because you didn't overwrite it). Environment binding (the runtime reads the config, not a hardcoded string). Fixtures (you can prove the rollback state is healthy). Identity (you can tell from telemetry whether the rollback worked).

## Storage substrate — git, registry, or database

The five guarantees are substrate-independent. Pick the substrate based on who edits prompts and how fast changes must land.

### Git-only

Prompts as files in the application repo. Deployed with the code. Versioned by git. Promoted via PR + merge + deploy.

- **Fits when**: only engineers author prompts, and the deploy cadence (< 1 hour) is fast enough to be a live channel for prompt tweaks.
- **Wins**: no extra infrastructure, full diff / review / rollback tooling from git.
- **Loses**: PMs and content designers can't ship a copy tweak without a code deploy; hot rollback still needs a config-only deploy path.

### Git for prompts + separate config for binding

The pattern above: prompts live in the app repo under `prompts/`, and a small `prompts.yaml` (also in git, or in a config store) binds versions to environments and experiments. This is the shape most teams should start with.

- **Fits when**: engineers author prompts, but the binding layer must be hot-changeable independent of code deploys (rollback in minutes, canary splits).
- **Wins**: strong review on prompt content (full PR review) + fast binding changes.
- **Loses**: PMs still can't self-serve without an engineer.

### Prompt registry (external service)

A dedicated product where prompts live: LangSmith Hub, LangFuse Prompts, PromptLayer, Vellum, Humanloop, Weave, Bedrock Prompt Management, Vertex AI Prompt Optimizer. Non-engineers can edit, versions are managed, evaluations often built-in.

- **Fits when**: multiple non-engineer roles (PM, content, designer, ops) need to author prompts, or you're managing enough surfaces that per-repo storage becomes a governance headache.
- **Wins**: multi-role authoring, built-in eval hooks, a UI for version comparison, provider-native for Bedrock / Vertex.
- **Loses**: dependency on the registry as a runtime, cross-provider portability if you pick a provider-native one, and the same guarantees discipline — a registry with mutable versions or no fixtures is worse than git.

Whichever substrate you pick, verify the five guarantees against it. A registry that lets you overwrite a published version breaks guarantee 2, and no amount of UI polish makes up for that.

## Rendering: templates, variables, and cache-safety

A prompt template is a function that turns inputs into a rendered prompt. Two shapes:

- **String-substitution templating** (`{tenant}` placeholders in Jinja, mustache, Python f-strings, `%s`). Simple, fast, safe when the inputs are trusted structural values.
- **Structural templating** (compose the prompt from typed blocks — a `Policy` block, a `Rubric` block, a `FewShotPack` block, a `RetrievalContext` block). Better for chapter-01 cache alignment because the block ordering is not a template author's judgment call.

Two rendering-time rules:

- **Never templatise inside a cache-key block.** If `{tenant}` renders into the system prompt block, you've broken the prompt-cache invariant from chapter 01. Push all template variables into the tail block (or fetch tenant policy in the conversation-scoped block that already accepts variation).
- **Deterministic ordering.** A `for` loop over a dict must sort by key before rendering. A JSON serializer inside a template must be canonicalised. Nondeterministic rendering breaks caching and reproducibility.

## Regression fixtures: what to record, what to compare

Fixtures for a prompt version have two axes: **what to run** and **what to compare**.

### What to run

- The pinned input dataset (`golden.jsonl`, etc).
- Against the exact model + provider + parameter set the surface uses in prod.
- With the exact tool schemas and retrieval context the surface uses in prod (frozen at fixture-capture time, not fetched live).

The last point matters: if your regression suite dispatches real retrieval, changes in the corpus poison the results. Freeze the retrieval context per fixture — capture the top-k retrieved documents when the fixture was authored, replay them on every run.

### What to compare

Structured extractions: field-level exact / fuzzy match against the reference. Report per-field precision / recall / F1 and the "all critical fields correct" rate.

Retrieval-with-citations: rubric-graded answer correctness, citation faithfulness (does each cited span support the sentence), refusal correctness on the "no good evidence" subset. mod-106 owns the rubric templates.

Co-pilot drafts and chat: rubric-graded on a fixed judge model. Do not use the same model you are evaluating as the judge; it will systematically prefer its own outputs.

Cost and cache: token counts and cached-token counts per fixture. A version that regresses cost-per-outcome even at the same task-success rate is a regression.

Latency: fixture runs are not a latency benchmark (they don't reflect production concurrency), but a step change in average tokens generated per response often correlates with latency at load.

### The promotion gate

```
if all([
    golden.task_success >= prod.task_success,
    regression.task_success >= prod.task_success,
    adversarial.attack_success <= prod.attack_success,
    cache.stable_prefix_bytes == prod.stable_prefix_bytes,
    cost_per_outcome <= prod.cost_per_outcome * 1.10,  # tolerance
]):
    promote(new_version)
else:
    block(reasons=[...])
```

The tolerance on cost lets you accept a small cost regression for a task-success win; there is no tolerance on the cache-hit invariant because that is a mechanical property, not a judgment call. Adjust to your team's risk tolerance and document the numbers in the promotion policy.

## Rollback in production

The rollback flow:

1. An alert fires (task-success dropped, cost-per-outcome spiked, retry-rate step-changed, over-refusal spiked — any of the mod-101 chapter 02 SLOs).
2. Oncall diagnoses whether the alert correlates with the most recent prompt promotion (the completion log has `prompt_version` on every row — group by it).
3. If yes, oncall reverts the binding: `prod: v8 → prod: v7` in the config.
4. Fleet picks up the config change within one refresh cycle (seconds to minute — this is your config-refresh SLO).
5. Alert-condition-should-clear latency is measured; if it doesn't clear in the expected window, escalate.
6. Post-incident, the failing version stays as an artefact (do not delete). The regression case that slipped past the fixtures gets *added* to the regression fixture so the next promotion catches it.

Step 6 is the loop that turns a rollback into learning. mod-110 owns the incident-to-fixture pattern in detail.

### What rollback does not fix

- **In-flight requests.** A request that started against v8 finishes against v8. Rollback affects new requests only. If v8 has a "poisons the conversation state" bug, users on that conversation must start a new one.
- **Cached prefixes on v8.** The provider cache still has v8's prefix; when traffic returns to v7, it re-seeds v7's cache. Expect a temporary cost bump.
- **Downstream side effects.** If v8 wrote wrong records to a database before rollback, rollback doesn't undo the writes. mod-107 covers surface-side "confirm before commit" for irreversible actions.

## What lives in the prompt vs what lives around it

A common failure mode: everything ends up in the prompt string because "it's the easy place to change it." A good version boundary matches how you want to change things.

| Belongs in the prompt version | Belongs in code | Belongs in config |
|---|---|---|
| Role definition, tone rules | Retry loop policy | Which prompt version is live per environment |
| Rubric / instructions | Schema (Pydantic model) | Which model / tier is used per surface |
| Few-shot examples | Tool dispatch, authorisation | Feature-flag / experiment splits |
| Refusal policy | Guardrails (in / out) | Rate limits, timeouts |
| Format / output-shape guidance | Validation logic | Provider region, quota routing |

The Pydantic model in particular is *code*, not a prompt asset. Changing the schema is a code change with type-checker consequences. If the prompt version references the schema, they should be version-compatible: prompt `v8` requires schema `v3.x`.

## Summary

- Prompts are code with a runtime cost. They need version control, immutable published versions, environment binding, regression fixtures, and hot rollback.
- Every rendered prompt has an identity — `(template_id, version, render_variables_hash)` — logged with every completion.
- Once published, a version is frozen; edits become new versions.
- Binding is config, not code — you switch which version is live via a config change so rollback takes minutes, not a redeploy.
- Fixtures live with the prompt and gate promotion. Golden, regression, adversarial, and cache-health checks all run against a pinned dataset.
- Substrate (git-only, git + config, external registry) doesn't matter as long as the five guarantees hold. A registry that permits published-version mutation is worse than git.
- Templating rules: no per-request variables inside cache-key blocks; deterministic ordering everywhere.
- Rollback is a config revert. In-flight requests, cached prefixes, and committed side effects are not undone by rollback — plan for them separately.
- The prompt version holds instructions, rubric, few-shot, format guidance. Schema, retry policy, dispatch, and guardrails live in code.

---

**End of mod-102 lectures.** You now have the five load-bearing pieces of production prompting: cache-safe layout, schema-safe outputs, honest streaming with cancellation, bounded retry and repair, and prompts as first-class versioned artefacts. mod-103 wires these into retrieval-with-citations; mod-104 into tool calling; mod-105 into product-shaped agents. mod-106 formalises the fixture-and-eval loop that gates every prompt change you'll ever ship on top of what you built here.
