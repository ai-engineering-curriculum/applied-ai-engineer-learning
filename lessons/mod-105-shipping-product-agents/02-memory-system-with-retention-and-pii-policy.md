# Memory System with Retention and PII Policy

## What "memory" even means in a product agent

"Memory" is the second most overloaded word in this module, after "agent." At the model-provider level it means the current message list plus the system prompt. At the framework level it means whatever the framework's `Memory` object stores. At the product level it means the user-facing feature ("ChatGPT remembers your name across chats") and the compliance contract behind that feature ("you can delete what it remembers"). This chapter uses the product-level definition and works its way down.

Concretely, a product agent needs to answer four questions, in this order, before you write a single line of memory code:

1. **What time-scale does the memory serve?** A turn? A session? A day? A year? The retention answers, the storage layer, and the eval strategy all follow from this.
2. **Whose data is it?** The user's? The workspace's (shared across users)? The company's? A regulated third party's? The permissions model follows from this.
3. **Which data classes may it hold?** Free text? Structured facts? PII? Health? Payment? The redaction / minimisation policy follows from this.
4. **What's the deletion / export contract?** GDPR / CCPA / your Terms of Service demand answers to "delete me" and "export my data." The storage layer follows from those requirements.

Skip these and you will build a memory system that ships facts about users that they cannot see, cannot correct, and cannot delete. That is a shipped compliance incident. Every mature product-agent memory design starts from these questions, not from "let's put the last N turns in the context."

## Three layers of memory, and one contract

Product-agent memory decomposes cleanly into three layers, each with its own retention window, storage substrate, and deletion contract. Names differ between providers (OpenAI's "Memory," Anthropic's "context editing" and per-project memory, LangChain's "short-term / long-term," LangGraph's "checkpointer / store"); the layers below are provider-neutral.

### Layer 1 — Session context

**Definition.** The message history and derived state of the *current conversation or task*. Ends when the session ends (a chat closes, a workspace task completes, a browsing task is dismissed).

**Purpose.** The agent knows what it just said, what the user just said, what tools it just called, and what those tools returned. Without this, no multi-turn behaviour is possible.

**Storage.** In-process for the duration of the session, plus a checkpoint layer if the session may span multiple hosts / restarts (LangGraph's `SqliteSaver` / `PostgresSaver`, OpenAI Agents SDK's `Sessions`, the Claude Agent SDK's session store, or your own equivalent). Serialised state must be encrypted at rest; every serialised session must be tied to the user's identity so that a "delete me" request can find it.

**Retention.** The default is "keep for the life of the session plus a debug window" — typically 7 to 30 days for support and observability. Anything longer is a policy decision; anything shorter is a debugability incident waiting to happen. The window must be a config value, not a hard-coded constant, so legal and privacy can move it without a release.

**Deletion contract.** A "delete this session" action must purge the message store, the tool-result cache, and any derived summarisations. It must not silently leave copies in a "deleted sessions" ledger; if regulatory retention requires you to keep an audit trail, keep only the metadata (session ID, user ID, deletion timestamp) not the content.

**What the applied engineer owns here.** The session boundary (when does one end and the next begin?), the checkpoint schema (what's in the state?), the summarisation strategy when the session exceeds context, and the "delete session" endpoint.

### Layer 2 — Per-user memory

**Definition.** Facts the agent knows about *this user* that persist across sessions. Names, preferences, prior projects, team affiliations, learned constraints ("prefers metric units"), and so on. Distinct from Layer 3 in that per-user memory is *scoped to the user's account* and is not derivable from a shared corpus.

**Purpose.** The user experiences continuity. They do not need to re-explain their team, their timezone, or their preferred coding style every session. This is the difference between "an agent" and "*my* agent" for the user.

**Storage.** A per-user store (relational, document, or vector, depending on retrieval shape) keyed by user ID. The store is subject to the user's data rights: read (they can see what's there), write (they can add), edit (they can correct), delete (they can remove either specific items or the whole store), and export (they can take a copy).

**Retention.** Indefinite by default (the user wants continuity), but the schema *per fact* should carry an explicit `source`, `first_seen_at`, `last_used_at`, and (for anything sensitive) `sensitivity`. Facts derived from a session should be marked as such; facts asserted directly by the user carry more weight than facts inferred by the model.

**Deletion contract.** "Delete this memory" removes the fact. "Delete my account" removes the whole store. Both operations must complete within a bounded, published SLA (30 days is the GDPR ceiling for a data-subject deletion request; most consumer products publish a much shorter SLA, and the OpenAI, Anthropic, and Google consumer product terms name theirs — see [resources.md](resources.md)).

**What the applied engineer owns here.** The write-to-memory decision surface (when does the agent decide "this is worth remembering"?), the extraction pipeline (what shape is the stored fact?), the read-from-memory retrieval (what facts get injected on each turn?), the review UI (how does the user see and edit their memory?), and the deletion pipeline.

### Layer 3 — Long-term memory (workspace / shared / corpus)

**Definition.** Facts that persist beyond a single user, either because they belong to a shared workspace (a team's Notion, a company's CRM), a shared product state (a support agent's knowledge of policy X), or a curated corpus.

**Purpose.** Continuity at the *workspace* level, not the user level. Two users of the same workspace should see the same facts. A support agent talking to two customers should apply the same policy.

**Storage.** Same substrates as Layer 2, but keyed by workspace / tenant / corpus ID with an ACL layer that maps individual users to their permitted subsets. The permissions model is not optional: a shared memory store that leaks facts a user shouldn't see is a data-exfiltration incident, and shared memory is the single most common source of such incidents in retrieval-with-actions surfaces (Shape C from [chapter 01](01-agent-shape-for-a-real-product-surface.md)).

**Retention.** Governed by the workspace's contract, not the user's. When a workspace is deleted, everything in Layer 3 for that workspace is deleted. When a user is deleted from a workspace, their *personal* Layer 2 memory is deleted; the Layer 3 memory the workspace still holds may reference their prior activity only if the workspace's contract permits it.

**Deletion contract.** A workspace delete is a cascading operation. A per-fact delete is a workspace-admin action, not an end-user action. Both must be logged; both must be reversible within a published grace period (30 days is common) before permanent purge.

**What the applied engineer owns here.** The tenancy boundary, the ACL enforcement (see the permissions retrieval discussion in mod-103), the "what belongs in shared memory vs. per-user memory" policy, and the workspace-admin surface for deletion / export.

### The contract underneath all three: PII, minimisation, and rights

Every layer above sits underneath a common contract that the applied engineer, not the peer role, owns:

- **Data-class awareness.** Facts stored in memory carry a `sensitivity` tag: `public`, `internal`, `pii`, `special-category` (health, financial, biometric, immigration status, and the other GDPR Article 9 categories). The tag is set at write time, indexed at read time, and drives the retention window and the access-control decisions.
- **Minimisation.** The default is "do not store what you do not need." An agent that scrapes free-text user messages into a `notes` blob is storing everything the user has ever said, including sensitive facts they never intended to persist. Extract structured facts with typed schemas; do not persist raw messages by default.
- **The four rights.** Access ("show me what you know"), rectification ("this is wrong, fix it"), erasure ("delete this"), portability ("give me a copy in a machine-readable format"). These are named in GDPR Articles 15, 16, 17, and 20; they are named in the CCPA / CPRA and in the LGPD; and even where no regulation reaches your surface, they are the honest floor of the user contract.
- **Auditability.** Every memory write is logged (who / what / when / why); every read that returns a sensitive fact is logged. The log is what lets you answer "why did the agent bring up my medication?" in an incident review.

Without this contract, the three-layer split is decorative. With it, the layers hold up under the questions your compliance, security, and legal teams will actually ask.

## A worked schema for per-user memory

The most contested layer in practice is Layer 2. Storing "your name is Priya" is fine; storing "your project has slipped and you're stressed about it" is a different kind of fact. Below is a schema shape that has held up in review across several product-agent designs. It's not the only schema; it is a *disciplined* schema, and the discipline is what matters.

```python
# memory/schema.py — a per-user memory record.
from datetime import datetime
from typing import Literal, Any
from pydantic import BaseModel, Field

Sensitivity = Literal["public", "internal", "pii", "special_category"]
Source      = Literal["user_asserted", "model_inferred", "tool_result", "admin_set"]

class MemoryFact(BaseModel):
    model_config = {"extra": "forbid"}

    fact_id:        str
    user_id:        str
    workspace_id:   str | None      # None = strictly per-user (Layer 2)
    key:            str             # e.g. "preferred_timezone", "team.name"
    value:          Any             # structured, not raw text
    source:         Source
    sensitivity:    Sensitivity
    confidence:     float = Field(ge=0.0, le=1.0)
    first_seen_at:  datetime
    last_used_at:   datetime
    retention_end:  datetime | None # None = retained indefinitely (subject to erasure)
    provenance:     dict[str, Any]  # session_id, tool_call_id, message anchor
```

Six choices to notice.

- **`user_id` and `workspace_id` together.** A fact known about a user *inside a workspace* (they're an admin here, they've been onboarded) is not the same fact as one known about the user across workspaces (their timezone). Splitting the key makes the ACL enforceable.
- **`key` is a controlled vocabulary.** Not free-text. The set of keys is versioned, reviewed, and mapped to explicit sensitivity classes. A new `key` needs a schema PR, not a `model.infer_new_key()` call at runtime.
- **`source` and `confidence`.** A `user_asserted` fact of confidence 1.0 is more durable than a `model_inferred` fact of confidence 0.6, and the retrieval layer weighs them accordingly. Model-inferred facts have expiry defaults; user-asserted facts do not.
- **`sensitivity`.** Drives which retrievers can read the fact, which prompts can include it, and how long it may be retained. A `special_category` fact never rides on a default system prompt; it must be explicitly requested by a policy-gated retriever.
- **`retention_end`.** Enforced by a background job. A fact whose end is past is deleted, not "soft-deleted with a `deleted_at`." Soft delete is not delete, and regulators know this.
- **`provenance`.** For every fact, you can point at the message or tool call that produced it. This is what makes rectification legible ("we learned this from your message on 2024-06-12; here it is") and what makes incident review possible.

An extractor that populates this schema is a typed, evaluatable module — not "just call the model with an extraction prompt." It is, per mod-102, a structured-output pipeline with its own eval fixture: given a session, does it extract the intended facts (recall) without inventing false ones (precision)?

## Retrieval on memory: not "always inject everything"

The temptation once you have a memory store is to inject everything relevant on every turn. This is wrong for four reasons: cost (tokens), latency (each fact fetched), safety (a sensitive fact injected into a context that will be logged externally), and user surprise (the agent volunteers a fact the user forgot they'd asserted).

The pattern that works:

- **Per-turn retrieval, not per-session load.** Retrieve the k facts relevant to the current message on each turn, the same way retrieval works over documents (mod-103). Memory retrieval is a *retriever*, not a *prelude*.
- **A separate retriever per sensitivity class.** The default retriever pulls `public` and `internal` facts. Pulling `pii` requires the tool or the user's request to name the reason. Pulling `special_category` requires an explicit policy gate.
- **A rate limit on volunteering.** Even facts the retriever surfaces should not be *volunteered* into every reply. The system prompt names when to bring up a stored fact ("when the user asks 'what did I say about X' or when it materially affects the answer") and when not to.
- **A visible-to-user "why did the agent know this?" affordance.** If the user asks, the surface names the source and the timestamp. This is what makes the feature legible instead of creepy.

The failure mode this discipline prevents is the one every "agent that remembers you" surface eventually hits: the agent recites something the user said months ago, out of context, and the user's reaction is "delete everything." That reaction is rational. The design that avoids it is per-turn retrieval, sensitivity-gated access, and legible provenance.

## The deletion / export contract in code

Building a memory system without a deletion pipeline is like building a bank vault without a door. Deletion is not a nice-to-have. Two pieces are load-bearing.

### Deletion, per user

```python
# memory/deletion.py — the "delete user" pipeline.
async def delete_user(user_id: str, reason: str) -> DeletionReceipt:
    receipt = DeletionReceipt.begin(user_id=user_id, reason=reason)

    # 1. Session layer — checkpoints, tool-result caches, message logs.
    async for session in sessions.for_user(user_id):
        await sessions.purge(session.id)
        receipt.record("session", session.id)

    # 2. Per-user memory (Layer 2). Cascade all facts.
    async for fact in memory.facts_for_user(user_id):
        await memory.purge_fact(fact.fact_id)
        receipt.record("fact", fact.fact_id)

    # 3. Workspace memory (Layer 3). Do NOT delete the workspace's facts;
    #    do delete provenance links that name this user.
    async for fact in memory.workspace_facts_referencing(user_id):
        await memory.strip_user_reference(fact.fact_id, user_id)
        receipt.record("stripped_reference", fact.fact_id)

    # 4. Downstream caches (embeddings, tool audit logs, model-provider caches).
    await embeddings.purge_by_user(user_id)
    await audit_log.tombstone_content(user_id, keep_metadata=True)
    await provider_cache.purge_by_user(user_id)

    return receipt.finalise()
```

Three properties to notice.

- **The steps are ordered and idempotent.** A retry that finds no session for a user still succeeds. Idempotency here is the same discipline as tool idempotency (mod-104).
- **Workspace memory is not blindly deleted.** A workspace's facts (a support policy, a team's project) do not belong to a single user, and the compliance contract with the workspace does not permit a single user to erase them. What can be stripped is the *provenance links* that point at that user. This is often the hardest legal / product conversation to have; do it once, at design time, not in the middle of an incident.
- **Downstream systems are named.** Embedding indices, audit logs, and provider-side caches all hold data. A "delete user" that leaves the embedding index full of vectors derived from the user's messages is not a delete. Anthropic's, OpenAI's, and Google's data-usage policies name their own retention windows; your pipeline has to interact with those windows honestly.

### Export, per user

Export is the shape of the read pipeline turned inside out. A machine-readable dump (JSON or CSV) of every session, every fact, every provenance link. Two rules:

- **The export names the same schema the app uses.** No hidden internal fields, no "internal metadata not shown." If a field is in the export, the user can act on it.
- **The export includes the *history* of a fact.** If a fact was updated three times, the export shows the three assertions with their timestamps. This is what "rectification" (Article 16) actually needs to be enforceable — a user must be able to see when their data changed.

## Provider-side memory features and the "who owns retention" question

Providers ship memory features of their own. ChatGPT's "Memory" surface, Anthropic's per-project memory in Claude Projects, Gemini's app-level memory, and the tenant-level "memory" features of MCP hosts each store data on your users' behalf on infrastructure you do not fully control.

Three questions to answer for every provider-side memory feature you enable:

1. **Where does the data live, and under whose terms?** The provider's storage, their retention, their subprocessors. If you are subject to GDPR, HIPAA, or SOC 2, this is a formal question with a formal answer.
2. **Who can trigger a deletion?** If only the provider's UI can trigger deletion, and your product cannot, you have taken on a compliance obligation you cannot fulfil.
3. **What does the provider index on?** A memory feature that indexes on user identifier alone means every session's data can be surfaced across product tenants unless you scope the identifier. Session-scoping identifiers is a design choice; the default is usually wrong.

The pragmatic pattern for a product that must own compliance: **turn provider-side memory off by default**, store memory in your own layer with the schema and contract above, and inject the relevant slice into the provider's prompt on each turn. You give up the "the provider remembers" feature; you keep the deletion / export contract intact.

## Anti-patterns

Six failures at the memory layer, all of which show up in real product post-mortems.

1. **The "put the last N turns in the prompt" memory.** No layering, no persistence, no user-visible surface, no deletion contract. The instant your surface has more than one session per user, this is broken.
2. **The unstructured `notes` blob.** A single free-text field per user that the agent appends to. The user cannot see it, cannot correct it, cannot search it, and neither can you. Every audit will ask "what does the agent know about me" and there will be no answer.
3. **Memory writes without a policy.** Anything the model deems interesting becomes a fact. Sensitive assertions ("I'm going through a divorce") get persisted alongside preferences ("I like dark mode"), with the same retention. The extraction pipeline needs sensitivity classification at write time, not after the fact.
4. **Soft delete instead of delete.** Rows tombstoned with `deleted_at=now()` are not deleted. Regulators, security auditors, and the users themselves treat this as non-compliance. Delete means the row is gone; the audit log records the deletion, not the content.
5. **No workspace / user boundary.** A shared workspace memory that includes user-personal facts. Removing a user requires deleting workspace facts; declining to remove them leaves the user's data in the workspace's memory. Split the layers at design time.
6. **The provider-side memory taken as free lunch.** You enable the provider's memory, feel productive, and don't notice that the deletion pipeline you owe your users only touches your own store. When a deletion request comes in, half the data is somewhere you can't purge on demand.

## Summary

- Product-agent memory has three layers — session, per-user, workspace — each with its own retention window, storage substrate, and deletion contract. Design each layer explicitly; do not conflate them into a generic "memory."
- Underneath all three layers sits a contract the applied engineer owns: data-class awareness, minimisation, the four data-subject rights (access, rectification, erasure, portability), and auditability. Design for the rights before you design the schema.
- A disciplined per-user memory record carries an explicit `key` from a controlled vocabulary, a `source`, a `sensitivity`, a `confidence`, retention timestamps, and provenance. Extract structured facts, not free text. Test the extractor as a structured-output pipeline (mod-102).
- Retrieve memory per turn, sensitivity-class-aware, with rate-limited volunteering and a visible-to-user "why did you know that?" affordance. Do not inject the store; retrieve from it.
- Ship the deletion and export pipelines as first-class code, not tickets. Delete means gone; export names the same schema the app uses; workspace boundaries are respected.
- Provider-side memory features are a shortcut with a compliance tax. Default them off if you must own the deletion contract; use them only where the tax is affordable.

[Chapter 03](03-human-in-the-loop-and-steering-ux.md) moves from what the agent remembers to what the user gets to see and steer: progress, cancel, undo, resume, hand-off, and the kill-switch that closes the loop.
