# exercise-01: Gateway Architecture Trade-Off Note

**Estimated effort:** 2 hours

## Objective

Produce the load-bearing artefact of chapter 01: a one-page, per-surface **gateway architecture trade-off note** that names the decision, ranks the four canonical shapes (LiteLLM / OpenRouter / Portkey / hyperscaler-native) on the six trade-off axes for *your* surface, and writes down the revisit trigger. The exercise is not "install a gateway." It is "make a defensible written decision that survives the next reorg."

## Learning outcomes

By the end you should be able to:

- Distinguish the four canonical gateway shapes by what each optimises for (control, breadth, policy density, residency).
- Rank a shape against a specific surface's constraints on the six trade-off axes (control, blast radius / residency, model breadth, feature velocity, unit economics, operational load).
- Name a legitimate *composition* (e.g. hyperscaler for prod + OpenRouter for experimentation) when a single shape does not fit — and justify it.
- Write a **revisit trigger** with a concrete condition that will re-open the decision, so the pick is not a fossil in two years.

## Prerequisites

- You have read [chapter 01](../01-gateway-architecture-and-the-trade-off-note.md).
- Access to at least one surface's production traffic profile (either your day-job's or the scaffold surface in the paired `applied-ai-engineer-solutions/mod-108/scaffold/` — see the module's solutions repo).
- Familiarity with at least one of the four gateway shapes at the "read the docs, run the quickstart" level.

## Problem statement

Your team currently calls provider SDKs directly from three features. Retry logic is copy-pasted, billing is one lump, and the last incident (a provider region 5xx) took the surface down because there was no fallback path. Leadership has asked you to "just pick a gateway." Your job is to *not* just pick one — your job is to produce the artefact that makes the pick defensible, revisitable, and readable to a peer six months from now.

In two hours, ship the trade-off note. **Do not deploy a gateway** as part of this exercise (chapter 02 and later exercises will do that with the pick in hand); the deliverable here is the written decision.

## Requirements

### Deliverable 1 — Surface characterisation

Under `docs/gateway-decision.md`, a **Surface** section that names, in concrete terms:

- The surface (feature or product area) the decision is for.
- The current volume: **peak QPS, average QPS**, and the direction of the trend (growing / flat / declining).
- The **model set currently in use** and the model set you expect to want in the next two quarters.
- The **residency posture** — do prompts / completions have a hard region constraint? A hard "must-not-leave-our-account" constraint? Cite the source of the constraint (compliance, customer contract, none).
- The **on-call posture** — is there an on-call rotation that can run a self-hosted service, or is the team small enough that a managed offering is strongly preferred?
- The **cost floor** — what does the current provider spend look like, and what markup would be disqualifying?

If any of these is genuinely unknown, name it as unknown and note who owns finding out. Do not paper over unknowns — the trade-off note is only defensible against a fact set you can name.

### Deliverable 2 — The trade-off matrix

In the same `docs/gateway-decision.md`, a **Trade-off matrix** section. The full table from [chapter 01](../01-gateway-architecture-and-the-trade-off-note.md), filled in for *your* surface — not in the abstract. Every cell is one of: `High`, `Med`, `Low`, or `N/A (constraint)` with a one-sentence justification underneath the table for anything not obvious.

The six axes: control, blast radius / residency, model breadth, feature velocity, unit economics at your traffic profile, operational load.

The four shapes: LiteLLM, OpenRouter, Portkey (with a note on whether you would run managed or self-hosted), hyperscaler-native (name which — Bedrock / Vertex / Foundry).

### Deliverable 3 — The decision and the "why not"

A **Decision** section: one sentence — "We are shipping shape X (or composition X + Y) for surface Z."

A **Why not the alternatives** section: one sentence per shape you did not pick, naming *the specific axis* that ruled it out for your surface. Not generic reasons — surface-specific ones.

### Deliverable 4 — The revisit trigger

A **Revisit trigger** section: at least two concrete conditions that would cause you to re-open the decision. Conditions must be *observable* — a specific QPS threshold, a specific residency change, a specific model availability change, a specific cost breach. "If things change" is not a trigger.

### Deliverable 5 — The peer-team dependencies

A **Peer-team dependencies** section that names who owns confirming any constraint the note relies on:

- Who signs off on the residency line (usually `privacy-engineer` / `data-security-engineer`).
- Who owns the on-call rotation for any self-hosted component (usually `sre` / `platform`).
- Who owns the cost sign-off (usually finance or an engineering director).

Empty rows are legitimate — write `n/a` and why.

## Starter guidance

- **Start from the constraints, not from the shapes.** If your surface has a hard "prompts must stay in eu-west-1" constraint, three of the four shapes are already ruled out and the exercise is much shorter. Constraint first, matrix second.
- **Do not fill in the matrix from vendor marketing pages.** Every gateway's marketing site claims to be strong on every axis. Ground the ranking in *your* team's ability to operate the shape — a "High" for feature velocity on a self-hosted OSS proxy is only true if your team has the SRE muscle to iterate on it.
- **Time-box the alternatives.** Spend no more than 15 minutes reading each of the four vendor docs. The trade-off is about your surface, not about becoming a gateway expert.
- **Be explicit about compositions.** If your honest answer is "hyperscaler for prod, OpenRouter for the experimentation surface," write that as the decision. Naming compositions up front is much cheaper than discovering the accidental one later.
- **Write the revisit trigger from the constraint side.** The constraint most likely to shift for your surface (residency, model availability, cost, volume) is where the trigger belongs.

## Acceptance criteria

- [ ] `docs/gateway-decision.md` exists, is one page (± 20%), and follows the section shape in [chapter 01](../01-gateway-architecture-and-the-trade-off-note.md).
- [ ] The **Surface** section names volume, model set, residency posture, on-call posture, and cost floor — with unknowns explicitly labelled.
- [ ] The **Trade-off matrix** has all four shapes × six axes filled in with cell-level justifications where the ranking is not obvious.
- [ ] The **Decision** is one sentence naming the shape (or composition).
- [ ] The **Why not the alternatives** section names, per rejected shape, the specific axis that ruled it out for your surface.
- [ ] The **Revisit trigger** section names at least two *observable* conditions.
- [ ] The **Peer-team dependencies** section names owners for every constraint the note relies on.
- [ ] The note is checked in and the commit message names the decision, not just "add gateway doc."
- [ ] The note carries a `date:` field and a `next-review:` field.

## Stretch goals

Pick any one:

- **Cost model.** Attach a `cost-model.md` with a back-of-envelope monthly cost estimate under each shape at your projected volume for the next two quarters. Show your assumptions; the goal is a defensible band, not a precise number.
- **A pilot spike.** Deploy the picked shape as a *single* route serving your lowest-stakes feature. Wire the retry / timeout / cost logging. Do not migrate other features. Prove the seam works before rolling it out.
- **Composition matrix.** Fill in the trade-off matrix for the two most-likely compositions (e.g. hyperscaler + OpenRouter, LiteLLM + hyperscaler) alongside the four single-shape rows. Compare the composition against the best-of-the-four.
- **Duplex / streaming audit.** Rank each shape's ability to proxy the transports your surface needs (WebSocket for realtime voice, SSE streaming for chat, standard HTTP for background). This is the specific check chapter 04 warns you not to skip.

## Deliverable

Submit as `mod-108/exercise-01/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/gateway-decision.md` with all sections filled.
- Optional stretch-goal artefacts (`cost-model.md`, `pilot/`, `composition-matrix.md`, `duplex-audit.md`).
- A short `README.md` naming the surface and pointing at the decision file.
