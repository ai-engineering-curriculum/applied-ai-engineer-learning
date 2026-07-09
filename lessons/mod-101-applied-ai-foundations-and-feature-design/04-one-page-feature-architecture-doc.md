# The One-Page Feature-Architecture Doc

## Why one page

The single artefact that binds the shape (chapter 01), the SLO table (chapter 02), and the trade-off note (chapter 03) into a shippable decision is a **one-page feature-architecture doc**. Three roles will sign off on it — a Product Manager, an ML engineer, and a governance owner — and none of them will read six pages before the next meeting. If it doesn't fit on a page, no one signs it, and the feature ships on an unwritten agreement that will be contested at the first incident.

A one-pager is not a design document. It is not a spec. It is not a runbook. It is a **decision record** — the smallest possible artefact that (a) answers the "what are we building and why" question in each reviewer's language, and (b) makes the accountable owner explicit for every dimension.

## The three sign-off perspectives

The reviewer test: can each of the three roles walk away with what they need?

- **PM**: what is being built? For whom? What is the success metric? What are we *not* doing? What's the kill criterion?
- **ML engineer**: what shape? what provider / tier / decoding config? what's the eval plan? what's the rollback lever? what's the router bake-off plan?
- **Governance owner**: what data touches what provider in what region? what's the safety envelope? what's the incident path? what human oversight exists? what quality dimensions have been considered?

The doc has to serve all three in the same page. That is a real constraint. Meet it.

## The template

Copy-paste and fill in. Where the template says *"link to X,"* the page holds a link, not the full artefact.

### Section 1 — Header (3 lines)

- **Feature name and surface** — one sentence naming the product surface and the user.
- **Owner + reviewers** — name of the engineer, PM, and governance reviewer signing off.
- **Version + review date** — semver-style version, sign-off date, next mandatory revisit date.

### Section 2 — What we're building (5 lines)

- **User** — one sentence: "our support agents," "our end-user policyholders," "our internal ops team."
- **Problem** — one sentence: "drafting a support reply takes ~7 min today because the agent hunts through the KB and past tickets."
- **Shape (chosen)** — from chapter 01, e.g. "co-pilot on the reply textarea, powered by retrieval-with-citations behind the scenes."
- **Shapes rejected** — one line naming the shapes rejected and the reason.
- **Non-goals** — one line listing what the feature will *not* do (e.g. "will not auto-send replies," "will not answer legal questions").

### Section 3 — SLO table (compressed)

The full table from chapter 02, one row per SLO. Compress the *evaluator* and *owner* columns to a handle each.

| Metric | Target | Window | Evaluator | Owner |
|---|---|---|---|---|
| task-success | ≥ 0.90 | 7 d | judge:v3 | applied-ai |
| groundedness | ≥ 0.95 | 7 d | judge:v3 | applied-ai |
| refusal (no-evidence) | ≥ 0.98 | 30 d | probe-set:v1 | applied-ai |
| over-refusal | ≤ 0.02 | 30 d | probe-set:v1 | applied-ai |
| cost / success | ≤ $0.04 | 7 d | billing | applied-ai |
| TTFT p95 | ≤ 800 ms | 1 d | client-tel | applied-ai |
| e2e p95 | ≤ 5 s | 1 d | client-tel | applied-ai |
| corpus lag p95 | ≤ 30 min | 1 d | ingest | data-eng |

### Section 4 — Initial provider + tier + region

One paragraph from chapter 03, plus a one-line "trade-off note: link to X."

- **Provider + model + version + region**: e.g. "Anthropic Claude Sonnet 4.5, `claude-sonnet-4-5`, us-east-1 for US tenants, eu-west-1 for EU tenants."
- **Structured output**: e.g. "native JSON tool call for the field-extraction step; unconstrained streaming for the drafted reply."
- **Fallback**: e.g. "self-hosted Llama 3.3 70B on the internal vLLM cluster; contract-tested in CI; expected quality regression documented in trade-off note."

### Section 5 — Data flow and residency (5 lines)

A prose paragraph or a compact bullet list:

- Where does the user's input come from, and what class of data is it (PII, PHI, confidential business data, public)?
- Which provider(s) does it touch, in which region(s)?
- What is logged, where, and with what retention and PII policy?
- Which sub-processors touch the payload (guardrails, judge, trace warehouse, feature-flag provider)?
- What is the deletion / export contract?

If the answer to any of these is "I don't know," the doc isn't ready for governance sign-off.

### Section 6 — Safety envelope (5 lines)

- **Guardrail layers** — one line per layer (provider moderation, guardrail model, programmable rails).
- **In-scope harms** — refer to a written safety policy version.
- **Human-in-the-loop** — what human confirms what, and where's the kill-switch?
- **OWASP LLM Top 10 and Agentic Top 10 mapping** — reference the mapping doc (owned in mod-107), not the full list.

### Section 7 — ISO/IEC 25059 quality checklist

The ISO/IEC 25059:2023 quality model extends the ISO/IEC 25010 software-product quality model with AI-specific dimensions. Use it as a checklist so the reviewers can see that each dimension was considered — even if the answer is "not applicable here, here's why."

At the one-pager altitude, the checklist is a table with one cell of prose per dimension. The classical 25010 dimensions still apply; 25059 adds explicit AI-related characteristics.

| Dimension | Note |
|---|---|
| Functional adequacy (25010) | The feature does the drafted-reply task; task-success SLO ≥ 0.90. |
| Performance efficiency (25010) | TTFT and end-to-end p95 SLOs; cost-per-successful-outcome SLO. |
| Compatibility (25010) | Runs inside existing helpdesk UI; API contract with retriever documented. |
| Interaction capability (25010) | Co-pilot UX with accept / edit / reject; citation panel always visible. |
| Reliability (25010) | Fallback provider path; retry / repair budget; graceful degradation on RAG-empty. |
| Security (25010) | PII scrub inbound / outbound; secrets policy; audit log. |
| Maintainability (25010) | Prompt versioned; eval fixtures pinned; rollback lever behind a feature flag. |
| Flexibility (25010) | Router-ready; open-weights fallback contract-tested; portability accounting on file. |
| Safety (25010) | Refusal and over-refusal SLOs; injection-resistance probe suite. |
| **Functional adaptability** (25059) | Prompt / retriever / model updates via feature flag without redeploy. |
| **User controllability** (25059) | User accepts / edits / rejects every suggestion; kill-switch by policy. |
| **Transparency** (25059) | "Drafted by AI" label; citation panel; model version and prompt hash surfaced in the trace and in a per-answer "why this answer" affordance. |
| **Robustness** (25059) | Injection-resistance suite; RAG-empty fallback; provider-outage fallback; content drift monitor. |
| **Intervenability** (25059) | On-call runbook; hot-rollback of prompt; per-tenant kill-switch. |

<!-- needs-research: verify the exact ISO/IEC 25059:2023 dimension list against the standard document before final sign-off; the dimensions above draw on the 25010 base plus the AI-specific extensions (functional adaptability, user controllability, transparency, robustness, intervenability) named in the 25059 abstract — confirm the wording of "intervenability" vs "interventionability" and the exact inclusion of "accountability" as a sub-characteristic when the standard is accessible. -->

The value is not that you fill in every cell with something elaborate — it is that every reviewer can see the cell was considered. An empty cell means "we haven't thought about this yet." A cell that says "N/A — this surface has no autonomous actions, so user-controllability collapses to accept/reject/ignore" is a defensible answer.

### Section 8 — Eval, rollout, and kill criteria (5 lines)

- **Eval-gated CI** — link to the offline suite; state which SLOs are enforced in CI vs monitored in production only.
- **Rollout plan** — shadow → 1 % canary → 10 % → 100 %, with per-stage exit criteria (from mod-111).
- **Kill criteria** — three or four concrete things that get the flag turned off: e.g. "groundedness < 0.85 on the 7-day rolling window," "cost / success > $0.08," "safety-refusal incident of any severity."
- **Owner on-call** — who gets paged and where the runbook lives.

## The sign-off flow

Land the artefact in a place all three reviewers can see and comment on (usually a Notion / Google Doc / repo-hosted markdown file with review). Route:

1. Draft owner circulates. Text-only comments only — no restructures without an in-person round.
2. PM signs Section 2, Section 3 targets, and Section 8 kill criteria.
3. ML engineer signs Section 4, Section 7 (technical dimensions), and Section 8 rollout.
4. Governance owner signs Section 5, Section 6, and Section 7 (AI dimensions).
5. Version bumps and next-review date locked in.

Sign-off is a checkbox with a name and a date. Not "I read it" — "I sign that my part is correct." If a reviewer can't sign, they say what would change their mind before they will.

## What a bad one-pager looks like

Common failure modes to catch in your own review:

- **Aspirational SLOs with no evaluator or window.** "High-quality answers" is not a number. If you can't compute it from telemetry weekly, it isn't an SLO.
- **A shape that isn't in the chapter-01 catalogue and isn't named as a blend.** "Agentic assistant with retrieval features" is not a shape. Pick one.
- **A trade-off note in the trade-off note field.** Section 4 links to the trade-off note; it doesn't try to contain it. If the section is a wall of text, the note isn't done.
- **A safety envelope that references a policy version that doesn't exist.** Version the policy before you cite it.
- **Kill criteria that are unfalsifiable.** "If quality is poor" is not a kill criterion; "if groundedness drops below 0.85" is.
- **No revisit date.** Trade-off notes and provider choices are perishable. Missing the revisit date is how a one-pager becomes a fossil.

## Summary

- The one-page feature-architecture doc is the single sign-off artefact that binds shape, SLOs, provider choice, safety, and rollout.
- Three roles sign: PM (Sections 2, 3, 8), ML engineer (Sections 4, 7 technical, 8), governance owner (Sections 5, 6, 7 AI).
- ISO/IEC 25059:2023 quality dimensions are a checklist — every cell either has a note or a defensible "N/A" reason.
- A one-pager without a revisit date is a fossil. Add the trigger and the mandatory date.
- Sign-off is not "I read it" — it's "I sign that my part is correct."

Chapter 05 closes the loop by naming the four failure modes of leaderboard-driven feature design so you can reject them in review — including your own.
