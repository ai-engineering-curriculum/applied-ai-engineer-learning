# exercise-04: Per-Feature Feature-Card Slice

**Estimated effort:** 3 hours

## Objective

Compose a **feature-card slice** for one real (or realistic) shipped applied-AI feature, drawing from the artefacts a mature substrate already produces: the signed charter, the trade-off notes, the eval fixtures, the safety policy, the incident-to-fixture close-outs, the cost ledger, and the runbook. The deliverable is not a marketing page and not a spec — it is the 3–6-page **structured evidence document** the internal review committee, the Governance-family peer roles, and a future audit reviewer read to answer *"what did we know about this feature when we shipped it?"*

The exercise proves you can *compose* the card in an afternoon from artefacts that exist, and — if a section resists composition — that you can name the substrate gap that made it hard rather than papering it over with prose.

## Learning outcomes

By the end you should be able to:

- Compose all nine sections of the feature card — **identity, intended use / non-goals, system architecture, evaluation evidence, safety evidence, incident evidence, cost / latency evidence, human oversight / intervention, known limitations** — by citing existing substrate artefacts rather than authoring fresh prose.
- Author a **YAML front-matter** block that machine-summarises the card in a shape the `ai-governance-analyst` aggregator can join across features.
- Distinguish an **evidence claim with a resolving link** from an unsupported assertion, and reject any claim in the card that cannot cite a source.
- Write a **known-limitations section** that is credible — populated with real gaps derived from the pre-mortem, the delegation graph, and the eval-set's uncovered surface — rather than "none known."
- Diagnose a **substrate gap** from the card composition process itself: any section the applied engineer cannot compose without inventing evidence is a gap in a mod-101-through-mod-111 substrate, not a writing task.
- Route the card through the three sign-offs (applied engineer, PM, governance owner) and record the sign-off in the artefact itself.

## Prerequisites

- You have read [chapter 04](../04-per-feature-model-card-slice.md) and the surrounding chapters ([01](../01-feature-charter-with-kill-criteria.md), [02](../02-incident-to-fixture-loop.md), [03](../03-build-vs-buy-across-three-surfaces.md), [05](../05-delegation-contracts-with-peer-specialists.md), [06](../06-governance-mapping.md)).
- A feature that you either own, are shipping, or can pattern realistically. Ideal: the same feature from [exercise-01](exercise-01-feature-charter-with-kill-criteria.md) and [exercise-03](exercise-03-build-vs-buy-matrix-gateway-eval-agent.md) — you already have the charter and the trade-off notes.
- Access to (or realistic patterned versions of) the substrate artefacts the card cites:
  - Charter and one-pager ([mod-101 chapter 04](../../mod-101-applied-ai-foundations-and-feature-design/04-one-page-feature-architecture-doc.md)).
  - Offline eval CI runs and judge-calibration report ([mod-106](../../mod-106-feature-side-evaluation/README.md)).
  - Safety policy, FP/FN report, OWASP mapping runbook ([mod-107](../../mod-107-guardrails-and-safety-on-the-surface/README.md)).
  - Cost ledger and CPSO monitoring ([mod-109 chapter 04](../../mod-109-cost-latency-slos-for-llm-features/04-per-feature-and-per-tenant-token-and-cost-accounting.md)).
  - Latency SLO dashboard ([mod-109 chapter 01](../../mod-109-cost-latency-slos-for-llm-features/01-latency-vocabulary-and-per-surface-slos.md)).
  - Runbook and incident close-out tickets ([mod-110](../../mod-110-observability-feedback-and-incidents/README.md)).
  - Kill-switch config and rollout plan ([mod-111](../../mod-111-shipping-experimenting-and-rollbacks/README.md)).
- A colleague or mentor willing to play the governance-owner reviewer, if you can arrange it. Self-review acceptable with the reviewer's-questions companion described below.

## Problem statement

Your feature is at (or approaching) GA. The internal AI risk committee has a review on the calendar in two weeks and wants "the feature card" on file 48 hours before. The `ai-governance-analyst` peer role has been asking for a machine-consumable slice they can join into the org-wide governance dashboard. The auditor for next quarter's ISO/IEC 42001 surveillance visit will pull the same document.

You are not writing this card from scratch — you are *composing* it from substrates you already run. If a section forces you to invent numbers, the substrate is the problem; if a section fits on a page because the substrate is present, the substrate is doing its job. Author the card, cite the evidence, sign it, and — where the composition surfaced a gap — file the gap so it lands in the next sprint rather than the next audit.

## Requirements

### Deliverable 1 — The feature card

Under `docs/features/<feature>/feature-card-v1.0.md`:

A single Markdown file, 3–6 pages when printed, with a **YAML front-matter** block and the **nine sections** from [chapter 04](../04-per-feature-model-card-slice.md). Path-stable — the location does not change on future revisions; only the version number in the filename bumps.

Front-matter (required fields, extend as your org's aggregator schema needs):

```yaml
---
feature: <feature-slug>
version: 1.0                # the shipped software version
card_version: 1.0           # this card's version, tracked separately
signed: YYYY-MM-DD
next_review: YYYY-MM-DD     # quarterly refresh at latest
owner: applied-ai-engineer:<handle>
pm: pm:<handle>
governance_reviewer: ai-governance-analyst:<handle>
charter: docs/features/<feature>/charter-vX.Y.md
one_pager: docs/features/<feature>/architecture-vX.Y.md

models:
  primary: <dated-model-id>          # e.g., gpt-4o-2024-08-06 — never bare aliases
  fallback: <dated-model-id>
  judge: <dated-model-id>
  embedding: <dated-model-id>

evals:
  offline_fixture_count: <int>
  regression_fixture_count: <int>
  task_success_score: <float>
  task_success_target: <float>
  over_refusal_rate: <float>
  over_refusal_target: "<= <float>"
  judge_calibration_date: YYYY-MM-DD
  judge_calibration_score: <float>

safety:
  policy_version: <version-string>
  owasp_mapping_findings: <int>
  owasp_mapping_open: <int>
  red_team_last_run: YYYY-MM-DD
  red_team_findings_total: <int>
  red_team_findings_open: <int>

cost:
  cpso: <float>
  cpso_target: <float>
  monthly_cost: <int>
  monthly_ceiling: <int>
  pricing_baseline: YYYY-MM

incidents_last_quarter:
  total: <int>
  p0: <int>
  p1: <int>
  p2: <int>
  p3: <int>
  loop_close_out_rate: <float>

governance:
  nist_ai_rmf_functions: [map, measure, manage]
  eu_ai_act_transparency: applicable | not-applicable | partial
  iso_42001_controls_hit: [<control-id>, ...]
---
```

The nine narrative sections (per [chapter 04](../04-per-feature-model-card-slice.md)):

1. **Feature identity** — name, version, surface, charter reference, architecture reference, owners.
2. **Intended use and non-goals** — cited from the charter; adds known misuse patterns and mitigations.
3. **System architecture** — model / provider / region pins, gateway, eval platform, agent SDK, retrieval stack, guardrail order (from the trade-off notes and the one-pager). Optional diagram.
4. **Evaluation evidence** — offline fixture set, offline results with delta, online eval loop details, A/B result (if the feature was rolled out through one), judge calibration.
5. **Safety evidence** — policy version, guardrail FP/FN report, over-refusal eval results, red-team summary, OWASP mapping table, PII / data handling summary. Names what has *not* been tested.
6. **Incident evidence** — count by severity for the review window, three-artefact loop close-out rate, open incidents, recurring patterns.
7. **Cost and latency evidence** — CPSO vs target, monthly cost vs budget, TTFT/e2e/tokens-per-second SLOs with current values, retry / hedging behaviour.
8. **Human oversight and intervention** — HITL points, escalation contract, kill switch (where it lives, when it was last exercised), user-facing controls, transparency affordances.
9. **Known limitations** — populations not calibrated for, failure modes not covered by the eval set, drift risks, non-goals reiterated, delegation dependencies. Not "none known."

Every claim in every section resolves to a **link** — a CI run, a dashboard, a PR, a runbook entry, a signed peer-role delegation contract, a Grafana query, a fixture file. Prose without a citation is not evidence.

### Deliverable 2 — The composition ledger

Under `docs/features/<feature>/feature-card-composition-ledger.md`:

A one-page ledger, mapping each of the nine sections to the artefact(s) it was composed from. Rows the composer *couldn't* fill without inventing evidence are called out as **gaps**, with a follow-up ticket or issue link. Example rows:

```markdown
| Section | Composed from                                                | Notes / gap                                     |
|---------|--------------------------------------------------------------|-------------------------------------------------|
| §1 Identity | charter v0.3, one-pager v1.0                              | —                                               |
| §4 Evals — offline | CI run #2481 at commit abc123, fixture set at eval/  | —                                               |
| §5 Safety — red team | `ai-risk-engineer` delegation contract §D-2, last run 2026-06-14 | —                                          |
| §6 Incidents — close-out rate | 3 incidents INC-2026-08-04-001…003 (all closed) | —                                               |
| §7 Cost — CPSO | Grafana `feature_cpso{feature=X}` panel                    | —                                               |
| §9 Limitations — populations | Pre-mortem doc; eval fixture locale distribution | GAP: no fixtures for `pt-BR`; ticket #4711 open |
```

The ledger is the diagnostic. If more than one row is a gap, the substrate — not the card — is what needs work before GA.

### Deliverable 3 — The reviewer's questions companion

Under `docs/features/<feature>/feature-card-reviewer-notes.md`:

Author the *four questions* each of the three reviewers is most likely to ask when reading the card, and show where in the card each answer lives (with section + line number, or an anchor link).

Suggested reviewer questions to cover (mix in your own):

- **Applied engineer's own preflight (self-review):** *Is every claim linked? Is any number more than a quarter old? Is the "known limitations" section credible or defensive?*
- **PM reviewer:** *Is the success metric the user-facing one from the charter? Did the A/B show a positive read on the primary metric without a regression on the guardrail metric? What is the CPSO story?*
- **Governance-owner reviewer:** *Is the safety scope from the charter still accurate? Are the handed-off harms actually being handled by the named peer role (link the contract)? Which EU AI Act transparency obligations apply and where is the affordance documented?*

Each answer is a pointer into the card, not a re-derivation. If an answer requires prose the card doesn't contain, edit the card.

### Deliverable 4 — The signature ritual

The signature block at the bottom of the card carries three lines with names, roles, and dates:

- **Applied-AI engineer** — signs on the entire card as the composer.
- **PM** — signs on §1, §2, §4 (evaluation evidence), and §7 (cost).
- **Governance owner** — signs on §5 (safety evidence), §6 (incident evidence), §8 (human oversight), and §9 (known limitations).

If reviewers are unavailable, write self-review notes in a companion file naming — from each role's perspective — what would change their mind. Attach as `docs/features/<feature>/feature-card-self-review.md`.

### Deliverable 5 — The next-review calendar entry

Add the card's `next_review` date to the same revisit calendar the trade-off notes from [exercise-03](exercise-03-build-vs-buy-matrix-gateway-eval-agent.md) use (`docs/features/<feature>/decisions/revisit-calendar.md`). The card's cadence: quarterly, or on any significant change, or on any P0/P1 incident.

## Starter guidance

- **Compose the front matter first.** Every field is a number the substrate already emits. If a field is `TBD`, that field is where you have a gap. Fill from the substrate, not from memory.
- **Section 3 (system architecture) is where the trade-off notes pay off.** If [exercise-03](exercise-03-build-vs-buy-matrix-gateway-eval-agent.md) is done, this section is three sentences plus a diagram — the choices and the version pins. Do not repeat the trade-off logic here; link to the notes.
- **Section 4 (evaluation evidence) needs a CI run at a specific commit.** "Task success is 0.923" without "at commit `abc123`, CI run #2481, fixture set `eval/v1.4`" is not evidence. Bake the discipline into the section from the first draft.
- **Section 5 (safety evidence) resists honesty.** The pull to write "no findings, all clear" is strong; a card that says that is a card no reviewer trusts. Include *what has not been tested* — locales, tenants, tool combinations, prompt shapes — and cite the pre-mortem doc if the safety eval is not comprehensive.
- **Section 6 (incident evidence) is a good place to show the substrate working.** If the incident-to-fixture loop from [chapter 02](../02-incident-to-fixture-loop.md) is running, this section reads: three incidents in the quarter, all closed with all three artefacts, one recurrence prevented by fixture INC-2026-XX-XX. That is the substrate earning its keep in prose.
- **Section 8 (human oversight) is where the EU AI Act transparency work lands.** If the feature interacts with users, name the "You're chatting with AI" affordance and cite [chapter 06](../06-governance-mapping.md) for the mapping.
- **Section 9 (known limitations) is the credibility section.** Write it last, then re-read it, then add one more limitation. If your fixture set has < 30 regression fixtures the feature has been in production for a quarter, that is a limitation. Non-covered locales, tenant sizes, tool combinations, agent depth caps — say them.
- **The composition ledger is the honest artefact.** A card with three gaps in the ledger and three follow-up tickets in flight is *better* than a card with none — because the alternative is a card that hides the gaps in prose. Reviewers respect the ledger.
- **Cite even the "not applicable" rows.** In section 8 the transparency affordance may be "N/A — no user-visible generative output; internal tool only." That is a valid citation with a rationale. `N/A` without a rationale fails governance review — see [chapter 06](../06-governance-mapping.md).
- **Card version ≠ software version.** The feature might be v1.3.2 while the card is v1.0. Bumps are independent; the card version bumps on card content changes, the software version bumps on feature changes. The front-matter distinguishes both.

## Acceptance criteria

- [ ] `docs/features/<feature>/feature-card-v1.0.md` exists as a single Markdown file, 3–6 pages when printed, with YAML front matter and nine narrative sections.
- [ ] Front matter contains all required top-level keys (`feature`, `version`, `card_version`, `signed`, `next_review`, `owner`, `pm`, `governance_reviewer`, `charter`, `one_pager`, `models`, `evals`, `safety`, `cost`, `incidents_last_quarter`, `governance`). Model IDs are **dated pins**, not bare aliases.
- [ ] Every claim in every section resolves to a link (CI run, dashboard, PR, delegation contract, runbook, Grafana query, fixture file, or another artefact). No number is orphaned from evidence.
- [ ] Section 9 (known limitations) lists at least three real limitations. "None known" is a reject.
- [ ] The composition ledger (`docs/features/<feature>/feature-card-composition-ledger.md`) maps each of the nine sections to the artefacts it drew from. Any row marked GAP has a linked follow-up ticket.
- [ ] The reviewer's-questions companion (`docs/features/<feature>/feature-card-reviewer-notes.md`) authors four questions per reviewer role and answers each with a pointer into the card.
- [ ] Signature block contains three lines (applied engineer, PM, governance owner) with names, roles, and dates. If self-reviewed, the self-review notes are attached as a companion.
- [ ] `docs/features/<feature>/decisions/revisit-calendar.md` (from [exercise-03](exercise-03-build-vs-buy-matrix-gateway-eval-agent.md)) has been updated with the card's `next_review` date.
- [ ] The card's `governance` front-matter block declares NIST AI RMF functions covered, EU AI Act transparency applicability, and the ISO/IEC 42001 controls the feature touches (defer detail to [exercise-05](exercise-05-delegation-contracts-with-peer-specialists.md) and [chapter 06](../06-governance-mapping.md)).

## Stretch goals

Pick any one:

- **The aggregator dry-run.** Author a tiny Python script (or a `jq` one-liner) that reads the YAML front matter from your card plus 2–3 stub cards from imaginary sibling features, and produces a one-page cross-feature dashboard (CSV or Markdown table). Prove the front-matter schema is machine-consumable. Land the script as `tools/aggregate-feature-cards.py` (or similar).
- **The stale-card game.** Take your v1.0 card and fast-forward the calendar by two quarters. Simulate three plausible drifts (a new provider on the fallback slot, a judge recalibration that moved the score, an incident that added a fixture) and produce card **v1.1** with a change-log block at the top of the file. The stretch is: which sections were easiest to update, which were hardest, and which need substrate work?
- **The audit walk-through.** Sit with the card open and simulate a 30-minute external auditor session. For every claim the auditor might challenge ("prove this," "show me the evidence"), record the click-through path from the card to the source artefact. Any path that takes more than three clicks is a linkability defect — fix it.
- **The two-altitude cross-reference.** Draft a **system card** (or system-card slice) at the *product-line* altitude that composes your feature card plus two hypothetical sibling features. Where does the aggregation naturally happen? Where does the system card need something the feature cards do not provide? This is the shape the `ai-governance-analyst` produces from many feature cards.
- **The known-limitations forcing function.** Send the card to a colleague and ask them to write the honest "known limitations" section on your behalf, from the linked evidence alone. Compare against yours. Where they saw a limitation you did not, you have work to do.

## Deliverable

Submit as `mod-112/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/features/<feature>/feature-card-v1.0.md` — the composed card.
- `docs/features/<feature>/feature-card-composition-ledger.md` — the composition ledger with any gap rows.
- `docs/features/<feature>/feature-card-reviewer-notes.md` — the reviewer's-questions companion.
- `docs/features/<feature>/feature-card-self-review.md` (if signatures were self-reviewed).
- `docs/features/<feature>/decisions/revisit-calendar.md` — updated with the card's next-review date.
- A short `README.md` naming the feature, the substrates the card cites (and which are patterned vs real), and any assumptions carried forward from earlier exercises.
