# exercise-02: Citations and Evidence UX

**Estimated effort:** 3 hours

## Objective

Design and implement the citations + evidence + "no good evidence" UX for a retrieval-grounded answer surface, end-to-end. Use a structured-output shape that enforces sentence-level grounding at parse time, wire the surface's guards, and demonstrate the six failure modes from [chapter 03](../03-citations-and-evidence-ux.md) fail loudly instead of silently.

## Learning outcomes

By the end you should be able to:

- Emit sentence-level grounded answers using a validated structured-output schema.
- Reject citation IDs that were not in the retrieved pack at parse time (not at render time).
- Render an evidence panel that lets a user get from claim to source in one click, with freshness and source metadata visible.
- Design a "no good evidence" branch that names what was checked and what was missing.
- Justify — with a rubric — a bucketed confidence signal rather than a raw model-produced number.

## Prerequisites

- You have read [chapter 03](../03-citations-and-evidence-ux.md).
- You are comfortable with Pydantic (or Zod / another schema validator) and can write a small React / htmx / Streamlit UI (any UI framework is fine; the visual richness is not the point).
- You have a small mock retriever that returns a fixed pack of 4–8 passages for any query, and a model provider you can call.

## Problem statement

Build the answer surface for an **internal policy Q&A bot** used by employees for HR / IT / legal questions. The bot answers "what is our policy on X" against a small corpus of policy documents. Users trust it at their peril — they are getting compliance-adjacent answers, and a hallucinated policy is a real risk.

Your surface must:

- Draft an answer that is **sentence-level grounded** to the retrieved passages.
- Show inline citations next to each grounded sentence.
- Render an evidence panel that highlights the cited span in each passage.
- Take the "no good evidence" branch when appropriate — either because the pack is empty, because the model tried to make ungrounded claims, or because coverage / freshness / ACL removed the sources that would have supported the answer.
- Never emit a substantive answer without at least one supported sentence and at least one legal citation.

You should assume the retriever client from exercise-01 is available and returns passages with `passage_id`, `doc_title`, `doc_url`, `text`, `updated_at`, and a per-source freshness state.

## Requirements

### Deliverable 1 — The structured-output schema

A Pydantic (or equivalent) schema for a `GroundedAnswer`. Requirements:

- `sentences: list[Sentence]`, where each `Sentence` has `text: str` and `supported_by: list[str]` (list of `passage_id`s from the pack).
- A distinguished linking-phrase case: an empty `supported_by` is allowed only for a small list of allowed connective phrases ("As a result,", "In summary,", etc.), and only if `sentence.is_linking is True`. Enforce this in a `@model_validator`.
- An `unsupported_reason: str | None` field that populates when the answer is the "no good evidence" branch.
- A validator that rejects the object if `sentences` is non-empty and `unsupported_reason` is also set.
- A validator that rejects the object if `sentences` is empty and `unsupported_reason` is `None` (the model produced nothing and gave no reason).

### Deliverable 2 — The parser + pack-check

A function `parse_and_validate(raw_output, pack)` that:

- Parses the model output into `GroundedAnswer`.
- Rejects any `passage_id` in any `supported_by` list that is not present in the retrieved pack. This is the citation-hallucination guard; log an entry to a counter every time it fires.
- Rejects any answer where a non-linking sentence has empty `supported_by`. This is the ungrounded-claim guard; log to a counter.
- On any rejection, either re-prompt the model with the specific violation (see mod-102 chapter 04 for the repair pattern) or drop the answer into the "no good evidence" branch. Pick one and document the choice.
- Returns a rendered payload for the UI: `{sentences: [...], citations: [...], evidence: [...], confidence_bucket: ..., no_good_evidence: ...}`.

### Deliverable 3 — The render layer

A minimal UI (Streamlit / small React page / htmx template — pick your poison) that renders:

- The answer text with inline citations rendered as pills / superscripts.
- An evidence panel on the right with:
  - Doc title, section (if any), and cited passage with the cited spans highlighted.
  - `last_updated` timestamp and a freshness badge (green / amber / red) driven by the freshness state.
  - A source link that opens in a new tab.
- A "Why this answer" expandable that shows the (possibly rewritten) query, the retrieval summary ("N passages considered from M sources; K cited"), and the confidence bucket.
- A distinct "no good evidence" layout when the branch fires — sources checked, an explanation, and a "file a policy-gap ticket" action stub.

The UI does not need to be pretty. It needs to be *legible enough that a reviewer can verify a claim in one click.*

### Deliverable 4 — The failure-mode demonstrations

A short `demos.md` that shows the surface handling each of the six failure modes from [chapter 03](../03-citations-and-evidence-ux.md) — screenshots or printed request/response pairs are both fine:

1. **Citation hallucination**: the model emits a passage ID not in the pack. The surface refuses to render it and takes the "no good evidence" branch (or repairs, per your choice in deliverable 2).
2. **Off-passage claim cited to a real passage**: the citation resolves but the passage does not actually support the claim. You do not need to auto-detect this in the exercise — instead, wire it into the online-eval hook (deliverable 5) so it is at least *countable*.
3. **Duplicate citations on the same claim** from two chunks of the same document — de-duplicated at the document level in the evidence panel.
4. **Citation to a document the current user cannot access** — the render refuses to show it; the surface either drops the citation and re-prompts or falls to the "no good evidence" branch.
5. **Silent degradation on empty packs** — an empty pack forces the "no good evidence" branch; the model is not called at all.
6. **Off-corpus answer** — the model tries to answer from parametric knowledge; the ungrounded-claim guard catches it.

Each demo should show the raw model output, the parser's decision, and the rendered UI state.

### Deliverable 5 — The counters and the confidence bucket

- A small `metrics.py` that exposes counters for `citation_hallucination_total`, `ungrounded_claim_total`, `no_good_evidence_total`, `cited_but_forbidden_total`, and `answers_rendered_total`.
- A `confidence_bucket(answer)` function that returns `low` / `medium` / `high` based on:
  - `cited_supported_ratio` = number of sentences with non-empty `supported_by` / total sentences.
  - The retriever's published score bucket for each cited passage (mock a `passage_score_bucket()` lookup).
- Document the bucket boundaries in `confidence.md` with the calibration you would run before promoting the bucket labels to the user-visible UI. You do not have to actually run the calibration in this exercise, but the plan should be defensible.

## Starter guidance

- **Prompt engineering matters here, but is not the primary lesson.** Use mod-102's structured-outputs discipline: emit JSON matching the schema, one shot per call, retry with a repair message on validation failure. The lesson is *what the schema forces the model to do*, not "how do I make the model emit JSON."
- **Do not resolve passage IDs in the model prompt.** Number the pack `P1..Pk` in the prompt, tell the model to cite by `P<n>`, and translate `P<n>` → the pack's `passage_id` at parse time. Never let the model see the URL, or it will fabricate URLs when it hallucinates a citation.
- **Cite in the sentence, not at the end.** The prompt instruction is: "Every sentence with a substantive claim must include `supported_by` referencing the passages that support it." A single trailing sources list is what [chapter 03](../03-citations-and-evidence-ux.md) warns against.
- **The "no good evidence" branch is a first-class output.** Model it as a valid `GroundedAnswer` where `sentences == []` and `unsupported_reason` is populated. Do not model it as a separate error path.
- **De-duplicate on `doc_id`, not `passage_id`.** In the evidence panel, if `P3` and `P7` are both chunks of the same document, the panel shows one entry with both spans highlighted; internally the surface keeps both IDs for the eval.
- **You can mock the ACL.** A `user_has_access(user_id, passage) -> bool` stub is enough. The point of the demo is that the render refuses the passage when the stub says no; production wiring is exercise-01's contract.
- **Freshness badge colours come from the freshness state in `index_freshness`.** No new metadata; consume what exercise-01's contract publishes.

## Acceptance criteria

- [ ] `GroundedAnswer` schema has the validators specified in deliverable 1 and rejects the two contradiction cases with clear errors.
- [ ] `parse_and_validate` rejects unknown citation IDs at parse time and increments a counter.
- [ ] `parse_and_validate` rejects ungrounded non-linking sentences and increments a counter.
- [ ] The rendered UI shows inline citations, an evidence panel with highlighted spans, a "Why this answer" expandable, and a distinct "no good evidence" state.
- [ ] The evidence panel de-duplicates passages that share a `doc_id`.
- [ ] The evidence panel refuses to render a citation the current user cannot access (mocked).
- [ ] The "no good evidence" state names the sources checked (from the manifest) and offers a next action.
- [ ] Empty-pack input takes the "no good evidence" branch without calling the model.
- [ ] The six failure-mode demos are recorded in `demos.md` with the raw output, parser decision, and rendered state.
- [ ] `confidence_bucket()` is deterministic given the same input, does not use a raw model-produced number, and its boundaries are documented with a calibration plan.

## Stretch goals

Pick any two:

- **Span offsets.** If the retriever provides sub-passage offsets, extend the schema so `supported_by` is a list of `(passage_id, start_char, end_char)` tuples; render the highlight against the actual span. State what breaks in the "sentence-level provenance" story if the model cites a passage but you can't localise the supporting span within it.
- **Live faithfulness spot-check.** Wire in an LLM-as-judge (a different model family from the generator) that scores 5 random rendered answers per day for cited-but-unsupported. Log the numbers and, on any hit, add the pair to a `redteam/` folder for gold-set candidacy.
- **Confidence calibration.** Actually run the calibration: label 100 rendered answers as correct / incorrect by hand and compute the accuracy per bucket. If your `high` bucket is not measurably better than `medium`, redesign the bucket function.
- **Voice mode / streaming.** Extend the render to support token-streaming answers where citations become clickable as soon as each sentence closes — but the ungrounded-claim guard still catches everything before it's shown. Discuss the trade-off between latency and post-hoc validation.
- **Adversarial prompts.** Ten queries designed to elicit each of the six failure modes (a query designed to trigger citation hallucination, a query designed to trigger off-passage claims, etc.). Show your guards catching each.

## Deliverable

Submit as `mod-103/exercise-02/` in the paired `applied-ai-engineer-solutions` repo:

- `schema.py` — `GroundedAnswer` and validators.
- `parser.py` — `parse_and_validate` + counter wiring.
- `render.py` or `app.py` — the UI.
- `metrics.py`, `confidence.py` — the counters and bucket function.
- `demos.md` — the six failure-mode demonstrations.
- `confidence.md` — the bucket-calibration plan.
- Any stretch-goal artefacts named alongside.
