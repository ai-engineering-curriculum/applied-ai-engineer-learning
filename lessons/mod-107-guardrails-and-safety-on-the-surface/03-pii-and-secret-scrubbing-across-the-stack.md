# PII and Secret Scrubbing Across the Stack

## Telemetry is a data-leak channel

The single largest surface-side incident category *besides* moderation and injection is data leakage — through prompts, through outputs, and above all through the traces and telemetry that capture both. The pattern is depressingly consistent: the team ships a feature with careful production-data handling, then instruments it with a tracing SDK that sends full request bodies to a hosted observability vendor, and now the customer's API keys, session tokens, and health disclosures live in a third-party's storage indefinitely.

Every serious incident review of an LLM product I have ever read includes a version of *"we didn't realise the tracer captured that."* This chapter is about closing that gap. The applied engineer owns four scrubbing points — inputs, prompts, outputs, and traces — and the discipline that keeps the four consistent.

The chapter also draws a boundary. The `data-security-engineer` and `privacy-engineer` peer roles own the legal / regulatory foundations (GDPR / CCPA / HIPAA / GLBA / PCI substrate, the DPIA process, cross-border transfer analysis). The applied engineer's job is to *ship* code that respects those foundations on their specific surface. This chapter is about the shipping.

## The four scrubbing points

Every LLM feature has four places where sensitive data can enter a system that will retain it longer than the request. Naming the four is the first move.

**Point 1 — Inputs, at the surface boundary.** The user's message enters the request pipeline. Anything sensitive in it — a full email, an API key pasted while asking for help, a paragraph containing a medical record — will now travel through every downstream system. Point-1 scrubbing catches secrets before they hit the LLM provider (whose retention is not yours), before they hit the prompt cache, and before they land in the tracer.

**Point 2 — The rendered prompt, before it leaves for the model provider.** The system prompt is templated; the user message is included; retrieved chunks are inlined; tool schemas are rendered. Anything sensitive in *any* of the four ends up on the provider's servers. Point-2 scrubbing is the last chance before the network hop. It is also where prompts constructed by *code you did not write* (framework internals, MCP server responses) get inspected for the first time.

**Point 3 — Outputs, before render.** The model's response can contain PII or secrets that were injected by retrieval, that the user themselves supplied earlier in the session, or that the model hallucinated but happens to be real (a synthesised-but-valid credit-card-number-shaped string). Output-side scrubbing catches these before render, prevents echoing customer A's data into customer B's session, and enforces the "no unmoderated bytes to the user" story from chapter 01.

**Point 4 — Traces and telemetry, before they land in the observability store.** The full request, the full prompt, the full response, the tool arguments, the tool results, the retrieval context — anything the tracer captures gets scrubbed here. This is the point most incident post-mortems name and the one most codebases forget.

A useful mental model: **treat the tracer with the same rigour as the model provider.** You would not send raw customer secrets to `example-llm-provider.com`. You should not send them to `example-tracing-vendor.com` either. The tracer is not free-of-obligation just because it wears an observability hat.

## What "sensitive" actually means on your surface

Scrubbing rules that are vague will produce scrubbing behaviour that is vague. Before writing any regex, the surface needs a **data classification** that names the categories:

- **Secrets.** Machine-generated credentials: API keys, OAuth tokens, session tokens, private keys, database URLs with embedded passwords, JWTs. High confidence to detect (the shapes are constrained); catastrophic if leaked; always redact in every direction.
- **Personal identifiers (PII).** Names, email addresses, phone numbers, physical addresses, IP addresses, government IDs (SSN, national ID, passport, tax ID), payment card numbers. Regulated in most jurisdictions; the redaction contract is bounded by the surface's legitimate need to see them.
- **Special-category personal data.** Health, financial, biometric, immigration status, sexual orientation, religion, precise geolocation. The regulatory anchors: [GDPR Article 9](https://gdpr-info.eu/art-9-gdpr/) names eight special categories (racial or ethnic origin; political opinions; religious or philosophical beliefs; trade-union membership; genetic data; biometric data processed to uniquely identify a person; data concerning health; data concerning sex life or sexual orientation); HIPAA defines [Protected Health Information at 45 CFR § 160.103](https://www.ecfr.gov/current/title-45/section-160.103) as individually identifiable health information held or transmitted by a covered entity or business associate in any form; [PCI DSS v4.0.1](https://www.pcisecuritystandards.org/) scopes "cardholder data" to the Primary Account Number (PAN) plus, when stored alongside the PAN, cardholder name, expiration date, and service code (Sensitive Authentication Data — full track, CVV2/CVC2/CID, PIN/PIN block — must never be stored post-authorisation); GLBA covers nonpublic personal financial information. Any surface touching these needs a compliance sign-off on the scrubbing design, not just an engineering sign-off.
- **Business-confidential data.** Internal customer names, internal metrics, unpublished pricing, unshipped roadmap items. Not always regulated, always damaging on leak. Often the category most under-scrubbed because it doesn't fit a regulator's checklist.
- **Contextually sensitive data.** Data that is fine in one destination and not another. A user's email is fine in a chat rendered to that same user; it is a leak in a support-agent transcript sent to a shared team channel. Contextual sensitivity is why "scrub PII from all logs" is coarse — the right rule is *scrub PII from logs that will be read by anyone other than the request's own user*.

For every category, the design names the **detector**, the **replacement**, and the **audit obligation**. A single "scrub PII" toggle is a lie; five categories is a working floor.

## Detection: the tool catalogue

Four common detection substrates. Most working systems use two or three in combination, exactly as chapter 01 uses two moderation layers.

- **Regex / heuristic detectors** — high precision on shape-constrained secrets (API-key prefixes, credit card LUHN checks, SSN patterns, common JWT structure). Fast, cheap, deterministic. Poor recall on free-text PII (names cannot be regexed reliably). Every secret-scanner — Yelp's [`detect-secrets`](https://github.com/Yelp/detect-secrets) (actively maintained), Trufflesecurity's [TruffleHog](https://github.com/trufflesecurity/trufflehog) (actively maintained, with provider-API verification), AWS Labs' [`git-secrets`](https://github.com/awslabs/git-secrets) (low activity but still the canonical reference for provider-key prefix patterns), and [GitHub's secret-scanning partner-pattern catalogue](https://docs.github.com/en/code-security/secret-scanning/introduction/supported-secret-scanning-patterns) — is a curated regex catalogue plus verification.
- **Statistical / ML PII detectors** — [Presidio](https://presidio.dataprivacystack.org/) (which has transitioned from Microsoft's org to community stewardship under the Data Privacy Stack project) is the reference open-source example: named-entity recognition plus recogniser rules plus a redaction stage. Its [supported entities list](https://presidio.dataprivacystack.org/supported_entities/) covers global entities (credit card, email, IP, phone, IBAN, crypto, person, URL, medical licence) plus country-specific recognisers (US SSN / DL / ITIN / passport, UK NHS / DL, and others for Spain, Italy, Poland, Singapore, Australia, India, and more). English is the default; [other languages](https://presidio.dataprivacystack.org/analyzer/languages/) are supported via swapped-in spaCy / transformers NLP engines plus localised context words. Complements regex: catches free-text PII that regex cannot.
- **Cloud DLP APIs** — AWS Comprehend PII detection, Google Cloud Sensitive Data Protection (formerly Cloud DLP), Azure AI Content Safety's PII detection surface. Managed, kept current, priced per call. Convenient when you already live in one cloud.
- **LLM-based detectors** — the model itself scores each span. Strong recall on subtle categories (a paragraph describing a health condition without ever naming a disease). Expensive; introduces its own hallucination risk (a detector that says "there is no PII here" when there is). Use it as a *second* layer, not a first.

The composition rule is the same as chapter 01: run cheap-and-precise first, then a broader ML detector, then (optionally) an LLM sanity-check for the highest-stakes flows. Fail closed on detector timeout in scrubbing paths — a trace that times out its scrubber is not stored unscrubbed; it is stored with a `[scrubbing_failed]` stub or dropped, and the failure alerts.

## Replacement: the four strategies and when to use each

Detecting the sensitive span is half the problem. What replaces it depends on the downstream use.

- **Redact** — replace with a fixed token (`[REDACTED]`, `[SECRET]`). Simplest; loses all information. Right for secrets in every context.
- **Mask** — replace with a shape-preserving placeholder (`4111-****-****-1111`, `s***@example.com`). Preserves that a value existed and roughly what shape it had; still non-reversible. Good for logs that need to be legible without leaking the value.
- **Tokenise / pseudonymise** — replace with a stable placeholder (`<EMAIL_1>`, `<PERSON_1>`) that maps consistently within a scope (a trace, a session). Preserves referential structure so a downstream analysis can still tell that "the same person appears twice." mod-106 chapter 02 uses exactly this in trace snapshots. Requires care: the mapping table itself is sensitive.
- **Hash / HMAC** — replace with a cryptographic digest of the value, keyed with a secret. Enables joining across systems (two logs of the same user hash to the same value) without revealing the plaintext. Necessary for user-ID handling in shared analytics stores. Do not use unkeyed hashes on low-entropy fields (an unkeyed SHA of an email address is trivially reversible via rainbow tables).

The default per category:

- Secrets → **redact**, always.
- PII in the model prompt → **tokenise** when the prompt is templated for a task that needs referential structure ("summarise this email including the sender's name"); **redact** or **mask** otherwise.
- PII in outputs to the user → **pass through** when the user is the data subject (they typed it in); **redact / mask** when the output goes to any other destination (analytics, support-agent transcripts, external channels).
- PII in traces → **tokenise** (so debugability survives) with a scoped mapping table, plus **hash** for the identifiers used in analytics rollups. Special-category PII → **drop the trace entirely** by default, unless a compliance-signed-off pipeline explicitly permits it (chapter 02 of mod-106 on special-category-data snapshots).
- Business-confidential data → surface-specific; often **tokenise** in prompts and **redact** in outputs when the output leaves the tenancy.

## Point-by-point: what to run where

### Point 1 — Input scrubbing

- **Secret detection is mandatory.** Users paste API keys into help messages surprisingly often ("here's my code, why isn't it working…"). A leaked secret in the prompt is now in the provider's storage. Redact secrets in the request path *before* the prompt is assembled.
- **PII detection is contextual.** For a surface where PII is expected (a support agent talking about *the user's own* account), do not aggressively scrub — you would break the feature. For a surface where PII is incidental (a code-completion tool), redact aggressively in the outbound direction.
- **Log the *fact* of scrubbing, not the *content*.** The trace records "3 secrets and 2 PII spans redacted at input," not the redacted values. This is enough for a support debug ("why did the response say `[SECRET]`?") without re-storing the leak.

### Point 2 — Prompt scrubbing

The rendered prompt includes system content, tool schemas, retrieval context, tool results, and message history. Two categories bite here that Point 1 cannot catch:

- **Retrieval-side leakage.** A chunk retrieved from the internal corpus contains a customer's PII from a stale document. The retrieval was in-scope; the *inclusion in this session's prompt* is out-of-scope. Filter chunks at retrieval time by tenant scope (mod-103's ACL discussion) and by sensitivity tag.
- **Tool-result leakage.** A tool returned a database row containing a full record — SSN, DOB, medical history — when the task only needed the name. Trim tool outputs at the tool boundary (mod-104), not in the model's context.

### Point 3 — Output scrubbing

The model's response can contain values the model was not authorised to produce, values it hallucinated but happen to be sensitive, or values that were legitimately in context but should not go to *this* destination.

- **Cross-tenant echo.** A hallucinated customer name from another tenant's data. Rare, catastrophic. Output scrubbing scans for identifiers that do not belong to the current tenant scope.
- **Structured leakage via free-text fallback.** The output was supposed to be a JSON payload of specific fields; the model produced free text containing everything in context. Enforce structured output (chapter 02, and mod-102) so this is impossible.
- **Format-shifted PII.** The user gave a name; the model rephrases it in an email template ("Hi $NAME") that then embeds in a downstream automated email. If the downstream template's rendering is a different destination than the user's chat, scrubbing has to fire between the model and the template.

### Point 4 — Trace scrubbing

The most important pass. Traces travel farther than prompts and are read by more people.

Two properties are load-bearing:

- **The tracing SDK is instrumented with the scrubber, not the other way around.** The [OpenTelemetry Python SDK's `SpanProcessor` interface](https://opentelemetry-python.readthedocs.io/en/stable/sdk/trace.html) exposes `on_start(span, parent_context)` (writable `Span`) and `on_end(span)` (`ReadableSpan`) hooks that run in-process before the exporter sees the span — the natural site for a scrubbing processor. Vendor-side hooks vary in capability: [Langfuse's masking docs](https://langfuse.com/docs/tracing-features/masking) describe both an OTEL-side `mask_otel_spans` and a legacy SDK-attribute `mask` hook; [Arize Phoenix](https://arize.com/docs/phoenix/tracing/how-to-tracing/advanced/masking-span-attributes) uses the OpenInference `OPENINFERENCE_HIDE_*` env vars (also settable via `TraceConfig`); [Braintrust](https://docs.braintrust.dev/guides/traces/privacy) offers ingestion-time detection plus SDK-side `maskingFunction` and `onSpanExport()` span customisers; [W&B Weave's `redact_pii=True`](https://docs.coreweave.com/weave/guides/tracking/redact-pii) in `weave.init()` runs Presidio client-side (Python only, 18 default entities). [OpenLLMetry](https://www.traceloop.com/docs/openllmetry/privacy/traces) is a cleaner-content case rather than a redactor — the `TRACELOOP_TRACE_CONTENT=false` env var (and the per-workflow override) suppresses prompt / completion content from spans entirely rather than rewriting it. Configure the scrubber at instrumentation time; do not rely on downstream stores to redact.
- **Every trace vendor gets the *same* scrubbing budget.** If Langfuse gets scrubbed traces and Datadog gets full traces because "it's just internal infra," the leak surface is Datadog. The scrubber runs *once*, upstream of *all* exporters.

A minimal instrumentation shape:

```python
# telemetry/scrub_span_processor.py — sketch.
from opentelemetry.sdk.trace import SpanProcessor
from telemetry.scrubbers import scrub_llm_span

class ScrubbingSpanProcessor(SpanProcessor):
    def on_end(self, span):
        scrubbed = scrub_llm_span(span)   # mutates in place; returns metadata
        span.set_attribute("scrub.applied", True)
        span.set_attribute("scrub.categories_hit", ",".join(scrubbed.categories_hit))
        span.set_attribute("scrub.spans_redacted", scrubbed.count)
        # DO NOT set any attribute containing the redacted value.

    def shutdown(self):
        pass
    def force_flush(self, timeout_millis=None):
        return True
```

Two properties. The scrubber runs *before* the span exports (`on_end` fires in-process). The scrubber records its own actions as span attributes, giving the observability side a metric for "scrubbing coverage" that the security team can watch.

## The DSAR / deletion pipeline: scrubbing is not deletion

Scrubbing prevents *storage* of sensitive data. It does not fulfil the user's right to delete data that is already stored. mod-105 chapter 02 develops the deletion pipeline for the memory layer; the observability substrate needs the equivalent.

Every trace store used in production must answer:

- **Can you find all traces for a given user?** ID-keyed indices on `user_id`, ideally on `tenant_id`, plus hashed identifiers for joined analytics.
- **Can you delete them within your published SLA?** GDPR names 30 days as the ceiling for a data-subject deletion request; many products publish shorter. The trace store's DELETE path is a first-class capability, not a "we can query it out of the DB manually" workaround.
- **Do exports for a user contain trace data?** GDPR Article 20 portability — trace data derived from the user's activity often qualifies. The applied engineer names the boundary explicitly with the privacy team.
- **Are provider-side caches purgeable?** Anthropic, OpenAI, and Google publish their own retention windows and deletion contracts for prompt caching and abuse-monitoring stores. If your DSAR pipeline stops at the tracing vendor and does not extend to the model provider, you have shipped an incomplete deletion contract.

## The FP/FN report for scrubbing

Same discipline as chapter 01, applied here. A scrubber without an FP/FN report is a scrubber that will over-redact quietly (the ML detector eats a customer's real name from the response and the support agent replies to "hi ") or under-redact quietly (the regex misses a new API-key prefix format and secrets leak for three months before anyone notices).

The eval set for a scrubber:

- **Labelled positives** — spans that *should* be redacted, per category. Sources: incident library (things that leaked before), public PII fixtures (Presidio ships some), and synthetic augmentation (`faker`-generated names and emails plugged into templated conversations).
- **Labelled negatives** — spans that must *not* be redacted (a discussion of "SSN" as an acronym in a policy document, an email address in an example section, a fake credit-card number in a fraud-training article). Sources: production traces sampled and reviewed.
- Per category: **precision** (of redactions, how many were legitimate), **recall** (of what should have been redacted, how much was), **false-drop rate** (for traces dropped as too-sensitive-to-store, how many were false alarms).

The report is re-run when: a detector is upgraded, a category is added, a new pattern lands in the incident library, or on a monthly cadence.

## Anti-patterns

- **"We scrub logs."** Only one point covered. The trace exports to a vendor without the same treatment; the model provider gets the raw prompt anyway.
- **The tracer captures request bodies "for now, we'll add scrubbing later."** "Later" always ships as an incident. If the scrubber is not in place at instrumentation time, do not instrument the sensitive fields yet.
- **Free-text `notes` fields.** The tracer captures a `notes` attribute the model wrote to. It contains, over time, every kind of data the surface handles. Prefer typed span attributes that name what they hold; a free-text attribute is a redaction bypass.
- **Client-side scrubbing.** The frontend scrubs before sending. A user with a modified client (or a competing product wrapping your API) doesn't. Server-side or nothing.
- **Unkeyed hashes of low-entropy PII.** SHA-256(email) is not a redaction; a rainbow table reverses it in seconds. Use HMAC with a rotating secret, or tokenise, or redact.
- **"The observability vendor is HIPAA-compliant, so we're fine."** A vendor's compliance posture is your foundation, not your ceiling. The BAA covers *their* obligations; *your* application still needs to minimise, scrub, and honour deletion. Sending unredacted PHI to a HIPAA-compliant vendor is still a Point-4 leak into a system whose analytics team can query.
- **Scrubbing that ships without a re-run schedule.** The detector was tuned six months ago against an incident library that has doubled since. Re-run monthly, or on every incident-library update, whichever comes first.

## Summary

- Telemetry is a data-leak channel. Treat the tracer with the same rigour as the model provider — the same scrubbing must fire before either sees the data.
- Four scrubbing points: **inputs** (secrets, contextual PII, at the surface boundary), **prompts** (retrieval-side leakage, tool-result over-fetch), **outputs** (cross-tenant echo, format-shifted PII), **traces** (the tracing-SDK-level scrub that fires before *every* exporter).
- Detection is a two-or-three-layer stack — regex/heuristic (cheap, precise, secret-shaped), ML/statistical (Presidio and equivalents), optional LLM-based sanity check on the highest-stakes flows. Fail closed on scrubber timeout in trace paths.
- Four replacement strategies — redact (default for secrets), mask (shape-preserving), tokenise (referential structure preserved, mapping table itself sensitive), hash / HMAC (keyed, for join analytics). Category and destination pick the strategy, not the codebase's habit.
- Scrubbing is not deletion. The DSAR pipeline covers every store the trace ever lands in — the tracing vendor, the analytics warehouse, the model provider's caches. The applied engineer names the deletion contract for each.
- An FP/FN report against a labelled eval set — the same discipline as chapter 01 — is what keeps the scrubber honest over time. Re-run on detector upgrades, category additions, and monthly.

[Chapter 04](04-tool-abuse-and-destructive-action-gates.md) turns from data flowing *out* of the surface to actions flowing *from* the surface: tool abuse, unauthorised tool use, destructive-action gates, and the per-tool budgets that keep an injection or a bug from becoming an incident.
