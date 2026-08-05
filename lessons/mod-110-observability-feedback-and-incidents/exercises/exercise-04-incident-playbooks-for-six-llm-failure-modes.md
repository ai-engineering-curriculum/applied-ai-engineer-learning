# exercise-04: Incident Playbooks for the Six LLM Failure Modes

**Estimated effort:** 3 hours

## Objective

Author **six written playbooks** — one per failure mode from [chapter 04](../04-incident-playbooks-for-six-llm-failure-modes.md) — specialised to your feature, and prove they work by running a **game day** on at least three of them. The exercise's artefact is a set of playbook markdown files that a rotation engineer (who is not you) can execute at 3 a.m., plus the game-day report that shows the mitigation actually works.

## Learning outcomes

By the end you should be able to:

- Write an incident playbook with the five-part shape (detection, triage, mitigation, root cause, prevention) specific to a feature, not generic.
- Identify which alert-vs-signal detects each failure mode on your feature, given the span vocabulary from [chapter 01](../01-otel-genai-and-openinference-span-vocabulary.md).
- Ship the versioned-artefact rollback paths (prompt, model pin, retrieval index, guardrail config, pricing table) that make the mitigation fast and reversible.
- Run a game day: simulate a failure, execute the playbook, time the mitigation, produce a written report.

## Prerequisites

- You have read [chapter 04](../04-incident-playbooks-for-six-llm-failure-modes.md).
- Exercises 01, 02, and 03 are complete (or in progress with enough substrate — spans, sampled trace stream, feedback pipeline — to detect the failure modes).
- Access to the eval-gated CI from [mod-106 chapter 03](../../mod-106-feature-side-evaluation/03-eval-gated-ci-end-to-end.md) or a plausible substitute.
- Access to a staging environment where you can deliberately break things without paging real users.
- Two teammates who can play "on-call receiving the page" during the game day (they can be from adjacent teams, they should not be the feature author).

## Problem statement

Your team has never written down what to do when: the prompt push regresses; the provider goes down; the eval judge drifts; the provider raises prices; a silent quality regression sets in; the guardrail starts blocking too much. The last time one of these happened (a bad prompt push a month ago), the on-call spent 90 minutes reading Slack and eventually rolled back a random deploy on gut feel. It worked, but nobody could reproduce the reasoning.

In three hours: write six playbooks that would let the *next* on-call cut that 90 minutes to 15, then prove three of them work by rehearsing them on staging.

## Requirements

### Deliverable 1 — Six playbooks

Under `runbooks/playbooks/`, six markdown files, one per failure mode:

- `01-bad-prompt-push.md`
- `02-provider-outage.md`
- `03-judge-drift.md`
- `04-price-hike.md`
- `05-silent-quality-regression.md`
- `06-guardrail-fp-spike.md`

Each file uses the five-part template from [chapter 04](../04-incident-playbooks-for-six-llm-failure-modes.md): **detection, triage, mitigation, root cause, prevention.** Each section is specialised to *your* feature (not the chapter's generic examples):

- **Detection** names the specific alert (or the specific query the on-call runs), with the current threshold. If the alert does not exist yet in your alerting substrate, the playbook links to an open ticket to create it.
- **Triage** is a numbered list of concrete commands / links / queries. Not "investigate the dashboard" — a link to the specific saved query.
- **Mitigation** is a one-command rollback. If the versioned-artefact substrate does not exist yet for this failure mode (e.g., prompts are not versioned), the playbook says so and links to the ticket to add it. That is a *finding*, not a gap you should hide.
- **Root cause** lists the likely causes (drawn from [chapter 04](../04-incident-playbooks-for-six-llm-failure-modes.md)) and the artefact-diff to identify each.
- **Prevention** names the eval-gate / rollout-canary / monitoring change that would have caught this pre-production.

### Deliverable 2 — Alert coverage matrix

Under `runbooks/alert-coverage.md`, a table:

| Playbook                        | Detecting alert(s)                            | Owner                | Status       |
| ------------------------------- | --------------------------------------------- | -------------------- | ------------ |
| 01-bad-prompt-push              | `<feature>.thumbs_down.rate.spike`, `structural_validity.drop` | @agent-team | shipped     |
| 02-provider-outage              | `<feature>.provider.error_rate.high`, `<feature>.retry_rate.spike` | @agent-team | shipped     |
| 03-judge-drift                  | `<feature>.judge.calibration.drift`           | @eval-team           | not shipped |
| 04-price-hike                   | manual (provider release notes subscription)  | @finops              | shipped     |
| 05-silent-quality-regression    | `<feature>.online_eval.score.drift`           | @agent-team          | shipped     |
| 06-guardrail-fp-spike           | `<feature>.guardrail.block_rate.spike`        | @safety-team         | shipped     |

Every `not shipped` alert has a linked open ticket to create it. This table itself is the applied-AI engineer's alerting-gap report to the platform team.

### Deliverable 3 — The versioned-artefact audit

Under `runbooks/artefact-audit.md`, a table:

| Artefact              | Versioned? | Rollback command / path             | Last tested rollback |
| --------------------- | ---------- | ----------------------------------- | -------------------- |
| Prompt templates      | yes        | `deploy prompt --pin=v13`           | 2026-03-02           |
| Model version pin     | yes        | update `pinned_model` config, deploy | 2026-02-15         |
| Retrieval index       | yes        | `deploy index --snapshot=2026-03-10` | never                |
| Guardrail config      | yes        | update `guardrail.yaml`, deploy      | 2026-03-08         |
| Pricing table         | yes        | update `pricing.yaml`, deploy        | not applicable      |
| Judge model + prompt  | partial    | judge model pinned; judge prompt not versioned | never |

For every "not versioned" or "never tested" row, an open ticket. This audit is often the highest-value output of the whole exercise: you find out how many of your artefacts do not actually have a rollback path.

### Deliverable 4 — Kill-switch inventory

Under `runbooks/kill-switches.md`:

- Every kill switch the feature has, the command that fires it, and the effect on the user experience.
- The date each kill switch was last exercised in staging. Any older than 90 days is flagged; anything never exercised is a finding.

### Deliverable 5 — The game day

Pick **three** of the six playbooks. For each, plan and execute a staging-only game day:

1. Set up a scenario that fires the failure mode (deploy a deliberately regressed prompt template to staging; block the primary provider's endpoint at the network layer; change the pricing table to a wrong value; etc.).
2. Have a teammate (who did not build the scenario) receive the alert / notice the signal and run the playbook cold — reading only the playbook and the runbook links, not asking you.
3. Time the response from alert to mitigation.
4. Write a **game-day report** under `runbooks/game-days/YYYY-MM-DD-<playbook>.md`:

```markdown
# Game day — <playbook> — <date>

## Scenario
<what you broke, and how>

## Alert fired at
<time>. Detected by: <alert name or manual observation>.

## Playbook executed by
<name>. Time to mitigation: <minutes>.

## What worked
<the parts of the playbook that were unambiguous and executable>

## What broke
<the parts that were ambiguous, out of date, or missing entirely>

## Actions
- [ ] <fix to the playbook>
- [ ] <fix to the alert>
- [ ] <fix to the versioning / rollback substrate>
```

The game-day report is the artefact that turns aspirational playbooks into real ones. The action items are inputs to the next iteration.

## Starter guidance

- **Do not write the playbook in the abstract.** Sit with the feature's code and the trace UI open. Every "check X" step in triage is a specific link or query you can point at.
- **The playbook length is not the point.** A one-page playbook with three sharp mitigations beats a five-page playbook with philosophical asides. On-call reads for keywords at 3 a.m.
- **The versioned-artefact audit is a finding, not a proof of goodness.** Most teams discover that at least one artefact — often the judge prompt or the retrieval index — is not versioned. The audit is how you decide to ship the versioning next.
- **The game day teammate must not have written the feature.** If the author runs the playbook, all the tacit knowledge is invisible and the report is falsely rosy.
- **Time the mitigation from the alert firing, not from the human noticing.** The two are different by 5–30 minutes on a healthy setup, longer on a bad one, and both matter.
- **Do not run the game day in production.** Staging only, on a scenario that cannot page real users. If your staging environment does not have enough traffic or fidelity to fire the alert, that itself is a finding.

## Acceptance criteria

- [ ] Six playbook files exist under `runbooks/playbooks/`, each using the five-part template, each specialised to the feature.
- [ ] Every triage step in every playbook links to a specific saved query, dashboard, or command — not a generic "investigate."
- [ ] Every mitigation is a one-command action or is called out as a finding with a ticket link.
- [ ] `runbooks/alert-coverage.md` shows every playbook's detecting alert with `shipped` / `not shipped` status and an owner.
- [ ] `runbooks/artefact-audit.md` shows every artefact's versioning status and last-tested-rollback date.
- [ ] `runbooks/kill-switches.md` inventories the kill switches with last-exercised dates.
- [ ] At least three game days have been run on staging, with a written report per game day.
- [ ] Each game-day report has action items filed as follow-up tickets.
- [ ] No game day ran against production or paged real users.

## Stretch goals

Pick any one:

- **Run all six game days.** Report which playbooks were fastest / slowest to mitigate and which alerts were noisiest / quietest. The comparative report is a leadership-audience artefact.
- **Chaos automation.** Wire a small chaos harness (a script that deploys a deliberately regressed prompt to staging, or blocks the provider endpoint, or bumps the guardrail threshold) so game days can be re-run monthly with one command. This is the substrate for a *continuous* rehearsal cadence, not a one-off exercise.
- **A cross-playbook alert consolidation.** Look at the six alerts. Do any of them fire on the same signal from different angles? Consolidate where legitimate; keep separate where the mitigation differs. This is alert hygiene work that most teams neglect.
- **A "playbook staleness" tripwire.** For each playbook, record the last time it was executed (either in a real incident or a game day). Alert when any playbook has not been touched in >6 months — that is the sign it is decaying.

## Deliverable

Submit as `mod-110/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `runbooks/playbooks/01-*.md` … `runbooks/playbooks/06-*.md`.
- `runbooks/alert-coverage.md`, `runbooks/artefact-audit.md`, `runbooks/kill-switches.md`.
- `runbooks/game-days/YYYY-MM-DD-*.md` (at least three).
- A short `README.md` naming the feature, the game-day cadence, and the top three findings from the audits.
