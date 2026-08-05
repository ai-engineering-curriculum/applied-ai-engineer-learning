# exercise-03: Batch Eligibility Rubric and Migration

**Estimated effort:** 2 hours

## Objective

Pick one candidate workload from your surface (or the scaffold), apply the three-question **batch eligibility rubric** from [chapter 03](../03-batch-apis-and-the-offline-eligibility-rubric.md), stage a dual-run migration to a provider batch API, and produce the artefacts that make the migration reviewable: the rubric doc, the dual-run diff report, and the fall-through plan for late batches. You are not shipping a full production cutover in two hours — you are producing a decision that a peer can approve or reject.

## Learning outcomes

By the end you should be able to:

- Apply the three-question eligibility rubric (freshness SLA, structural fit, economics) to a real workload and write a defensible decision.
- Author a batch-input JSONL and submit / poll / reconcile a batch job against a real provider batch API.
- Design a dual-run shadow-diff between the sync path and the batch path so you can measure drift before cutting over.
- Name the fall-through path for late or failed batches — and prove it in a test.

## Prerequisites

- You have read [chapter 03](../03-batch-apis-and-the-offline-eligibility-rubric.md) and skimmed the current OpenAI Batch (or equivalent) API docs.
- A workload with either meaningful volume (hundreds of requests / day at least) or an obvious offline shape (background enrichment, nightly summarisation, embedding a corpus), OR the scaffold.
- A provider account with batch access. The exercise deliberately uses a small model on a small sample to keep cost trivial.

## Problem statement

Your team runs a "background" enrichment: every hour, a scheduled job takes the last hour of user-generated content, calls an LLM to classify each item into a taxonomy, and writes the classification back to the database. Two things are quietly wrong:

1. Nobody knows whether the *product* SLA on the classification is really "within an hour," within four hours, or within a day. The cron just runs hourly because someone set it that way.
2. The workload has grown to ~50k requests / day at ~$0.001 each. If it is truly batch-eligible, migrating it saves ~$25 / day (~$9k / year) at 50% off — enough to be worth doing but not enough to justify a botched migration.

Your job over two hours: apply the rubric, stage the dual-run, and produce the reviewable artefacts.

## Requirements

### Deliverable 1 — Rubric doc

Under `docs/batch-eligibility.md`, the artefact from [chapter 03](../03-batch-apis-and-the-offline-eligibility-rubric.md):

```markdown
# Batch eligibility — <workload> — <date>

## Workload
<one paragraph: input shape, output shape, consumer of the output>

## Q1 — Freshness SLA
Maximum staleness: <N minutes / hours / days>
Source of the SLA (ticket, contract, product decision): <cite>
Batch-eligible on freshness? <yes | conditional | no — reason>

## Q2 — Structural fit
Independent at the line level? <yes | no — inner-loop only | no>
Any cross-request state? <none | describe>

## Q3 — Economics
Current monthly sync cost: $<X>
Projected monthly batch cost: $<X × 0.5, verify>
Projected monthly savings: $<S>
Migration + operational cost estimate: <engineer-weeks>
Payback horizon: <months>

## Decision
<Ship batch | Keep sync | Sync w/ batch-eligible inner loop>

## Fall-through
Late-batch handling: <retry as sync, alert, drop, degrade>

## Peer sign-off
- Product: <name>
- Data owner: <name>
- Finance / cost owner: <name>
```

Do not hand-wave Q1. If the current freshness SLA is "we don't know," name that as the answer and identify who owns finding out. A rubric with a hand-waved Q1 is not defensible.

### Deliverable 2 — Batch JSONL generator

Under `code/build_batch.py`, a script that reads a sample of records from your workload's queue (or a fixture) and produces a valid batch input file for your chosen provider. Requirements:

- `custom_id` is namespaced per submission (`<batch_seq>-<row_id>`) so collisions across days cannot happen.
- The request body is byte-identical to what the sync path sends for the same record (so downstream diff is meaningful).
- The script is idempotent on re-run for the same input.

### Deliverable 3 — Submit / poll / reconcile

Under `code/run_batch.py`, a runner that submits the JSONL, polls until completion (with a *deadline* shorter than the workload's freshness SLA so a fall-through can fire), and writes results back keyed by `custom_id`. Requirements:

- Handles the `completed` case (extract results per line).
- Handles the line-level partial-failure case (some lines have an `error` field; do not silently drop).
- Handles the `expired` / `failed` batch case (invoke the fall-through).
- Emits observability spans with `app.batch.batch_id`, `app.batch.status`, `app.batch.line_error_ratio`, and the usual `app.cost.*` fields.

Run this against a sample of at least a few hundred records so the metrics are meaningful.

### Deliverable 4 — Dual-run shadow diff

Run the *same sample* through both the sync path and the batch path. Under `docs/dual-run-diff.md`, report:

- Total records.
- Line-level error ratio on the batch path.
- Fraction of records where the batch response and sync response *differ*.
  - Structural diff (different response shape).
  - Semantic diff (same shape, different content).
- Cost delta observed (sync $X vs batch $Y for the same sample).

Any diff rate above the workload's eval-suite tolerance (from mod-106) is a blocker. Name it if you see it.

### Deliverable 5 — Fall-through test

Simulate a late batch (e.g. cancel the batch job or set the deadline to 1 second). Show that the fall-through path re-issues the affected records against the sync endpoint, that the ledger reflects the fall-through cost, and that no records are silently lost.

## Starter guidance

- **Do not skip Q1.** The freshness SLA is what makes or breaks the rubric; most teams fail this step. If the answer today is "we don't know," write that and identify the owner.
- **Use a cheap model** (e.g. `gpt-4o-mini`, `claude-3-5-haiku`) for the sample. Batch is the exercise, not model choice.
- **Namespace `custom_id`s** — the most common reconciliation bug is two submissions colliding on the same key.
- **Set the poll deadline shorter than the freshness SLA**, not shorter than the batch contract. If the workload tolerates 4 hours and the provider promises 24, poll for 3 hours and fall through at 3.
- **Log the discount landing**, not just the total cost. Effective per-request cost on batch vs sync is the number that proves the migration was worth it (chapter 04 substrate).
- **Do not try to migrate three workloads at once.** One workload, done well, is the goal.

## Acceptance criteria

- [ ] `docs/batch-eligibility.md` answers all three rubric questions with real numbers (or explicit "unknown, owner: X").
- [ ] `code/build_batch.py` and `code/run_batch.py` run end-to-end against a real batch API for at least a few hundred records.
- [ ] `docs/dual-run-diff.md` reports error ratio, diff rates (structural + semantic), and observed cost delta.
- [ ] The fall-through path is demonstrated (a late / failed batch is caught and re-issued sync; ledger reflects the fall-through).
- [ ] Observability spans on the batch path use the same vocabulary as the sync path (chapter 04 fields present).
- [ ] The rubric doc has peer-sign-off rows filled (or explicitly names who has not yet signed).

## Stretch goals

Pick any one:

- **Second batch provider.** Author a second `run_batch_anthropic.py` (or Bedrock, or Vertex) and rerun the same sample. Compare per-provider batch cost, per-provider line-error ratio, and per-provider p99 batch completion time.
- **Rate-limit interaction.** Deliberately submit a batch large enough to touch your account's TPM / RPM cap on the underlying model. What does the provider return? Fold the finding into your rubric.
- **Cache-across-batch check.** For a workload that shares a system prompt across all lines, confirm whether prompt caching (chapter 02) applies inside the batch job. If it does not on your provider, note the write-cost penalty on the rubric.
- **Inner-loop batch on a sync workload.** Pick an *interactive* workload with an embarrassingly-parallel inner step (e.g. batch-embed all retrieved documents) and migrate the inner step to batch while keeping the outer surface interactive.

## Deliverable

Submit as `mod-109/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/batch-eligibility.md`, `docs/dual-run-diff.md`.
- `code/build_batch.py`, `code/run_batch.py`, and the fall-through implementation.
- The raw sample input, the raw batch output file, and the reconciled results.
- A short `README.md` naming the workload and pointing at the rubric.
