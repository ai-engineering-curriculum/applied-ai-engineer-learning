# Trace Snapshots as Regression Fixtures

## The most valuable fixture you don't have yet

Ask any applied engineer whose product has real traffic what their best regression test would be, and the honest answer is *"whatever the model did for that user on Tuesday who complained."* Real production traces are the closest thing you will ever have to a hostile user, a realistic input distribution, and a labelled failure — because someone actually complained. Yet most product-agent teams do not have a working trace-to-fixture pipeline; they replay a bug once, fix the prompt, and never think to freeze the trace as a regression test.

This chapter is about closing that gap. A trace snapshot is a production request captured with enough surrounding context — inputs, retrieval, model outputs, tool calls, decoding config, model / prompt / retriever versions — that a future run can be replayed against a new model, prompt, or retriever and answer *"would the change under review have handled this trace at least as well as production did?"* That is a fixture in the same sense that `expected == actual` is a fixture: it is a specific past behaviour, pinned in a repo, that the next change is compared against.

The applied engineer owns this pipeline. The upstream observability plumbing (mod-110) captures the raw traces; this chapter is about *harvesting* those traces into a versioned fixture set that runs in the offline suite (chapter 01) and gates PRs (chapter 03).

## Two things a trace snapshot must be

**A snapshot must be replayable.** If you cannot re-run a snapshot against a new model/prompt/retriever and observe a new outcome, you have not built a regression fixture; you have built a screenshot. Replayability requires that everything the model saw at generation time is captured in a form the harness can feed to a candidate configuration — including retrieval context, tool outputs, the exact prompt template with variables substituted, and the seed / temperature.

**A snapshot must be judgeable.** The eval harness has to be able to decide *whether the new outcome is at least as good as the recorded one*. The judgement can be programmatic (does the new tool-call sequence end in the same customer-record state?), rubric-based (does the new answer score at least as well on the four dimensions the rubric names?), or comparison-based (a judge ranks the two responses head-to-head, with position controls from chapter 01). The point is that "judgeable" is a property of the fixture, not of the tooling: if the recorded trace doesn't include enough context for a downstream check to fire, it is not a fixture yet.

## Lineage: the six fields every snapshot pins

The single most common way a trace snapshot rots is that the team can no longer tell *what produced the recorded output.* Six months later the prompt has been rewritten, the retriever swapped, the model version bumped, and the fixture is comparing a candidate configuration to a ghost.

Every snapshot therefore pins, at minimum, this **lineage**:

- **Model identity.** Provider, model ID, and version pin — e.g. `anthropic/claude-opus-4-7`, `openai/gpt-4.1-2025-04-14`, `google/gemini-2.5-pro`. When a provider "silently" updates a model behind an alias, you have lost your ground truth; always pin to a dated or otherwise pinned-snapshot model ID. Each major provider documents its own pinning contract — Anthropic treats every Claude model ID as a pinned snapshot (with pre-4.6 aliases pointing to a dated ID), OpenAI exposes dated snapshot IDs alongside moving aliases, Google Gemini splits IDs into stable / preview / latest / experimental lanes, and Azure OpenAI lets you lock a deployment to a specific version — see `resources.md` for the docs pages.
- **Prompt lineage.** The exact prompt template ID and version (`support_agent.system.v14`), plus the fully-rendered prompt with variables substituted, plus the hash of the rendered prompt. The rendered prompt is what the model saw; the template ID is what the human debugs from.
- **Retriever lineage.** The retriever's identity (`bm25+cohere-rerank/v3.2`), the index snapshot ID or timestamp, the query as issued to the retriever, and the exact chunks returned with their doc IDs and scores. Without this, "the RAG regressed" is undebugable.
- **Tool catalogue lineage.** The tool schema versions the model was offered, the tool implementations' versions, and the recorded tool-call outputs (including any errors). A tool that changed its output shape between capture and replay is a silent breakage; the schema hash catches it.
- **Decoding config.** Temperature, top-p, top-k, max tokens, stop sequences, seed (where the provider supports it). A change in decoding that is not captured makes the replay a coin flip.
- **Environment / feature flags.** Which feature flags were on for this user, which A/B arm the request landed in, and which routing decision produced the model choice (mod-108's territory, but the flag values need to be captured *here*).

A snapshot without these fields is a lower-value artefact. A snapshot with them is a first-class regression fixture — and, crucially, one whose *diff* between capture and replay is legible to a code reviewer.

## PII scrubbing: the non-optional step

Production traces are, by construction, full of real user data. Freezing them into a fixture that ships in a git repo — or a shared eval store — is a data-handling decision the applied engineer must not shortcut.

The scrubbing pipeline has three passes.

**Pass 1 — field-level minimisation.** Some fields are almost never load-bearing for the eval and can be dropped wholesale: user IDs (replace with a stable per-fixture hash), email addresses (replace with `user@example.com`-shape placeholders), phone numbers, physical addresses, IP addresses, payment tokens, session IDs. This is done at capture time, before the trace ever lands in the fixture store.

**Pass 2 — free-text redaction.** The user's messages, the model's outputs, the retrieved chunks, and the tool results all contain free text. Use a PII detector — Microsoft Presidio, a hosted DLP service (Google Cloud Sensitive Data Protection, AWS Comprehend PII, Azure AI Language PII), an LLM-trace-specific scanner such as `llm-guard`, or a small locally-run model — to tag PII spans and replace them with typed placeholders (`<PERSON_NAME_1>`, `<EMAIL_1>`, `<POSTCODE_1>`); `resources.md` lists the current docs for each. Use *stable* placeholders — the same real value maps to the same placeholder within a trace — so downstream checks that depend on referring-back-to-a-name (a common failure mode in support-agent replies) still work.

**Pass 3 — review before publication.** The scrubbing pipeline is imperfect. A human reviews every trace before it is added to the fixture set. This is not "scale-blocking bureaucracy" — a fixture set of even a few hundred traces is bounded work, and a single leaked medical fact in a fixture is a very expensive incident.

For any surface handling **special-category** data (health, financial, biometric, immigration status, sexual orientation — GDPR Article 9 categories, or the equivalents in HIPAA / GLBA / PCI), the default is *not to snapshot the trace at all*, even scrubbed. Build synthetic-but-realistic fixtures instead, and use production traces only as inspiration for the synthetic ones. Compliance review, not the applied engineer alone, signs off on the exceptions.

Independent of jurisdiction, a scrubbed snapshot must never contain: named individuals other than the account owner (a support agent's reply "Priya from Acme wanted…" leaks a third party), free-text credentials (passwords, API keys pasted into chat), or anything the user explicitly asked to be forgotten under a deletion request. The deletion-request audit trail from mod-105 chapter 02 must include a step that purges matching entries from the fixture store.

## Sampling: which traces to freeze

You cannot freeze every trace. The applied engineer's job is to pick a *representative and diagnostic* small set. Four buckets, drawn intentionally:

1. **Complaints and thumbs-down.** Traces attached to negative feedback, cancellation reasons, or support tickets are pre-labelled failure candidates. Even if the current output is technically acceptable, the user's dissatisfaction is a signal worth freezing.
2. **High-blast-radius happy paths.** The five or ten most common request shapes the surface handles. A regression on these is a regression a lot of users will see. Freeze a handful per shape.
3. **The long tail.** A stratified sample by request length, tool-call depth, language, retrieval hit count, and any other axis your surface varies on. The long tail is where distribution shift bites first.
4. **The incident library.** Every incident that produced a fix (chapter 01, mode 5) mints one or two frozen traces. These are the "critical" tag in the pre-registration file — they must not regress.

Two anti-patterns to avoid:

- **The vanity sample.** Fifty carefully chosen "shows the product at its best" traces. Passes every gate; catches nothing.
- **The oversampled-recent sample.** Only last week's traces. The suite tracks whatever bug is currently in production and misses the seasonal or long-tail regressions.

A working ratio for a shipped surface is roughly: 30% complaints/negative-feedback, 30% high-frequency happy paths, 30% stratified long-tail, 10% incident library. Rotate 10-20% of the fixture set per quarter; deprecate traces whose behaviour is no longer representative (schema changed, feature removed).

## Snapshot schema: a worked example

Enough abstraction. Below is a concrete on-disk shape a trace snapshot can take. Illustrative, not literal — the details vary per surface.

```yaml
# fixtures/traces/support-agent/2026-07-14/T-4713-refund-out-of-window.yaml
snapshot_id: T-4713-refund-out-of-window
captured_at: "2026-07-14T09:41:22Z"
source:
  request_id: req_01HZ...              # opaque; the raw prod request ID is elsewhere
  feedback:
    thumbs: down
    reason_code: "did_not_help"
    tag: complaint
tags: [complaint, high_priority, refund_flow, critical]
lineage:
  model: anthropic/claude-opus-4-7
  prompt:
    template_id: support_agent.system.v14
    template_hash: sha256:2ab8...
    rendered_hash: sha256:c04e...
    rendered_path: rendered/T-4713.txt   # committed alongside
  retriever:
    id: bm25+cohere-rerank/v3.2
    index_snapshot: policy_kb_2026_07_10
  tools:
    catalogue_hash: sha256:9fd3...
    versions:
      search_orders: v2.1
      issue_refund: v1.4
  decoding:
    temperature: 0.2
    top_p: 1.0
    max_tokens: 1024
    seed: 424242
  flags:
    experiment.refund_ui_v2: on
    routing.arm: opus_default
inputs:
  user_message: |
    hi, i'd like a refund on order <ORDER_ID_1> please, it's been a while
  session_history:
    - role: user
      content: "hey"
    - role: assistant
      content: "Hi <PERSON_1>! How can I help?"
recorded_output:
  transcript_path: transcripts/T-4713.txt
  tool_calls:
    - name: search_orders
      arguments: {order_id: "<ORDER_ID_1>"}
      result_hash: sha256:a1b2...
    - name: issue_refund
      arguments: {order_id: "<ORDER_ID_1>", amount_cents: 3999}
      result: {status: "ok"}
  final_answer_hash: sha256:5e9a...
graders:
  # How to score a replay against this snapshot.
  - kind: state_check
    check: "no refund issued when order is > policy window"
    on_snapshot: fail_recorded_by_design   # the recorded behaviour is the bug
    expected_on_replay: no_refund_issued
  - kind: rubric
    rubric_id: support_reply.v3
    dimensions: [polite, policy_compliant, actionable]
    thresholds: {polite: 4, policy_compliant: 5, actionable: 3}
  - kind: judge_pairwise
    baseline: recorded_output.final_answer_hash
    judge_model: anthropic/claude-opus-4-8   # pinned; see chapter 04 on judge drift
    require_swap_agreement: true
```

Two properties are worth calling out. First, `graders` is a first-class field: the snapshot names *how it will be judged*, not just what happened. Second, a snapshot can encode "the recorded behaviour was the bug" (see `fail_recorded_by_design` above) — a fixture derived from a complaint should expect the replay to *diverge* from the original, not match it.

## Replay: how the fixture actually runs

The replay flow, as a pseudocode sketch:

```python
# eval/replay.py — sketched, not a library recommendation.
def replay(snapshot, candidate_config):
    ctx = build_context(
        model=candidate_config.model,
        prompt_template=load_template(candidate_config.prompt_template_id),
        retriever=build_retriever(candidate_config.retriever),
        tools=build_tool_catalogue(candidate_config.tool_versions),
        decoding=candidate_config.decoding,
        flags=snapshot.lineage.flags,        # preserve the flag arm the user was in
    )
    result = run_surface(
        user_message=snapshot.inputs.user_message,
        session_history=snapshot.inputs.session_history,
        ctx=ctx,
    )
    return {g.kind: score(g, snapshot, result) for g in snapshot.graders}
```

Two subtleties.

**Retriever pinning vs. re-issuing.** For most RAG regressions, you want to replay the retrieval against the current index — that is the point. But if the *index* has drifted since capture (a document was updated, a chunker was retuned), a replay that scores differently could reflect either a generation change or a retrieval change. The workaround: capture the retrieved chunks in the snapshot (as above) and offer two replay modes — *live retrieval* (issue the query against the current retriever) and *frozen retrieval* (feed the captured chunks directly to the generation step). A regression that appears under live retrieval but not frozen retrieval is a retrieval-side change.

**Tool-call replay vs. tool-call re-issue.** Same problem, one level up. For a tool-using agent, the recorded tool outputs are part of the fixture. A replay that re-issues the tool calls against the real backend will produce different outputs on any tool that isn't perfectly deterministic. Default to *tool-call replay* (feed the captured tool outputs back to the model as if the tool had returned them) for offline gating; reserve *tool-call re-issue* for end-to-end integration tests against a staging tenant (mod-105 chapter 04).

## Versioning: fixture set as first-class artefact

A snapshot set is a suite version, not a directory of files. Every fixture ships with:

- A `snapshot_id` that is stable for the lifetime of the fixture (renaming is a breaking change).
- A `captured_at` timestamp (never edited).
- A tag set that the CI runner can filter on (`critical`, `safety`, `long_tail`, `complaint`, etc.).
- Provenance: who curated it, from which production request, when it was added.

The fixture directory ships with a manifest:

```yaml
# fixtures/traces/support-agent/MANIFEST.yaml
suite_version: 2026.07.3
snapshot_count: 214
tag_counts:
  critical: 12
  safety: 20
  complaint: 84
  happy_path: 63
  long_tail: 35
rotation_policy:
  quarterly_rotation_ratio: 0.15
  last_rotation: "2026-07-01"
retention:
  deletion_request_ttl_days: 30    # cascaded from user deletion requests
```

The manifest is what CI compares against a baseline. A PR that adds a snapshot bumps `suite_version` by convention; a PR that changes a snapshot's `graders` or `lineage` is a *breaking* change and gets a distinct version bump.

## Anti-patterns

Six worth naming.

1. **Screenshots-not-fixtures.** The team saves prompt/output pairs as `.txt` files with no lineage. Six months later they cannot replay anything.
2. **Prod-tenant replay.** The eval hits the real production backend. Passing eval means someone got a duplicate refund. Replay against staging or against captured tool outputs, never against prod.
3. **PII in fixtures.** A single real customer name in a committed fixture is a compliance incident when the repo is compromised. Scrub at capture time, review before publication, honour deletion requests.
4. **The unpinned model.** The lineage says `gpt-4-turbo`; the provider silently rolls the alias forward; last quarter's fixture no longer means what it meant. Pin dated IDs where available; flag alias-only pins as tech debt.
5. **The scoring-drift-hidden fixture.** The `graders` block is edited casually. Now the fixture judges by a different rubric than it did last month; regressions are invisible. Grader changes are suite-version bumps, not edits.
6. **The one-off snapshot.** A trace is added, never rotated out, never tagged. Six months later nobody remembers why it's there. Every snapshot carries provenance and belongs to a bucket.

## Summary

- Production traces are the highest-signal fixtures a shipped surface has. Freezing them as regression tests is the applied engineer's job; the raw trace plumbing lives in mod-110.
- A snapshot must be *replayable* (all context the model saw is captured) and *judgeable* (grader spec is part of the fixture).
- Every snapshot pins six lineage fields: model, prompt, retriever, tool catalogue, decoding config, flags. Without them, "the fixture regressed" is undebugable.
- PII scrubbing is a three-pass pipeline (field minimisation, free-text redaction, human review) plus a deletion-request cascade. Special-category data is synthetic-first, snapshot-never by default.
- Sample traces from four buckets — complaints, high-frequency happy paths, stratified long-tail, incident library — and rotate ~15% quarterly.
- Ship a manifest with the fixture set. Suite version, snapshot count, tag counts, rotation policy. The manifest is what CI compares against a baseline.

Chapter 03 wires these snapshots — and the six offline modes from chapter 01 — into an eval-gated CI pipeline using Promptfoo, DeepEval, and the trace-store runners (Braintrust, Langfuse, Weave, Phoenix, LangSmith).
