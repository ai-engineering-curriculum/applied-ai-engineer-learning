# exercise-02: Memory System with Retention and PII Policy

**Estimated effort:** 4 hours

## Objective

Build the memory layer for a product agent under the discipline of [chapter 02](../02-memory-system-with-retention-and-pii-policy.md). You will implement the three layers (session, per-user, workspace), the disciplined fact schema, the per-turn retrieval, and — most importantly — a deletion / export pipeline that actually works end-to-end. The point of the exercise is not to demonstrate that you can store JSON. It is to demonstrate that when a compliance auditor asks "show me what you remember about user X, and now delete it," you can execute both operations in under a minute, with a receipt.

## Learning outcomes

By the end you should be able to:

- Separate session, per-user, and workspace memory into three stores with different retention and permission contracts.
- Design a typed per-user fact schema with sensitivity classification, provenance, source, confidence, and retention timestamps.
- Ship a per-turn memory retriever that gates access by sensitivity class and rate-limits volunteering.
- Build the four data-subject rights operations (access, rectification, erasure, portability) as first-class endpoints, tested against a real deletion receipt.
- Trade off provider-side memory features against a compliance contract you can enforce.

## Prerequisites

- You have read [chapter 02](../02-memory-system-with-retention-and-pii-policy.md).
- You have finished [exercise-01](exercise-01-agent-shape-decision-for-real-surface.md) or picked a shape independently; this exercise assumes a shape and a target surface.
- You have Python 3.11+ (or TypeScript / Node 20+) with Pydantic (or Zod), SQLite (for a lightweight persistent store), and a vector index (FAISS, Chroma, or LanceDB — pick one).
- You have `pytest` (or Vitest) and can run a small evaluation harness.

## Problem statement

You are the applied-AI engineer on **an internal legal-ops assistant** for a mid-sized company's in-house counsel team (Shape E from [chapter 01](../01-agent-shape-for-a-real-product-surface.md), with retrieval-heavy Shape C overlap). The surface helps lawyers draft contract clauses, look up prior deals, and answer regulator-facing questions.

Business context you inherit:

- Users are lawyers. They will assert facts to the agent (their jurisdiction, the deal type they specialise in, boilerplate they prefer) and expect the agent to remember them across sessions.
- The workspace is the legal team; the corpus contains contract precedent, internal memos, and a policy library. Facts about deals are workspace-shared; facts about lawyers' personal preferences are per-user.
- Legal / privacy has already communicated the retention policy: session state kept 30 days for debug, per-user facts retained until the user deletes them or leaves the company, workspace memory retained per the team's contract with the company.
- The company operates in the EU and California — the GDPR and CCPA data-subject rights apply. Sensitive data is possible: some lawyers work on employment matters and may assert facts about themselves (health, disability accommodations) that fall under GDPR Article 9 special categories.
- The company's in-house AI team is deciding whether to enable the provider's own memory feature; you have been asked to make a recommendation.

The MVP scope is a working three-layer memory system, the four rights operations, and a design memo for the provider-memory question.

## Requirements

### Deliverable 1 — The three memory layers

Under `memory/`, three storage modules with distinct schemas and boundaries.

#### 1a. Session store (Layer 1)

- Persistent (SQLite is fine) so a session survives a process restart.
- Keyed by `(session_id, user_id)`. A session with no user is anonymous and expires more aggressively.
- Stores the message history, tool calls / results, and any derived summarisations.
- Retention: configurable, default 30 days from last activity. A background sweeper deletes expired sessions.
- Encrypted at rest for the message content (a simple Fernet / libsodium wrapper is fine for the exercise; the point is that decryption is a code path, not an assumption).

#### 1b. Per-user memory store (Layer 2)

- The typed schema from [chapter 02](../02-memory-system-with-retention-and-pii-policy.md), implemented in Pydantic (or Zod). At minimum: `fact_id`, `user_id`, `workspace_id: str | None`, `key` (from a controlled vocabulary), `value`, `source` (`user_asserted` | `model_inferred` | `tool_result` | `admin_set`), `sensitivity` (`public` | `internal` | `pii` | `special_category`), `confidence`, `first_seen_at`, `last_used_at`, `retention_end: datetime | None`, `provenance`.
- A **controlled `key` vocabulary** committed to `memory/keys.py` — the list of known keys and their default sensitivity. New keys require a schema change.
- Storage: SQLite for the structured columns; a small vector index for semantic retrieval on `key` + `value` (for retrieving facts by proximity to the user's message).

#### 1c. Workspace memory store (Layer 3)

- Same schema shape with `workspace_id` required and `user_id` optional. Facts assert something about the workspace, not the user.
- ACL enforcement: reads are gated by the caller's workspace membership. A user removed from a workspace loses read access to workspace facts.
- Deletion: workspace-admin only, not end-user.

### Deliverable 2 — The extraction and retrieval pipelines

Under `pipelines/`.

#### 2a. Fact extractor

- Given a session (messages, tool calls), produce a list of candidate `MemoryFact` records with the correct `source`, `sensitivity`, `key`, and `provenance`. Do not persist directly; return candidates.
- Implemented as a structured-output pipeline per mod-102 — the extractor's system prompt + typed schema forces the model to only produce facts whose `key` is in the controlled vocabulary. An extraction that would produce a new key returns a "new key requested" candidate instead, for admin review.
- A separate `promote_candidates(candidates)` step is the decision surface for whether a candidate becomes a stored fact. Model-inferred facts require a confidence threshold; user-asserted facts pass straight through; special-category facts always require explicit user confirmation, not automatic promotion.

#### 2b. Per-turn retriever

- Given a user message + current session context + user identity, produce the top-k facts to inject into the prompt.
- **A separate retriever per sensitivity class.** The default retriever pulls `public` and `internal` facts. `pii` facts are pulled only when the message names a reason (e.g., the user asks "what did I say my timezone was?"). `special_category` facts are pulled only through a policy-gated code path that logs the access.
- A **rate limit on volunteering**: an injected fact does not automatically get repeated in the response. The system prompt (write it) governs when the model may bring up a stored fact vs. when it should keep the fact latent.

### Deliverable 3 — The four rights operations

Under `rights/`, one endpoint (a plain Python function or an HTTP handler — your choice) per right.

#### 3a. Access (`get_user_memory(user_id) -> UserMemoryExport`)

Returns everything Layer 2 holds about the user (all facts, all provenance) plus a summary of Layer 1 sessions (session IDs, timestamps, no content in the summary — content is in the full export).

#### 3b. Rectification (`update_fact(fact_id, patch) -> MemoryFact`)

The user (or an admin acting on the user's behalf) can update a fact's `value`, `confidence`, or `sensitivity`. The prior version is preserved in a history table so the fact's edit history is recoverable.

#### 3c. Erasure (`delete_user(user_id, reason) -> DeletionReceipt`)

The full deletion pipeline from [chapter 02](../02-memory-system-with-retention-and-pii-policy.md):

- Purge every Layer 1 session for the user.
- Purge every Layer 2 fact.
- Strip provenance links in Layer 3 that reference the user; do not delete Layer 3 facts themselves.
- Purge derived caches: vector index entries, LLM-provider cache identifiers (stub the provider-side call if you don't have a real account), audit log content (keep metadata).
- Return a receipt: what was deleted, how many rows, and a signed / hashed proof.

The deletion is idempotent — a second call for the same user returns a receipt showing zero rows deleted, not an error.

#### 3d. Portability (`export_user_memory(user_id) -> bytes`)

A JSON dump containing every Layer 1 session (with content), every Layer 2 fact (with history), every Layer 3 provenance link referencing the user, and every derived-cache entry. The export names the same schema the app uses — no hidden internal fields. The JSON is signed / hashed to detect tampering after export.

### Deliverable 4 — Tests

Under `tests/`, at minimum:

- **Schema tests.** New-key rejection. Sensitivity classification defaults. `retention_end` sweep behaviour.
- **Extraction tests.** Given a fixture conversation, does the extractor produce the expected candidate facts? Does it correctly refuse to invent new keys?
- **Retrieval tests.** Given a message and a store state, does the retriever return the expected facts? Does the `pii` retriever refuse without a reason? Does the `special_category` retriever require the policy gate?
- **Rights tests.**
  - Access returns everything. Nothing hidden.
  - Rectification preserves history.
  - Erasure removes Layer 1 sessions, Layer 2 facts, and Layer 3 provenance links; does not remove Layer 3 facts themselves.
  - Erasure is idempotent.
  - Export contains a complete round-trip: import into an empty store produces the same state.

### Deliverable 5 — The provider-memory design memo

`memo.md`. One page. Answer:

- **Turn on or off?** State the decision. Reference the compliance surface (GDPR, CCPA, the company's DPA with the provider).
- **If off, why?** Which of your rights operations breaks if the provider persists user data on its side?
- **If on, how do you fulfil the erasure right?** Name the concrete provider API call, its SLA, and how it composes with your `delete_user` pipeline.
- **What changes if you're on Anthropic, OpenAI, and Google?** Two sentences per provider on the current published retention posture. Cite the docs; do not guess.<!-- needs-research: link the current pages from Anthropic, OpenAI, and Google on data-usage / retention for API traffic. -->

## Starter guidance

- **Start with the schema.** The Pydantic type and the controlled key vocabulary set the shape of everything downstream. If you find yourself changing the schema during the retrieval work, stop and stabilise it.
- **Do NOT store raw messages as facts.** The extractor's job is to turn raw messages into structured facts, not to `notes.append(message)`. If you find yourself hitting this shape, revisit [chapter 02](../02-memory-system-with-retention-and-pii-policy.md)'s "unstructured notes blob" anti-pattern.
- **Write the deletion first, then the writes.** If the deletion doesn't work, no amount of writes are safe. Deletion tests are a first-class citizen, not an afterthought.
- **Encrypt Layer 1 content.** Even if it's a demo, this is muscle memory you want. The unencrypted-message-store is one of the more common real production incidents.
- **Do not use production data.** Everything in this exercise is synthetic. Real user conversations do not belong in exercises.
- **Sensitivity classification is a *pre-write* step.** Do not persist a fact and *then* classify it. The classifier runs during extraction; the persistence layer refuses to store an unclassified fact.
- **Idempotency in the deletion.** Sneak the same discipline you learned in mod-104 chapter 01 in here. `delete_user` may be retried; the receipt tells the caller what happened this time.

## Acceptance criteria

- [ ] The three layers exist as separate stores with the retention windows and permission boundaries from chapter 02.
- [ ] The per-user schema carries `key` (controlled vocabulary), `source`, `sensitivity`, `confidence`, `first_seen_at`, `last_used_at`, `retention_end`, and `provenance`.
- [ ] The extractor produces typed candidates and refuses to invent new keys; a candidate for a new key is queued for admin review, not silently dropped.
- [ ] The per-turn retriever is sensitivity-class-aware; the `pii` retriever requires a reason; the `special_category` retriever requires a policy gate that logs the access.
- [ ] `get_user_memory` returns everything the app holds about the user, in the app's schema, with no hidden fields.
- [ ] `update_fact` preserves the fact's history; no in-place overwrite.
- [ ] `delete_user` deletes Layer 1 sessions, Layer 2 facts, derived cache entries, and provenance links from Layer 3, but does not touch Layer 3 facts themselves. Idempotent.
- [ ] `export_user_memory` produces a machine-readable dump that can be round-trip-imported into an empty store.
- [ ] Layer 1 message content is encrypted at rest.
- [ ] The deletion receipt is verifiable (signature or hash-chained) so an auditor can prove it wasn't fabricated.
- [ ] Test coverage includes all four rights operations, the sensitivity gating, and the extractor's key discipline.
- [ ] The provider-memory memo makes a call, cites the docs, and names the concrete erasure API for the on-case.

## Stretch goals

Pick any two:

- **Admin surface for new keys.** A CLI command that lists pending "new key requested" candidates and lets an operator approve them (with a default sensitivity) or reject them. The workflow is what keeps the controlled vocabulary evolvable without being uncontrolled.
- **The user-facing "what do you know about me?" UI.** A small web page (Streamlit / Gradio / plain Flask) that reads `get_user_memory` and shows a legible list of facts, their sources, their timestamps, and edit / delete buttons wired to `update_fact` and per-fact delete. This is where "the four rights" stops being abstract and starts being product.
- **Right-to-know timing SLA.** Instrument the deletion pipeline; measure p50 and p99 latency for `delete_user` on a store of 10,000 facts. Publish the SLA. Add a load test that fails if the pipeline crosses the SLA.
- **Provider-side integration.** If you have a provider account, wire `delete_user` to also call the provider's deletion endpoint (if any) and record the provider's receipt in your own receipt. Reason about what happens if the provider's deletion fails — is your erasure valid?
- **A synthetic PII fixture.** A set of conversations that contain a mix of `internal`, `pii`, and `special_category` assertions. Run your extractor against them; measure precision / recall on the sensitivity classification. This is a mod-102 structured-output eval applied to the extractor.
- **Cross-workspace isolation test.** A user in workspace A must not be able to read workspace B's facts, even through a subtle bug in the retriever. Write a fuzz test that constructs pairs of workspaces and confirms isolation on 100 random queries.

## Deliverable

Submit as `mod-105/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `memory/session.py`, `memory/user.py`, `memory/workspace.py`.
- `memory/schema.py`, `memory/keys.py`.
- `pipelines/extractor.py`, `pipelines/retriever.py`.
- `rights/access.py`, `rights/rectification.py`, `rights/erasure.py`, `rights/portability.py`.
- `tests/` with the coverage in Deliverable 4.
- `memo.md` — the provider-memory design decision.
- A short `README.md` linking the files together, naming the shape you chose, and stating which provider you targeted for the memo.
- Any stretch-goal artefacts named alongside.
