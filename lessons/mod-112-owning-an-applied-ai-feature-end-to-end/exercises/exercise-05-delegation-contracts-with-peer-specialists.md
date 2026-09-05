# exercise-05: Delegation Contracts With Peer Specialists

**Estimated effort:** 2 hours

## Objective

Author the **delegation-contract inventory** for one real (or realistic) applied-AI feature: seven contracts, one per peer role the applied engineer routinely delegates to, each on the six-field shape from [chapter 05](../05-delegation-contracts-with-peer-specialists.md). Every contract is bounded in scope, names its inputs and outputs, sets a revisit cadence, and — where a peer counter-party is available — is signed by both parties. The deliverable is the *inventory*, not any single contract; the exercise proves you can name the entire delegation graph, spot the gaps, and reject vague scope statements before they cost an incident.

## Learning outcomes

By the end you should be able to:

- Name the seven peer roles the applied-AI engineer routinely delegates to and describe, in one sentence each, what depth those roles own that the applied engineer does not.
- Write a **bounded scope statement** for each hand-off — one that a reviewer can sign onto or push back on, not a category ("safety") the two sides read differently.
- Author both sides of the interface: the **applied-engineer inputs** the peer role needs to do their work, and the **peer-role outputs** the applied engineer needs back, each with a format and a cadence.
- Distinguish **what does not delegate** from what does, and use the applied-engineer's own-scope list from [chapter 05](../05-delegation-contracts-with-peer-specialists.md) as a check.
- Detect a **delegation gap** — a peer-role dependency the feature card cites without a matching contract on file — and either close the gap or add a "known limitation" in the card.
- Refresh a contract on cadence (or on an event trigger) and write the change log so the previous version's counter-party can see what changed.

## Prerequisites

- You have read [chapter 05](../05-delegation-contracts-with-peer-specialists.md), and ideally the peer-role sections in the other chapters that discuss which specialist owns which depth.
- The feature you used for [exercise-01](exercise-01-feature-charter-with-kill-criteria.md), [exercise-03](exercise-03-build-vs-buy-matrix-gateway-eval-agent.md), and [exercise-04](exercise-04-per-feature-model-card-slice.md). The delegation graph plugs the feature into the surrounding org; the same feature carries through.
- A short mental (or explicit) list of the peer specialists in your org, if you have them. If not, patterned counter-parties (`agentic-ai:$name`, `ai-risk-engineer:$name`, and so on) are acceptable; the exercise is about the contract shape, not about naming your specific colleagues.
- The feature card from [exercise-04](exercise-04-per-feature-model-card-slice.md), in particular §9 (known limitations) — the delegation-contract inventory is what §9 will reference.

## Problem statement

Your feature card is composed. Section 9 (known limitations) says "delegation dependencies: red-team programme (`ai-risk-engineer`), serving performance (`ai-infra-performance`), authz architecture (`ai-infra-security`)." The governance reviewer flips the card, follows the section 9 pointer, and asks: *"Show me the contracts."*

There are three possibilities. (1) The contracts exist, signed, on file — the reviewer moves on. (2) The contracts do not exist — the reviewer flags the feature. (3) The contracts exist but the scope statements are vague ("owns safety") and the sign-off dates are 18 months old — the reviewer flags the feature and asks who is currently on-call for the hand-off. Only option (1) survives the review.

Author the inventory. Seven contracts, or explicitly-marked "not applicable" for the peer roles this feature does not need. Where a real peer counter-party is available, negotiate and sign. Where not, self-review from the peer role's perspective and note what you would push back on.

## Requirements

### Deliverable 1 — The seven contracts

Under `docs/features/<feature>/delegations/`, one Markdown file per peer role:

- `agentic-ai-engineer.md` — framework depth
- `ai-eval-engineer.md` — program-level eval
- `model-evaluation-engineer.md` — benchmark and statistical rigour
- `ai-risk-engineer.md` — harm models and red-team data
- `ai-infra-mlops.md` (or `ai-infra-ml-platform.md`, matching your org's naming) — deploy and platform
- `ai-infra-performance.md` — serving performance
- `ai-infra-security.md` (or `application-security-engineer.md`) — deep security

Each contract is 1–2 pages of Markdown in the six-field shape from [chapter 05](../05-delegation-contracts-with-peer-specialists.md):

1. **Peer role** — named role, team, on-call counter-party (or patterned `role:$name`).
2. **Scope handed off** — bounded description of *what* is delegated. Not "safety" — "adversarial red-team programme for the <feature> surface, quarterly cadence, feedback into the eval fixture set."
3. **Applied-engineer inputs** — the specific artefacts the applied engineer commits to provide, in a specific format, on a specific cadence.
4. **Peer-role outputs** — the specific artefacts the peer role commits to return, in a specific format, on a specific cadence.
5. **Escalation and revisit** — how the two sides handle disagreement; the calendar cadence and the event triggers that force a re-sign.
6. **Signed** — both parties, with date. If self-reviewed, the applied engineer signs and the peer signature line is annotated "self-reviewed on <date> — pending counter-party sign".

If a peer role does *not* apply to this feature (e.g., `ai-infra-performance` for a fully hosted / provider-API-only feature), the file for that role still exists, contains a one-paragraph "not applicable, rationale" statement, and is signed by the applied engineer. Absence-with-rationale is a valid answer; silent absence is a gap.

### Deliverable 2 — The inventory README

Under `docs/features/<feature>/delegations/README.md`:

A single-page index that lists all seven files with a one-line summary of what is delegated (or the not-applicable rationale), a status column (signed / self-reviewed / pending / N/A), and a last-signed date. The README is the file the feature card ([exercise-04](exercise-04-per-feature-model-card-slice.md)) §9 links to.

Working shape:

```markdown
# Delegation inventory — <feature>

| Peer role                     | Scope in one line                                       | Status         | Last signed |
|-------------------------------|----------------------------------------------------------|----------------|-------------|
| agentic-ai-engineer           | LangGraph internals; MCP protocol integration            | Signed         | 2026-08-04  |
| ai-eval-engineer              | Shared judge substrate; program-level eval standards     | Signed         | 2026-07-22  |
| model-evaluation-engineer     | A/B statistical analysis; benchmark methodology          | Self-reviewed  | 2026-08-04  |
| ai-risk-engineer              | Quarterly red-team programme; harm model                 | Signed         | 2026-07-14  |
| ai-infra-mlops                | k8s cluster, CI/CD, secrets rotation                     | Signed         | 2026-06-30  |
| ai-infra-performance          | N/A — hosted provider only, no self-served inference     | Signed (N/A)   | 2026-08-04  |
| ai-infra-security             | Authz architecture; MCP-server security review           | Signed         | 2026-07-14  |
```

Any row not signed or explicitly N/A is a **delegation gap**; the applied engineer is on the hook for that depth until it closes.

### Deliverable 3 — The own-scope statement

Under `docs/features/<feature>/delegations/OWN-SCOPE.md`:

The counterpart to the inventory: a single-page list of the things the applied engineer explicitly does *not* delegate, drawn from the own-scope list in [chapter 05](../05-delegation-contracts-with-peer-specialists.md). Each bullet ends with "…owned by the applied engineer; do not delegate." Include, at minimum:

- The feature's **charter, one-pager, and feature card** (from [chapters 01](../01-feature-charter-with-kill-criteria.md), [04](../04-per-feature-model-card-slice.md); [mod-101 chapter 04](../../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md)).
- The **prompt registry** and prompt-level rollout discipline (from [mod-102 chapter 05](../../mod-102-production-prompting-and-structured-outputs/05-prompts-as-versioned-artefacts.md) and [mod-111 chapter 04](../../mod-111-shipping-experimenting-and-rollbacks/04-hot-rollback-of-a-bad-prompt.md)).
- The **feature-level eval fixtures** and the incident-to-fixture loop (from [chapter 02](../02-incident-to-fixture-loop.md); [mod-106](../../mod-106-feature-side-evaluation/README.md)).
- The feature's **guardrail configuration, safety policy, and OWASP mapping runbook rows** (from [mod-107](../../mod-107-guardrails-and-safety-on-the-surface/README.md)).
- The feature's **cost accounting and CPSO** (from [mod-109 chapter 04](../../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md)).
- The feature's **on-call runbook, incident response, and three-artefact close-out** (from [mod-110 chapter 05](../../mod-110-observability-feedback-and-incidents/05-per-feature-status-page-and-on-call-runbook.md)).
- The **flag layer, canary plan, and rollback SLA** (from [mod-111](../../mod-111-shipping-experimenting-and-rollbacks/README.md)).
- The **delegation contracts themselves** — the applied engineer authors, negotiates, and refreshes.

The own-scope file is short (one page). Its job is to make the boundary visible on both sides — if a peer role tries to hand back one of these bullets, the applied engineer has a written rejection.

### Deliverable 4 — The gap analysis

Under `docs/features/<feature>/delegations/GAPS.md`:

A brief companion that lists any delegation gap you found while writing the inventory:

- Peer roles whose contract is unsigned (self-reviewed only), with the plan for getting the counter-party signature.
- Peer roles whose scope statement is currently vague (you flagged it during drafting) and the tightening you plan.
- Feature-card §9 dependencies that do not match any contract on file — either close the gap by writing the contract, or push the dependency into "known limitations" with a rationale.

Each row has an owner (the applied engineer or, if the gap is a scope negotiation, the peer counter-party) and a target close date.

### Deliverable 5 — Refresh drill for one contract

Pick one contract (any of the seven), fast-forward the calendar by 6 months, and produce **v0.2** as if a triggering event had occurred (an SDK-version bump for `agentic-ai-engineer`, a red-team finding for `ai-risk-engineer`, a cluster capacity event for `ai-infra-mlops`, and so on). Store as `docs/features/<feature>/delegations/<peer-role>-v0.2.md` alongside the v0.1 file.

The v0.2 file includes a **change-log block** at the top:

```markdown
## Changes from v0.1 (signed 2026-08-04) → v0.2 (signed 2027-02-12)

- Triggering event: LangGraph v4 major-version release, breaking checkpointing API.
- Scope: unchanged (still LangGraph internals + MCP integration).
- Inputs: added — feature's list of LangGraph primitives used, refreshed at each release.
- Outputs: added — compatibility PR from peer for v4 upgrade, tested against feature's regression fixtures.
- Cadence: unchanged (per major-version bump + incident triggers).
```

The refresh drill proves the contract is a living artefact — a fossil signed at GA and never touched is a fossil the incident will not respect.

## Starter guidance

- **Start with the scope statement.** Every contract lives or dies on Section 2. Draft it once, then re-read it and ask: could a reasonable colleague read this and disagree with what is delegated? If yes, rewrite. "Safety" is a category, not a scope; "adversarial red-team programme for the support-refund-composer surface, quarterly cadence, feedback into the eval fixture set" is a scope.
- **Draft inputs before outputs.** Applied-engineer inputs are what you *commit to provide* — the feature's control-flow spec, the fixture folder, the traffic model, the tool catalogue. These are the concrete artefacts you can point at. Peer-role outputs are what you *ask back*; that side is easier once your side is clear.
- **Format matters.** "We will share the trace vocabulary" is vague; "the tracer emits `app.prompt.template_hash` and `app.experiment.arm` per the vocabulary in [mod-110 chapter 01](../../mod-110-observability-feedback-and-incidents/01-otel-genai-and-openinference-span-vocabulary.md)" is a hand-off the peer can accept or push back on.
- **Cadence matters.** "Regularly" is not a cadence; "quarterly, plus on any P0/P1 incident tagged `framework`" is a cadence a calendar can enforce. Every contract has one.
- **The "not applicable" case is a first-class outcome.** If your feature is fully-hosted / provider-API only, the `ai-infra-performance` contract is a one-paragraph rationale saying the peer role has no work here for this feature, and it is signed. Leaving the file absent is worse than writing "N/A" — the reviewer cannot tell if you forgot.
- **The peer role signs, or you write down what you would say if you were them.** A contract with only one signature is a draft. If the peer counter-party is unreachable, the honest thing is a "self-reviewed pending sign" annotation with the outreach plan on the gap ledger.
- **Do not author the peer role's work.** The applied engineer red-teaming their own feature is a scope violation of the *role*, not just of the contract. The whole point of delegation is that the peer role brings depth *plus* independence.
- **Do not hand off the surface.** User-visible incidents on the feature route to the applied engineer. The platform team can help mitigate an underlying cluster event; they cannot reason about a support-refund-composer's user. The own-scope file names this boundary in writing.
- **The delegation graph is a peer-review artefact.** When you complete the inventory, walk it through with a colleague. Every "I thought X owned that" you catch during peer review is one you avoid catching at incident time.

## Acceptance criteria

- [ ] Seven files exist under `docs/features/<feature>/delegations/`, one per peer role (or the org-specific naming). Every file follows the six-field shape from [chapter 05](../05-delegation-contracts-with-peer-specialists.md).
- [ ] No contract has a scope statement that reads as a bare category ("safety," "quality," "infra"). Every scope statement is bounded.
- [ ] Every contract names inputs and outputs in a specific format on a specific cadence. "Regularly" and "as needed" are rejects.
- [ ] Contracts that do not apply to this feature carry an explicit "not applicable, rationale" statement, signed by the applied engineer. Files are never silently absent.
- [ ] The inventory `README.md` lists all seven with a status column and a last-signed date.
- [ ] The `OWN-SCOPE.md` file names what does *not* delegate, with each bullet linked to the substrate that produces it.
- [ ] The `GAPS.md` file names any unsigned contract or unresolved scope negotiation, with an owner and a target close date.
- [ ] At least one contract has been refreshed to v0.2 with a change-log block at the top; both v0.1 and v0.2 files are on disk.
- [ ] The feature card ([exercise-04](exercise-04-per-feature-model-card-slice.md)) §9 (known limitations) resolves its delegation-dependency mentions to the contracts in this inventory. No orphan references in either direction.

## Stretch goals

Pick any one:

- **The counter-party rehearsal.** Send one contract (any of the seven) to a colleague who plays the peer role and asks them to negotiate the scope, inputs, and cadence as if they were the actual counter-party. Land the amended version. The stretch is the *diff* — what changed, and what you would have missed without the rehearsal?
- **The scope-drift audit.** Take three arbitrary bullets from the "applied-engineer inputs" section across your seven contracts. For each, prove — with a link to a substrate, a dashboard, or an artefact — that you actually can produce it today. Any bullet you cannot back up is a scope-drift gap; add it to `GAPS.md`.
- **The org-altitude view.** Imagine your org has ten features, each with a delegation inventory in this shape. Author a short "who owes whom what" map (a table or a simple diagram) that aggregates the peer-role loads. Where is one peer role (say `ai-risk-engineer`) load-bearing across many features? That is a peer-role capacity conversation the applied engineer's inventory feeds.
- **The gone-tomorrow drill.** For one peer role, imagine the counter-party leaves the org tomorrow. Write the two-paragraph on-boarding note that would let their successor pick up the contract without a re-negotiation. If you cannot, the contract is under-documented — fix it.
- **The delegation-contract quality bar.** Take one anti-pattern from [chapter 05](../05-delegation-contracts-with-peer-specialists.md)'s anti-patterns list and prove your inventory does not exhibit it — for every contract. Attach the check as a companion doc.

## Deliverable

Submit as `mod-112/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/features/<feature>/delegations/` — seven contracts plus `README.md`, `OWN-SCOPE.md`, `GAPS.md`.
- The v0.2 refresh of one contract.
- A short `README.md` at the exercise root naming the feature, the peer roles that apply (and those that don't), and any assumptions about the org's peer-role naming.
