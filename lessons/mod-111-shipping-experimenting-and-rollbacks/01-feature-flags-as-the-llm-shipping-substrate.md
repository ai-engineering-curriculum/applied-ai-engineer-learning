# Feature Flags as the LLM Shipping Substrate

## Why a flag layer, before anything else in this module

Every other rollout technique in this module — canary, shadow, A/B, hot rollback — assumes there is a runtime switch you can flip *without shipping code*. If the only way to change the prompt is a code deploy, canaries run at deploy cadence (not traffic cadence), A/B assignment is compile-time, and rollback is another deploy. That was already too slow for a stateless CRUD service in 2018. It is untenable for an LLM feature where the failure mode is "the tone of the model changed overnight after a provider alias swap and you need the new prompt live for support tier 1 in ten minutes."

The applied-AI engineer's job is not to reinvent the flag substrate. The industry has settled on two working answers — **LaunchDarkly** as the dominant commercial platform and **OpenFeature** as the CNCF-hosted vendor-neutral SDK/API standard. Either one is a fine choice; picking neither and rolling your own is the anti-pattern. This chapter names the shape of the flag layer LLM features need, what makes LLM flags different from CRUD flags, and how the flag decision integrates with the observability substrate from [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md).

## What "feature flag" means here — four flag shapes

Not every LLM flag is the same shape. Confusing them is the first source of rollout mishaps.

- **Boolean release flags** — "is the new support-agent surface on for this user?" — the canonical on/off toggle. Roll from 0% → 1% → 5% → 25% → 100%. Rollback is flipping to `false`.
- **Multivariate flags** — "which prompt version does this turn use?" — the flag value is a string identifier (`support_answer_v14`, `support_answer_v15_canary`) and the code branches on it. This is the shape most LLM prompt / model / chain rollouts take.
- **Configuration flags** — "what is the current `max_tokens` for this feature?" — a scalar or JSON blob delivered by the flag SDK. Cheaper than a redeploy, treated as first-class code by the flag platform.
- **Kill-switch flags** — "is this feature disabled entirely, falling back to a static or degraded surface?" — a boolean whose *default* semantics matter: if the flag SDK cannot fetch, the kill switch must **fail closed** (surface stays alive, feature is off).

Every LLM feature this module ships uses at least the multivariate flag for prompt/model selection and the kill-switch flag for graceful degrade. The boolean release flag is often layered on top: cohort A gets the feature *at all*, cohort B does not, and inside cohort A the multivariate flag picks the prompt version.

## Why LaunchDarkly or OpenFeature and not "a YAML file in a repo"

The pull to build your own flag layer is strong. It is one HTTP GET, right? Two failure modes take out that build-your-own version:

1. **Latency and availability.** A flag lookup is on the critical path of every LLM turn. If your flag layer is a synchronous HTTP call to a config service, you added an SLO-eating dependency to a code path that already has a 500–3000 ms provider call attached to it. Both LaunchDarkly's SDK and every OpenFeature-conformant provider ship with an **in-memory cache** that streams updates from the control plane (Server-Sent Events for LaunchDarkly, provider-specific for others). Lookups are microseconds, not milliseconds, and survive a control-plane outage.
2. **Targeting rules and audit.** By the third rollout you will need "50% of internal users, 5% of tier-2 accounts, and every account tagged `pilot`, but not in the EU region." That is not code you should be writing; it is what commercial and CNCF-standard flag systems already do, with per-flag audit logs required by SOC 2 and enterprise procurement.

**LaunchDarkly** is the reference commercial implementation: server-side SDKs in every mainstream language, targeting rules editor, experimentation product, audit trail, and a well-documented failure model. **OpenFeature** is the CNCF-graduated open standard: a vendor-neutral SDK/API layer that lets you swap providers (flagd, GrowthBook, LaunchDarkly's OpenFeature provider, Flagsmith, ConfigCat) without rewriting call sites. Use OpenFeature at the code seam when you want portability; use LaunchDarkly directly when you have picked it and want the fastest path to the full targeting product. Do not build both.

The rest of this chapter shows the same code shape written to the OpenFeature SDK because it is the vendor-neutral surface; the semantics carry over to the LaunchDarkly SDK 1:1.

## The call-site shape: `evaluate(flag, context) → value`

Every OpenFeature-shaped flag lookup takes a **flag key** and an **evaluation context** and returns a typed value. The evaluation context is where the LLM-specific fields live.

```python
from openfeature import api
from openfeature.evaluation_context import EvaluationContext

client = api.get_client()

def choose_prompt_version(user_id, tenant_id, surface, locale):
    ctx = EvaluationContext(
        targeting_key=user_id,                # stable hash key — controls sticky bucketing
        attributes={
            "tenant_id": tenant_id,
            "surface": surface,               # "chat_panel" / "inline_completion" / ...
            "locale": locale,
            "internal": user_id in INTERNAL_ALLOWLIST,
            "account_plan": plan_for(tenant_id),
        },
    )
    return client.get_string_value(
        flag_key="support_agent.prompt_version",
        default_value="support_answer_v14",   # LKG version if the flag layer fails
        evaluation_context=ctx,
    )
```

Three things about this signature repay careful attention.

**The `targeting_key` is the sticky bucketing key.** LaunchDarkly and every serious OpenFeature provider hash `targeting_key` into `[0, 1)` and route deterministically. If you pass `user_id`, one user always lands in the same variant across turns — which is what you want for A/B and for a coherent support conversation. If you pass a per-request UUID, every turn gets a fresh variant and the trace timeline is unreadable. This is the single most common mis-wire on LLM rollouts.

**The `default_value` is the last-known-good.** If the flag SDK cannot reach the control plane and its cache is cold (fresh process, network partition, credential rotation), the SDK returns `default_value`. For a prompt-version flag, the default must be the current stable version — not `None`, not an empty string, and not the *new* version you are rolling out. Getting this wrong means a network partition ships the new prompt to 100% of users by accident. Chapter 04 makes this the central invariant of the rollback story.

**Attributes are lower-cardinality than they look.** LaunchDarkly rate-limits contexts by attribute cardinality; if you emit an attribute per session ID, the platform will complain (or bill you). Attributes are for targeting rules, not for identifiers — put IDs in `targeting_key` and put cohort tags in `attributes`.

## What makes LLM flags different from CRUD flags

Flag mechanics are the same. The failure modes are not.

- **The flag value is often a version identifier, not a boolean.** A prompt/model/chain rollout is naturally multivariate. The flag returns a string like `"support_answer_v15_canary"` and the feature code looks that up in a **prompt registry** ([chapter 04](04-hot-rollback-of-a-bad-prompt.md)) to fetch the actual template. Do not embed the prompt text in the flag value; the flag layer is not a CMS.
- **A rollback wants to shift 100% of traffic in seconds, not the next deploy.** Prompt bugs surface in production because there is no unit test for "did the tone change." When they do, the rollback SLA is minutes, not hours. This is exactly what the flag layer buys — the reason to invest in it is the rollback path, not the rollout path.
- **The variant needs to appear on the trace span.** Every LLM span emits `app.prompt.id` and `app.prompt.template_hash` (see [mod-110 chapter 01](../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)). Add `app.experiment.arm` — the flag value that produced this turn — so any downstream analysis can join spans by cohort without a second table.
- **The provider changes what the flag value points to under you.** `gpt-4o` is an alias that Anthropic-style silently rolls; if your flag value is `gpt-4o` the flag layer will loyally return `gpt-4o` while the provider swaps in a new snapshot. Pin dated model IDs in the flag value (`gpt-4o-2024-08-06`) so the flag is what actually changes.
- **Cost tracks the cohort.** A canary at 5% that costs 3× the control accumulates the daily cost of a 15% general rollout of the control. The flag layer is the seam where you can *both* size the cohort *and* budget its cost — see the CPSO discipline in [mod-109 chapter 04](../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md).

## Targeting rules for LLM cohorts

The LaunchDarkly and OpenFeature targeting rule editors let you compose a rollout by cohort without redeploying. The four cohort shapes an LLM feature actually uses:

1. **Internal-first.** Rule: `internal == true` → new variant. Rule everyone else: control. Ship the new prompt to your own employees before customers touch it. Cheap; blast radius is a room of people who can DM the on-call.
2. **Named allowlist / design partner.** Rule: `tenant_id IN {…}` → new variant. Rule everyone else: control. Design partners are pre-briefed; they know they are on the new thing.
3. **Percentage rollout by stable key.** Rule: `bucket(user_id) in [0, 5)` → new variant. Rule: `bucket(user_id) in [5, 100)` → control. 5% canary. This is where the observation regime from [chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md) plugs in.
4. **Segment intersection.** Rule: `tenant_id IN pilot_segment AND locale == "en-US" AND account_plan != "enterprise"` → new variant. Slice the exposure to what you actually want to observe. Every additional predicate is a trade between *insight* (this cohort looks like the population you care about) and *statistical power* (fewer users → wider confidence intervals).

Targeting rules are edited in the flag platform's UI, not in a config repo, and every edit is audit-logged. That audit trail is load-bearing for the incident retro in [chapter 04](04-hot-rollback-of-a-bad-prompt.md) — when the bad-prompt push is post-mortemed, the flag-platform audit log is the primary source of truth for who changed what when.

## The kill-switch flag

Every LLM feature ships with a **feature-level kill switch** — a boolean flag whose sole purpose is "turn this feature off entirely and route to the fallback surface." The fallback is *not* the previous prompt version; the previous version is what a rollback goes to. The kill switch is what fires when the feature itself is causing incident-shaped damage (guardrail false-positive spike, provider outage cascading, cost explosion) and the correct answer is "stop calling the model for this feature altogether."

```python
if not client.get_boolean_value("support_agent.enabled", default_value=True, evaluation_context=ctx):
    return static_fallback(user_query)   # canned copy: "sorry, this feature is briefly unavailable"
```

Two invariants:

- The kill switch **default matches the current state**. If the feature is normally on, default is `True`; the flag SDK's fallback then keeps the feature on during a control-plane outage. Flip the default only when the feature is normally off.
- Every LLM feature has one. It is written on day one of the feature, not the day of the incident. The incident is when it fires, not when it is designed.

The kill switch is the safety net every subsequent chapter assumes. It is also the interface to the guardrails work in [mod-107 chapter 04](../mod-107-guardrails-and-safety-on-the-surface/README.md) — the guardrail's "hard stop" is a kill-switch flip.

## Wiring the flag decision into the trace

The trace substrate from [mod-110](../mod-110-observability-feedback-and-incidents/README.md) needs to know which variant every turn was served. Emit it on the outer turn span and let it propagate:

```python
prompt_version = choose_prompt_version(user_id=user.id, tenant_id=tenant.id,
                                       surface="chat_panel", locale=user.locale)

with tracer.start_as_current_span("support_agent.turn") as span:
    span.set_attribute("app.feature", "support_agent")
    span.set_attribute("app.prompt.id", prompt_version)               # the flag's return value
    span.set_attribute("app.experiment.arm", prompt_version)          # same field name across features
    span.set_attribute("app.flag.evaluated_at", int(time.time()))     # so late variant changes are visible
    template = prompt_registry.get(prompt_version)
    span.set_attribute("app.prompt.template_hash", template.sha256)
    ...
```

`app.experiment.arm` is the join key downstream analysis uses to compare variants. Emit it whether or not the variant is part of a formal experiment — the analytical shape is the same and having the field always present means the analyst does not have to reason about when it is and is not populated.

Both LaunchDarkly and OpenFeature emit an **evaluation event** (LaunchDarkly calls it a flag event; OpenFeature calls it a `ProviderEvaluationResult`) with the variant, the rule that matched, and the evaluation context. Route these events into your data warehouse alongside the trace log; that is what makes the "which cohort saw which variant" retrospective query possible.

## Anti-patterns

- **A per-request random UUID as the `targeting_key`.** Every turn gets a fresh variant. A/B is impossible, sticky rollout is impossible, and support conversations flip prompts mid-turn. The `targeting_key` must be stable across a user's session — usually `user_id` or `session_id`, whichever the surface's identity model supports.
- **The prompt text embedded in the flag value.** The flag becomes a CMS. Prompt edits happen in the flag UI (no code review, no diff, no versioning). Use a prompt registry ([chapter 04](04-hot-rollback-of-a-bad-prompt.md)) and let the flag value be a *pointer* into it.
- **`default_value` is the new variant.** A network partition ships the new thing to 100% of traffic while your dashboards show 0% rollout. The default is always the last-known-good stable version.
- **No `app.experiment.arm` on the span.** The rollout is invisible after the fact. The support ticket says "the answer was weird yesterday" and you cannot tell which variant produced it. Every LLM span carries the variant.
- **A "config repo" instead of a flag layer.** Rollouts and rollbacks happen at the cadence of the deploy pipeline. This is what the flag substrate exists to prevent — do not skip building it because your first flag was "just a boolean."
- **No kill switch.** When the incident hits, you page for a deploy to disable the feature. The runbook item that should have taken 15 seconds takes 40 minutes.
- **Cardinality abuse in `attributes`.** Emitting per-session IDs as attributes overwhelms the flag platform. Attributes are for targeting rules; IDs are for `targeting_key`.

## Summary

- **Adopt a flag layer before you need a rollout.** LaunchDarkly (commercial) or OpenFeature (CNCF-hosted vendor-neutral SDK) — both give in-memory-cached lookups, targeting rules, and audit logs. Do not roll your own.
- LLM features use four flag shapes: **boolean release flags** (feature on/off), **multivariate flags** (prompt/model/chain version), **configuration flags** (max_tokens, top_p), and **kill-switch flags** (fail-closed graceful degrade). Every feature ships with at least a multivariate flag and a kill switch.
- The call site is `client.get_string_value(flag_key, default_value, ctx)`. The `targeting_key` is a **stable user or session ID**, not a per-request UUID. The `default_value` is the **last-known-good** — never the variant being rolled out.
- **Pin dated model IDs**, not provider aliases, in flag values. Otherwise the provider changes what the flag points to and you cannot tell.
- Every LLM span emits `app.experiment.arm` (the flag value that produced the turn), whether or not the variant is part of a formal experiment. This is the join key for [chapter 03](03-ab-experiment-interface-to-the-experimentation-platform.md).
- Every feature ships with a **kill switch** whose default matches the current state. The kill switch is the interface guardrails, cost governance, and the on-call runbook all trigger.
- The flag platform's **audit log** is the primary source of truth for the post-incident retro. Rules edited in the UI carry more forensic weight than commits, because incidents happen at flag cadence.

[Chapter 02](02-canary-and-shadow-launches-with-pre-registered-gates.md) turns from "how to size a cohort" to "how to observe a cohort with pre-registered promotion criteria" — canary and shadow launches with quality, safety, and cost gates decided *before* the rollout starts.
