# exercise-01: Retriever Contract With the Rag Engineer

**Estimated effort:** 3 hours

## Objective

Negotiate and write down the retriever contract for a specific product surface — freshness SLOs, coverage manifest, and ACL enforcement pattern — as a versioned document that both the applied-AI and `rag-engineer` peer tracks can commit to. Wire the contract's runtime checks into a thin retriever client the surface calls.

## Learning outcomes

By the end you should be able to:

- Draw the applied-AI / retriever boundary on a real feature and name what crosses it.
- Turn a "the retriever is fast and accurate" wish into per-source freshness SLOs, a coverage manifest with named exclusions, and one of the three ACL enforcement patterns.
- Design a retriever-client interface that requires `tenant_id`, `user_id`, and `surface_id`, and returns freshness / version metadata every call.
- Wire the contract's *what happens on breach* clauses into the surface (freshness gate, coverage gate, ACL fail-closed).
- Explain, when a stakeholder asks, why the pattern you picked is the one that survives contact with the Red Team.

## Prerequisites

- You have read [chapter 01](../01-consuming-a-retriever-you-do-not-own.md) and [chapter 02](../02-freshness-coverage-permission-contract.md).
- You have a mock retriever or a very small real one you can call (a `Retriever` class with an in-memory list of documents is enough — the contract exercise is about the interface, not the ANN index).
- You are comfortable writing YAML and Python (or TypeScript).

## Problem statement

You are the applied-AI engineer on a **B2B multi-tenant helpdesk co-pilot**. The retriever team is spinning up. They have a working prototype: BM25 + a hosted embedder + Qdrant, no reranker yet, with a simple `retrieve(query, top_k)` API and no ACL enforcement — they filter by tenant only if you pass `tenant_id` as a metadata filter, which is optional today. There are three sources:

- `helpcenter.acme.com` — public help articles, updated a few times a day, ~2,000 documents.
- `zendesk.tickets` — the tenant's own solved/closed tickets, updated on a webhook, ~10,000/tenant.
- `salesforce.knowledge` — internal-only reference articles, updated by the CS team, ~500/tenant.

Business context: two enterprise tenants are onboarding next quarter under signed data-processing agreements that specifically require **fail-closed ACL enforcement** and **≤ 60 s revocation propagation**. Your PM has committed to those numbers without checking with the retriever team.

The retriever team has ~4 weeks. Your job is not to build their retriever; your job is to **write the contract that governs their build** and to wire the surface so it degrades gracefully when the contract is stressed.

## Requirements

Produce four deliverables.

### Deliverable 1 — The contract document (`retriever-contract.yaml`)

A single YAML file, per the shape in [chapter 02](../02-freshness-coverage-permission-contract.md). Requirements:

- Every source lists a **freshness SLO** with p95 + p99 in seconds and a measurement clock explicitly named.
- Every source lists a **coverage** block with an inclusion rule (what's in), a list of known exclusions (what's out), and a target coverage stat.
- The **ACL** block names one of the three patterns from chapter 02 and justifies the choice in a comment (retriever-enforced is the answer for this exercise, but write the justification as if you were defending the choice in review).
- The **ACL revocation propagation SLO** meets or beats the 60 s number the PM committed to.
- Every SLO has a **`on_degraded`** row naming what the surface does on breach.
- Every SLO has an owner tag (`applied_ai:` / `rag:`).
- The document is versioned (`version: 1`) with a change log stub.

### Deliverable 2 — The retriever client interface

A file `retriever_client.py` (or `.ts`) that:

- Defines `RetrieveRequest`, `Passage`, `RetrieveResponse` per [chapter 01](../01-consuming-a-retriever-you-do-not-own.md).
- Makes `tenant_id`, `user_id`, and `surface_id` **required** — a call missing any of them raises at the client, before the network hop.
- Rejects unknown filter keys against a small `FILTER_SCHEMA` constant. The schema is versioned; passing a key not in the schema is a client-side error, not a retriever-side one.
- Returns, on every response, an `index_freshness` dict per-source and a `versions` dict (`embedding`, `index`, `reranker`, `contract_version`).
- Refuses to hard-code any numeric score threshold. If the surface wants to threshold, it looks up a bucket the retriever team publishes (mock this with a `SURFACE_SCORE_BUCKETS` dict).

### Deliverable 3 — The three surface gates

Add three small guards on the surface side and prove they fire correctly:

- **Freshness gate**: if any source cited in a hypothetical answer is in `degraded` state, add a `stale_source_warning: True` flag to the response the surface returns. Include a unit test with a stubbed `index_freshness` that exercises both `healthy` and `degraded` states.
- **Coverage gate**: if any source in the manifest reports coverage below its target, add a `low_coverage_warning: True` flag and identify the sources. Unit test with a stubbed manifest.
- **ACL fail-closed**: if the ACL check errors out or times out, the surface returns `no_evidence: True` with a "we couldn't verify permissions" reason and does **not** proceed to generation. Unit test the failure path.

### Deliverable 4 — A one-page review memo

A `review.md` you would take into a joint review with the retriever team, containing:

- One paragraph on why you picked the ACL pattern you did (given the enterprise commitments).
- A named list of what the surface refuses to reach into and why — the anti-patterns from [chapter 01](../01-consuming-a-retriever-you-do-not-own.md).
- The change-management proposal: how the contract version bumps, what constitutes a breaking change vs. a compatible one, and how the retriever team signals a change to the surface team.
- One question you would ask the retriever team where the eval readout (the joint gold set from [chapter 04](../04-retrieval-quality-instrumentation.md)) is the deciding evidence.

## Starter guidance

- **Start from the YAML.** It is tempting to open the client file first. Don't. The interface follows the contract. Draft the YAML, walk it through with a peer role-playing the retriever engineer, then design the interface to serve it.
- **Make the "on_degraded" row concrete.** "Show a warning" is not enough. Which warning, where, in what UX element? A gate the reader can't picture in the product is a gate that won't be built.
- **The ACL pattern for this exercise is retriever-enforced.** The other two patterns (post-retrieval filter, physical isolation) are defensible in different contexts; write the memo as if you had to defend rejecting them here. Consult [chapter 02](../02-freshness-coverage-permission-contract.md)'s ACL section for the trade-offs.
- **Refuse to design in numeric score thresholds.** If your instinct is `if p.score < 0.72`, stop. Look up a bucket. The bucket comes from the retriever team, keyed by `surface_id`.
- **The filter schema is a client-side validation, not a runtime check on the retriever.** The retriever will validate too, but the client catches early. This is the same pattern as Pydantic on structured outputs (mod-102 chapter 02).
- **Do not build a real vector store.** Any function that returns hard-coded passages is fine. The exercise is about the contract and the client, not the retrieval implementation.
- **Write the tests before the guards.** Each guard's unit test is one paragraph of setup + a one-line assertion; if the test is hard to write, the guard's semantics aren't clear yet.

## Acceptance criteria

- [ ] `retriever-contract.yaml` covers all three sources with per-source freshness SLOs (p95, p99, measurement clock) and per-source coverage (inclusion rule, known exclusions, target).
- [ ] The ACL block names a single pattern, meets the 60 s propagation number, and specifies a fail-closed failure mode.
- [ ] Every SLO has an `on_degraded` row that names a specific surface behaviour, not a generic banner.
- [ ] The retriever-client interface enforces required `tenant_id`, `user_id`, `surface_id` before the network hop.
- [ ] The client's `FILTER_SCHEMA` rejects an unknown filter key with a clear error at the client boundary.
- [ ] Every mock response includes both `index_freshness` and `versions` — the surface can log both without special-casing.
- [ ] No numeric score threshold is hard-coded in the surface. A `SURFACE_SCORE_BUCKETS` lookup mediates.
- [ ] Freshness-gate, coverage-gate, and ACL-fail-closed unit tests all pass; each exercises both the healthy and the degraded path.
- [ ] `review.md` names the rejected ACL patterns with one-line justifications each, and states a change-management proposal.

## Stretch goals

Pick any two:

- **A propagation-latency test.** Simulate an ACL revocation event and prove your surface stops returning that user's passages within the 60 s SLO. A `time.time()`-driven stub is enough.
- **A contract-drift CI check.** Write a CI script that diffs the current `retriever-contract.yaml` against `main` and, on any change, prints a summary of "what breaks for callers" — a new required filter key, a tightened SLO, an added ACL pattern. Fail the build unless a `# breaking-change:` block is present in the PR.
- **The Red Team memo.** Write a one-page memo (`redteam.md`) enumerating five ways your retriever contract could still leak — passages in logs, error messages that reveal existence, timing side-channels on ACL denials, etc. — and what the surface does about each.
- **Multi-surface reuse.** Extend the client so a second `surface_id` ("policy-qa") can call the same retriever with a different `k`, different filter schema, and different score buckets. Show that adding the second surface required zero changes to the first surface's code path.
- **A contract-change dry run.** Write a memo describing a specific breaking change (e.g., a new mandatory `principal_groups` field for ACL) and the deprecation window you would use, in a shape both teams could sign off on.

## Deliverable

Submit as `mod-103/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `retriever-contract.yaml`
- `retriever_client.py` (or `.ts`)
- `test_gates.py` (or equivalent)
- `review.md`
- Any stretch-goal artefacts named alongside.
