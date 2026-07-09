# The Freshness / Coverage / Permission Contract

## Why a contract, not a spec

"The retriever is fast and accurate" is not a contract. It is a wish. Under pressure — a launch, an outage, a re-embedding rollout, a corpus migration — an unwritten wish loses to whatever the retriever team was already working on. The contract is the artefact that survives that pressure. It states, in numbers and in enforceable interfaces, what the surface can trust and what it must handle itself.

Three families of guarantees matter enough to write down. Everything else the retriever team can change without a heads-up.

- **Freshness**: how stale can an indexed document be before it stops counting as "current" for this surface?
- **Coverage**: which sources are in scope, and what fraction of each source is actually in the index?
- **Permission**: who is allowed to see which documents, and how is that enforced end-to-end?

This chapter is the applied-AI engineer's template for pinning all three, per surface, in a way both teams can commit to.

## Freshness

### The user-visible failure mode you're preventing

The failure mode isn't "the index is stale" — indices are always stale, by some amount, in some places. The failure mode is that the surface **presents a stale answer as if it were current**. A support agent drafts a refund reply from a policy that changed last week. A sales rep quotes a spec that was corrected two days ago. A compliance answer cites a superseded regulation.

The contract's job is to make that failure mode either impossible or visible.

### The two clocks

Every freshness contract runs on two clocks and you must be explicit about which one you're pinning:

- **Ingest lag**: source-of-truth → chunker → embedder → index. From "the document was published (or updated) in the source system" to "it is queryable from the vector store."
- **Refresh lag**: source-of-truth → *notification of an update*. Systems that push (webhooks, change data capture, event streams) have near-zero refresh lag; systems that poll have a bounded refresh lag equal to the polling interval.

Total staleness the surface sees = refresh lag + ingest lag. The retriever team is responsible for both; the surface is responsible for stating a bound on the sum.

### Writing the freshness SLO

A freshness SLO has five fields:

| Field | Example |
|---|---|
| **Source** | `helpcenter.acme.com`, `salesforce.knowledge`, `zendesk.tickets` |
| **Bound** | p95 ≤ 15 minutes, p99 ≤ 60 minutes |
| **Measurement clock** | source `updated_at` → passage queryable |
| **Population** | production traffic, all documents ingested in the trailing 24 h |
| **What happens on breach** | the surface renders a "may be out of date" banner *and* the surface's coverage panel shows the affected source with a red freshness state ([chapter 03](03-citations-and-evidence-ux.md)) |

Two things about the "what happens on breach" row are load-bearing. First, a breach is a **product event**, not a retriever-team paging event: the surface must degrade gracefully because you cannot promise the retriever team will fix freshness inside the user's session. Second, the surface must be able to *know* it's in breach — which means the retriever's response must expose per-source last-refresh timestamps and current lag estimates, not just an index-wide "last touched" number.

### The index-freshness manifest

The retriever returns, on every response, a small manifest keyed by source:

```json
{
  "index_freshness": {
    "helpcenter.acme.com": {
      "last_refresh_completed_at": "2026-07-09T13:52:14Z",
      "current_lag_p95_seconds": 640,
      "state": "healthy"
    },
    "zendesk.tickets": {
      "last_refresh_completed_at": "2026-07-09T14:03:07Z",
      "current_lag_p95_seconds": 90,
      "state": "healthy"
    },
    "salesforce.knowledge": {
      "last_refresh_completed_at": "2026-07-08T22:14:00Z",
      "current_lag_p95_seconds": 60000,
      "state": "degraded"
    }
  }
}
```

The surface can then answer three questions locally, without a Slack thread:

- "Do we mark this answer as possibly stale?" — yes, when any source cited in the answer is in `degraded` state.
- "Do we suppress this source entirely?" — yes, when a per-source policy says so (a compliance answer may refuse to use a source that's more than 60 minutes stale even if it's cited).
- "Do we render a coverage warning in the evidence panel?" — yes, when the user's query is likely to match a currently-degraded source.

### Per-document freshness — the harder version

Some surfaces need finer than per-source freshness. A single Zendesk ticket that was updated 30 seconds ago is more important than the source's average lag. The finer contract is: **each returned passage carries its own `updated_at`**, and the surface can enforce per-passage rules ("do not draft a policy answer from a passage older than 30 days if the corpus has fresher matches"). Not every retriever supports this; when it does, use it.

## Coverage

### Coverage is not "we indexed it"

"Yes, the KB is indexed" is the first answer, and it's rarely the whole answer. Real coverage questions look like:

- Which document types are indexed? Just the current versions, or the historical revisions too?
- Are internal-only articles indexed? Draft articles? Archived articles?
- Are non-English pages indexed and, if so, how are cross-language queries handled?
- Are attachments (PDFs, images with text) indexed?
- Is a document counted as "covered" if only the title made it in, or must the body have been chunked and embedded?

The contract has to answer these — or the surface will silently miss cases it looks like it should handle.

### The corpus manifest

The retriever publishes a **corpus manifest** for each `surface_id` — a versioned document that lists every source in scope and, per source:

- The specific slice indexed (a URL glob, a Salesforce report, a Zendesk brand ID).
- The doctypes included (`article`, `pdf`, `ticket`, `transcript`).
- Doctype-specific inclusion rules (`article.state != 'archived'`, `ticket.status in ['closed', 'solved']`).
- The refresh schedule and mechanism (push, poll every 5 minutes, nightly).
- The current coverage stat: `documents_in_source`, `documents_indexed`, and the % ratio. A source that reports 92 % coverage is telling you something specific.
- Known exclusions ("images-only PDFs are not OCR'd", "tickets before 2024 were not backfilled") — these matter because the surface's failure modes cluster around known exclusions.

Versioning the manifest lets the surface log the version alongside every retrieval and correlate an incident against a corpus change ("coverage of `helpcenter/api-reference` dropped from 96 % to 71 % when the manifest went from v14 to v15 last Tuesday").

### Surface-side coverage tests

The applied-AI engineer's job is to write coverage tests that exercise the manifest as a spec:

- **Presence tests**: for each surface, name 20–50 documents that must always be retrievable by a known query. Run this in CI against staging and on a schedule against production. A regression means the manifest is either wrong or the corpus is broken.
- **Freshness tests**: publish a canary document into each source with a timestamped body, poll the retriever for it, and record end-to-end lag. This validates the freshness SLO with a real reproducer.
- **Doctype tests**: a query that should surface a PDF, a query that should surface a ticket, a query that should surface a KB article. Confirms doctype-level coverage isn't silently degrading.

These tests belong to the applied-AI engineer because they are anchored to product cases. They live in the shared eval repo referenced in [chapter 01](01-consuming-a-retriever-you-do-not-own.md).

## Permission — the ACL contract

### What "ACL enforcement" means at the retriever

Naive versions of retrieval-augmented product features have been shipping with permission bugs since 2023. The pattern is: an internal search UI or a support chatbot ingests documents from many sources, and the retriever returns the top-k regardless of who's asking. The prompt then surfaces content that the specific user shouldn't be able to see — sometimes literally quoting it in the answer, sometimes just citing a URL that reveals its existence.

The contract's job is to make that impossible by construction, not by prompt discipline.

There are three defensible enforcement patterns, in order of preference.

#### Pattern A — Retriever-enforced ACL (preferred)

The retriever performs the ACL check as part of the query. Every request must pass `tenant_id` and `user_id` (and typically a groups list); every indexed document carries its ACL as document metadata; the retriever filters at query time so only allowed passages can be returned. The surface can't accidentally leak because it can't accidentally *receive* an unauthorised passage.

This is the pattern that survives contact with a Red Team. It also requires the retriever team to build a real ACL layer, so it takes the most work.

Consequences for the applied-AI engineer:

- The retriever's `RetrieveRequest` mandatorily includes `user_id` (or an equivalent principal). No optional field.
- The retriever's response includes an ACL-version stamp per passage or per index, so an incident can be correlated to an ACL rebuild.
- Deletions and access revocations propagate on a bounded SLO (e.g., "user removed from group ⇒ their retrieval no longer returns that group's docs within 60 seconds"). This is a real SLO with a real testing burden — write it down.

#### Pattern B — Post-retrieval filter (defensible)

The retriever returns candidates; the surface (or an ACL microservice adjacent to it) filters them against the current user's permissions before they reach the prompt. Works when the ACL system is authoritative in a separate place from the retriever and the retriever team can't feasibly re-implement it.

The dangers:

- The surface must **fail closed**: if the ACL service is unreachable, return no passages, do not fall through to unfiltered results. An outage in the ACL service must not become an information-disclosure incident.
- The retriever must not return the *text* of forbidden passages to the surface at all; only their IDs, so the surface can drop them before rendering. If forbidden passages leave the retriever process, they leave a memory-inspection or logging path that leaks them.
- The prompt must never see the forbidden passages. Filtering after prompt construction is not filtering.

#### Pattern C — Physical index isolation

Per-tenant (or per-security-domain) indices. Tenant A queries index A; tenant B queries index B. Simple to reason about; wasteful of storage; hard to scale beyond a few dozen tenants. Appropriate for high-security niches (regulated industries, on-prem installs) and for very small tenant counts.

Consequences:

- The retriever client must fail loud if a request comes in without a `tenant_id`.
- Cross-tenant queries must be an explicit second API with its own authorisation, not a "just pass a list of tenant IDs" flag on the same call.

### Failure modes the ACL contract must name

Regardless of which pattern you pick, the contract enumerates the failure modes and what the surface does in each:

- **Unknown user** — treat as unauthorised, return no passages. Never fall through to public content unless the contract explicitly declares a public-fallback surface.
- **ACL service unavailable** — fail closed. Log the outage, present a "we couldn't verify permissions" state to the user, do not draft an answer from an unfiltered retrieval.
- **ACL data stale** — bound the staleness. "ACL propagation ≤ 60 s p95" is a real SLO; without it you cannot promise revocations take effect in time.
- **Cross-tenant identifier collision** — namespaces on IDs. `doc_id` alone is not a global key.
- **Test / staging leakage into production** — separate indices, separate credentials, separate embedder namespaces if you're paranoid.
- **Logging leakage** — passage text logged to a shared trace store becomes a covert channel around the ACL. Structure your logging so passage text is either redacted or per-tenant-scoped.

## Composing the three: the contract document

The three contracts fit into a single document per surface, roughly two pages long. Keep it in the shared repo, versioned, reviewed by both teams.

```yaml
# retriever-contract.yaml — surface_id: helpdesk-reply-copilot
version: 3
owners:
  applied_ai: "@josh.f"
  rag: "@team-retrieval"

sources:
  - id: helpcenter.acme.com
    freshness_slo:
      p95_seconds: 900
      p99_seconds: 3600
      measurement: "source updated_at -> passage queryable"
    coverage:
      inclusion_rule: "state == 'published' AND locale in ['en']"
      known_exclusions: ["images without OCR", "beta articles"]
      last_reported_coverage: 0.96
    acl_pattern: retriever_enforced
    on_degraded: "surface shows 'may be out of date' banner; source usable"

  - id: zendesk.tickets
    freshness_slo:
      p95_seconds: 120
      p99_seconds: 600
      measurement: "ticket updated_at -> passage queryable"
    coverage:
      inclusion_rule: "status in ['solved', 'closed'] AND updated_at > 2024-01-01"
      known_exclusions: ["attachments not indexed"]
      last_reported_coverage: 0.99
    acl_pattern: retriever_enforced
    on_degraded: "surface suppresses source; drafts refuse to cite tickets"

acl:
  primary_pattern: retriever_enforced
  revocation_propagation_p95_seconds: 60
  failure_mode: fail_closed
  logging_policy: "passage text redacted at surface log boundary"

overall:
  p95_retriever_latency_ms: 400
  k_max: 8
  reranker_included: true
```

The document is boring on purpose. Boring means both teams can point at a line and settle a dispute in seconds instead of days.

## What the surface does on breach

None of the SLOs above matter unless the surface behaves differently when they're breached. Wire the following in:

- **A freshness gate.** If any source cited in an answer is in a `degraded` state, the surface adds a visible staleness marker to that citation ([chapter 03](03-citations-and-evidence-ux.md)).
- **A coverage gate.** If the user's query pattern-matches a source that is currently below its coverage floor, prepend a short banner ("some product data is unavailable right now").
- **An ACL fail-closed.** If the ACL check fails or the ACL service is unreachable, the surface refuses to draft the answer rather than draft it from an unfiltered retrieval.
- **A metric per gate.** Count each gate firing per surface per day. A gate that never fires is either dead code or dead detection; a gate that fires constantly is a contract that needs renegotiation, not a UX to hide.

## Anti-patterns

1. **The "trust me" retriever.** A retriever that promises freshness / coverage / ACL verbally and returns no metadata to prove it. The surface can't gate, can't degrade, can't correlate incidents. Fix: require the manifest and the freshness stamps in the response.
2. **The one-size SLO.** A single freshness bound across all sources. The tickets index needs seconds; the compliance PDF corpus is fine at 24 hours. Force per-source SLOs.
3. **Post-retrieval ACL that fails open.** When the ACL service is down, the surface renders passages anyway "so we don't break." That is not a UX degradation; that is a policy violation. Fail closed.
4. **ACL in the prompt.** "You are talking to a user who has access to tenants A and C; do not surface documents from tenants B or D." Prompts do not enforce permissions. This has been demonstrated to leak in adversarial evals every year since 2023. Fix: enforce at the retriever or in a filter between retriever and prompt.
5. **A coverage number as a marketing metric.** "97 % of your KB is indexed!" — great, which 3 % is missing, and does it correlate with what users actually ask? Force the manifest to name known exclusions and the surface to test presence on real queries.
6. **The manifest that never changes.** If the corpus manifest hasn't been updated in six months, it is stale documentation, not a contract. Wire it into the retriever's own release process so a corpus change is a manifest change.

## Summary

- The retriever contract has three parts: freshness, coverage, permission. Each is a per-surface, per-source document with numeric bounds and named breach behaviour.
- Freshness is two clocks (refresh + ingest) summed. Per-source SLOs, per-response manifest, per-passage `updated_at` where the retriever supports it.
- Coverage is a versioned corpus manifest listing inclusion rules, known exclusions, and current stats. The surface writes presence tests against it.
- ACL enforcement belongs at the retriever (preferred), a fail-closed post-retrieval filter (defensible), or physical index isolation (niche). Never in the prompt.
- Every SLO comes with a "what the surface does on breach" line. A gate without a UX response is a gate that isn't there.
- Six named anti-patterns collapse the contract: the "trust me" retriever, one-size SLOs, fail-open ACLs, ACL in the prompt, coverage-as-marketing, and manifests that don't change.

Chapter 03 turns the freshness and coverage state into UX — how the surface renders "here is what we found, and here is how confident we are that it's the right thing."
