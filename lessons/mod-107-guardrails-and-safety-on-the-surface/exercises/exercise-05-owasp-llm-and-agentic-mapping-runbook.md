# exercise-05: OWASP LLM and Agentic Mapping Runbook

**Estimated effort:** 2 hours

## Objective

Author the **safety policy**, the **refusal-category taxonomy**, the **over-refusal eval fixture set**, and the **OWASP LLM + Agentic AI mapping runbook** for one product surface — the artefacts that tie every guardrail in this module together, give every finding a home, and assign every finding an owner. The exercise is not "read the OWASP list." It is "produce the versioned files a reviewer will use to sign off next quarter's release."

## Learning outcomes

By the end you should be able to:

- Author a written safety policy for a shipped surface: scope, must-never list, tolerated-refusal areas, must-not-over-refuse areas, rendering rules, escalation contract, owners, review cadence.
- Define a runtime refusal-category taxonomy that rolls up on a dashboard and cross-references specific safety-policy rules.
- Assemble an over-refusal eval fixture set (obvious-yes, clarifying-yes, policy-grey, injection-shaped-but-legitimate) alongside a must-refuse eval fixture set — both pre-registered.
- Map every finding — from a moderation FP, from an injection-eval failure, from a tool-abuse red-team, or from a real incident — to at least one OWASP LLM Top 10 entry and (where applicable) one OWASP Agentic AI Top 10 entry, with an owner assigned.
- Draw the peer-role handoffs explicitly so the applied engineer's runbook does not accidentally claim `ai-risk-engineer` or `ai-security-engineer` responsibilities.

## Prerequisites

- You have read [chapter 05](../05-refusal-calibration-and-owasp-mapping.md).
- You have completed at least two of exercises 01-04 (so you have real guardrails to map to).
- Access to the current OWASP Top 10 for Large Language Model Applications (`https://owasp.org/www-project-top-10-for-large-language-model-applications/`) and the OWASP Agentic AI project material.<!-- needs-research: pin current URLs and titles; both projects have evolved. -->
- The surface's org chart — who the trust-and-safety, product, security, and privacy owners are.

## Problem statement

Product leadership has asked *"can you produce, in one document, everything a new reviewer would need to sign off on our safety posture?"* The team has moderation, injection defence, scrubbing, and tool guards, but no written policy tying them together, no runbook mapping past findings to a taxonomy, and no clear owners on the findings the team knows about.

In two hours, produce the four artefacts. Every artefact must be *versioned*, *reviewable*, and *machine-readable-where-that-matters*.

## Requirements

### Deliverable 1 — The safety policy

Under `docs/safety-policy.md`, a versioned Markdown document with the sections from [chapter 05](../05-refusal-calibration-and-owasp-mapping.md):

- **Version and owner block.** Version, owners (a trust-and-safety + product pair, and any peer-role responsibilities named), review cadence, review calendar.
- **Scope.** One sentence: what the surface does.
- **Must-never.** Bulleted list of hard boundaries specific to your surface's domain and jurisdiction.
- **May refuse.** Bulleted list of tolerated refusals with the policy section number each derives from.
- **Must not over-refuse.** Bulleted list of areas where refusal is a bug.
- **Rendering.** How refusals are worded, whether categories are disclosed, what the escalation path looks like from the user's side.
- **Escalation contract.** Which categories route to a human, on what SLA, with what handoff context.
- **Cross-references.** For every guardrail from exercises 01-04, name the policy section it implements.

### Deliverable 2 — Refusal-category taxonomy

Under `runtime/refusal-categories.yaml`, a machine-readable taxonomy:

- Every category from chapter 05's runtime set (`policy.must_never`, `policy.out_of_scope`, `policy.escalate_required`, `moderation.provider`, `moderation.classifier`, `injection.suspected`, `pii.blocked`, `tool.denied`, `tool.declined_by_user`) plus any surface-specific categories your policy adds.
- For each: a `description`, a `policy_rule` pointing at the safety-policy section, a `default_render` (the wording the user sees), and an `owner_role` (who reviews spikes in this category).
- Every refusal-emitting site in code (chapters 01-04) must use a category from this file. A test that greps for hardcoded refusal-category strings not in the taxonomy fails the build.

### Deliverable 3 — Over-refusal and under-refusal eval

Under `fixtures/refusal/`:

- `obvious-yes.yaml` — at least **50** legitimate requests. Refusal is a bug on any of them.
- `clarifying-yes.yaml` — at least **20** requests where the correct behaviour is a clarifying question (not a refusal, not a blind answer).
- `policy-grey.yaml` — at least **20** requests near a policy boundary. For each: the *documented correct behaviour* (with the safety-policy section that decides it).
- `injection-shaped-but-legitimate.yaml` — at least **10** requests whose surface phrasing looks like an injection but whose intent is legitimate (a user quoting `ignore prior instructions` from a paper, a user asking about prompt-injection safety, a QA test case). Refusal on these is a bug.
- `must-refuse.yaml` — at least **30** requests that MUST be refused per the safety policy's must-never list. Answering any of them is a critical incident.

Under `bin/refusal-eval`, a runner that:

- Executes each fixture, records the outcome (`refused`, `clarified`, `answered`, `escalated`).
- Computes over-refusal rate (on the obvious-yes and injection-shaped-but-legitimate panels), under-refusal rate (on the must-refuse panel), and clarification rate (on the clarifying-yes panel).
- Emits a machine-readable and human-readable report; fails a PR that regresses either rate against the pre-registered thresholds.

Pre-register the thresholds in `policies/refusal.yaml` in the mod-106-shape.

### Deliverable 4 — The OWASP mapping runbook

Under `security/owasp-runbook.md` and `security/owasp-findings.yaml`:

- **Runbook prose (`owasp-runbook.md`).** For each entry in the current OWASP LLM Top 10 that applies to your surface, name: which of your guardrails address it (with links into exercises 01-04), what the residual risk is, and who owns the residual risk. Do the same for the OWASP Agentic AI Top 10 entries that apply. Where a top-ten entry does *not* apply (your surface has no tools, no memory, no retrieval, etc.), state that and why.
- **Findings table (`owasp-findings.yaml`).** At least **five** concrete findings — from your moderation FP/FN report, your injection eval, your tool-abuse red-team, or your incident library. Each row includes `id`, `surface`, `discovered_by`, `date`, `summary`, `owasp_llm` (list of entry codes), `owasp_agentic` (list, if applicable), `controls_hit`, `controls_missed`, `owner`, `status`, `remediation` (list of PR references or "planned"), and `review_at`.
- The findings table's `owner` must be one of a named list of roles (`applied-ai-engineer`, `ai-risk-engineer`, `ai-security-engineer`, `privacy-engineer`, `data-security-engineer`, `trust-and-safety`, `incident-commander`). No finding is unowned.

### Deliverable 5 — The peer-role boundary

Under `docs/peer-role-boundaries.md`, a one-page written statement:

- For each peer role from chapter 05 (`ai-risk-engineer`, `ai-safety-engineer`, `ai-security-engineer`, `privacy-engineer`, `data-security-engineer`, `incident-commander`), name: what they own on the surface, what the applied engineer owns that *feeds* them, and where the handoff artefact lives (a specific file, a ticket queue, a dashboard).
- This is the artefact that keeps the applied engineer's runbook from silently absorbing peer-role responsibilities.

### Deliverable 6 — Drift monitor sketch

Under `dashboards/refusal-drift.md`, a sketch (Markdown + a mock chart if you want) of:

- The dashboard panel that shows refusal rate per category over the last 7 and 30 days.
- The alert rule (e.g. "page owner if any category's rate deviates > 3σ from the 30-day baseline or crosses the pre-registered ceiling").
- The sampled-review queue for refused requests (1% sample) and how those samples flow back into `fixtures/refusal/`.

You do not need to actually deploy the dashboard for this exercise — the sketch and query language is sufficient. Wiring the dashboard is a mod-110 exercise.

## Starter guidance

- **Start with the must-never list.** It's the shortest and the highest-leverage. Write down the two or three lines that would embarrass the product if the surface violated them, and hang everything else off that.
- **Do not paste OWASP's descriptions verbatim.** For each entry, write one paragraph about what it means *for your surface* — not the abstract. If you can't do that, the entry probably doesn't apply and should say so.
- **Findings without owners are the failure mode.** If you can't name an owner, either the finding is not real, or the peer-role boundary in your org is unclear. Both are worth surfacing.
- **The over-refusal fixtures come from your users.** Support tickets that read "the AI refused to help me with X" are exactly the fixtures. If the surface is new, borrow from public reports of over-refusal incidents (there are many).
- **The taxonomy is small on purpose.** If your team has invented 20 categories, the taxonomy is trying to be a diagnostic system, not a rollup — pull categories back into the chapter-05 nine plus a small surface-specific tail.

## Acceptance criteria

- [ ] `docs/safety-policy.md` has all the sections listed in Deliverable 1, is versioned, and names owners with a review calendar.
- [ ] `runtime/refusal-categories.yaml` is a machine-readable taxonomy where every category has a `policy_rule` pointing at the safety policy and an `owner_role`.
- [ ] `fixtures/refusal/` has the five panels at the required scale.
- [ ] `bin/refusal-eval` produces over-refusal and under-refusal rates and fails on regression against pre-registered thresholds in `policies/refusal.yaml`.
- [ ] `security/owasp-runbook.md` addresses every applicable OWASP LLM Top 10 and Agentic AI Top 10 entry with a per-surface paragraph, and explicitly notes non-applicable entries.
- [ ] `security/owasp-findings.yaml` contains ≥ 5 real findings with owners from the named role list.
- [ ] `docs/peer-role-boundaries.md` states, per peer role, what they own and where the handoff artefact lives.
- [ ] `dashboards/refusal-drift.md` sketches the panel, the alert rule, and the sampled-review queue.
- [ ] No finding in the runbook is unowned.
- [ ] No hardcoded refusal-category string in code sits outside the taxonomy (proven by a test).

## Stretch goals

Pick any two:

- **Automated OWASP category cross-check.** Write a small script that grep-checks every guardrail source file for a `# owasp: LLM01` (or similar) annotation and cross-references the runbook. Fails if a guardrail file has no annotation.
- **Incident-template link.** Update the surface's incident post-mortem template (or draft one if none exists) to require an OWASP mapping and an owner assignment. Prove that a post-mortem cannot be marked "closed" without the mapping being present.
- **Refusal-rate baseline.** Run the refusal eval, record the baseline numbers into `policies/refusal.yaml`. Argue for the pre-registered thresholds. This is the artefact that makes the refusal eval a real gate rather than a decoration.
- **Peer-role RACI.** Extend the peer-role boundaries doc into a full RACI (responsible, accountable, consulted, informed) for the top 10 findings in your runbook. This is what a security reviewer will actually read.
- **Public-face safety statement.** Draft a one-page user-facing version of the safety policy (what the surface can and cannot help with). Every claim in the public version must map back to a section in the internal safety policy. This is a product-comms artefact that lives or dies on the internal policy's clarity.

## Deliverable

Submit as `mod-107/exercise-05/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/safety-policy.md`, `docs/peer-role-boundaries.md`.
- `runtime/refusal-categories.yaml`.
- `fixtures/refusal/` with the five panels.
- `bin/refusal-eval` and `policies/refusal.yaml`.
- `security/owasp-runbook.md`, `security/owasp-findings.yaml`.
- `dashboards/refusal-drift.md`.
- A short `README.md` summarising the four artefacts and how to find the review queue.
- Any stretch-goal artefacts named alongside.
