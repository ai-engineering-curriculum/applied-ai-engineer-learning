# exercise-03: User Feedback Triage with Argilla

**Estimated effort:** 3 hours

## Objective

Build the end-to-end feedback pipeline from [chapter 03](../03-user-feedback-triage-pipeline.md) for the feature you instrumented in [exercise-01](exercise-01-otel-genai-instrumentation-for-a-feature.md): capture thumbs / structured / free-text feedback, join it to the trace, dedup and cluster, route to owners, expose in an Argilla (or Label Studio) workspace, and export the expert-reviewed records into the eval dataset shape from [mod-106 chapter 01](../../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md). The exercise ends when you can point at a fixture that started as a real (or scaffold-synthesised) thumbs-down and made it through the pipeline into the eval set.

## Learning outcomes

By the end you should be able to:

- Design a three-signal feedback capture (thumb, structured category with a versioned taxonomy, optional scrubbed free text) that carries `trace_id` as the join key.
- Join feedback events to trace records via the OTel trace ID.
- Dedup by `(feature, category, prompt_id, retrieval_ids, output_hash)` first, embedding-similarity on comments second, time-window clustering third.
- Route triage entries to owners via a `feature × category` routing table with SLAs.
- Stand up an Argilla (or Label Studio) workspace with the triage entries as annotation records, and export reviewed records to the eval-dataset schema.
- Instrument the trace with the feedback attribute (`app.feedback.signal` / `.category` / `.entry_id`) so tail sampling and dashboards can key on it.

## Prerequisites

- You have read [chapter 03](../03-user-feedback-triage-pipeline.md).
- [exercise-01](exercise-01-otel-genai-instrumentation-for-a-feature.md) is done — you have an instrumented feature emitting spans with a `trace_id` reachable from the client.
- Argilla (self-hosted or SaaS) or Label Studio (self-hosted or SaaS) — either is acceptable; pick the one you can stand up in an hour.
- A small storage layer for feedback events (SQLite, Postgres, DuckDB — the exercise does not care).
- An embedding model for comment dedup (any small model — `sentence-transformers/all-MiniLM-L6-v2`, OpenAI `text-embedding-3-small`, or provider-equivalent).

## Problem statement

Your feature ships a thumbs-up / thumbs-down under each response. The clicks land in a table. Nobody reads the table. When last week's incident produced a spike in thumbs-downs on the same underlying failure ("wrong answer to reset password"), the on-call had no idea it was the *same* failure across 40+ different users — the table showed 40 rows with no clustering. The eval set from [mod-106](../../mod-106-feature-side-evaluation/README.md) has not gained a new fixture in two months because nobody has a repeatable way to turn user complaints into labelled eval items.

In three hours: build the pipeline that turns the raw feedback into a triage queue an owner can work, and out the other end into an eval fixture ready for the CI gate.

## Requirements

### Deliverable 1 — The capture surface

Add or extend the feature's UI (or, if you cannot ship UI, a backend `/feedback` endpoint) to accept the three-signal shape from [chapter 03](../03-user-feedback-triage-pipeline.md):

- **Binary thumb** — `signal ∈ {thumbs_up, thumbs_down, reported}`.
- **Structured category** — on thumbs-down, a required choice from a 6–8-option enum. The enum is defined in `docs/feedback/taxonomy-v1.yaml` with a version field. Choose an enum that fits the feature; document it.
- **Optional free-text comment** — scrubbed on ingress (reuse the scrubber from [exercise-02](exercise-02-sampling-and-pii-scrubbing-policy.md)). Store the scrubbed text AND a hash of the raw text (never the raw text).

Every feedback record carries: `trace_id`, `turn_span_id`, `feature`, `tenant_id`, `user_id_hash`, `client_version`, `taxonomy_version`, plus the fields above.

### Deliverable 2 — The join and dedup pipeline

Under `feedback/`:

- `join.py` — reads feedback events; for each event, fetches the trace context (from the trace backend's API, or from a mirror of the sampled span log) and produces a joined **feedback event with trace summary** in the shape from [chapter 03](../03-user-feedback-triage-pipeline.md).
- `dedup.py` — clusters joined events into triage entries using the three-key strategy: exact match on `(feature, category, prompt_id, top_retrieved_doc_ids, output_hash)` first; embedding cosine similarity ≥ 0.85 on comments second; time-window clustering (3× baseline in 10 min → same incident) third.
- Idempotent — running the pipeline twice on the same input produces the same triage entries.

### Deliverable 3 — The routing table and SLA

Under `docs/feedback/routing.yaml`, the routing rules from [chapter 03](../03-user-feedback-triage-pipeline.md):

- `features:` — the owning team + on-call rotation per feature.
- `category_overrides:` — `hallucinated_source` co-notifies retrieval-team; `unhelpful_refusal` co-notifies safety-team with a priority boost; `jailbreak_success` pages on `incident_suspected`.
- `sla_by_priority:` — triage-within and resolve-within for `incident`, `high`, `normal`.

Under `feedback/route.py`, the code that reads the routing table, applies rules to each new triage entry, records the assigned owner, and starts the SLA clock.

### Deliverable 4 — The Argilla (or Label Studio) workspace

Stand up an Argilla workspace (or Label Studio project) and land the triage entries as annotation records. Each record surfaces:

- The scrubbed input (user question).
- The scrubbed output (assistant response).
- The retrieval hits (IDs, scores, truncated content).
- The user's category and free-text comment.
- The number of users who hit the same cluster (`count`, `distinct_users`).

The annotation UI collects, per record:

- **Confirm / correct the category** (the same 6–8 enum plus `not_a_defect`).
- **Optional gold answer** (what the response *should have been*).
- **Reviewer notes** (any observations for the postmortem).

Document how to run the workspace locally in `docs/feedback/workspace-setup.md`.

### Deliverable 5 — The export to the eval dataset

Under `feedback/export.py`:

- Pull reviewed records from Argilla / Label Studio.
- Transform to the eval-dataset schema from [mod-106 chapter 01](../../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md) (input, expected-output-if-any, tags, provenance-of-origin-trace-id).
- Write to `eval/fixtures/from-feedback/YYYY-MM-DD.jsonl`.
- Idempotent and versioned — a fixture never disappears from an old export.

The exported fixture chain must be traceable back: eval fixture → triage entry → feedback events → trace IDs → traces.

### Deliverable 6 — The trace-side write-back

Add a stream processor that writes `app.feedback.signal`, `app.feedback.category`, `app.feedback.entry_id` back onto the trace record (either as post-hoc span attributes if your backend supports it, or as a join-table linked by `trace_id`). Show that a tail-sampling policy from [exercise-02](exercise-02-sampling-and-pii-scrubbing-policy.md) can now filter on `app.feedback.signal = thumbs_down` directly.

### Deliverable 7 — The end-to-end walkthrough

Under `docs/feedback/walkthrough.md`, a narrative:

1. Show a synthetic (or scaffold-produced) thumbs-down click landing.
2. Show it being joined to its trace.
3. Show it being deduped with two other clicks into a single triage entry with `count = 3`.
4. Show it being routed to the owner's rotation with the correct SLA.
5. Show it appearing in the Argilla workspace as an annotation task.
6. Show a reviewer confirming the category and writing a gold answer.
7. Show the exported fixture appearing in `eval/fixtures/from-feedback/`.
8. Show the trace record now carrying `app.feedback.signal = thumbs_down`.

## Starter guidance

- **Do not build a bespoke annotation UI.** Argilla and Label Studio are both mature; either does the job. The applied-AI engineer's leverage is the pipeline, not the annotation tool.
- **The taxonomy enum is a small file.** Do not put it in a database or a config panel where it can drift ungoverned. `docs/feedback/taxonomy-v1.yaml` gets diffed and reviewed.
- **Dedup runs on a schedule, not in the click handler.** Every minute is fine. Per-click dedup is what most teams try first; it collapses to slow, brittle, and impossible to reason about.
- **The embedding for comment dedup is small.** MiniLM is fast enough to run on CPU. Do not spin a GPU model for the dedup step.
- **`not_a_defect` is a first-class review outcome.** Roughly a third of thumbs-downs should be marked this way. If your workflow does not surface it prominently, the eval dataset becomes contaminated.
- **Every triage entry links back to the traces it deduped from.** The gold answer is much easier to write when the reviewer can open the actual traces and see the failure in context.
- **Scrub free text at ingress.** Do not store raw comments. The scrubbed version is what the pipeline works with; the hash is what dedup uses.

## Acceptance criteria

- [ ] The three-signal capture is shipped (UI or endpoint); every record carries a `trace_id`.
- [ ] `docs/feedback/taxonomy-v1.yaml` names the enum and is versioned.
- [ ] `feedback/join.py` and `feedback/dedup.py` are idempotent and produce the triage-entry shape from [chapter 03](../03-user-feedback-triage-pipeline.md).
- [ ] `docs/feedback/routing.yaml` and `feedback/route.py` route entries to owners with an SLA clock that starts on assignment.
- [ ] Argilla (or Label Studio) workspace is running locally with at least 10 real / synthetic triage entries loaded.
- [ ] The annotation flow lets a reviewer confirm / correct category, mark `not_a_defect`, and optionally write a gold answer.
- [ ] `feedback/export.py` produces `eval/fixtures/from-feedback/YYYY-MM-DD.jsonl` in the mod-106 chapter 01 schema; the file has at least 5 exported items.
- [ ] The trace record carries `app.feedback.signal` / `.category` / `.entry_id` on rated turns (or a documented join-table equivalent).
- [ ] `docs/feedback/walkthrough.md` shows the eight-step chain from click to eval fixture.
- [ ] No raw user comments or unhashed user IDs are stored anywhere.

## Stretch goals

Pick any one:

- **LLM-as-judge suggestion in the annotation UI.** Use Argilla's `SuggestionValue` (or Label Studio's ML backend) to pre-populate a suggested category from an LLM judge; the reviewer confirms or corrects. Report the reviewer-agreement rate on the first 50 suggestions.
- **A feedback dashboard.** Publish a small dashboard (Grafana / Metabase / Superset — pick one you already have) with: thumbs-down rate per feature per day, category mix, triage queue size by state, mean-time-to-triage, mean-time-to-resolve, `not_a_defect` rate. This is the artefact the PM reads weekly.
- **Auto-file GitHub issues.** For each triage entry moving to `assigned`, open a GitHub issue in the feature's repo with the trace links, the category, and the count. Close the issue when the triage entry closes. This is the glue between the feedback pipeline and the engineer's real task queue.
- **A "reported" channel via Slack.** Wire a Slack app command (`/report-response`) that a support agent runs while chatting with a customer; it submits a `signal = reported` feedback event with the trace ID pulled from the conversation. Prove it lands in the pipeline the same way as user-side reports.

## Deliverable

Submit as `mod-110/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- The capture-surface changes (UI diff or endpoint code).
- `feedback/` with `join.py`, `dedup.py`, `route.py`, `export.py`.
- `docs/feedback/taxonomy-v1.yaml`, `docs/feedback/routing.yaml`.
- `docs/feedback/workspace-setup.md`, `docs/feedback/walkthrough.md`.
- Argilla / Label Studio config or fixture files sufficient to reproduce the workspace.
- `eval/fixtures/from-feedback/YYYY-MM-DD.jsonl` (with at least one dated export).
- A short `README.md` naming the feature and how to run the pipeline end-to-end.
