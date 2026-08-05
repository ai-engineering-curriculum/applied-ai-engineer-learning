# User Feedback Triage: From Thumbs-Down to Eval Dataset

## Why user feedback needs a pipeline, not a Slack channel

The default LLM-product feedback loop, at every organisation, looks like this: the surface renders a thumbs-up / thumbs-down under each response. The clicks stream into a table nobody reads. Once a week, someone opens the table, sees "42% thumbs-down last week, up from 38%," has no idea which response the user was reacting to, no way to sort the mass of clicks by feature, tenant, or turn, and closes the tab. The dashboard is a lagging indicator of feeling. It changes no code, opens no ticket, and improves no prompt.

The observability substrate from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) tells you *what the system did*. User feedback tells you *how the user reacted to what the system did*. Neither is useful alone. Joined — feedback attached to the exact trace that produced it, deduped and triaged, routed to an owner, and eventually landing in the eval dataset from [mod-106](../mod-106-feature-side-evaluation/README.md) — it becomes the ground-truth stream the whole feedback loop runs on.

This chapter names the pipeline: the **capture surface** (what to ask the user), the **join to the trace**, the **dedup and triage** stage, and the **expert-review substrate** — Argilla or Label Studio, both mature enough to run at team scale without a bespoke build.

## The capture surface: three signals, one join key

The pipeline needs three feedback shapes. Every one attaches to the same trace ID, which is what makes them joinable downstream.

### Signal 1 — the binary thumb

The thumbs-up / thumbs-down under the response. The rule: **there is only one bit of information**, and treating it as ordinal or continuous is over-reading. What it does well: the click rate is cheap, dense, and stable enough to be a trend indicator. What it does poorly: it does not tell you *why*.

The capture:

```typescript
type BinaryFeedback = {
    trace_id: string;              // the OTel trace ID from the response that was rated
    turn_span_id: string;          // which turn in a multi-turn conversation
    feature: string;               // "support_agent_v3"
    tenant_id: string;             // or hashed tenant
    user_id_hash: string;          // hashed at the SDK boundary — see chapter 02
    signal: "thumbs_up" | "thumbs_down";
    timestamp: string;             // ISO 8601
    client_version: string;        // surface build the user was on
};
```

The `trace_id` is the join key. If your surface renders the response and does not know the trace ID, that is a bug — the trace ID must be threaded from the backend to the client as part of the response envelope, so that a click can reference the exact span.

### Signal 2 — structured categorical feedback

When the user thumbs-down, prompt them for a category. Not free text — a small enum drawn from the taxonomy your feature already uses. For a support agent: `wrong_answer`, `missed_context`, `unhelpful_refusal`, `too_long`, `too_short`, `hallucinated_source`, `other`. For a code assistant: `wrong_syntax`, `wrong_api`, `off_target`, `slow`, `interrupted_my_flow`, `other`.

Two design constraints.

**Keep the enum short — six to eight options.** Longer lists fatigue the user and collapse to "other." Six well-chosen options catch 80% of the meaningful signal and force you to *name* your top failure modes explicitly.

**Change the enum only through a versioned migration.** `feedback_taxonomy_v2.yaml` is a file with a diff, not a config-panel change. When the enum changes, dashboards that group by category need to know a new value appeared; downstream eval datasets need to know old labels may need re-mapping.

```typescript
type StructuredFeedback = BinaryFeedback & {
    signal: "thumbs_down";       // structured only follows a thumbs-down
    category: "wrong_answer" | "missed_context" | "unhelpful_refusal"
            | "too_long" | "too_short" | "hallucinated_source" | "other";
    taxonomy_version: "v2";
};
```

### Signal 3 — free-text open comment

The optional textarea attached to the thumbs-down UI. Density is low (maybe 5–15% of thumbs-downs include text) but signal density is high — the text is where the user says the specific thing the enum flattened.

Free text is the field the PII scrub matrix from [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) matters most on the way in. Users type customer names, real emails, sometimes secrets. Scrub on ingress before the text lands in storage; store both the scrubbed text (for triage) and a hash of the raw (for dedup) — never the raw content.

```typescript
type FreeTextFeedback = StructuredFeedback & {
    comment_scrubbed: string;     // regex-scrubbed on the way in
    comment_hash: string;         // sha256 of the raw text, for dedup only
    comment_length_original: number;
};
```

### The "unsolicited-report" channel

A fourth capture path worth calling out: a user-invoked "report this response" button (or Slack app command, or in-IDE keybinding) that fires even when the user did not click thumbs-down — often used to flag "this was fine but weird" or "this is a policy issue." Treat these as thumbs-down for the purposes of tail sampling ([chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md)) but tag them separately (`signal = "reported"`) so triage can route them differently.

## Joining feedback to the trace

The join is trivial in shape and load-bearing in practice. Every feedback record carries the `trace_id`; every trace already has the full context from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md).

The join produces a **feedback event** with the full trace context attached:

```json
{
    "feedback_id": "fbk_01HXYZ...",
    "trace_id": "8f3a...",
    "feature": "support_agent_v3",
    "signal": "thumbs_down",
    "category": "hallucinated_source",
    "comment_scrubbed": "The article it linked doesn't exist.",
    "trace_summary": {
        "prompt_id": "support_answer_v14",
        "gen_ai.request.model": "gpt-4o-2024-08-06",
        "gen_ai.response.model": "gpt-4o-2024-08-06",
        "gen_ai.usage.output_tokens": 234,
        "retrieval": {
            "index": "support-articles-v3",
            "top_hits": [
                {"id": "art_012", "score": 0.42},
                {"id": "art_099", "score": 0.31}
            ]
        },
        "tool_calls": [{"name": "search_kb", "status": "ok"}],
        "guardrail": {"decision": "allow"}
    },
    "input_summary": "user asked: how do I reset my WPA2 password on the ...",
    "output_summary": "the assistant answered: to reset your WPA2 password, go to ..."
}
```

Two things this shape makes possible that the raw thumbs-down does not.

**Filterable by anything on the trace.** "Show me every `hallucinated_source` thumbs-down where the top-1 retrieval score was under 0.5" — that is one query, and the answer is often the fix.

**Directly reproducible.** The trace has the prompt and the retrieval hits. An engineer can re-run the exact turn against a new prompt version or a new model and see whether the change fixes it.

The join is the point where two upstream disciplines — the span vocabulary from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) and the tail-sampling policy from [chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md) — pay off. Without the span, the feedback has no trace to attach to. Without tail sampling that force-keeps thumbs-down traces, the trace is gone before the feedback arrives.

## Dedup: the same complaint from a hundred users

At scale, the same underlying failure produces the same category from many users on many trace IDs. The pipeline must dedup so triage sees "300 users complained that the WPA2 reset flow returns a hallucinated article" as **one** entry with a count, not 300.

Three dedup keys, applied in order:

1. **Exact match on `(feature, category, prompt_id, top_retrieved_doc_ids, output_summary_hash)`.** Same feature, same failure category, same prompt, same retrieval hits, same output = the same bug. Merge.
2. **Near-match on comment embedding when free text is present.** If two comments have cosine similarity above a threshold (~0.85 with a small embedding model), they are the same complaint in different words. Merge, keep both original comments as evidence.
3. **Time-window clustering per feature.** If the thumbs-down rate for a feature spikes above 3× its baseline in a 10-minute window, everything in that window is presumed to be one incident. Merge into one entry, tag as `incident_suspected: true` and page the on-call (this is the guardrail-FP-spike playbook trigger from [chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md)).

The output of the dedup stage is a **triage entry**, not a feedback event:

```json
{
    "entry_id": "tri_01HXYZ...",
    "cluster_key": "support_agent_v3|hallucinated_source|support_answer_v14|...",
    "signal": "thumbs_down",
    "category": "hallucinated_source",
    "feature": "support_agent_v3",
    "count": 47,
    "distinct_users": 31,
    "first_seen": "2026-03-15T09:12:00Z",
    "last_seen": "2026-03-15T14:47:00Z",
    "trace_ids": ["8f3a...", "9b21...", ...],       // representatives, capped at 20
    "example_input": "...",
    "example_output": "...",
    "state": "unassigned",
    "owner": null
}
```

The triage entry is the unit of work an engineer picks up. Not 47 individual thumbs-down clicks.

## Owner-assigned triage — who gets paged for what

Triage entries need routing rules so they land in front of the right person without a human dispatcher. Two dimensions determine the owner:

- **Feature** — every feature has an owning team defined in a routing table (`docs/observability/feedback-routing.yaml`). The team's on-call rotation from [chapter 05](05-per-feature-status-page-and-on-call-runbook.md) is the default owner.
- **Category** — some categories cut across features and route differently. `hallucinated_source` on any RAG feature routes to the retrieval-owning team plus the feature owner. `unhelpful_refusal` on any feature routes to the guardrail owner (from [mod-107 chapter 05](../mod-107-guardrails-and-safety-on-the-surface/05-refusal-calibration-and-owasp-mapping.md)) plus the feature owner. `wrong_answer` stays with the feature owner.

The routing table shape:

```yaml
# docs/observability/feedback-routing.yaml
features:
  support_agent_v3:
    owner_team: "agent-team"
    on_call_rotation: "agent-primary"
  batch_summariser:
    owner_team: "batch-team"
    on_call_rotation: "batch-primary"

category_overrides:
  hallucinated_source:
    also_notify: ["retrieval-team"]
  unhelpful_refusal:
    also_notify: ["safety-team"]
    priority_boost: true                # a slow-to-triage refusal is an over-blocking incident
  jailbreak_success:
    also_notify: ["safety-team"]
    priority_boost: true
    page_on: incident_suspected         # if the cluster is spiking, page immediately

sla_by_priority:
  incident:  { triage_within: "15m",  resolve_within: "4h" }
  high:      { triage_within: "1d",   resolve_within: "1w" }
  normal:    { triage_within: "1w",   resolve_within: "4w" }
```

The routing table is a policy artefact. Changes go through review; new categories require a `also_notify` decision explicitly, not by default.

### The triage state machine

Each triage entry has a small state machine an engineer walks it through:

```
unassigned  →  assigned  →  investigating  →  reproduced  →  fix_open  →  fix_shipped  →  closed
                                                  ↓                            ↓
                                             not_reproduced              regression_expected
                                                  ↓                            ↓
                                                closed                 closed_with_note
```

- `assigned`: an owner has been named. Timer starts on the SLA.
- `investigating`: the engineer has opened the trace, read the input/output.
- `reproduced`: the failure is confirmed on the current prod stack (not a stale trace from a shipped fix).
- `not_reproduced`: cannot repro — either the trace is stale, the user's context was unusual, or the failure is intermittent. Close with a note.
- `fix_open`: a PR / prompt-version change / retrieval-corpus fix exists.
- `fix_shipped`: the change is in production.
- `closed`: the SLA clock stops.

The state transitions are events, not free-form status fields. Dashboards read the state distribution ("how many entries have been in `investigating` for more than the SLA?") to surface stalled triage.

## The expert-review substrate: Argilla or Label Studio

Structured feedback + free text + trace context is a dataset that wants a real annotation UI, not a spreadsheet. Two open-source tools are mature enough to adopt at team scale:

- **Argilla** — LLM-native from the start, integrates with Hugging Face datasets, supports LLM-as-judge suggestions, has a first-class notion of feedback records with multiple response fields per record. Best fit when the review team is comfortable with the LLM ecosystem and wants to hand-annotated data straight into training / eval workflows.
- **Label Studio** — general-purpose annotation platform, older and more mature ML-Ops tooling around it, supports arbitrary UI configs (thumbs, ratings, category picks, span highlighting, comparative rankings). Best fit when the review team already uses it for other data-annotation work.

Both do the same job for this pipeline: **an expert opens a triage entry, sees the input / output / retrieved documents / feedback comment, and produces a structured judgment.** The judgment might be:

- Confirming the category (the user said `hallucinated_source`; the expert says yes it is a hallucination).
- Correcting the category (the user said `wrong_answer`; the expert says actually the retrieval was wrong upstream).
- Producing a **gold answer** — what the response *should have been*. This is the highest-value artefact, because gold answers become the eval dataset from [mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md).
- Marking the entry as `not_a_defect` — the user was wrong to thumbs-down; the response was correct. These are as important as confirmed defects because they calibrate over-triage.

### The dataset export

The expert review produces a stream of labelled records. Weekly (or at any cadence the eval-owning team prefers), the labelled records are exported to the eval dataset:

```
raw feedback events
    → dedup
    → triage entries
    → owner-assigned triage
    → expert review (Argilla / Label Studio)
    → labelled records
    → eval dataset (mod-106 chapter 01)
```

Every step in that pipeline is versioned. The eval dataset's row count grows over time; every new row cites the triage entry it came from; every triage entry cites the feedback events it deduped from; every feedback event cites the trace. If somebody two quarters from now asks "why is this fixture in the eval set," the answer is a chain of clicks back to a real user complaint from a real trace.

### What the expert-review step actually catches

The reason to run expert review — instead of just piping the raw feedback into the eval dataset directly — is that raw user feedback is *noisy*. Users click thumbs-down for wrong reasons (they clicked the wrong button, they were angry about something unrelated, they misread the response). Users pick the wrong category (they think "wrong answer" means "unhelpful"). Free text is sometimes uninterpretable without context.

Empirically, on a healthy feedback pipeline, roughly a third of thumbs-downs are `not_a_defect` on review — either the response was correct, or the failure was in a different sub-step than the user identified, or the "correct" behaviour was in fact what the user got. Skipping expert review means the eval dataset is contaminated with ~33% noise; every improvement then has to beat the noise floor.

## Instrumentation the pipeline needs

The observability substrate from [chapter 01](01-otel-genai-and-openinference-span-vocabulary.md) grows two attributes to close the loop:

```python
# On the LLM span of the response the user rated.
span.set_attribute("app.feedback.signal", "thumbs_down")     # populated after-the-fact via async join
span.set_attribute("app.feedback.category", "hallucinated_source")
span.set_attribute("app.feedback.entry_id", "tri_01HXYZ...")
```

These attributes are set **not on the span at emission time** — the user has not clicked yet — but by a stream processor that joins feedback events to trace spans and updates the trace record. Whether your backend supports post-hoc span updates varies (Honeycomb, Datadog, and most vendor tools have "derived columns" or "attributes-from-events" that fill the same role); if yours does not, keep the join in a separate join table and let the trace UI link to it by trace ID.

The reason to want the feedback attribute *on the trace itself*: it makes tail sampling ([chapter 02](02-sampling-pii-scrubbing-and-trace-budgets.md)) trivial — "always keep traces where `app.feedback.signal = thumbs_down`" fires from the trace record, not from a cross-system query.

## Anti-patterns

- **Feedback dashboard as a lagging indicator.** Weekly thumbs-down percentage on a chart. Nobody knows which response any of the clicks refers to. No trace join = no action.
- **Enum with 30 options.** Users pick "other" every time. The signal collapses. Six to eight categories is the working number.
- **Free-text comments stored raw.** Every ingest is a PII incident waiting. Scrub on the way in; keep a hash of the raw for dedup.
- **No dedup.** Triage queue has 300 entries; 297 of them are the same bug. Engineer burns half a day realising it. Dedup on `(feature, category, prompt_id, retrieval_ids, output_hash)` first, embedding-similarity second.
- **Piping raw feedback straight into the eval dataset.** The dataset is ~1/3 noise. Every model or prompt change looks worse than it is because the baseline is contaminated. Expert review is not optional; it is the noise filter.
- **Feedback with no owner.** Triage queue grows monotonically. Nobody knows they are on the hook. Route on `(feature, category)` to a named team's on-call rotation with an SLA.
- **`not_a_defect` never recorded.** The engineer resolves the entry as "closed" without noting the user was wrong. Over-triage rate becomes unmeasurable; the pipeline drifts toward "everything is a defect" which is a form of over-blocking.
- **Trace ID not surfaced to the client.** The thumbs-down click has no way to reference the response it was about. Every downstream join is impossible. Thread the trace ID through the response envelope from day one.
- **Enum change without a taxonomy version bump.** Old thumbs-downs categorised as `wrong_answer` and new ones as `factually_incorrect` are the same thing labelled differently. The dashboard shows a "spike" that is a schema change. Version the taxonomy explicitly.

## Summary

- Feedback is three signals joined to one key: the **binary thumb** (dense, no reason), a **short structured category** (six to eight options, versioned taxonomy), and an **optional free-text comment** (scrubbed on ingress, hash retained for dedup). All three carry the `trace_id` of the response they refer to.
- The join to the trace is the point where feedback becomes *actionable* — filterable by prompt version, retrieval hit scores, tool status, guardrail decision. Without the join, feedback is a mood ring.
- **Dedup** collapses the same underlying failure across many users into one triage entry with a count. Exact-match on `(feature, category, prompt_id, retrieval_ids, output_hash)` first, embedding-similarity on comments second, time-window clustering for incident-shaped spikes third.
- **Route triage entries to an owner** by `(feature, category)`. `unhelpful_refusal` and `jailbreak_success` co-notify the safety team; `hallucinated_source` co-notifies the retrieval team; a per-entry SLA governs triage and resolution time.
- **Expert review** (Argilla or Label Studio) is the noise filter. Roughly a third of thumbs-downs are `not_a_defect` on review; skipping the step contaminates the downstream eval dataset with that noise. Reviewed records — including gold answers — become the eval fixtures from [mod-106 chapter 01](../mod-106-feature-side-evaluation/01-offline-eval-suites-a-pr-can-run.md).
- **Post the feedback signal back onto the trace** (`app.feedback.signal`, `app.feedback.category`, `app.feedback.entry_id`) so tail sampling and dashboards read from the trace record directly.

[Chapter 04](04-incident-playbooks-for-six-llm-failure-modes.md) turns from steady-state feedback to the six recurring incident modes an LLM feature actually breaks along, and the written playbook the on-call reaches for when the alert fires.
