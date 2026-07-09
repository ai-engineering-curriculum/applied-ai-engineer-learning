# Leaderboard-Driven Failure Modes

## Why this chapter exists

Public leaderboards — MMLU, MMLU-Pro, GPQA, HumanEval, SWE-bench, LMArena, Chatbot Arena, WebArena, τ-bench, and the constantly-refreshing lab announcements — are useful *shortlist* signals. They tell you which models are in the conversation for a *class* of task and which are behind. They do not tell you which model will hit your surface's SLO on your production distribution.

Confusing the two is the most common failure mode in applied-AI-engineer interviews and in real feature reviews. It also costs money. This chapter names the four failure modes and gives you the review question that catches each one before it ships.

Read this chapter as the checklist you use on **your own** trade-off note before circulating it, and as the checklist you use on **someone else's** proposal before you sign it.

## Failure mode 1 — SOTA-worship

**The pattern.** "We should use the top-of-the-leaderboard model because it has the highest score." The score in question is on a public benchmark (MMLU, GPQA, LMArena, SWE-bench) whose task distribution is not your task distribution and whose grading rubric is not your grading rubric.

**Why it fails.** Leaderboards optimise for legibility across many models on a fixed task; your surface has one distribution and one rubric. A model that is 3 points ahead on MMLU can be *worse* on your extraction task, worse on your groundedness metric, or worse in exactly the failure mode your users complain about. The public score is a hint at membership in the shortlist, not evidence for the choice.

**The concrete risk.** You choose the more expensive model because it's SOTA, ship it, and your cost-per-successful-outcome SLO is missed by a factor of two. Or, worse, you underspec your own eval slice because "the SOTA benchmark score is high enough" and never catch that your surface's rubric is different.

**The review question.** *What is the model's score on our internal eval slice, on our production-shaped inputs, using our task-success rubric?* If the answer is "we haven't measured that yet," the choice is not defensible.

**How to reject it in review.** "I hear that Model X is SOTA on GPQA. What's the score on our eval slice? Until we have that number, I'm not comfortable signing Section 4 of the one-pager."

## Failure mode 2 — Benchmark-vs-surface mismatch

**The pattern.** The benchmark measures a *related* skill, not the skill your surface actually requires. HumanEval measures function-level Python correctness; your surface is a multi-file code-review co-pilot on a private monorepo. MMLU measures four-way multiple-choice on academic questions; your surface is open-ended customer support with citations. Chatbot Arena measures side-by-side preference on prompts a random Discord user typed; your surface is a regulated-industry Q&A whose users have expert domain knowledge.

**Why it fails.** Skill-transfer from benchmark to surface is not uniform. A model that beats the field on the benchmark can be middling on your surface because the benchmark rewards patterns that don't show up in your inputs, or fails to reward patterns that do.

**The concrete risk.** A frontier model that wins on Chatbot Arena but produces breezy, unsourced prose is a bad choice for a legal-QA surface where a well-cited "I don't have evidence for that" is worth more than a confident-sounding answer. You'll ship it, users will lose trust, and the incident will be blamed on "hallucination" when the real cause is a benchmark that never measured the failure mode you cared about.

**The concrete risk (extraction).** HumanEval-style code benchmarks reward function-level output that runs against a hidden test; your extraction surface needs field-level precision on a schema. High HumanEval does not predict good extraction, or good tool-argument generation, or good long-context recall.

**The review question.** *What skill does the benchmark measure, and how close is that skill to the skill our surface exercises?* If the two don't line up, the leaderboard is a shortlist hint at best.

**How to reject it in review.** "SWE-bench Verified measures multi-file bug-fix on public GitHub issues under a specific harness. Our surface is a code-review co-pilot on our private monorepo with our lint rules and our reviewer conventions. The transfer isn't obvious. I want a slice on our repos before I sign."

## Failure mode 3 — One-model lock-in

**The pattern.** The prompt, the structured-output layer, the caching strategy, the guardrail plumbing, and the eval fixtures are all coupled to one specific provider's dialects and quirks. Swapping any part out for a bake-off (mod-108) or an incident-time fallback (mod-110) requires a rewrite.

**Why it fails.** Providers have outages. Providers change prices. Providers deprecate models on their own timeline (a mid-tier model can be retired with 3–6 months notice; sometimes less). Providers roll out new terms, new regions, new residency rules. If your feature can only run on one provider, every one of those events is a potential incident, and every one of them costs you optionality on price and quality.

**The concrete risks.**

- **Provider outage.** Frontier providers have all had multi-hour incidents. If the fallback is untested, the outage becomes yours.
- **Price change.** A 20 % price hike on the primary provider changes the cost-per-successful-outcome SLO overnight. If routing is not possible, the SLO is missed until the next redeploy.
- **Model deprecation.** A production model getting deprecated in 90 days is a real forcing function. If the feature is coupled to that model's specific quirks, the migration is a full re-eval.
- **Router bake-off is impossible.** mod-108 assumes portability. Without it, "let's try a mid-tier model at 5× less cost" becomes a rewrite, not an experiment.

**The review question.** *If our primary provider went down for four hours right now, what does the user experience? What SLOs regress and by how much?* If the answer is "the feature is offline" and there is no offline mode, that is a coupling risk you are living with.

**How to reject it in review.** "Section 4 says the structured output uses Provider A's native tool-call format. Have we contract-tested the fallback path with Instructor + Pydantic on the open-weights fallback? Section 8 says we shadow for a week — does that include the fallback provider, or only the primary?"

## Failure mode 4 — Unmeasured cost

**The pattern.** The feature ships with an unstated cost model. Nobody knows what the per-successful-outcome cost actually is, because it isn't a first-class metric. When someone finally does the math, the number is 5× higher than the informal budget the PM was holding in their head.

**Why it fails.** LLM calls are metered. Retries are metered. Guardrail calls are metered. Judge calls (when a judge is used online for anything, mod-106) are metered. Multi-step agents call the model many times per request; tool-calling features call the provider more than once per user turn. Streaming looks free but costs the same. Prompt caching is a legitimate lever, but only if the system prompt is stable enough to hit the cache — every prompt change is a cost-model change.

**The concrete risks.**

- **Runaway feature cost.** An unmonitored autonomous-agent surface can burn its yearly budget in a week the first time an injected instruction sends it into a tool-loop.
- **Wrong tier locked in.** A frontier model was picked without measuring cost-per-successful-outcome; when the number is finally computed, the mid-tier model would have hit the same success rate at 20 % of the cost.
- **Per-tenant blindness.** A single power tenant drives 40 % of the cost. Without per-tenant accounting (mod-109), the median tenant looks fine while the business bleeds.
- **Judge inflation.** The online judge is a frontier model; nobody thought to add it to the cost model. It costs more than the primary model.

**The review question.** *What is our cost per successful outcome today, and what is our target? What is the target's basis?* If either is missing, the feature is not ready to ship at a scale where cost matters.

**How to reject it in review.** "Section 3 has a cost-per-successful-outcome SLO of ≤ $0.04. What is it today on our shadow traffic, and what's the breakdown across primary calls, retries, guardrail calls, and any judge calls? I want the number before I sign."

## The failure modes interact

The four are not independent — they reinforce each other.

- SOTA-worship makes one-model lock-in worse (the SOTA model is the *reason* to couple to that provider).
- Benchmark-vs-surface mismatch makes unmeasured cost worse (you're paying for a capability the benchmark measured but your surface doesn't use).
- One-model lock-in and unmeasured cost together make provider-price-change incidents unsurvivable.
- SOTA-worship and unmeasured cost together produce the classic "we spent our whole feature budget on tokens for a quality gain users can't feel" outcome.

If a proposal shows one, look for the others.

## Rejecting them in your own review

A defensible one-pager (chapter 04) contains the anti-patterns' antidotes in-line:

1. **Section 3** (SLOs) states the internal eval slice and rubric explicitly, not "matches SOTA on X." (Anti-SOTA-worship, anti-benchmark-mismatch.)
2. **Section 3** includes a concrete cost-per-successful-outcome target with a basis. (Anti-unmeasured-cost.)
3. **Section 4** (provider + tier) names the fallback and states that it is contract-tested in CI. (Anti-one-model-lock-in.)
4. **Section 4** includes the portability accounting from chapter 03 — which prompt patterns, structured-output mechanisms, and caching strategies are provider-specific. (Anti-one-model-lock-in.)
5. **Section 8** (kill criteria) includes a cost-based trigger, not only a quality-based one. (Anti-unmeasured-cost.)

If any of these are missing, the doc is not defensible against a serious review, even if it looks tidy.

## Summary

- Four failure modes to reject in review — in others' work and your own:
  - **SOTA-worship** — public benchmark scores as evidence rather than shortlist hint.
  - **Benchmark-vs-surface mismatch** — a benchmark that measures a related-but-different skill.
  - **One-model lock-in** — a prompt / structured-output / caching stack coupled to a single provider's dialects.
  - **Unmeasured cost** — no first-class cost-per-successful-outcome metric with a basis.
- The four interact. Presence of one is a signal to look for the others.
- The one-pager (chapter 04) is the vaccine — internal eval slice and rubric, explicit cost target, portability accounting, contract-tested fallback, cost-based kill criterion.
- Leaderboards are for building the shortlist. Your eval slice is for deciding the choice.

That closes the module. You now have — for the next feature you're asked to review — a shape catalogue, an SLO framework, a provider-tier trade-off shape, a one-page decision doc, and a review checklist. The rest of the track (mod-102 through mod-112) turns that decision into a shipped feature and operates it in production.
