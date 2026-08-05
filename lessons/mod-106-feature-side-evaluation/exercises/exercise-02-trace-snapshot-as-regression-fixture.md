# exercise-02: Trace Snapshot as Regression Fixture

**Estimated effort:** 3 hours

## Objective

Build a **trace-to-fixture pipeline** for a shipped (or shipped-shaped) feature: capture a production trace with full lineage, scrub the PII, freeze it as a regression fixture with a specified grader, and prove it *replays* correctly against a candidate configuration. The output is not a single fixture; it is a repeatable pipeline that could produce hundreds of fixtures over the life of the surface.

## Learning outcomes

By the end you should be able to:

- Capture a trace with the six lineage fields from [chapter 02](../02-trace-snapshots-as-regression-fixtures.md) (model, prompt, retriever, tools, decoding, flags) and know why each field matters.
- Design and run a three-pass PII scrubber (field minimisation → free-text redaction → human review) plus a deletion-request cascade.
- Author a snapshot file with `graders`, `tags`, and `lineage` such that the replay is unambiguous.
- Implement two replay modes — *live retrieval / live tools* vs. *frozen retrieval / captured tool outputs* — and know which to use for gating.
- Version the fixture set with a manifest that CI can compare against a baseline.

## Prerequisites

- You have read [chapter 02](../02-trace-snapshots-as-regression-fixtures.md) and have completed [exercise-01](exercise-01-offline-eval-suite-with-preregistered-thresholds.md) (or have an equivalent suite runner).
- You have Python 3.11+ and access to at least one PII-detection tool (Microsoft Presidio, a cloud DLP API, or a local classifier).
- You have a real trace store or a staging store you can replay from (Langfuse, LangSmith, Braintrust, Phoenix, or a simple JSONL file of captured requests).
- Access to at least one model provider for the replay.

## Problem statement

A user complained last Tuesday. The team fixed the prompt. Two months later a similar complaint arrives; nobody remembers whether the fix would have caught it. This exercise is the pipeline that ensures the second incident never has to guess. You will build the *machinery* — capture, scrub, freeze, replay — and prove it on at least three real traces (or three realistic synthetic traces if no real trace is available).

## Requirements

### Deliverable 1 — Trace capture with lineage

Under `capture/`, a capture routine that turns a trace store record into a `RawTrace` object with **all six lineage fields**:

- `model` — provider + dated model ID (e.g. `anthropic/claude-opus-4-7`).
- `prompt` — `template_id`, `template_hash`, `rendered_hash`, and the rendered prompt itself.
- `retriever` — `id`, `index_snapshot`, `query`, and the returned chunks with doc IDs and scores.
- `tools` — `catalogue_hash`, per-tool versions, and captured tool outputs (with output hashes).
- `decoding` — temperature, top-p, top-k, max_tokens, stop, seed.
- `flags` — feature-flag values and routing arm.

If your trace store is missing any of these, that is the exercise: instrument the capture path to emit them (typically by adding structured attributes to the OpenTelemetry / observability span, mod-110 territory).

Ship *three sample raw traces* under `capture/samples/` — they can be synthetic if no real traces are available, but they must be realistically shaped and complete on lineage.

### Deliverable 2 — The three-pass PII scrubber

Under `scrub/`, a pipeline function `scrub(raw_trace: RawTrace) -> ScrubbedTrace` that runs, in order:

1. **Pass 1 — field-level minimisation.** Drop or hash: user IDs, session IDs, IPs, emails, phone numbers, payment tokens. Publish the field list in `scrub/policy.md`.
2. **Pass 2 — free-text redaction.** Run a PII detector over free-text spans (user messages, model outputs, tool result strings, retrieved chunks). Replace detected PII with **stable typed placeholders** (`<PERSON_1>`, `<EMAIL_1>`, `<ORDER_ID_1>`) — the same real value maps to the same placeholder within one trace.
3. **Pass 3 — human-review checkpoint.** The pipeline emits the scrubbed trace to a review queue (`scrub/review-queue/`) with a diff highlighting redacted spans; only a human-approved trace is written to `fixtures/traces/`.

Include a *deletion-request cascade* — a `scrub/delete_by_user_hash.py` that removes any snapshot whose `source.user_hash` matches a supplied hash, and records the deletion in an audit log.

### Deliverable 3 — Three snapshot files

Under `fixtures/traces/<feature>/`, at least three snapshot YAML files, each in the shape from [chapter 02](../02-trace-snapshots-as-regression-fixtures.md):

- One `complaint`-tagged trace (a user thumbs-down or support ticket).
- One `high_frequency_happy_path` trace (a common request shape).
- One `long_tail` or `edge_case` trace.

Each snapshot has:

- Full lineage (from Deliverable 1) with the values pinned at capture time.
- `graders` block naming at least one programmatic check and one rubric-based judge check (position-swapped, chapter 01).
- `tags` including any relevant `critical`, `safety`, or persona segment tags.
- `source.feedback` (thumbs, reason code) where applicable.
- Placeholder-substituted content; no unscrubbed PII.

### Deliverable 4 — Replay in two modes

Under `replay/`:

- `replay.py` implementing two modes:
  - **`--mode=live`** — the retriever re-issues the query against the current index; tools are re-invoked against the staging backend.
  - **`--mode=frozen`** — the captured retrieved chunks are fed directly to the generation step; tool calls are matched to the captured tool outputs by argument hash.
- Both modes produce a `ReplayResult` with the new outputs plus the grader scores from the snapshot's `graders` block.

Demonstrate on the three snapshots that both modes run without error, and that `frozen` mode is deterministic given a seed.

### Deliverable 5 — Fixture-set manifest

Under `fixtures/traces/<feature>/MANIFEST.yaml`:

- `suite_version`.
- `snapshot_count` and `tag_counts`.
- `rotation_policy` (quarterly rotation ratio; last rotation date).
- `retention.deletion_request_ttl_days` (the SLA under which deletion-cascaded snapshots are purged).

A script `bin/manifest_check` regenerates the manifest from the fixture directory and fails if the committed manifest is out of sync — the manifest is the source of truth CI compares against a baseline, so a stale manifest is a broken suite.

### Deliverable 6 — The write-up

Under `docs/pipeline.md`, one page:

- Which trace store this integrates with and where the lineage fields come from.
- Which PII detector you used and its known blind spots (every detector has some).
- How the deletion-request cascade fits into your surface's overall deletion contract (mod-105 chapter 02).
- Which replay mode you'd default to for CI gating (usually `frozen` for the deep gate; `live` for a nightly integration test) and why.

## Starter guidance

- **Start with a stubbed trace.** Get the pipeline running end-to-end on a JSON blob of your own composition before you point it at real traces.
- **Scrub before you commit.** The one trace you commit unscrubbed is the one that will end up in a screenshot in a public deck.
- **Match tool outputs by argument hash, not by name.** In frozen replay, two calls to `search_orders` with different arguments must not match to the same captured output.
- **Placeholder stability matters.** If two different real names both map to `<PERSON_1>` within one trace, a downstream check that depends on the reply saying the right name back to the right person becomes untestable.
- **Deletion is a real feature.** A user-deletion request from mod-105 chapter 02 must reach the fixture store. Wire the cascade before you have the incident that needs it.
- **Don't over-index on real traces yet.** If you cannot capture real traces today, use realistic synthetic ones — the exercise is the *machinery*, not the volume.

## Acceptance criteria

- [ ] `capture/` produces `RawTrace` objects with all six lineage fields populated.
- [ ] `scrub/` runs all three passes; no snapshot in `fixtures/` bypasses the human-review gate.
- [ ] `scrub/delete_by_user_hash.py` removes matching snapshots and writes an audit-log entry.
- [ ] At least three snapshot YAMLs under `fixtures/traces/<feature>/` covering complaint, happy-path, and long-tail tags.
- [ ] Every snapshot's `graders` block specifies at least one programmatic and one judged check.
- [ ] `replay --mode=live` and `replay --mode=frozen` both run cleanly on all three snapshots.
- [ ] `frozen`-mode replay is deterministic given a pinned seed.
- [ ] `MANIFEST.yaml` is generated by `bin/manifest_check` and matches what's on disk.
- [ ] No snapshot contains raw PII (grep for common patterns: emails, phone numbers, and any test-string you scrubbed).
- [ ] `docs/pipeline.md` names the trace store, PII detector, deletion cascade, and default CI replay mode.

## Stretch goals

Pick any two:

- **A trace-diff reviewer.** A CLI (`bin/trace-diff`) that takes a snapshot and a `ReplayResult` and pretty-prints the diff between the captured behaviour and the replay — token-diff of the answer, tool-call-sequence diff, retrieved-chunk-set diff. This is the tool your future self will use in every regression review.
- **A special-category-data guard.** A pre-commit hook (or CI check) that fails if a fixture under `fixtures/traces/` contains any of a configured list of special-category patterns (medication names from an open list, credit-card patterns, medical-record numbers). The check refuses to let the fixture merge.
- **A trace-cohort curator.** A script that, given a batch of scored online traces (from [exercise-04](exercise-04-sampled-online-eval-loop-with-judge-drift-monitor.md)), suggests which ones to promote to the fixture set, weighted by feedback signal, novelty against the existing set, and coverage-gap of tags.
- **A cross-provider replay.** Replay the same snapshot against two providers (e.g. Anthropic and OpenAI). Report the delta on the graders. This is the routing-decision honesty check for mod-108.
- **A retriever-frozen vs. retriever-live consistency check.** For every snapshot, run both replay modes and report the score delta. Snapshots whose two modes diverge sharply are diagnosing a retriever-side change; snapshots whose modes agree isolate the change to the generation step.

## Deliverable

Submit as `mod-106/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `capture/` with the capture code + three sample raw traces.
- `scrub/` with the three-pass pipeline, the PII policy file, and the deletion-cascade script.
- `fixtures/traces/<feature>/` with ≥ 3 snapshot YAMLs, `MANIFEST.yaml`, and any rendered prompt / transcript sidecar files.
- `replay/` with `replay.py` and both modes.
- `bin/manifest_check` and any other CLIs.
- `docs/pipeline.md`.
- Any stretch-goal artefacts named alongside.
