# Batch APIs and the Offline Eligibility Rubric

## Why "batch" is a first-class shipping surface

Provider batch APIs — OpenAI's `/v1/batches`, Anthropic's Message Batches API, Bedrock's batch inference jobs, Google Vertex AI's batch prediction — trade wall-clock time for money. You send a file of thousands (or millions) of requests, the provider schedules them against spare capacity over a window (currently up to 24 hours on the mainstream shapes), and the responses come back at roughly half the per-token cost of the same requests run synchronously.<!-- needs-research: confirm current published discount rates: OpenAI Batch is ~50% off standard Chat Completions pricing with a 24h SLA; Anthropic Message Batches is ~50% off with a 24h SLA; Bedrock batch inference and Vertex batch prediction each have their own envelopes. --> On workloads that qualify, this is the single largest cost win the applied-AI engineer can execute without changing the model or the prompt.

The trap: teams either never migrate anything to batch (because the online path is what they already know) or migrate the wrong things (a "background" workload that turns out to have a two-minute SLA nobody wrote down). The savings are real but they are gated on a *written eligibility rubric* — the deliverable of this chapter — and on a migration pattern that survives the case where the batch job fails, is late, or returns partial results.

Peer-role boundary: `ai-infra-performance` owns the deep serving-side scheduling story (how the batch window fills against provider fleet capacity). The applied engineer owns the *shipping-side* question: which of our workloads belong on the batch path, how do we move them there without regressions, and how do we run both paths side-by-side during migration.

## The four canonical batch APIs

### OpenAI Batch API

The reference shape. You upload a JSONL file where each line is a self-contained `{custom_id, method, url, body}` record, create a batch that references the file, and poll the batch object until it moves to `completed` (with an output file) or `failed`. Requests are billed at a discount (currently ~50%) versus the same synchronous endpoints, subject to a nominal 24-hour completion window.<!-- needs-research: confirm current OpenAI Batch supported endpoints (chat completions, embeddings, responses) and the exact discount / SLA. -->

**What it optimises for.** Cost and simple orchestration. The API surface is minimal — upload, create, poll, download — and slots naturally into any cron-driven or event-driven job.

**Limits.** SLA is a *window*, not a deadline; most batches finish in minutes to hours, but the contract is 24 hours. Batch requests cannot use provider-side streaming, cannot exercise stateful features (assistants API), and inherit whatever per-model rate limit already applied to your account. Partial failures happen at the *request line* level — the batch itself succeeds, and individual lines contain errors you must handle.

### Anthropic Message Batches API

Same shape, different envelope. A batch is a JSONL of `{custom_id, params}` records where `params` is a Messages API request. Same nominal 24-hour window, same ~50% discount versus the synchronous Messages API, same request-level partial-failure model.<!-- needs-research: confirm Anthropic Message Batches current pricing, request-count cap per batch, and total-token cap per batch. --> Supports Anthropic's tool use, vision, and prompt caching (though caching across batch requests works differently from the online path — verify current behaviour).

### AWS Bedrock batch inference

Bedrock exposes batch inference jobs against a subset of hosted models. You stage input records in S3, create a `ModelInvocationJob` pointing at the input prefix and an output prefix, and Bedrock writes the results back to S3. Pricing and SLAs are per-model — Anthropic-on-Bedrock and Amazon Nova each publish their own batch envelope.<!-- needs-research: confirm current Bedrock batch-inference supported models and per-model discount envelope. --> IAM, region, encryption, and residency are inherited from your Bedrock configuration — the batch path stays inside your account boundary.

### Google Vertex AI batch prediction

Vertex batch prediction takes a source (BigQuery table or GCS JSONL), a destination, and a model, and processes rows in a scheduled job. The Gemini models support batch prediction with a discount versus online prediction; older PaLM / non-Gemini models had different envelopes.<!-- needs-research: confirm current Vertex batch discount for Gemini models and any minimum job size. -->

### The common substrate

All four batch APIs share:

- **Sync-vs-async unit price gap** — batch is materially cheaper (canonically ~50%, but always verify against current provider docs).
- **Delivery is a window, not a deadline.** Any request whose value expires inside the window is not batch-eligible.
- **Line-level partial failures.** The batch job "succeeds" and individual records inside can still error out. Your code must reconcile.
- **Line-level idempotency via `custom_id`.** Every request line carries a customer-supplied ID that binds the output back to your row.
- **No streaming; no interactive tool use.** Batch is fire-and-forget by construction.

Learn one API and the other three read as syntactic variations.

## The eligibility rubric

The single question that decides batch eligibility: **can any user wait 24 hours for this specific request?** If yes, it is a batch candidate. If no, it is an online request. The apparent simplicity hides three sub-questions the rubric must answer explicitly.

### Q1 — What is the freshness SLA?

Every workload has a maximum staleness beyond which the result is worthless. Nightly analytics tolerate 24 hours; a "background enrichment" that feeds a live UI element tolerates minutes; a "batch" that a user is polling for their own upload tolerates seconds. Write the freshness SLA down as a number, in the workload's ticket.

Rules of thumb:

- **≥ 24 hours** — batch-eligible for the main provider batch endpoints without qualification.
- **1–24 hours** — batch-eligible with care. Provider SLAs are 24 hours upper-bound; most batches finish sooner, but the contract does not guarantee sooner. Fall-through to sync if the batch is late (see the migration pattern below).
- **< 1 hour** — not batch-eligible on the main endpoints. Some providers offer "flex" or "priority" tiers between sync and batch (verify current availability); worth considering but not the same as batch.

### Q2 — Is the workload embarrassingly parallel at the record level?

Batch APIs are throughput-optimised, not orchestration-optimised. Each line in the JSONL is independent — no shared state across requests inside a batch, no way to have request N depend on the output of request N-1. If your workload is a for-loop over independent items (embedding a corpus, classifying tickets, summarising documents, generating alt-text for images), it is a natural fit. If your workload is a multi-turn agent, a tool-calling chain, or a request whose prompt depends on the previous response, it is not.

Sub-question: some agentic workloads have an *inner* embarrassingly-parallel step (batch-embed all retrieved documents, batch-summarise all outputs) inside an otherwise interactive flow. Migrate those steps, not the whole flow.

### Q3 — Is the workload cost-elastic in a way that batch pays off?

Batch is worth the migration work when the per-request savings times the request volume exceeds the cost of building and operating the batch path. Rough math: on a workload sending 1M requests/day at $0.02/request average, batch at 50% saves $10k/day — obviously worth building. At 1k requests/day at $0.002/request, batch saves $1/day — not worth building.

Break-even points depend on your team's operational cost of running the batch path (the S3 buckets, the poller, the reconciliation code, the on-call for batch failures). A useful lower-bound heuristic: **if projected monthly savings are less than one engineer-week of migration cost, do not migrate**. The savings are real but the maintenance drag is not.

### The rubric as an artefact

```markdown
# Batch eligibility — <workload> — <date>

## Workload
<one paragraph: what is this workload, who consumes its output, what shape are its requests>

## Q1 — Freshness SLA
Maximum staleness the consumer tolerates: <N minutes / hours / days>.
Source of the SLA: <ticket / contract / product decision>.
Batch-eligible on freshness? <yes | conditional | no — reason>.

## Q2 — Structural fit
Are requests independent at the line level? <yes | no — inner-loop only | no>.
Any state that must span requests? <none | describe>.

## Q3 — Economics
Current monthly cost of the sync path: $<X>.
Projected monthly cost on batch (× 0.5 as a first cut): $<X/2>.
Projected monthly savings: $<S>.
Estimated migration + operational cost: <engineer-weeks>.
Payback horizon: <months>.

## Decision
<Ship batch | Keep sync | Sync with a batch-eligible inner loop>

## Fall-through
If we ship batch: how do we handle a late batch (past the 24h contract)?
<retry as sync, alert, drop, degrade>.

## Peer sign-off
- Product: <name>
- Data / analytics owner (if relevant): <name>
- Finance / cost owner: <name>
```

Two properties. First, the rubric is **written per workload**, not per team. A team might have one workload on batch and three on sync; the answer belongs on the workload's page. Second, the rubric answers *how to fall through* when the batch is late — the migration is not done until that path exists.

## Migration pattern: dual-run before you cut over

The safe migration from a sync path to a batch path is not "flip the switch on Monday." It is a dual-run window in which both paths run in parallel and you compare outputs before retiring the sync path.

### Phase 1 — mirror the sync path into a batch file

For every request the sync path handles, write an equivalent JSONL line to a rolling batch file. Do **not** yet send the batch; just accumulate the file and validate that the lines are well-formed. This catches every edge case the sync path handles that the batch shape cannot express (streaming, per-request tool_choice mid-conversation, dynamic tool availability).

### Phase 2 — schedule the batch, keep the sync as source of truth

Once a day, submit the accumulated file as a real batch. When it completes, compare the batch output for each `custom_id` against the sync response that was already returned to the user. Track the diff rate (structural diff, semantic diff, cost delta). This is a shadow run — the user always sees the sync response; the batch response is metrics-only.

Watch for three failure modes:

- **Structural drift** — the batch output has a different response shape than sync (rare, but possible when features like structured outputs or tool use differ across endpoints).
- **Semantic drift** — same shape, materially different content (model-version drift, non-deterministic sampling; both expected at some rate, and the eval suite from mod-106 defines what "acceptable" looks like).
- **Missing / errored records** — batch line-level failures. The reconciliation code must know what to do (usually: re-queue for the next batch, or drop to sync if truly urgent).

### Phase 3 — cut over with a fall-through

Once the shadow-diff rate is inside eval-suite tolerance, flip the primary path to batch. The critical addition: a **fall-through** that catches any request whose freshness deadline is approaching before the batch completes, and re-issues it against the sync endpoint. Metric: fall-through rate. If it is above the number you wrote in the rubric (a few percent, typically), the freshness SLA in the rubric was wrong; go back and fix the rubric before scaling.

### Phase 4 — retire the sync path (or don't)

Retire the sync code only if the fall-through rate is stably below your ceiling. In many workloads, the answer is *don't* — you keep a small sync capacity as insurance, and it costs almost nothing because it almost never fires. The dual-path shape is legitimate; a paranoid fall-through is cheaper than a botched cutover.

## A skeleton in code

The OpenAI shape, chosen because the pattern is minimal and translates directly to Anthropic, Bedrock, and Vertex with request-shape edits.

```python
import json
import time
from pathlib import Path
from openai import OpenAI

client = OpenAI()

def build_batch_file(records, path: Path):
    """Write a JSONL file where each line is a self-contained request."""
    with path.open("w") as f:
        for rec in records:
            line = {
                "custom_id": rec["id"],
                "method": "POST",
                "url": "/v1/chat/completions",
                "body": {
                    "model": rec["model"],
                    "messages": rec["messages"],
                    "temperature": rec.get("temperature", 0),
                },
            }
            f.write(json.dumps(line) + "\n")

def submit_batch(path: Path):
    up = client.files.create(file=open(path, "rb"), purpose="batch")
    batch = client.batches.create(
        input_file_id=up.id,
        endpoint="/v1/chat/completions",
        completion_window="24h",
    )
    return batch.id

def poll_until_done(batch_id, poll_interval_s=60, deadline_s=24 * 3600):
    t_start = time.time()
    while True:
        batch = client.batches.retrieve(batch_id)
        if batch.status in ("completed", "failed", "expired", "cancelled"):
            return batch
        if time.time() - t_start > deadline_s:
            raise TimeoutError(f"batch {batch_id} exceeded deadline")
        time.sleep(poll_interval_s)

def read_results(batch):
    """Yield (custom_id, response_body, error) tuples from the output file."""
    if not batch.output_file_id:
        return
    contents = client.files.content(batch.output_file_id).text
    for raw in contents.splitlines():
        line = json.loads(raw)
        yield line["custom_id"], line.get("response"), line.get("error")

# Reconciliation: apply the batch results back to whatever store owns the workload.
# Any custom_id with an error becomes a fall-through candidate.
```

Three things to notice. The `custom_id` is the join key — every reconciliation step keys on it. The poll loop has a *deadline* that is shorter than the workload's freshness SLA, so a hanging batch triggers the fall-through before it becomes a user-visible incident. The result reader treats line-level errors as first-class output, not as an exception — because the batch object is `completed` even when individual lines failed.

## The observability seam

The mod-110 substrate consumes the same span vocabulary from the batch path that it does from the sync path. Emit:

- `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*` — one span per completed batch line, populated from the batch output file.
- `app.batch.batch_id`, `app.batch.enqueued_at`, `app.batch.completed_at` — for latency / throughput accounting on the batch itself.
- `app.batch.status` — `completed`, `partial`, `expired`, `failed`.
- `app.batch.line_error_ratio` — errors / total lines.
- `app.cost.usd` and `app.cost.discount_ratio` — so finance can see the batch discount landing in the per-workload attribution developed in chapter 04.

The dashboard's headline number for the batch path is **effective per-request cost** — total spend on the workload divided by total successful requests — plotted next to the same number for the sync fall-through. If they converge, something is wrong (batch is not paying off). If batch is materially lower, the migration was worth it.

## Anti-patterns

- **Batch as a synonym for "background."** Not every background job is batch-eligible. A background job with a 60-second product SLA is a sync job that happens not to block the UI; batch is 24-hour SLA. Read Q1 of the rubric before claiming eligibility.
- **No fall-through.** The batch is late (or the batch file is malformed and rejects at submission time), and there is no plan. On workloads that quietly matter — recommendation freshness, alerts, digests — the outage is silent for hours before someone notices. Build the fall-through before the cutover.
- **Custom-id collisions.** Two jobs on the same day, both using `<row_id>` as their `custom_id`, and the reconciliation step mixes results. Namespace `custom_id`s per submission (`<batch_seq>-<row_id>` or similar).
- **Silent line-level errors.** The reconciliation code reads only lines with a `response` field and drops lines with an `error` field on the floor. The rate slowly climbs; nothing alerts because "the batch completed." Track `line_error_ratio` and alert on it.
- **Prompt-cache thrashing across batch and sync.** A workload split half batch, half sync, with the same system prompt but subtly different framing across the two paths, means the cache never warms on either side. Keep the prompt shape byte-identical; chapter 02 is the constraint.
- **Batch API used to duck rate limits.** Batch inherits your rate limit; it does not multiply your capacity. Teams that migrate to batch expecting "more throughput" and get the same TPM cap are usually surprised. Read the current rate-limit docs before assuming batch is a capacity lever.

## Summary

- Batch APIs (OpenAI Batch, Anthropic Message Batches, Bedrock batch inference, Vertex batch prediction) trade wall-clock (up to a 24-hour window) for materially lower per-token cost (canonically ~50% off).
- Eligibility is decided per workload by a written rubric — freshness SLA, structural fit, economics — not by whether the workload feels "background."
- Migration is dual-run: mirror into a batch file, shadow-diff against sync, cut over with a fall-through path for late batches. The fall-through is not optional.
- Line-level failures are first-class output. Reconcile them; alert on the error-line ratio; never assume "batch completed" means "every request succeeded."
- Observability spans on the batch path use the same vocabulary as the sync path — the mod-110 substrate reads both without translation. The headline chart is **effective per-request cost** for batch vs sync on the same workload.
- Peer-role handoff: `ai-infra-performance` owns the serving-side batch scheduling story; the applied engineer owns the shipping-side rubric and the migration pattern.

[Chapter 04](04-per-feature-and-per-tenant-token-and-cost-accounting.md) turns cost from a monthly line item into a per-feature, per-tenant, per-outcome metric — the accounting layer that makes "cost per successful outcome" a real product number.
