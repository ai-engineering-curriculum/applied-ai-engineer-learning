# exercise-02: Incident-to-Regression-Fixture Walkthrough

**Estimated effort:** 2 hours

## Objective

Simulate a real incident on an LLM feature and close the incident-to-fixture loop end-to-end: capture the failing traces, redact them, land a regression fixture that fails the pre-incident prompt and passes the post-incident prompt, wire an alert that would have fired the incident earlier, and update the runbook. Three merged artefacts, one loop, one working substrate. The deliverable is not "we fixed the bug"; it is the *three-artefact close-out* that stops the shape from recurring.

## Learning outcomes

By the end you should be able to:

- Capture 5–20 failing traces from an incident window and pin them into an incident-scoped bucket the retention policy will not sweep.
- Apply the production PII scrubber to fixture material and get a human sign-off before the fixture merges.
- Author a regression fixture in YAML that asserts *shape* (tool called, refusal category, presence of a content class) rather than a specific model output string.
- Wire a metric alert with a source query, a threshold rationale, a routing, and a runbook link — all four in the PR.
- Amend the feature runbook with a new or updated playbook entry keyed to the incident.
- Verify the fixture with the pre-incident version fails and with the post-incident version passes, in the eval-gated CI.

## Prerequisites

- You have read [chapter 02](../02-incident-to-fixture-loop.md) and [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md).
- A working LLM feature with an offline eval suite (from your day job, or the scaffold used in the mod-106 / mod-110 exercises, or the flag layer from [mod-111 exercise-01](../../mod-111-shipping-experimenting-and-rollbacks/exercises/exercise-01-feature-flag-driven-prompt-rollout.md)).
- An eval-gated CI job you can add a fixture to (from [mod-106 chapter 03](../../mod-106-feature-side-evaluation/03-eval-gated-ci-end-to-end.md)).
- A tracing UI or log substrate that surfaces `app.prompt.template_hash`, `app.experiment.arm`, and input/output payloads (from [mod-110 chapter 01](../../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)).
- A PII scrubber you can invoke on trace payloads (from [mod-107 chapter 03](../../mod-107-guardrails-and-safety-on-the-surface/03-pii-and-secret-scrubbing-across-the-stack.md)).
- A metrics substrate (Prometheus, Grafana Cloud, Datadog, Honeycomb, or equivalent) where you can add an alert rule.

## Problem statement

You are going to *stage* an incident. Pick one of two shapes — either simulate a **bad prompt push** (change the prompt to one you know will fail on a specific class of input), or simulate a **guardrail false-positive spike** (change a classifier threshold so a benign class of input starts getting blocked). Either shape produces user-visible symptoms without a code bug.

Drive traffic that exposes the failure. Observe the shape in the tracing UI. Then close the loop:

- Capture the failing traces before you fix anything.
- Fix the incident (roll the prompt back or restore the threshold).
- Author the regression fixture from the captured traces.
- Wire the alert that would have caught the shape.
- Amend the runbook.

The exercise is graded on the *three artefacts merged*, not on the mitigation time. A real incident's mitigation is minutes; the loop close-out is days. The exercise runs the whole loop compressed into two hours.

## Requirements

### Deliverable 1 — The incident narrative

Under `docs/incidents/INC-<yyyy-mm-dd>-<slug>/incident.md`:

A short structured narrative — one page or less — covering:

- **Chosen shape.** Bad prompt push or guardrail false-positive spike, and the specific injection.
- **Symptom.** What the failing traces showed. Prefer prose plus a small evidence table (request ID, input digest, output digest, thumbs-down category, template hash).
- **Traffic driver.** How you drove traffic to expose the shape. Include the count of turns driven and the count of failing turns observed.
- **Mitigation.** The action taken to stop the bleeding (rolled prompt back to hash X, restored threshold from Y to Z), with the wall-clock timestamps of first-symptom, mitigation-applied, and mitigation-verified.
- **Root cause.** One paragraph.
- **Close-out coordinator.** The one person who owns "all three PRs land." Even in a solo exercise, name the role.

### Deliverable 2 — The trace pinning

Under `docs/incidents/INC-<yyyy-mm-dd>-<slug>/traces/`:

The 5–20 failing traces you pinned, plus 5–10 known-good traces from the same window for contrast. Each trace is a self-contained file (JSON dump, OpenTelemetry export, or your tracer's native artefact) with:

- The full input and output payloads (pre-redaction — you'll redact in the next step).
- The full span attribute vocabulary (`app.prompt.template_hash`, `gen_ai.system`, `gen_ai.response.model`, `app.experiment.arm`, `app.feature`, plus feature-specific attributes).
- The request ID and timestamp.

A `README.md` in the folder documents the pin: which trace was the first symptom, which cluster share the failure mode, which known-good traces are the contrast set.

### Deliverable 3 — The redaction pass

Under `docs/incidents/INC-<yyyy-mm-dd>-<slug>/redacted/`:

Post-scrub versions of the pinned traces. Includes:

- The output of the automated PII scrubber (same scrubber the surface uses in production — cite the module path).
- A `redaction-review.md` where a human (you, or a colleague if you can arrange it) reviewed each scrubbed trace and noted anything the automatic scrubber missed. If a manual redaction was made, the review file records it.
- A `reverse-map.enc` (or equivalent placeholder) representing the reversible-mapping key. **Do not commit real reversible-mapping data in the exercise;** for the exercise, either use synthetic PII in the pre-redaction traces or drop a placeholder file noting where the real key would live.

### Deliverable 4 — The regression fixture PR

Under `eval/fixtures/regressions/INC-<yyyy-mm-dd>-<slug>-<n>.yaml`:

One or more YAML files, one per fixture derived from the incident cluster, in the shape from [chapter 02](../02-incident-to-fixture-loop.md). At minimum:

- Stable `id` linked to the incident.
- `input` block with the redacted request payload.
- `expected` block asserting *shape* (tool called, refusal category present/absent, content-class present/absent). Uses the judge substrate from [mod-106 chapter 02](../../mod-106-feature-side-evaluation/02-trace-snapshot-as-regression-fixture.md) for judge-based assertions.
- `failure_mode_at_incident` prose paragraph.
- `severity` (P0-P3) and `owasp_mapping` where applicable.

Prove the fixture is doing its job:

- **CI run against the pre-incident version** — the fixture *fails*. Include the CI log artefact.
- **CI run against the post-incident version** — the fixture *passes*. Include the CI log artefact.
- **A single PR** on the branch that (a) adds the fixture and (b) contains the changes needed to make the fixture pass (or references the mitigation PR).

### Deliverable 5 — The alert PR

Under `monitoring/alerts/<feature>-<shape>.yaml` (or your monitoring stack's format):

- The alert definition — source query, threshold, evaluation window, severity.
- A short `docs/alerts/<feature>-<shape>.md` companion that documents:
  - The source query in the metrics substrate (PromQL, Grafana query, Datadog query — whichever).
  - The threshold rationale ("2σ above the 30-day baseline of X" or "absolute threshold from charter kill criterion Y").
  - The routing (which on-call rotation, which channel, at what severity).
  - The runbook link (which resolves to Deliverable 6's runbook entry).

The alert must, when replayed against the incident's time window, have fired *earlier* than the human noticed the incident. Include a short "replay result" note in the companion doc showing the alert would have fired at wall-clock time T, N minutes before the human first noticed.

### Deliverable 6 — The runbook amendment PR

Under `docs/features/<feature>/runbook.md`:

Either (a) a new playbook entry for a novel failure mode, or (b) an amendment to an existing playbook entry for a variant, or (c) a known-issue callout. The entry follows the shape from [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md): detection, triage, mitigation, root-cause, prevention.

The runbook link from Deliverable 5's alert must resolve to this entry.

### Deliverable 7 — The three-artefact close-out ticket

Under `docs/incidents/INC-<yyyy-mm-dd>-<slug>/close-out.md`:

A short close-out document with a checklist:

```markdown
## Close-out

- [x] Fixture PR merged — link to PR, link to CI logs (pre-incident fail, post-incident pass).
- [x] Alert PR merged — link to PR, link to replay evidence.
- [x] Runbook PR merged — link to PR, link to the runbook entry.

Coordinator: <name / role>. Signed off: <date>.
```

Only when all three checkboxes are ticked does the incident ticket close.

## Starter guidance

- **Do the staging carefully.** For the bad prompt push, edit one line of the prompt in a way that breaks a *specific* class of input — e.g., remove the "you may issue refunds for medical-reason returns" instruction so refund requests mentioning a health issue get incorrectly refused. For the guardrail FP spike, tighten a classifier threshold enough that a benign class starts blocking. Both should be one-line changes so you can roll back cleanly.
- **Drive traffic with a small synthetic driver.** 40–100 requests is enough — a mix of the failure-triggering class and normal traffic. Use synthetic personas / test tenants so you never touch real user data.
- **Pin the traces before you mitigate.** This is the discipline the chapter names. If you mitigate first, the failing traces may age out of your default retention window and the fixture cannot be built.
- **Use the same PII scrubber the feature uses in production.** Reusing the exact same regexes and NER model is what makes the fixture reflect what the pipeline actually processes. Mismatched scrubbers produce fixtures that pass locally but fail in production replay.
- **Assert shape, not strings.** If you assert `output == "I've processed your refund"`, the fixture flakes on temperature variance. Assert `refund_processed == True` via the judge substrate, or `refusal_category is None`, or `tool_calls contains 'process_refund'`. Judge-based assertions are the [mod-106 chapter 02](../../mod-106-feature-side-evaluation/02-trace-snapshot-as-regression-fixture.md) substrate.
- **Pick the alert's source query carefully.** For a bad prompt push, `thumbs_down_rate{template_hash=X} > baseline + 2σ` is the shape. For a guardrail FP spike, `guardrail_block_rate{category=Y} > baseline + 3σ`. Reference [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md) for the recurring shapes.
- **Alerts without a runbook link fail review.** Wire the runbook link before the alert PR merges; the entry in the runbook can be a stub that says "see incident INC-… for context, remediation not yet documented" if the runbook edit is still in flight — but the link resolves.
- **The one coordinator matters.** In a real org, three people can each own one PR and none of them own "all three land." Naming a coordinator — even for the solo exercise — is the discipline that prevents the incident closing on one artefact merged.

## Acceptance criteria

- [ ] `docs/incidents/INC-<...>/incident.md` narrates the staged incident with symptom, driver, mitigation timeline, root cause, and coordinator.
- [ ] `docs/incidents/INC-<...>/traces/` contains 5–20 failing traces and 5–10 contrast traces, with a README that names the shared failure mode.
- [ ] `docs/incidents/INC-<...>/redacted/` contains post-scrub versions with a `redaction-review.md` naming the reviewer and any manual redactions.
- [ ] `eval/fixtures/regressions/INC-<...>-*.yaml` contains one or more regression fixtures asserting *shape*, not strings; each has `id`, `input`, `expected`, `failure_mode_at_incident`, `severity`, `owasp_mapping` (where applicable).
- [ ] Pre-incident CI run **fails** the new fixture; post-incident CI run **passes** it. Both CI logs are linked from the close-out doc.
- [ ] Alert PR (`monitoring/alerts/<...>`) has a source query, a threshold rationale, a routing, and a resolving runbook link. Replay evidence shows it would have fired earlier than the human noticed.
- [ ] Runbook PR (`docs/features/<feature>/runbook.md`) adds a new entry or amends an existing one in the [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md) shape.
- [ ] Close-out ticket lists all three PRs merged with a coordinator name and sign-off date.
- [ ] Zero raw (unredacted) PII in any merged file, including the fixture. Where synthetic PII was used, the incident narrative names it.

## Stretch goals

Pick any one:

- **Two shapes, one incident.** Stage an incident where the root cause is one thing but *two* failure modes surface (a bad prompt push that also causes an over-refusal spike). Close the loop with two fixtures and one alert, and show that the runbook amendment covers both.
- **The pre-mortem pass.** Before staging the incident, write down which of the six recurring modes from [mod-110 chapter 04](../../mod-110-observability-feedback-and-incidents/04-incident-playbooks-for-six-llm-failure-modes.md) your feature is exposed to and which of them already have a regression fixture. Then use the staged incident to add a fixture for one mode that was missing. The stretch is: at the end, ship 5 additional pre-mortem fixtures for shapes you have not yet had incidents in.
- **The redaction round-trip.** Prove that the reverse-mapping key is reversible: pick one redacted fixture, use the mapping to reconstruct the original (in a private local environment, not committed), and verify that the reconstructed input reproduces the failing behaviour on the pre-incident version. Document the round-trip discipline (who is authorised to reverse, in what workflow, with what audit trail).
- **The judge-drift complication.** Stage the incident with a *judge* on the assertion side that is subject to drift. Prove the fixture is robust: run it 20 times with the judge; measure the pass/fail flake rate; if it flakes, tighten the assertion shape until it does not.
- **The game-day version.** Coordinate with a colleague to stage the incident without them knowing which shape you chose. They play the on-call role and close the loop; you play the incident commander. Debrief on how long each stage took and where the substrate creaked.

## Deliverable

Submit as `mod-112/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/incidents/INC-<...>/incident.md`, `traces/`, `redacted/`, `close-out.md`.
- `eval/fixtures/regressions/INC-<...>-*.yaml` plus CI log artefacts.
- `monitoring/alerts/<...>` plus `docs/alerts/<...>.md` companion.
- `docs/features/<feature>/runbook.md` diff.
- A short `README.md` naming the feature you staged the incident on, the shape you chose, and how you drove the traffic.
