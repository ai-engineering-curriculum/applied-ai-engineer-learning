# exercise-01: Feature Shape Decision Rubric

**Estimated effort:** 2 hours

## Objective

Given three real-shaped product specs, apply the seven-shape catalogue from [chapter 01](../01-choosing-the-feature-shape.md) and produce a defensible feature-shape decision for each — including the explicit rejection of the six shapes you did not pick.

## Learning outcomes

By the end you should be able to:

- Read a spec fragment (as a PM or a founder would write it) and name the surface, the user, the artefact, and the success signal.
- Walk the seven-shape catalogue in order and score each shape as **fits**, **partial**, or **rejects** with a one-line reason.
- Name blends explicitly (e.g. "retrieval-with-citations behind a co-pilot surface") rather than leaving the ambiguity in prose.
- State the consequence-of-being-wrong signal you would monitor to know if the shape choice needs revisiting.

## Prerequisites

- You've read [chapter 01](../01-choosing-the-feature-shape.md).
- You've read a real product spec once in your life. If not, look at any well-known open-source project's roadmap or feature RFC before starting.

## Problem statement

You are the applied-AI engineer on a mid-sized SaaS team. Three fresh specs land in your queue in the same week. Each is one paragraph long. For each, produce a feature-shape decision record.

### Spec A — Sales research assistant

> "Our AEs spend the first 30 minutes of a call prep researching the prospect: reading the account brief, cross-referencing recent news, pulling in the last three ticket threads. We want to cut that to 5 minutes. Ideally, the AE opens the account page in our CRM, and gets a compact summary they can trust — with links to the raw material — before they join the call."

### Spec B — Contract clause extraction

> "Legal ops receives ~200 vendor contracts per week. Our downstream tools need 14 specific fields extracted per contract: vendor name, effective date, termination clause presence, auto-renewal, governing law, and so on. Today it's manual. We want to run this at ingest time and expose the extracted fields to the vendor-management system."

### Spec C — Internal ops helper

> "Our finance / legal / IT teams field a constant stream of one-off questions from the rest of the company. Most are already answered somewhere in our internal wiki, but people don't find them. We want a Slack bot that answers policy questions from the wiki when it can, and *explicitly says it doesn't know* when it can't."

## Requirements

For each of the three specs, produce a decision record with these sections:

1. **Surface / user / artefact / success signal** (four lines).
2. **The seven-shape walk** — a table with one row per shape (chat, assistant, co-pilot, autonomous agent, one-shot generation, retrieval-with-citations, structured extraction) and columns `fits? (yes / partial / no)` and `reason`.
3. **Chosen shape** — one to two lines, blends named explicitly (e.g. "structured extraction as the primary shape, wrapped in a one-shot generation for the human-readable summary field").
4. **Rejected shapes** — one line per rejected shape (one line per row of the walk that is not the chosen shape or a named blend component).
5. **Consequence-of-being-wrong signal** — the one product-visible signal you would monitor that would tell you the shape choice needs to be revisited.

## Starter guidance

- Start each spec with the artefact question — *what leaves the model?* Prose? A form? A tool call? A summary next to citations? That answer eliminates three or four shapes immediately.
- Watch for the "assistant vs co-pilot" trap. If the primary product action is *producing* an artefact, it's a co-pilot. If the primary action is answering a user's question about an artefact they already have, it's an assistant. Both being "sidebars in a product" doesn't make them the same.
- On the retrieval-with-citations shape, decide what the surface does when the retrieval turns up nothing sufficient. If the answer is "the surface must say 'I don't have evidence for that'" (Spec C), retrieval-with-citations is likely the shape.
- On the autonomous agent shape, the test is whether the *user has consented to delegation* and whether the actions the agent would take are reversible. If either is no, reject the shape.

## Acceptance criteria

Your submission is defensible if a reviewer can answer **yes** to each of the following, on each of the three specs:

- [ ] The chosen shape is one of the seven (or a named blend of them). "Agentic AI feature" is not a shape.
- [ ] Every one of the seven shapes appears in the walk with a `fits?` value and a reason.
- [ ] The rejection of each non-chosen shape is stated in one sentence with a reason grounded in the spec, not in "not enough time" or "seems worse."
- [ ] The blend, if any, names the primary shape and the component shape(s) — not "combination of the above."
- [ ] The consequence-of-being-wrong signal is a specific observation, not a vibe (e.g. "acceptance rate < 30 % on the drafted-summary card" not "users seem unhappy").
- [ ] Total length per spec ≤ ~1 page of markdown.

## Stretch goals

Pick any two of the following:

- **Interview shape.** Rewrite one of your decisions as a 5-minute whiteboard answer to the interview question "how would you design this feature?" — no notes, one page.
- **Adversarial reviewer.** For one spec, write a 5-line "second reviewer" note that disagrees with your shape choice and argues for a different one. Then defend or update your choice.
- **Blend visibility.** Draw the actual surface UX for Spec A as a wireframe (paper sketch or plain markdown ASCII art is fine). The point is that the shape reads off the wireframe without needing prose.
- **Fourth spec.** Take any real feature you use daily (Cursor's Composer, GitHub Copilot Chat, Notion AI, Perplexity's answer surface, Anthropic's Claude Projects, ChatGPT's browsing) and produce the same decision record for it *as if you were pitching it* — inferring the shape from the observed surface.

## Deliverable

A single markdown file per spec (or one file with three sections), submitted to the paired solutions repo (`applied-ai-engineer-solutions`) under `mod-101/exercise-01/`.
