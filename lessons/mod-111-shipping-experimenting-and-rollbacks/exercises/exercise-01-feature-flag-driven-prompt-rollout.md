# exercise-01: Feature-Flag-Driven Prompt Rollout

**Estimated effort:** 3 hours

## Objective

Wire an OpenFeature (or LaunchDarkly) SDK into one LLM feature so that a new prompt version can be turned on for a specific cohort at a specific percentage without any code deploy. Prove the rollout works by flipping between three prompt versions under load, without touching source control. The deliverable is the flag layer, not the eventual A/B result — the exercise is about the substrate every subsequent chapter builds on.

## Learning outcomes

By the end you should be able to:

- Configure an OpenFeature-conformant client (or LaunchDarkly SDK) in your feature service with an in-memory cache and a live update channel.
- Model a prompt / model / kill-switch flag hierarchy at four levels of granularity: feature-on/off, cohort %, prompt version, model version.
- Use a **stable `targeting_key`** and a **last-known-good `default_value`** so cohort assignment is sticky and fail-closed on a control-plane outage.
- Emit `app.experiment.arm`, `app.prompt.id`, `app.flag.evaluated_at` on every LLM span so the trace substrate ([mod-110 chapter 01](../../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)) can slice by variant.
- Roll traffic between three prompt versions live, and verify the shift in the trace UI within seconds — not deploys.

## Prerequisites

- You have read [chapter 01](../01-feature-flags-as-the-llm-shipping-substrate.md).
- A working LLM feature (from your day job, from the `mod-104/scaffold/`, or the `mod-110/scaffold/` used in earlier exercises). At minimum: a prompt template, one LLM call per turn, a request-scoped user identity.
- An OpenFeature-conformant provider or LaunchDarkly account. Free options: **flagd** (CNCF OpenFeature reference provider, runs locally), **GrowthBook** (self-hostable), **LaunchDarkly free tier**, **ConfigCat free tier**. The exercise assumes flagd + OpenFeature for the code samples; the shape carries over.
- The OTel Gen-AI + OpenInference span wrapper from [mod-110 exercise-01](../../mod-110-observability-feedback-and-incidents/exercises/exercise-01-otel-genai-instrumentation-for-a-feature.md), or an equivalent.

## Problem statement

Your feature currently has one prompt version, checked into source control at `prompts/support_answer.md`. Any change to the prompt requires a PR, a review, a CI build, and a deploy — call it 40 minutes best case, 3 hours worst case. Your PM wants to trial two alternatives (`v15_concise`, `v15_thorough`) against the current (`v14`), starting with internal users on `v15_concise` today and expanding to 5% of a specific customer segment tomorrow. Under the current setup, that trial takes a week of engineering time. Fix it.

Over three hours: introduce a flag layer, model the four-flag hierarchy, wire the prompt-selection call to consult the flag layer at every turn, ensure sticky assignment and fail-closed defaults, emit the variant on the trace span, and demonstrate an in-production version flip that lands in the trace UI in seconds.

## Requirements

### Deliverable 1 — Flag layer setup

Under `flags/`:

- The provider config: for **flagd**, a running instance and a `flags.json` seeded with the four flags in the hierarchy. For **LaunchDarkly** or another hosted provider, screenshots or exported flag definitions from the UI.
- A short `docs/flag-catalog.md` describing each flag: key, type (bool / string / int / JSON), targeting attributes it uses, default value, and *why*. Every flag has a **written rationale for its default**; the rationale explains what happens when the SDK cannot reach the control plane.

Flag hierarchy:

```
support_agent.enabled           bool         default true    (kill switch)
support_agent.cohort            int (0-100)  default 0       (percentage rollout)
support_agent.prompt_id         string       default "support_answer_v14"  (LKG)
support_agent.model_id          string       default "gpt-4o-2024-08-06"   (LKG dated)
```

### Deliverable 2 — Client wiring

Under `app/flags.py` (or `app/flags.ts`):

- An OpenFeature client initialised once at process start, with the in-memory cache enabled and the control-plane update channel wired.
- A `choose_variant(user, tenant, surface, locale) -> Variant` helper that composes the four flag lookups into a single call site. Uses `EvaluationContext` with a stable `targeting_key` (the `user.id` or `session.id` — pick one and justify).
- A short test suite (5–10 tests) that asserts:
  - Assignment is deterministic for a given `targeting_key`.
  - The `default_value` is returned when the SDK is disconnected (simulate a network partition).
  - The kill switch returns `False` → the caller routes to a static fallback.
  - Two different `targeting_key` values land in *different* cohort buckets on a 50/50 rollout (not guaranteed for one pair, but the hash distribution is well-known and testable at ~1000 samples).

### Deliverable 3 — Feature integration

Modify the feature's turn handler so that:

- Every turn calls `choose_variant(...)` *before* selecting the prompt template.
- The variant's `prompt_id` is looked up in a small prompt registry (a `dict[str, str]` mapping IDs to templates in `prompts/registry.py` is fine for this exercise — the full registry is the next chapter's exercise).
- If `support_agent.enabled` is `False`, the handler returns a static "briefly unavailable" response — no LLM call.
- The turn span carries `app.feature`, `app.experiment.arm` (the `prompt_id`), `app.prompt.template_hash` (the SHA of the template text), and `app.flag.evaluated_at` (ms timestamp of the flag lookup).

### Deliverable 4 — Three prompt versions in the registry

Author three prompt versions in `prompts/registry.py`:

- `support_answer_v14` (control — the current prompt, verbatim).
- `support_answer_v15_concise` (a variant that tightens the system prompt to prefer concise answers).
- `support_answer_v15_thorough` (a variant that broadens it to include an evidence-summary paragraph).

Each is a real prompt. Include a `metadata` block with author, date, and a one-line intent statement per version.

### Deliverable 5 — The live rollout demonstration

Drive at least 200 turns through the feature. Record a `docs/rollout-log.md` with the following four transitions, each timestamped and each proven by a trace-UI query:

1. **T=0**: all traffic on `v14`. Confirm 100% of spans in the last minute carry `app.experiment.arm == "support_answer_v14"`.
2. **T+5 min**: flip `support_agent.prompt_id` to `v15_concise` for `internal == true` in the flag UI. Drive traffic from an internal-tagged and a non-internal-tagged identity. Confirm the internal traffic sees `v15_concise` while the non-internal traffic still sees `v14`. Time-to-effect (TTE) — the wall-clock time between the flag flip and the first span carrying the new variant — is recorded.
3. **T+15 min**: flip `support_agent.cohort` to 5% *and* `support_agent.prompt_id` to `v15_thorough` for the cohort. Drive enough traffic that at least 20 turns are in the 5% cohort. Confirm the split ratio is roughly 5% and both variants are represented in the trace.
4. **T+25 min**: flip `support_agent.enabled` to `False`. Drive traffic; confirm no LLM spans are emitted and the caller returned the static fallback.

For each transition, the log includes: the flag change made, the wall-clock timestamp, the observed TTE, and a screenshot / query result from the trace UI showing the shift.

### Deliverable 6 — Fail-closed proof

Under `docs/fail-closed.md`, prove that a control-plane outage does NOT roll a new variant to users. Suggested method: point the OpenFeature client at an unreachable endpoint, restart the process (cold cache), and drive traffic. Confirm every span carries the `default_value` prompt (`v14`) — not the *most recent* value in the (now-unreachable) control plane. Include the wire-capture or log evidence.

## Starter guidance

- **Use OpenFeature + flagd for the free path.** Install `openfeature-sdk` and `openfeature-provider-flagd` (Python) or the Node equivalents. Run flagd as a local process reading `flags.json`. Every reference in this exercise uses the OpenFeature call shape; substituting a hosted provider is one initialiser line.
- **Pick your `targeting_key` deliberately.** For a support surface, `user.id` is usually right — a user's cohort should not flip mid-conversation. For an anonymous surface, `session.id` (with the session cookie as the persistence layer) is the fallback. Document the choice in `docs/flag-catalog.md`.
- **Do not embed prompt text in flag values.** The flag value is a version identifier; the registry is where the text lives. Even for this exercise, treat the two as separate concerns.
- **Test the disconnected-SDK path early.** It is the most common wire-up mistake — the SDK "works" against the control plane and silently fails when disconnected. Stub the network out in a test *before* your first live rollout.
- **The trace attribute name `app.experiment.arm` is not negotiable.** [mod-110](../../mod-110-observability-feedback-and-incidents/README.md) and [chapter 03](../03-ab-experiment-interface-to-the-experimentation-platform.md) both consume it. Name it consistently or the join across systems breaks.
- **Reserve the kill switch for real emergencies.** It is not the rollback lever for a bad prompt — that's the multivariate flag. The kill switch is the "the feature is causing incident-shaped damage, take it out of user surface entirely" lever.

## Acceptance criteria

- [ ] `flags/` contains the four-flag hierarchy definition (JSON, YAML, or provider-exported artefact) and `docs/flag-catalog.md` documents each flag's key, type, default, and rationale.
- [ ] `app/flags.py` (or equivalent) initialises an OpenFeature / LaunchDarkly client with in-memory caching and exposes a `choose_variant` helper.
- [ ] The helper's test suite passes: determinism, disconnected default, kill-switch routing, and bucket distribution.
- [ ] The feature's turn handler consults the flag layer on every turn and skips the LLM call when `support_agent.enabled` is `False`.
- [ ] Every LLM span carries `app.experiment.arm`, `app.prompt.template_hash`, and `app.flag.evaluated_at`.
- [ ] `docs/rollout-log.md` documents all four live transitions with timestamps, TTE observations, and trace-UI evidence.
- [ ] `docs/fail-closed.md` proves the disconnected-SDK path returns the last-known-good default, not the new variant.
- [ ] No prompt text is stored inside a flag value.
- [ ] The `targeting_key` is stable across a user's session (documented and tested).

## Stretch goals

Pick any one:

- **Attribute-based cohort.** Add a `pilot_segment` attribute on the evaluation context and target the prompt flag by `tenant_id IN <pilot-segment-list>`. Demonstrate the flip from percentage-based to segment-based targeting without a redeploy.
- **Feature-scoped kill switches for two independent features.** Add a second feature (or a mock second feature) with its own kill switch, and verify that flipping one does not affect the other. Prove blast-radius containment.
- **Warm-cache SLA.** Measure the time between control-plane update and process-cache reflection under three network conditions (LAN, 100 ms RTT, 500 ms RTT). Chart the distribution. Confirm all three are under 5 seconds.
- **`bucket()` verification.** Instrument the assignment call to log the raw bucket value alongside the assigned variant. Over 10k assignments, confirm the empirical distribution matches the configured split within a chi-squared p > 0.05.

## Deliverable

Submit as `mod-111/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `flags/` with the four-flag definition and provider config.
- `app/flags.py` (or equivalent) plus tests.
- `prompts/registry.py` with the three prompt versions.
- The instrumented feature diff or scaffold.
- `docs/flag-catalog.md`, `docs/rollout-log.md`, `docs/fail-closed.md`.
- A short `README.md` naming the feature, the flag provider, and how to run the traffic driver.
