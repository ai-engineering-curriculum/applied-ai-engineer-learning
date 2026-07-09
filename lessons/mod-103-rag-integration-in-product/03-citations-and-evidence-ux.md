# Citations and Evidence UX

## What the user is actually doing

A user reading a retrieval-grounded answer is doing one of three things — often within the same session:

- **Consuming**: they trust the answer and want to *do the thing*. They skim the summary and move on.
- **Verifying**: they want to check the answer against the source before they act on it. They click a citation, land on the paragraph, and read it.
- **Correcting**: they suspect the answer is wrong or the source is missing. They want to see what the system had, and what it didn't.

A good citations UX serves all three without penalising any. A bad one either drowns the consumer in footnotes or hides the sources from the verifier. Getting this right is not a "polish" step at the end of the project. It is the primary difference between a demo and a shipped feature.

This chapter is the applied-AI engineer's job description for citations, evidence panels, "why this answer" surfaces, and — most importantly — the *no good evidence* path.

## Grounding is a UX contract, not a footnote

The temptation is to append a small "sources" section at the bottom of the answer. It hides three problems:

- **Sentence-level provenance is lost.** The reader can't tell which claim came from which source. Skim-verification breaks — you have to re-read every citation to check one sentence.
- **Non-grounded claims blend in.** A sentence the model made up on its own looks identical to a sentence it lifted from a passage. The reader has no visual signal.
- **"No good evidence" cannot be represented.** A footer-only pattern implies every claim in the body is grounded because there are citations at the end. When retrieval failed, the surface has no honest place to say so.

The pattern that survives contact with real users treats citations as **structural**, not decorative:

- Every load-bearing claim is anchored to a passage the model saw.
- The anchoring is legible in the layout — not "there is a citation somewhere on the page."
- The absence of anchoring is *also* legible — a claim that was not backed by evidence is either marked or was never emitted.
- The evidence panel lets the verifier get from the sentence in the answer to the sentence in the source in one click, in the same tab.

## The four surfaces

Not every product needs all four. Every product needs to name which ones it has and which it doesn't.

### 1. Inline citations

A superscript, bracketed number, or pill next to each grounded claim. Clicking it either scrolls the evidence panel to the passage or, in a compact surface (mobile, inline chat), pops a small quote preview.

The engineering:

- **Emit citations in the model's output as opaque IDs**, not URLs. The IDs are the `passage_id` values the retriever returned in the pack. The surface resolves them to titles + URLs + snippets at render time.
- **Refuse citations to IDs that weren't in the pack.** A pack is a list of retrieved passages; every citation must map to one of them. Reject at parse time — this catches "citation hallucination," where the model invents a plausible-looking source that was never retrieved. Increment a counter when it happens; a high rate is a signal to fix the prompt (or the model) before shipping.
- **Cap the number of citations per sentence.** Two or three, ordered by descending faithfulness of the sentence to each source. Long citation lists on every sentence indicate a prompt problem: the model is refusing to commit to a source and covering itself.

### 2. The evidence panel

A pane (sidebar, drawer, expanding tray) that shows the retrieved passages the answer was drawn from. Not the whole retrieval pack — the ones actually cited. The panel is the verifier's home.

Minimum content per passage:

- Document title.
- Section / heading, if available.
- The passage text, with the sentence(s) that were cited highlighted.
- Source URL (opens in a new tab).
- The freshness state for this source (from [chapter 02](02-freshness-coverage-permission-contract.md)) — usually a small icon + a `last updated` timestamp, red / amber / green.
- A "not what you were looking for?" link that files feedback into the eval loop.

The panel behaves differently when the freshness state is `degraded`: the timestamp goes red, and a short note explains why. When the user's ACL is the reason a source is missing, don't show a masked entry — that leaks the existence of forbidden documents. The evidence panel shows only what the user is entitled to see, and the surface's coverage warning (below) handles the rest.

### 3. "Why this answer" / the reasoning trace

Beyond citations, users increasingly want a legible chain: *what did the system decide, and why?* Especially on high-stakes surfaces — regulated answers, incident triage, medical / legal Q&A.

The applied-AI engineer's version is not the model's raw chain-of-thought (which is often a liability — noisy, sometimes wrong, sometimes leaks intermediate scratchpad). It is a **structured trace** the surface controls:

- **The query, as rewritten.** If [chapter 05](05-feature-side-levers.md)'s rewriter changed the user's query, show the rewrite. The user should be able to see "I asked X; the system searched for Y."
- **The retrieval summary.** "8 passages considered from 3 sources; 4 cited." Not the raw scores.
- **The classification decision.** If the query was gated ("this is a policy question, so we retrieved from the policy corpus and skipped the tickets corpus"), name that.
- **The evidence used per claim.** This is the inline-citations layer, restated as a table for reviewers.
- **The confidence signal.** A user-facing bucketed confidence (Low / Medium / High) tied to a measurable input — average faithfulness of cited passages, or a calibrated recall estimate. Never a raw model-produced number that varies across generations.

Two failure modes to design against:

- **Reasoning-trace theatre.** A pretty accordion that says "I searched, I found, I answered" without exposing anything falsifiable. If the trace does not let the user notice a mistake, it is not a trace, it is decoration.
- **Reasoning-trace as spec leak.** Detailed system-prompt content, internal tool names, or corpus paths in the trace teach users what to poke at. Redact anything you wouldn't publish on a status page.

### 4. The coverage / no-evidence surface

Some queries return nothing usable — the retriever found no passages above threshold, or all candidates were suppressed by freshness / ACL / a source outage. The surface must say so, in a way the user can act on.

Bad patterns:

- **Emitting the answer anyway.** "According to our knowledge base…" followed by an answer that came from the model's parametric knowledge. This is the classic hallucination path and destroys the surface's trust story in one click.
- **A generic "I don't know."** The user learns nothing about *why* and cannot fix their query.
- **A silent empty state.** The panel shows nothing and the user assumes the feature is broken.

The pattern that actually helps:

- Say explicitly: **"I couldn't find evidence in *[sources checked]* that would answer this."** Name the sources — the coverage manifest gives you these strings.
- Suggest the next action: "Try rephrasing," "Check that this topic is covered by *[source]*," "File a request for this to be added." The suggestion is grounded in the corpus manifest, not invented.
- If the retriever *did* return borderline passages that were filtered out, offer them as "possible partial matches — not confident enough to draft an answer." The user can then decide whether to expand the query themselves.
- If ACL or freshness suppressed a source, say **that a source was unavailable**, without naming the specific forbidden documents. "One of your sources is currently degraded; results may be incomplete." The user knows something is off and knows to escalate.

The "no good evidence" surface is the ally of the eval loop in [chapter 04](04-retrieval-quality-instrumentation.md): it turns silent hallucination-shaped failures into legible refusals you can count.

## The engineering of grounded generation

The UX above requires the prompt to actually produce sentence-level citations against a fixed pack of retrieved passages. Two implementation shapes are common on the applied-AI side; both work; know the trade-offs.

### Shape A: Post-hoc attribution

Generate the answer freely against the retrieved pack, then run a second pass — deterministic or LLM-based — that attributes each sentence back to the passages that support it. Simpler prompt; adds a step; the attribution can miss on subtle paraphrases.

### Shape B: In-line citation prompting

Instruct the model to emit citations as it writes, using a structured token: `Refunds are available within 30 days [P12].` The pack passages are numbered `[P1]…[Pk]`; the parser strips the tokens on the way to the surface and resolves them to passage IDs. Simpler pipeline; needs the model to actually cooperate; benefits from a schema (mod-102's structured-outputs discipline) that enforces citation shape.

A minimal structured-output schema that composes with either shape:

```python
from pydantic import BaseModel, Field

class Sentence(BaseModel):
    text: str
    supported_by: list[str] = Field(
        default_factory=list,
        description="passage_ids from the retrieved pack, or [] for chit-chat sentences",
    )

class GroundedAnswer(BaseModel):
    sentences: list[Sentence]
    unsupported_reason: str | None = Field(
        default=None,
        description="If the answer is 'no good evidence', this holds the human-visible reason.",
    )
```

At render time the surface iterates `sentences`, glues them together, and renders each `supported_by` list as inline citations. A sentence with an empty `supported_by` is either a linking phrase ("As a result,") — allowed — or an unsupported claim — flagged and either rendered with a "no evidence" marker or dropped, depending on the surface's tolerance. The `unsupported_reason` field is the "no good evidence" branch above.

Whichever shape you pick, **validate every emitted `passage_id` against the pack** before you render. This is a one-line defence against citation hallucination and the counter you increment when it happens becomes a first-class metric ([chapter 04](04-retrieval-quality-instrumentation.md)).

## Confidence, without theatre

Users treat any surfaced confidence value as authoritative. A miscalibrated confidence — "High" on a wrong answer — actively harms trust. The applied-AI engineer's job is to surface confidence only when it's grounded in something measurable.

Two defensible signals:

- **Coverage of the claims by the pack.** "3 of 4 sentences supported by cited passages" → bucketed to High. "1 of 4" → Low. This is cheap to compute and hard to fake; it composes with the structured-output shape above.
- **Retrieved evidence strength.** The retriever's scores can be bucketed via a threshold the retriever team publishes for this surface. Not raw numbers (chapter 01's anti-pattern) — bucketed labels the retriever attaches.

Two undefensible signals to avoid:

- **A model-produced "confidence: 0.87" number.** Not calibrated, wildly variable across generations, means whatever the model felt like emitting.
- **Number of retrieved passages as confidence.** More passages ≠ better answer. It correlates with the surface's `k`, not with truth.

Whatever signal you use, **calibrate it against your gold set**. If your "High" bucket has 80 % correctness on the labelled gold and your "Low" bucket has 25 %, the label is doing real work. If both buckets score the same, the label is theatre — remove it until you have a signal that predicts.

## Attribution failure modes to design against

Six specific failure modes appear in real deployments. Design against each explicitly.

1. **Citation hallucination.** The model emits a citation to a passage that was not in the pack. Guard: reject unknown IDs at parse time; increment a counter; alert if it exceeds a threshold.
2. **Off-passage claims cited to a real passage.** The citation ID resolves, but the passage does not actually support the claim. Guard: sample a small % of live traffic through a faithfulness judge ([chapter 04](04-retrieval-quality-instrumentation.md)); ratio of "cited but unsupported" is a hard metric with a clear target.
3. **Paragraph-level citations for sentence-level claims.** The user clicks a citation and lands on a 500-word passage where the relevant sentence is buried. Guard: highlight the span in the evidence panel; if the retriever supports sub-passage offsets, use them.
4. **Duplicate citations on the same claim.** The model cites P3 and P7, which are two chunks of the same document. Guard: de-duplicate at the document level in the evidence panel; still keep both passage IDs internally for the eval.
5. **Citation to a document the user cannot access.** Should be impossible if [chapter 02](02-freshness-coverage-permission-contract.md)'s ACL contract is honoured, but check anyway: the surface must refuse to render a citation whose passage's ACL doesn't include the current user.
6. **Silent degradation on empty packs.** The retriever returned nothing; the model produced an answer anyway. Guard: if `pack.passages` is empty, the surface *must* take the "no good evidence" branch. Do not send an empty-pack prompt to the model and hope.

## A worked example: the policy Q&A surface

**Spec**: an internal-tools bot answers "what is our policy on X" for HR / IT / legal questions.

**Retrieval pack**: 6 passages from the current HR / IT / legal handbooks (per-tenant, ACL-scoped, per [chapter 02](02-freshness-coverage-permission-contract.md)).

**Prompt**: structured-output shape from above. The system instruction says: "Answer only from the retrieved passages. Every claim must cite a passage id from the pack. If no passage supports the claim, set `unsupported_reason` and produce no sentences."

**Surface**:

- Inline citations as small `[HR-14]` pills next to each grounded sentence.
- Evidence panel on the right: document title, section, the cited passage with the cited sentence highlighted, freshness stamp, source link.
- "Why this answer" collapsed by default: expands to show the (rewritten) query, the retrieval summary ("6 passages from 2 sources; 4 cited"), and the confidence bucket ("High: 4 of 4 sentences supported by cited passages").
- "No good evidence" branch: renders `unsupported_reason` verbatim + the sources checked + a "file a policy-gap ticket" link.

**What the surface refuses to do**:

- Cite a passage ID that was not in the pack.
- Render an answer when the pack was empty.
- Cite a source whose freshness state is `degraded` beyond the per-source suppression rule.
- Cite a passage whose ACL doesn't include the current user.
- Show a numeric confidence.

## Summary

- Citations are structural, not decorative. Sentence-level grounding is the difference between a demo and a shippable feature.
- Four surfaces cover the space: inline citations, the evidence panel, "why this answer," and the coverage / no-evidence surface. Name which ones you have and which you don't.
- The engineering follows a structured-output shape: emit sentence-plus-supporting-passage-IDs; validate every ID against the pack at parse time; the surface resolves IDs to titles + URLs + snippets at render time.
- Confidence is welcome, but only when it's grounded in something measurable and calibrated against a gold set. Numeric model-produced confidences are theatre.
- Six failure modes recur — citation hallucination, off-passage claims, paragraph-level citations, duplicates, ACL leakage, silent degradation on empty packs — and each has a specific guard.
- The "no good evidence" branch is not a nice-to-have. It is the alternative to hallucination and the ally of the eval loop.

Chapter 04 turns the guards above into concrete metrics — recall@k on a gold set, hit rate on live traffic, faithfulness and answer-relevancy sampling, and the RAG-triad readout per surface.
