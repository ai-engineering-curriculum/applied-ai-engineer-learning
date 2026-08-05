# exercise-05: Cross-Provider Contract Tests

**Estimated effort:** 2 hours

## Objective

Build the **cross-provider contract test** discipline from chapter 05 for the router you shipped in exercises 02 and 04: a fixture set that pins what is under contract, a runner that exercises every provider on every route's fallback list, per-provider **golden traces** pinned to dated model IDs, and CI wiring that blocks a merge on a hard-fail diff. Prove that "we support provider X as a fallback" is a fact your CI checks, not a promise in a runbook.

## Learning outcomes

By the end you should be able to:

- Name what is under contract for a route (envelope, message shape, tool-call semantics, usage, finish reason, refusal shape, streaming, latency / cost envelope) and what deliberately is not.
- Write **shape + must-contain + bounded-numeric** fixtures instead of equal-string assertions.
- Capture **per-provider golden traces** pinned to dated model IDs and version them explicitly.
- Diff a new run against goldens with **hard-fail / soft-warn bands** that survive legitimate non-determinism.
- Wire the contract-test job into CI so a route or provider change that breaks the contract blocks merge.

## Prerequisites

- You have read [chapter 05](../05-cross-provider-contract-testing.md).
- Exercises 02 and 04 shipped — this exercise contract-tests *their* routes.
- Access to every provider on every route's fallback list (mock providers are acceptable for CI, but the initial golden capture must run against real providers).
- CI substrate that can run a fixture-based test and post a report to a PR (GitHub Actions, GitLab CI, Buildkite — pick whatever your team uses; the scaffold ships a GitHub Actions workflow you can adapt).

## Problem statement

Your router (from exercise-02) has a fallback list. Your multi-modal router (from exercise-04) has fallbacks per modality. But nothing has *ever run* against those fallbacks except during the manual proof in each exercise. If Anthropic's tool-call shape changes, if a fallback STT provider silently stops returning speaker labels, if a swap from OpenAI to Google returns a different `finish_reason` enum — your feature will break in production and the first symptom will be a downstream crash. In two hours, close the loop.

## Requirements

### Deliverable 1 — What is under contract

Under `contract/scope.md`, one page:

- For each route you contract-test, name **what is under contract** (envelope fields present, message content shape, tool-call name and argument-key equality, usage fields non-null, finish reason enum, refusal shape, streaming shape if applicable, latency / cost envelope).
- Name **what is deliberately not under contract** (word-for-word content equality, deep semantic quality, exact token counts).
- One paragraph on the hard-fail vs soft-warn policy — which categories block merge, which surface a warning for reviewer sign-off.

### Deliverable 2 — Contract fixture set

Under `contract/fixtures/<route-name>/`, at least **6 fixtures per contract-tested route** in the yaml shape from [chapter 05](../05-cross-provider-contract-testing.md). Composition suggestions:

- A **factual-qa** fixture — text out, one must-contain check, bounded length.
- A **structured-output** fixture — JSON schema, hard-fail on schema-invalid.
- A **tool-call** fixture — tool definition, expected tool name, argument-key equality.
- A **refusal** fixture — a prompt the surface's policy expects the model to refuse; the fixture asserts the refusal is legible (not "assert refuses" — assert the *shape* of a refusal — e.g. a `content_filter` finish reason, or a policy-language message with a specific keyword).
- A **streaming** fixture (if the route streams) — assert chunk shape and delta semantics.
- One **modality-specific** fixture per non-text route (e.g. STT: expected transcript, timestamps within band; image gen: expected size, non-empty bytes, safety-filter behaviour).

### Deliverable 3 — Contract-test runner

Under `contract/run.py`:

- Loads the route table (exercise-02 / -04 output), enumerates providers on each route's fallback list.
- For each (route, provider, fixture) combination: dispatches the fixture, records the response, evaluates the fixture's `expect` block (shape assertions, semantic checks, cost/latency bands).
- Captures a **trace** — the full response, normalised, sensitive-data-scrubbed — for every combination.
- Emits a **contract report** `reports/YYYY-MM-DD.yaml` and `.md` with per-provider per-fixture pass / soft-warn / hard-fail.

### Deliverable 4 — Per-provider golden traces

Under `contract/goldens/<route-name>/<fixture-id>/<provider>.golden.yaml`:

- Capture goldens for every (route, provider, fixture) combination.
- Every golden pins the **dated model ID** (not `latest`), the runner version, and the capture date.
- Provide a `bin/capture-goldens` entry point to refresh goldens; refreshing is a **reviewed PR**, not a silent action.

### Deliverable 5 — Diff bands

The runner diffs new traces against goldens per the chapter-05 policy:

- **Hard-fail**: missing envelope fields, missing content when golden had content, tool-call name mismatch, semantic must-contain miss, cost outside ±30% of golden.
- **Soft-warn**: content length outside ±30%, token counts outside ±20%, latency p95 outside ±50%.
- The bands are named in a config file (`contract/diff-policy.yaml`) so the reviewer argues about them in code.

### Deliverable 6 — CI wiring

Under `.github/workflows/contract-tests.yaml` (or the equivalent for your CI):

- Runs the contract-test runner on every PR that touches `routing/**`, `contract/**`, or `src/features/**`.
- Runs on a **nightly / weekly** schedule against the current pinned provider models (this catches provider-side drift under the same dated ID).
- Posts the contract report as a PR comment with pass / soft-warn / hard-fail counts and a link to the full report.
- **Blocks merge on any hard-fail.**

### Deliverable 7 — Two proofs

Under `proofs/`:

- `contract-caught-it.md` — deliberately introduce a shape divergence (e.g. remove a normaliser in the router that transforms Anthropic's tool-use into the OpenAI shape). Run the contract-test runner. Show the hard-fail diff. Restore the normaliser. Show the runner passing.
- `golden-refresh.md` — refresh one golden trace and show the PR shape a reviewer would see: `+ new golden` next to the diff against the previous. Prove the refresh is reviewed, not silent.

## Starter guidance

- **Shape > equality.** Every time you are tempted to write `assert response == expected`, stop. Write `assert type(response.content) == str and 10 < len(response.content) < 500` and one `must_contain`. Equal-string assertions are the anti-pattern from chapter 05.
- **Pin dated model IDs everywhere.** `openai/gpt-4o` is not a model — it is an alias. `openai/gpt-4o-2024-08-06` is a model. Fixture, golden, config: all pin dated IDs. This is the specific failure mode the chapter is built around.
- **Refusal fixtures are the most fragile.** Refusal shape drifts sharply per provider and per model version. Assert the *shape* of a refusal, not the exact phrasing. If your surface has a canonical refusal-detection rule, use that.
- **Include the streaming fixture if you have a streaming surface.** Streaming shape divergence is the biggest silent bug in cross-provider fallbacks.
- **Do not use real user data in fixtures.** Fixtures are synthetic or explicitly consented. This is not paranoia — a fixture is checked in and lives forever.
- **The nightly job is the point.** A contract test that only runs on PR misses provider-side drift under the same dated ID. Wire the schedule at the same time as the PR trigger.

## Acceptance criteria

- [ ] `contract/scope.md` names what is under contract, what deliberately is not, and the hard-fail / soft-warn policy.
- [ ] `contract/fixtures/<route>/` has ≥ 6 fixtures per contract-tested route including at least: factual, structured-output, tool-call, refusal, streaming (if applicable), and modality-specific (if applicable).
- [ ] `contract/run.py` runs every fixture against every provider on the route's fallback list — the primary is not the only path exercised.
- [ ] `contract/goldens/` has per-provider goldens pinned to dated model IDs; `bin/capture-goldens` refreshes them via reviewed PR.
- [ ] `contract/diff-policy.yaml` names the hard-fail vs soft-warn bands.
- [ ] `contract/reports/` shows at least one baseline report.
- [ ] CI job runs on PR + on a schedule; posts the report as a PR comment; blocks merge on hard-fail.
- [ ] `proofs/contract-caught-it.md` demonstrates a real shape divergence caught by the runner.
- [ ] `proofs/golden-refresh.md` demonstrates a reviewed golden refresh.
- [ ] No fixture contains real user data; every fixture is synthetic or explicitly labelled as consented.

## Stretch goals

Pick any one:

- **Contract-test the composed realtime fallback.** From exercise-04's composed STT → text-LLM → TTS path — write a fixture that plays a fixed audio blob through the composed fallback and asserts (a) the STT transcript's shape, (b) the text-LLM response's shape, (c) the TTS audio's shape (duration band, format). This is a multi-hop contract test and is the harder variant.
- **Streaming-chunk contract.** Extend the streaming fixture to assert the *sequence* of chunk shapes — e.g. the first chunk has role metadata, subsequent chunks have delta.content, the final chunk has a finish_reason. Different providers stream slightly different first-chunk / last-chunk semantics.
- **Cost-envelope regression.** Add a fixture whose sole purpose is to assert cost stays within band. When a provider silently doubles its pricing (or your prompt shape shifts to a more expensive tier), the regression is caught at CI time, not at month-end.
- **Cross-region golden.** Capture goldens for the same route × provider in two regions (e.g. us-east-1 and eu-west-1). The diff between them is often more interesting than the diff between providers.
- **Trace-snapshot integration.** Feed the contract-test golden traces into the mod-106 chapter-02 trace-snapshot substrate so the two disciplines share machinery. This is a real duplicate-substrate risk worth resolving.

## Deliverable

Submit as `mod-108/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `contract/` with `scope.md`, `fixtures/`, `run.py`, `goldens/`, `diff-policy.yaml`, `reports/`.
- `bin/capture-goldens`.
- CI workflow file (`.github/workflows/contract-tests.yaml` or equivalent).
- `proofs/contract-caught-it.md` and `proofs/golden-refresh.md`.
- Any stretch-goal artefacts named alongside.
- A short `README.md` naming the routes under contract and how to run the runner locally and interpret the report.
