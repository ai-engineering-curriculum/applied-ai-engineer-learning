# exercise-04: Hot Rollback of a Bad Prompt

**Estimated effort:** 2 hours

## Objective

Build the substrate that makes a **hot rollback of a bad prompt** a 30-second pointer flip, not a 20-minute deploy. That means: an append-only prompt registry, a rollback runbook the on-call can read at 3 AM, a published rollback SLA with measured TTD / TTDe / TTE, and a **rehearsed rollback drill** with the numbers written down. The deliverable is the machinery and the drill, not a live incident to recover from — the whole point is to have the drill land in the repo *before* you need it.

## Learning outcomes

By the end you should be able to:

- Design a prompt registry with **append-only** semantics, content-addressed versioning, and process-cached reads.
- Emit `app.prompt.id` and `app.prompt.template_hash` on every LLM span so that a rollback is provable in the trace.
- Decompose the **rollback SLA** into TTD / TTDe / TTE and measure each against a target.
- Compose a flag hierarchy that lets the smallest containment lever fire first (prompt → cohort → kill-switch), avoiding "kill switch as only lever."
- Write a **one-page runbook** with a copy-pastable rollback command that lives next to the code and is linked from every alert.
- Run and document a **rollback drill** that exercises the whole path — pager, credentials, dashboard, flag flip, verification — not just the flag flip.

## Prerequisites

- You have read [chapter 04](../04-hot-rollback-of-a-bad-prompt.md).
- The flag layer from [exercise-01](exercise-01-feature-flag-driven-prompt-rollout.md) exists.
- The canary and burn-rate alerting substrate from [exercise-02](exercise-02-canary-and-shadow-launch-with-gates.md) exists.
- The mod-110 span wrapper exists so LLM spans can carry `app.prompt.id` and `app.prompt.template_hash`.
- The paging system routing (Pagerduty, Opsgenie, or a stubbed webhook) is reachable from your alerting stack.

## Problem statement

Your team's `support_agent_v15_concise` variant merged and rolled to 5% at 14:00 UTC. At 14:07 a support engineer notices a spike in refusal responses. Under the current setup, the rollback would take: 20 minutes for a hot-fix PR, 5 minutes for CI, 10 minutes for the deploy pipeline, and 3 minutes for the fleet to pick up the new artefact. That's 38 minutes on the clock — during which real users are hitting the bad prompt.

Over two hours: build the append-only registry, wire it, publish the rollback SLA, write the runbook, and run one full rollback drill end-to-end. The measurable target: **TTE (time-to-effect) ≤ 60 seconds** and **TTDe (time-to-decision) ≤ 3 minutes with a fresh on-call reading the runbook cold**.

## Requirements

### Deliverable 1 — Prompt registry with append-only semantics

Under `prompts/registry.py` (or your language's equivalent) or as a small service:

- A key–value store of `(prompt_id, template_text, metadata)` entries, with `prompt_id` derived from `template_text` via SHA-256 truncation (`sha256(template).hex[:12]` is a working choice) OR from a monotonic integer suffix (`support_answer_v14`, `v15`, …) — pick one and justify.
- Entries are **immutable**: attempting to overwrite an existing `prompt_id` raises. Adding a *new* entry with a new `prompt_id` is fine.
- Every entry carries metadata: `author`, `created_at_utc`, `intent` (one-line human summary), and — for deprecated entries — `deprecated_reason` (a human note; the entry is never removed from the store).
- The registry is loaded into process memory at process start and refreshed on flag-layer signal. A registry read on the hot path is a single dict lookup, no I/O.
- Tests assert: overwrite raises, cold-cache load returns known entries, deprecated entries are still readable, hash-collision detection is present (two templates that hash to the same short ID must fail loudly).

Under `prompts/versions/` (or the registry's equivalent), commit three concrete versions: `v14` (the current stable), `v15_concise` (the candidate from prior exercises), and `v15_bad` (a deliberately-broken version — e.g., one that starts with "I cannot help with any request" — you will roll to this in the drill).

### Deliverable 2 — Span attribution

Modify the LLM-call wrapper so that every LLM span carries:

- `app.prompt.id` — the value returned by the flag layer (matches the registry key).
- `app.prompt.template_hash` — `sha256(template_text)` full hex, for post-hoc diffing across two versions.
- `app.prompt.registry_source` — either `"live"` (loaded from the registry) or `"default"` (fell back to the SDK default because the registry couldn't resolve).

Add a test that asserts these attributes are always present on the span and never `None` — even in the fall-back-to-default path.

### Deliverable 3 — Rollback SLA doc

Under `docs/rollback-sla.md`:

- **TTD target** (time-to-detection): the minutes between a bad prompt reaching a user and the on-call being paged. Depends on your burn-rate alerts from exercise-02.
- **TTDe target** (time-to-decision): the minutes between the page and the on-call deciding to roll back.
- **TTE target** (time-to-effect): the seconds between the flag flip and every worker's process reflecting the change.
- The **fail-closed** default for each flag in the hierarchy, restated. The `default_value` for `support_agent.prompt_id` is the current LKG.
- The **flag hierarchy** and which layer's flip corresponds to which rollback shape: bad prompt → `prompt_id` flip, bad model → `model_id` flip, bad rollout state → `cohort` drop to 0, incident-shaped damage → `enabled = false`.
- The **owner** of the SLA (a specific person / team) and the **review cadence** (quarterly recommended).

Publish specific numbers (e.g., TTD ≤ 10 min, TTDe ≤ 3 min, TTE ≤ 60 sec). If your feature can't hit them today, name what's blocking and file it as an action item.

### Deliverable 4 — The runbook

Under `docs/runbook-rollback.md`, a **one-page** runbook. Sections:

- **Header**: feature name, SLA numbers, last-drilled date, on-call rotation.
- **Step 1 — Identify the bad variant** with a specific trace-UI query URL (or the query itself for a Prometheus/Grafana stack).
- **Step 2 — Roll back**: the exact flag key, the exact target value, the exact CLI command as an alternative to the UI. Copy-pastable.
- **Step 3 — Verify**: which metric to watch, what threshold means the rollback landed, what to do if it didn't (escalate to `enabled = false`).
- **Step 4 — Notify**: the specific channel(s) and the template message.
- **Where the LKG version is documented**: `docs/lkg.md` (deliverable 5) or an equivalent.

Keep the runbook to one printed page. If it grows to two, cut. The runbook is for humans at 3 AM.

### Deliverable 5 — LKG version doc

Under `docs/lkg.md`, a machine-readable and human-readable list of the current last-known-good version per feature:

```yaml
support_agent:
  prompt_id: support_answer_v14
  model_id: gpt-4o-2024-08-06
  last_updated: 2026-08-05
  last_updated_by: <name>
  notes: v14 has been stable since 2026-06-12; passes all four gates.
```

The doc is updated as part of the launch process — every time a new version successfully promotes to 100%, the LKG entry gets updated (and the previous LKG stays in the git history).

### Deliverable 6 — Rehearse the rollback drill

Run the drill and record it in `docs/rollback-drill.md`:

- **Setup**: temporarily set `support_agent.prompt_id` to `v15_bad` at a small cohort (5% or lower — this is a drill on a *test* feature or with synthetic traffic, not on real users if you can avoid it).
- **Alert**: your burn-rate alert from exercise-02 should fire. Note the wall-clock time from the flag flip to the alert firing. This is the observed TTD.
- **Page**: the alert routes to a pager. Note the wall-clock time from alert firing to on-call ack. This is part of TTDe.
- **Runbook**: the on-call reads the runbook cold. Note the time from ack to the flag-flip command being executed. This is the rest of TTDe.
- **Flip**: the flag flips. Note the wall-clock time from the flip to the first span carrying `v14` in the trace. This is TTE.
- **Verify**: the refusal metric returns to baseline. Note the observation.
- **Notify**: the notification message is posted.

For each measured number, compare to the SLA target. Any component that missed target gets a follow-up action item. Every drill produces at least one improvement (a broken link, a stale flag name, a slow SDK cache-refresh setting, a runbook step that took longer than expected).

### Deliverable 7 — Retro

At the bottom of `docs/rollback-drill.md`, a 5-sentence retro: what worked, what didn't, one thing to fix by next month's drill, and the date of the next scheduled drill.

## Starter guidance

- **Content-addressed prompt IDs are the easy-mode append-only.** If your `prompt_id = sha256(template)[:12]`, an "overwrite" is mechanically impossible — the ID *is* the content. If you prefer monotonic integer suffixes for human ergonomics (`v14`, `v15`), enforce the immutability at the registry layer with a "raise on overwrite" check.
- **Bake the drill into a script**, not an "ad hoc test with the on-call." A script that flips the flag, waits for the alert, waits for the ack, and prints the timing table is repeatable — you can run it every month for the cost of an hour of on-call attention.
- **The runbook is the artefact that pays for itself.** If the on-call at 3 AM has to read a wiki page, click three links, and cross-reference two Slack messages to find the rollback command, they will get it wrong. Copy-pastable command, in one place, linked from the pager alert.
- **TTE is often the slowest to move.** Flag SDKs have a cache-refresh cadence (LaunchDarkly's SSE stream is near-instant; some providers poll on a 30-second interval). If your TTE is above target, the fix is usually in the SDK config, not in the feature code.
- **Do not delete `v15_bad` from the registry after the drill.** Its presence in the registry, marked `deprecated_reason: "rollback drill 2026-08-05"`, is the whole point of append-only. A retro on a real incident will need the exact template that was live.

## Acceptance criteria

- [ ] The prompt registry supports append-only writes, enforces immutability, and reads from process memory on the hot path. Tests pass.
- [ ] Every LLM span carries `app.prompt.id`, `app.prompt.template_hash`, and `app.prompt.registry_source`.
- [ ] `docs/rollback-sla.md` publishes concrete TTD / TTDe / TTE numbers, the fail-closed defaults, the flag hierarchy, and the owner / review cadence.
- [ ] `docs/runbook-rollback.md` is one page, has a copy-pastable rollback command, and is linked from every burn-rate alert.
- [ ] `docs/lkg.md` names the current LKG prompt and model per feature.
- [ ] `docs/rollback-drill.md` documents an executed drill with measured TTD, TTDe, TTE compared to SLA targets, plus the retro.
- [ ] `v15_bad` is present in the registry, marked `deprecated_reason`, and the drill's trace exports show the successful rollback back to `v14`.
- [ ] Any SLA component that missed target has a filed follow-up action item.

## Stretch goals

Pick any one:

- **Auto-rollback bot.** Wire the fast-burn alert to an auto-rollback action (via the flag provider's API) that flips `prompt_id` to the LKG value from `docs/lkg.md` without a human. Rehearse; measure the fully-automated TTD → TTE. Note any guardrails that must be in place before you'd enable auto-rollback in production (e.g., "only if the burn is on a *known-safe* rollback target").
- **Cross-feature blast-radius test.** Flip `support_agent.enabled = false` and verify that a second unrelated feature is not affected. Prove the flag hierarchy contains blast radius as designed.
- **Registry as a service.** Move the prompt registry from an in-process dict to a small service (Redis-backed, or a tiny FastAPI over a SQLite db). Preserve the append-only semantics. Measure the effect on TTE — a network hop on the flag-refresh path is not free.
- **Provider-caching alignment.** For features using OpenAI or Anthropic prompt caching (mod-109 chapter 02), verify that a prompt rollback also correctly re-uses the cached prefix of `v14`. If not, the cost cliff on rollback is real; note it.

## Deliverable

Submit as `mod-111/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `prompts/registry.py` and `prompts/versions/` (or the equivalent).
- The updated LLM-call wrapper emitting the new span attributes.
- `docs/rollback-sla.md`, `docs/runbook-rollback.md`, `docs/lkg.md`, `docs/rollback-drill.md`.
- Trace exports from the drill: one before the flip (carrying `v15_bad`), one after (carrying `v14`).
- A short `README.md` naming the feature, the flag provider, the paging system used for the drill, and how to re-run the drill.
