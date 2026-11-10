# Eval Suites from Benchmark Shapes

## The right way to consume a public benchmark

Public agent benchmarks — τ-bench, SWE-bench (and its Verified subset), WebArena, AgentDojo — are among the most important artefacts in the field. They give the community a common yardstick, they let providers publish comparable numbers, and they inspire the tooling that trickles down into every product agent. They are also, in almost every case, the wrong evaluation to run on your product surface.

The applied-AI engineer's move is to consume public benchmarks as **shape templates**, not as targets. You steal the *structure* — how they define a task, what the environment looks like, what constitutes success, how they simulate a user — and you rebuild the fixture from your own product tasks, your own tool catalogue, and your own success criteria. The score you care about is on your suite, not theirs. The public suite tells you the shape; the internal suite tells you if you're shipping.

This chapter walks through the four benchmarks named in the module's learning objectives, extracts the shape template from each, and shows how to fold that template into an internal eval fixture for a product agent. It is deliberately not a "how to top τ-bench" chapter — mod-106 goes deep on the eval loop, and the peer track handles benchmark-topping research. This is the applied engineer's move: **steal the shape, throw away the target.**

## The shape a good agent benchmark exposes

Every benchmark worth stealing from exposes the same five parts, however differently they name them:

1. **A task specification.** What the agent is being asked to do, as a natural-language instruction or a structured intent. Public benchmarks make this precise and adversarial; you will make yours precise and *representative of real user asks*.
2. **An environment.** Files on disk, a browser DOM, an inbox, a customer record. The environment has a definite state before the task and a definite state after.
3. **A tool catalogue.** The concrete tools the agent may call. Public benchmarks fix the catalogue so that runs are comparable; you fix the catalogue to match production, then vary it deliberately for ablations.
4. **A success criterion.** What counts as done. This is where public and internal suites most sharply diverge: public benchmarks lean on programmatic checks (does this test pass? is this DOM state reached?); product suites lean on outcome checks that map to user-visible artefacts.
5. **A user simulator (or a scripted user).** For interactive shapes, the benchmark needs to simulate the user's replies. τ-bench uses an LLM-driven simulator; WebArena uses fixed intents; SWE-bench Verified uses static PR bodies. You will pick a simulator strategy per shape.

Any benchmark that gives you all five is a template you can steal. Any internal suite that omits one of them is missing a load-bearing piece.

## τ-bench — the customer-experience template

**What it is.** Sierra's τ-bench (also written "tau-bench") is a benchmark for tool-using agents in customer-support-style tasks. It exposes two environments, `retail` and `airline`, each with a domain-realistic tool catalogue (order lookup, cancellation, exchange, refund; reservation lookup, seat change, refund) and a policy document the agent must obey. A user simulator (an LLM prompted with a customer persona and an intent) drives the conversation. Success is a programmatic check with two parts: the paper's reward is the product of a database-state check (does the final backend state match the annotated goal state?) and an output check (does the agent's response convey the required information back to the user?) — see [Yao et al., 2024](https://arxiv.org/abs/2406.12045) and the reference implementation at [`sierra-research/tau-bench`](https://github.com/sierra-research/tau-bench).

**Why it matters.** τ-bench is the closest public artefact to the CX shape from [chapter 01](01-agent-shape-for-a-real-product-surface.md). It captures three things a real product needs:
- A conversation that can span many turns.
- Tools that mutate real-shaped state.
- A policy layer the agent must follow (refund only in these conditions; exchange only within N days).

**The shape template.** From τ-bench, steal:

- **The "environment + policy + tools + user simulator" package.** A τ-bench-shaped internal suite has *all four*. Environments come from staging copies of production systems (never from production; policy failures in eval must not touch real state). Policy is a document the retrieval layer can find (mod-103). Tools are the same tools production uses. User simulators are separate LLMs — often a smaller, cheaper model — prompted with production-representative personas.
- **The "goal state" success criterion.** For a support agent, the outcome is the *state of the customer record after the conversation*, not the transcript quality. A refund of the correct amount to the correct account is the check. Transcript quality is a separate axis, evaluated separately.
- **The pass^k metric.** τ-bench emphasises reliability under repeated runs — an agent that succeeds 4/5 times on a task is not the same product as one that succeeds 1/5, even if the average is the same. Internal suites should measure both mean success and consistency; a low-variance failure is easier to debug than a high-variance flake.

**What to throw away.** The public tasks themselves. Your customers do not have τ-bench's `retail` or `airline` schemas; they have yours. The stolen shape is the *structure*, not the data.

**Adapting for a real CX surface.** Concretely, a τ-bench-shaped internal suite for a support agent for a subscription product looks like:

- 50-200 tasks curated from real ticket data (with PII scrubbed), each specifying a user's initial message, the ground-truth "goal state" of the account after resolution (e.g., "subscription paused for 30 days; support case closed with reason `pause_request`"), and the policy the agent must obey.
- A user simulator — a smaller model prompted with the customer's persona and one of a small library of "conversational styles" (short-tempered, technically fluent, non-native speaker, distracted). Run each task with multiple simulators to test robustness.
- The same production tool catalogue. Not a mock; the real tools pointed at a staging tenant of the real backend.
- Automated success checks against the staging tenant's state after each run. A transcript rubric (LLM-as-judge, calibrated as in mod-102) runs in parallel to score tone / policy compliance.

## SWE-bench Verified — the coding-agent template

**What it is.** SWE-bench is a benchmark of real GitHub issues paired with the PRs that resolved them. An agent gets the issue text and the repo state before the fix; it must produce a patch that makes the hidden test suite pass. SWE-bench Verified is a 500-task filtered subset that the SWE-bench team and OpenAI validated to remove problematic tasks (broken environments, spec-only tasks with unresolvable issues, hidden tests that are actually broken) — see OpenAI's announcement [Introducing SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/). Verified is the number to quote when you're comparing coding agents seriously.

**Why it matters.** SWE-bench Verified is the closest public artefact to the workspace tier of Shape A from [chapter 01](01-agent-shape-for-a-real-product-surface.md) — a coding agent that reads a spec, navigates a repo, produces a diff. It captures:

- A real, cluttered repository (not a leetcode-style small file).
- A test suite as the success criterion.
- A minimum viable environment (a runtime and a build system) the agent has to work inside.

**The shape template.** Steal:

- **The "issue → hidden tests → patch" flow.** For an internal coding-agent suite, gather issues from your own tracker, pair each with a *hidden* test the correct fix would make pass, and evaluate the agent's patch by running your test suite. The hidden test is what keeps the eval honest — the agent cannot memorise the fix if the fix isn't in-distribution to it.
- **The "cluttered environment" fidelity.** The eval runs against a real checkout of the repo at a real prior commit, with real dependencies. A minimal-repro fixture is a unit test, not an agent eval.
- **The Verified discipline.** Before you rely on a number from your internal suite, someone reviews the tasks. Broken tasks, mislabelled tasks, and unresolvable spec-only tasks get removed. Otherwise your headline number is measuring your fixture quality, not your agent quality.

**What to throw away.** The specific SWE-bench repos. Your team's coding agent is judged by how well it works on *your* codebase, not `psf/requests` and `sphinx-doc/sphinx`. Steal the flow; supply your own repos.

**Adapting for a real coding-agent surface.** For a workspace-tier coding agent shipped to an internal engineering team:

- Curate 50-200 real historical issues from your team's tracker. Each is paired with the ground-truth fix (from the merged PR) and, ideally, the hidden test that failed before and passes after.
- Snapshot the repo state at the issue's report time and run the agent against that snapshot in a container.
- Evaluate: does the agent's patch pass the hidden test? Does it pass the pre-existing test suite? Does it modify only the intended files (a proxy for scope discipline)? Does it match the reviewer's stylistic conventions (a rubric metric, not a hard gate)?
- Track pass^k the same way τ-bench does. A coding agent that solves a task 3/5 times is not the same product as one that solves it 5/5 — even for pair-programming shapes, retry cost is real.

## WebArena — the browsing-agent template

**What it is.** WebArena is a benchmark of 812 realistic web tasks against self-hosted copies of real-world applications (e-commerce, Reddit-style forum, GitLab, an OpenStreetMap clone) — see [Zhou et al., 2023](https://arxiv.org/abs/2307.13854) and [`web-arena-x/webarena`](https://github.com/web-arena-x/webarena). Tasks are given as natural-language intents ("comment on the latest post in the /r/foo subreddit"); success is execution-based — for action tasks the harness inspects the resulting environment / URL state, and for information-seeking tasks it compares the agent's answer against a reference via exact-match, must-include, or fuzzy-match. VisualWebArena extends the idea to visually-driven pages with 910 additional tasks ([Koh et al., 2024](https://arxiv.org/abs/2401.13649)).

**Why it matters.** WebArena is the closest public artefact to Shape D from [chapter 01](01-agent-shape-for-a-real-product-surface.md) — a browsing agent driving a real DOM. It captures:

- A real (self-hosted, realistic) web application, not a toy sandbox.
- Task specifications close to how a user would phrase a delegated task.
- Success criteria grounded in application state, not chain-of-thought quality.

**The shape template.** Steal:

- **The "hosted realistic app" discipline.** A browsing agent's internal eval runs against a hosted copy of the target site (or a fixture built to look like it), not against a live production page. Live pages A/B test, rate-limit, and change nightly; you cannot run a suite of 200 tasks against them and reason about the results.
- **The intent-to-state success criterion.** The eval checks whether the *state of the site* matches the intended outcome after the agent's run. Screenshots and transcripts are diagnostic; state checks are the pass/fail gate.
- **The task diversity discipline.** WebArena categorises tasks by verb (create, delete, edit, find), by depth (number of pages navigated), and by domain (which app). Internal suites should mirror this: an eval that only measures easy find-and-read tasks will not catch regressions on multi-step form fills.

**What to throw away.** The specific WebArena environments unless you happen to have a product that browses e-commerce or GitLab in the same shape. For most product browsers, you will build your own hosted-copy fixture.

**Adapting for a real browsing-agent surface.** For a browsing agent that automates a category of external tasks (say, job applications or purchase-order forms):

- Host 3-5 realistic exemplars of the target site type. If you can, host mirrors of the real vendors' pages with staging accounts; if not, build clones that match the DOM shape.
- Curate 30-100 tasks that span the verb / depth / domain matrix. Each specifies the initial URL, the intent, the ground-truth end-state, and any user-authoritative pieces (login credentials for a test account, a payment token, etc.).
- Instrument every run with screenshots and a DOM diff at each step so a failed task is debuggable.
- Track a *safety metric* alongside success: how many runs took an action the user did not authorise (an off-goal purchase, a form submission the task did not require)? A benchmark that ignores unauthorised actions is not measuring what a browsing-agent product cares about.

## AgentDojo — the adversarial-tool-use template

**What it is.** AgentDojo (from Zurich's SPY Lab) is a benchmark for evaluating an agent's robustness to prompt-injection attacks delivered through tool results. The agent is given a benign user task; the tool responses contain adversarial payloads (in field values, in file contents, in email bodies, in webpage text) attempting to redirect the agent. The benchmark reports three distinct metrics: **benign utility** (did the agent solve the user task with no attack present?), **utility under attack** (did the agent solve the user task without also executing the attacker's side-effects?), and **targeted attack success rate** (did the attacker's goal get executed?) — see [Debenedetti et al., 2024](https://arxiv.org/abs/2406.13352).

**Why it matters.** AgentDojo is the applied engineer's honest reminder that **eval is not just about capability, it's about robustness**. Every product agent that reads user-controlled content (email, documents, webpages, customer messages, tool outputs) is exposed to prompt injection. mod-107 owns the guardrails story; this chapter's contribution is the eval fixture that measures whether your guardrails work.

**The shape template.** Steal:

- **The "utility + security" dual metric.** Every eval task carries a utility check (did the agent do the thing?) and, in the adversarial mode, a security check (did the agent stay on-task under the attack?). The two are reported separately; a "high-utility" agent that is fully hijackable is a shipped incident.
- **The "attack surface" catalogue.** AgentDojo enumerates where injections can come from: tool result fields, document contents, email bodies, external HTTP responses. Internal suites should keep an equivalent catalogue for their surface: which tools return content the agent will read, where does that content come from, and what's the attack posture on each?
- **The "targeted goal" attacks.** Injections that just say "ignore your instructions" are easy; injections with realistic hostile goals ("transfer $100 to this account") are the hard cases. Internal red-teaming should include the hard cases.

**What to throw away.** The specific attack corpus. Your surface has a specific set of tools with specific result shapes; an internal AgentDojo-shaped suite carries injection payloads targeted at *your* tools' outputs, not the public benchmark's fixtures.

**Adapting for a real product surface.** For any product agent that reads user- or third-party-controlled content:

- Catalogue the read surfaces: which tools' outputs will be interpreted by the model? For each, note the content's origin (the user, another user, a third-party site, an internal system).
- Build an injection corpus per read surface: benign-looking payloads that attempt to redirect the agent, escalate its privileges, or leak content it shouldn't disclose.
- Run every eval task in two modes: clean (tools return their normal outputs) and adversarial (payloads injected at one or more read surfaces). Report utility-clean, utility-under-attack, and hijack rate as three separate numbers.
- Rotate the corpus as your guardrails improve. A static injection suite that the guardrails have memorised is not measuring robustness; it's measuring memorisation.

## Composing an internal suite from the four templates

For a real product agent, the four templates rarely apply cleanly one at a time. A retrieval-with-actions surface (Shape C) needs τ-bench discipline for tool use and AgentDojo discipline for the documents it reads. A coding co-pilot with browsing (Shape A + D) needs SWE-bench discipline for the diff evaluation and WebArena discipline for the browsing portion.

The composition rule: **one internal suite per shape, drawing on one or more public shape templates.** For each shape you ship, the suite specifies:

- The task list (curated from real user tasks, PII-scrubbed).
- The environment (staging or hosted copy, never production).
- The tool catalogue (production tools pointed at staging).
- The success criteria (state checks on the environment, plus rubric metrics for user-facing tone).
- The user / attack simulator strategy (LLM-driven for interactive shapes, static intent for browsing, adversarial for read surfaces).
- The reporting shape (mean success, pass^k, security metrics where applicable, cost per task, latency per task).

The suite is versioned. New tasks land through PRs. Task removals require justification (task was broken, task no longer represents production traffic). The suite runs on every model / prompt change (mod-106 owns this loop) and its results are what gate a release, not the public benchmark score.

## The applied-vs-agentic split on eval

This is the second place in the module where the split with the agentic-AI peer role becomes concrete.

**The peer role** owns the agent engine's benchmark story — topping τ-bench and SWE-bench Verified, publishing WebArena results, evaluating new planner / executor architectures against AgentDojo. Their metrics are on public suites; their work is legible to the research community; their releases are gated on benchmark progress.

**The applied engineer** owns the *product* agent's eval — task lists derived from real user tasks, environments staged from real backends, success criteria that map to user-visible outcomes, cost / latency measured in the shape of the SLO. Their metrics are on internal suites; their work is legible to the product org; their releases are gated on user outcomes.

Two consequences worth naming.

- **A benchmark number is not a product signal.** A model that jumped from 40% to 55% on τ-bench in a provider release may or may not help your CX agent; you learn that from *your* suite, not theirs. This is the honest reason "please rerun eval on the new model before we roll out" is a mod-106 discipline, not paranoia.
- **Building the internal suite is a durable investment.** Public benchmarks change. Providers update. Tasks get retired. Your internal suite is the artefact that lets you tell whether the agent got better on the work your users actually do. Ship it early, keep it small, and grow it deliberately.

## Anti-patterns

Six failures at the eval-composition layer.

1. **"We're at 62% on τ-bench so we're ready to ship."** τ-bench measures τ-bench. Your users have never opened it. The provider score correlates with the shape template's difficulty, not with your surface's health.
2. **The single-task fixture.** A suite with 5 hand-picked tasks that the agent passes every time. Any regression that misses those 5 tasks is invisible. Aim for at least 50; more is better.
3. **The static gold-answer fixture.** Every task has one exact string the agent must produce. Most product tasks have many acceptable outcomes; measure against *state*, not string.
4. **The prod-tenant eval.** Running the eval suite against production data because "staging isn't representative." Every eval run touches real users. This is not eval; this is a controlled outage.
5. **The utility-only eval.** No adversarial mode, no security metric, no injection corpus. The first exploit lands live.
6. **The mock-tool eval.** The suite mocks every tool response. The agent's tool-use behaviour is tested against a fixture, not against the actual tool's schema and errors. This catches nothing except mock drift.

## Summary

- Public benchmarks (τ-bench, SWE-bench Verified, WebArena, AgentDojo) are shape templates for internal eval, not targets. Steal the structure, throw away the data.
- Every good agent benchmark exposes five parts: task spec, environment, tool catalogue, success criterion, and user (or attacker) simulator. An internal suite that omits one is missing a load-bearing piece.
- τ-bench → the CX shape template: environment + policy + tools + user simulator, with state-based success and pass^k reliability metrics. Ship one per CX product.
- SWE-bench Verified → the coding-agent template: issue + hidden test + repo snapshot + patch evaluation. Curate from your own tracker; run against real repo state, not minimal repros.
- WebArena → the browsing-agent template: hosted realistic site + intent + state-based success + task diversity across verb / depth / domain. Never eval against a live production page.
- AgentDojo → the adversarial-tool-use template: utility + security dual metric, injection corpus per read surface, targeted goals not just "ignore instructions." Rotate the corpus as guardrails improve.
- Compose one internal suite per shipped shape, drawing on one or more templates. The suite is versioned and gated on real user outcomes; the public benchmark is a signal to the peer role, not a release gate for you.

[Chapter 05](05-public-applied-agent-patterns.md) closes the module by looking at four public applied-agent patterns — Cursor Composer, GitHub Copilot Workspace, Sierra CX agents, and Harvey — and extracting the design choices that actually shipped. The applied-engineer moves in this chapter are the same moves those teams made, sometimes hard-won.
