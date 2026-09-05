# Resources for mod-107 — Guardrails and Safety on the Product Surface

Curated primary sources for the topics covered across the five chapters. Provider documentation, standards bodies, and the canonical open-source classifier and evaluation projects come first; practitioner writeups and secondary references are named as such. Every one of these surfaces — provider moderation endpoints, classifier release trains, red-team corpora, OWASP category revisions — moves on a monthly or quarterly cadence. Consult the current live docs before pinning behaviour, and add an accessed-on date to any citation you carry into a design review.

## Cross-cutting framing (all chapters)

- **OWASP — Top 10 for Large Language Model Applications** — the canonical category framework this module maps every finding into (chapter 05). Includes prompt injection, sensitive information disclosure, improper output handling, excessive agency, unbounded consumption, and other entries relevant to a product surface. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`. <!-- needs-research: pin the current version (2025 revision); the entry list and codes have changed twice since 2023. -->
- **OWASP — GenAI Security Project** — the umbrella that houses the LLM Top 10, the Agentic AI Top 10, the LLM AI Cybersecurity & Governance Checklist, and adjacent working groups. `https://genai.owasp.org/`. <!-- needs-research: confirm current project name and URL. -->
- **OWASP — Agentic AI Threats and Mitigations** — the companion taxonomy for agent-specific failure modes (memory poisoning, tool misuse, identity spoofing, cascading tool-chain failure, plan / goal manipulation). `https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/`. <!-- needs-research: pin current project name — it has evolved through "Agentic Threats," "Agentic AI Top 10," and related names. -->
- **NIST — AI Risk Management Framework 1.0** — the govern / map / measure / manage frame the OWASP mapping runbook (chapter 05) plugs into for a regulated surface. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **NIST — AI 600-1: Generative AI Profile** — the GenAI-specific companion to the RMF; the closest thing to a category catalogue that a compliance reviewer will accept. `https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf`.
- **MITRE ATLAS** — adversarial-technique catalogue for ML systems (including LLM-integrated ones), useful when the OWASP entries do not decompose a finding cleanly. `https://atlas.mitre.org/`.
- **Anthropic — Building effective AI agents** — the workflow-vs-agent framing that decides which surfaces need the module's guardrails at what depth. `https://www.anthropic.com/engineering/building-effective-agents`.
- **OpenAI — A practical guide to building agents** — provider-side applied framing, including safety and guardrails sections. `https://cdn.openai.com/business-guides/practical-guide-to-building-agents.pdf`. <!-- needs-research: confirm current stable URL. -->
- **Chip Huyen — *AI Engineering* (book)** — the "safety, security, and defence in depth" chapter is the closest published statement of the module's overall discipline. `https://huyenchip.com/`.
- **Simon Willison — Blog and long-running commentary on LLM safety** — practitioner reference for prompt injection variants, provider moderation changes, and shipped incidents. `https://simonwillison.net/`.

## Chapter 01 — Layered moderation with a provider + classifier split

### Provider moderation endpoints

- **OpenAI — Moderation guide** — `omni-moderation-latest` and the category taxonomy (`sexual`, `violence`, `harassment`, `hate`, `self-harm`, `illicit`, plus their per-modality flags). `https://platform.openai.com/docs/guides/moderation`. <!-- needs-research: pin the current dated model ID and the current category list — both have changed. -->
- **OpenAI — Safety best practices** — the surface-level pattern guide OpenAI publishes alongside the endpoint. `https://platform.openai.com/docs/guides/safety-best-practices`. <!-- needs-research: confirm URL. -->
- **Anthropic — Content moderation with Claude** — Claude-native pattern for input / output classification, with per-category scoring examples. `https://docs.anthropic.com/en/docs/about-claude/use-case-guides/content-moderation`. <!-- needs-research: confirm current URL after Anthropic docs restructuring. -->
- **Anthropic — Usage policies** — the policy document Claude's own safety filters are aligned to; the applied engineer's surface-level policy usually starts from this shape. `https://www.anthropic.com/legal/aup`.
- **Google — Gemini safety settings and safety ratings** — the `safetyRatings` response fields and `safetySettings` request fields, per category and threshold. `https://ai.google.dev/gemini-api/docs/safety-settings`. <!-- needs-research: confirm URL. -->
- **Google — Gemini responsible AI safety filters** — the deeper policy documentation behind the categories. `https://ai.google.dev/gemini-api/docs/safety-guidance`. <!-- needs-research: confirm URL. -->
- **AWS — Bedrock Guardrails documentation** — first-class guardrail API with content filters, denied topics, sensitive info filters, and word filters. `https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html`.
- **Azure — AI Content Safety documentation** — standalone content-safety SDK with text / image / groundedness / prompt-shield / protected-material APIs. `https://learn.microsoft.com/en-us/azure/ai-services/content-safety/`. <!-- needs-research: confirm URL and current API surface list. -->

### Classifier guardrails (open and hosted)

- **Meta — Llama Guard 3** — the reference small-parameter safety classifier used by the two-layer split in this chapter. Model card includes the hazard taxonomy. `https://huggingface.co/meta-llama/Llama-Guard-3-8B`. <!-- needs-research: confirm current version — Llama Guard 4 was announced; pin latest stable. -->
- **Meta — Llama Guard 4** — successor / multimodal variant, if applicable to your surface. `https://huggingface.co/meta-llama/Llama-Guard-4-12B`. <!-- needs-research: confirm URL and current model ID. -->
- **Meta — Prompt Guard** — small classifier specifically for prompt-injection / jailbreak detection referenced by chapter 02's Layer 1. `https://huggingface.co/meta-llama/Prompt-Guard-86M`. <!-- needs-research: confirm current version and model ID. -->
- **Meta — Purple Llama project** — the umbrella for Llama Guard, Prompt Guard, CyberSecEval, and CodeShield. `https://github.com/meta-llama/PurpleLlama`.
- **IBM — Granite Guardian** — an alternative small-parameter safety classifier family with a documented category set. `https://huggingface.co/ibm-granite/granite-guardian-3.0-8b`. <!-- needs-research: confirm current model ID and version. -->
- **ProtectAI — LLM Guard** — a scanner framework combining regex, ML, and classifier scanners for input and output moderation, PII, and injection. `https://llm-guard.com/`.
- **NVIDIA — NeMo Guardrails** — a rails / policy framework that composes classifier + regex + LLM-based checks around agent flows. `https://docs.nvidia.com/nemo/guardrails/`. <!-- needs-research: confirm URL after NVIDIA docs restructuring. -->
- **Guardrails AI — Hub** — a catalogue of validators (many of them classifier-backed) applied to LLM outputs. `https://hub.guardrailsai.com/`.

### Red-team corpora and hazard taxonomies

- **Gehman et al. — *RealToxicityPrompts*** — the canonical toxicity-elicitation prompt corpus for FP/FN evaluation of moderation. `https://arxiv.org/abs/2009.11462`, `https://allenai.org/data/real-toxicity-prompts`.
- **Parrish et al. — *BBQ: A Hand-Built Bias Benchmark for Question Answering*** — the canonical bias-fixture set for the "does the moderator over-block minoritised groups?" false-positive audit. `https://arxiv.org/abs/2110.08193`.
- **Zou et al. — *Universal and Transferable Adversarial Attacks on Aligned Language Models* (AdvBench)** — the AdvBench harmful-behaviour and harmful-string corpora, widely reused as jailbreak fixtures. `https://arxiv.org/abs/2307.15043`, `https://github.com/llm-attacks/llm-attacks`.
- **MLCommons — AILuminate benchmark** — the hazard-taxonomy-driven safety benchmark that many current classifier releases align to. `https://mlcommons.org/benchmarks/ai-safety/`. <!-- needs-research: confirm current URL and version — the benchmark is versioning quickly. -->
- **AI2 — WildGuardMix / WildGuard** — a broader safety fixture set covering harmful requests, benign refusals, and jailbreaks; useful for the benign-decoy panel. `https://huggingface.co/datasets/allenai/wildguardmix`. <!-- needs-research: confirm URL and current version. -->
- **Anthropic — Sample red-teaming datasets (HH-RLHF companion)** — foundational corpus for harmful-request evaluation. `https://github.com/anthropics/hh-rlhf`.

### Streaming and retract-on-flag patterns

- **OpenAI Agents SDK — Guardrails documentation** — the input / output guardrail primitives referenced in the streaming section (Pattern C, retract-on-flag). `https://openai.github.io/openai-agents-python/guardrails/`. <!-- needs-research: confirm URL and current semantics — the SDK is still stabilising. -->
- **Anthropic — Streaming messages** — the streaming API surface that a chapter-01 buffer / boundary / retract pattern is layered on top of. `https://docs.anthropic.com/en/api/messages-streaming`.
- **Google — Gemini streaming responses** — the parallel provider surface. `https://ai.google.dev/gemini-api/docs/text-generation#stream`. <!-- needs-research: confirm URL. -->

## Chapter 02 — Prompt injection defence at the surface

### Foundational framing and taxonomy

- **Simon Willison — *Prompt injection attacks against GPT-3*** — the canonical 2022 post that named the class of attack. `https://simonwillison.net/2022/Sep/12/prompt-injection/`.
- **Simon Willison — Prompt injection tag** — ongoing field notes on new variants and provider responses. `https://simonwillison.net/tags/prompt-injection/`.
- **Greshake et al. — *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection*** — the foundational indirect-injection paper the retrieval-side layer is designed against. `https://arxiv.org/abs/2302.12173`.
- **OWASP LLM01 — Prompt Injection** — the canonical taxonomy entry the mapping runbook connects findings to. `https://genai.owasp.org/llmrisk/llm01-prompt-injection/`. <!-- needs-research: pin the current version's entry code and URL. -->
- **OWASP LLM05 — Insecure Output Handling** — the entry the Layer 3 content-type-safe rendering discipline maps to. `https://genai.owasp.org/llmrisk/llm05-insecure-output-handling/`. <!-- needs-research: confirm current entry number — the numbering has shifted across revisions. -->

### Benchmarks and eval harnesses

- **Debenedetti et al. — *AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents*** — the canonical agent-shaped injection benchmark the exercise-02 fixture set follows. `https://arxiv.org/abs/2406.13352`, `https://agentdojo.spylab.ai/`.
- **Zhan et al. — *InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated Large Language Model Agents*** — the sibling benchmark for indirect injection through tool results. `https://arxiv.org/abs/2403.02691`, `https://github.com/uiuc-kang-lab/InjecAgent`.
- **Toyer et al. — *Tensor Trust: Interpretable Prompt Injection Attacks from an Online Game*** — the crowdsourced attack-corpus paper that supplies the "attackers reliably beat system-prompt-alone defences" evidence. `https://arxiv.org/abs/2311.01011`, `https://tensortrust.ai/`.
- **Schulhoff et al. — *Ignore This Title and HackAPrompt* (EMNLP 2023)** — the HackAPrompt competition and corpus, another canonical adversarial dataset. `https://arxiv.org/abs/2311.16119`, `https://www.hackaprompt.com/`.
- **Rebuff** — an open-source prompt-injection detection framework combining heuristics, vector similarity, an LLM check, and a canary-token trap. `https://github.com/protectai/rebuff`. <!-- needs-research: confirm the project is still maintained. -->

### Provider-side injection guidance

- **Anthropic — Mitigating jailbreaks and prompt injection** — Claude-native guidance on system-prompt discipline and distrust markers. `https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks`. <!-- needs-research: confirm URL. -->
- **Anthropic — System prompts** — reference for role-separation and system-prompt best practice. `https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts`. <!-- needs-research: confirm URL. -->
- **OpenAI — Prompt injection defence (cookbook)** — worked examples of segregation markers and defensive prompting. `https://cookbook.openai.com/examples/how_to_use_guardrails`. <!-- needs-research: confirm current URL and article — the cookbook restructures often. -->
- **Google — Gemini prompt-injection guidance** — provider-side safety-guidance URLs on protecting the model from adversarial inputs. `https://ai.google.dev/gemini-api/docs/safety-guidance#prompt_injection`. <!-- needs-research: confirm URL. -->
- **Microsoft — Prompt Shields (Azure AI Content Safety)** — a hosted detector for user-prompt and document-prompt injection. `https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection`. <!-- needs-research: confirm URL and current product name. -->

### Structured-output enforcement (Layer 3)

- **OpenAI — Structured Outputs guide** — the `strict: true` schema-constrained decoding surface. `https://platform.openai.com/docs/guides/structured-outputs`.
- **Anthropic — Tool use with `input_schema`** — schema-constrained tool arguments as the structured-output escape-hatch control. `https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview`. <!-- needs-research: confirm URL after Anthropic tool-use docs reorganisation. -->
- **Google — Gemini `responseSchema` / `responseMimeType`** — Gemini's schema-constrained decoding. `https://ai.google.dev/gemini-api/docs/structured-output`. <!-- needs-research: confirm URL. -->
- **Instructor (jxnl/instructor)** — the popular OSS Python library layering Pydantic-validated structured output over multiple providers. `https://python.useinstructor.com/`.

## Chapter 03 — PII and secret scrubbing across the stack

### PII detection

- **Microsoft Presidio — Documentation** — the reference OSS PII detector combining NER + recognisers + anonymiser. `https://microsoft.github.io/presidio/`.
- **Microsoft Presidio — Supported entities and language support** — the coverage matrix chapter 03 references for the "non-English recall degrades" caveat. `https://microsoft.github.io/presidio/supported_entities/`, `https://microsoft.github.io/presidio/analyzer/languages/`. <!-- needs-research: confirm URLs. -->
- **Microsoft Presidio — Anonymizer custom operators** — the reference for building the stable typed-placeholder / tokenisation scheme. `https://microsoft.github.io/presidio/anonymizer/`.
- **Google Cloud — Sensitive Data Protection (formerly Cloud DLP)** — the hosted PII / DLP surface referenced as a cloud-DLP alternative. `https://cloud.google.com/sensitive-data-protection/docs`.
- **AWS — Comprehend PII detection** — hosted alternative on AWS. `https://docs.aws.amazon.com/comprehend/latest/dg/pii.html`.
- **Azure — AI Language PII detection** — Microsoft's hosted alternative outside the Content Safety surface. `https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview`.

### Secret detection

- **Yelp — `detect-secrets`** — enterprise-scale secret scanner with plugin architecture, useful as a source-of-truth for high-precision secret regexes. `https://github.com/Yelp/detect-secrets`.
- **AWS Labs — `git-secrets`** — the original pre-commit secret scanner; still a useful reference for provider-key prefix patterns. `https://github.com/awslabs/git-secrets`.
- **Trufflesecurity — TruffleHog** — high-recall secret scanner with verification against provider APIs. `https://github.com/trufflesecurity/trufflehog`.
- **GitHub — Secret scanning documentation** — the canonical list of provider secret patterns GitHub scans for, useful as a starting regex catalogue. `https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning`.
- **GitHub — Secret scanning partner program (patterns list)** — the maintained provider-pattern catalogue. `https://docs.github.com/en/code-security/secret-scanning/introduction/supported-secret-scanning-patterns`. <!-- needs-research: confirm URL. -->

### Tracing SDKs and redaction hooks

- **OpenTelemetry — GenAI semantic conventions** — the emerging standard span attributes (`gen_ai.prompt`, `gen_ai.completion`, etc.) the scrubber operates on. `https://opentelemetry.io/docs/specs/semconv/gen-ai/`. <!-- needs-research: confirm URL — the spec is moving out of experimental. -->
- **OpenTelemetry — Span processors (SDK)** — the extension point the `ScrubbingSpanProcessor` sketch is built on. `https://opentelemetry.io/docs/languages/python/instrumentation/#creating-spans`. <!-- needs-research: pin the current SpanProcessor extension-point docs URL. -->
- **OpenLLMetry (Traceloop)** — the popular OTel-conformant LLM tracing library, with redaction / masking hooks. `https://www.traceloop.com/openllmetry`, `https://github.com/traceloop/openllmetry`.
- **OpenInference (Arize)** — the parallel semantic-convention spec for LLM traces, adopted by Phoenix. `https://github.com/Arize-ai/openinference/tree/main/spec`.
- **Langfuse — Masking / redaction documentation** — the hosted-vendor's built-in redaction hooks for the trace-side scrubber. `https://langfuse.com/docs/tracing-features/masking`. <!-- needs-research: confirm URL. -->
- **LangSmith — Sensitive data handling** — LangChain's guidance for masking traces before export. `https://docs.smith.langchain.com/observability/how_to_guides/masking`. <!-- needs-research: confirm URL. -->
- **Arize Phoenix — Redaction** — Phoenix's redaction hooks. `https://arize.com/docs/phoenix/tracing/how-to-tracing/masking-span-attributes`. <!-- needs-research: confirm URL. -->
- **Braintrust — Data privacy and redaction** — Braintrust's masking configuration. `https://www.braintrust.dev/docs/guides/traces/privacy`. <!-- needs-research: confirm URL. -->
- **Weights & Biases Weave — Sensitive-data controls** — Weave's masking / opt-out settings. `https://weave-docs.wandb.ai/guides/tracking/redact-pii`. <!-- needs-research: confirm URL. -->

### Regulatory foundations (the compliance envelope)

- **GDPR — Regulation (EU) 2016/679** — the substrate the applied engineer's scrubbing runs inside. Articles 9 (special-category data), 17 (erasure), 20 (portability) are referenced explicitly. `https://eur-lex.europa.eu/eli/reg/2016/679/oj`.
- **EDPB — Guidelines and opinions** — the European Data Protection Board's authoritative interpretation of GDPR clauses that matter to LLM surfaces. `https://www.edpb.europa.eu/our-work-tools/general-guidance_en`.
- **California — CCPA / CPRA** — Californian data-subject rights that most US-facing surfaces are subject to. `https://oag.ca.gov/privacy/ccpa`, `https://cppa.ca.gov/regulations/`.
- **US HHS — HIPAA overview** — the regulatory frame for PHI on any health-adjacent surface. `https://www.hhs.gov/hipaa/index.html`.
- **PCI SSC — PCI DSS v4.0** — the payment-card standard for any surface where a card number could enter a prompt or trace. `https://www.pcisecuritystandards.org/document_library/`. <!-- needs-research: confirm current v4.x version. -->
- **US FTC — GLBA privacy rules** — the financial-services counterpart. `https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act`.

### Provider retention and DSAR contracts

- **Anthropic — Zero Data Retention** — the enterprise / commercial-endpoint contract that removes the provider-side retention leak point. `https://privacy.anthropic.com/en/articles/10440198-does-anthropic-offer-zero-data-retention-zdr`. <!-- needs-research: confirm URL. -->
- **Anthropic — Privacy policy and data usage** — the general retention posture and DSAR contact path. `https://www.anthropic.com/legal/privacy`.
- **OpenAI — Enterprise privacy and data controls** — the ZDR / retention-window contract on the OpenAI side. `https://openai.com/enterprise-privacy/`.
- **OpenAI — API data usage policies** — how long non-enterprise API data is retained and for what purpose. `https://platform.openai.com/docs/models/how-we-use-your-data`. <!-- needs-research: confirm URL. -->
- **Google — Gemini API data usage** — parallel documentation for Google-hosted models. `https://ai.google.dev/gemini-api/terms`. <!-- needs-research: confirm URL and the current data-usage clause. -->
- **AWS Bedrock — Data protection** — the AWS-side contract for prompts and responses. `https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html`.
- **Azure OpenAI — Data, privacy, and security** — the Microsoft-side contract. `https://learn.microsoft.com/en-us/legal/cognitive-services/openai/data-privacy`.

## Chapter 04 — Tool-abuse guards and destructive-action gates

### OWASP entries this chapter maps to

- **OWASP LLM06 — Excessive Agency** — the primary entry chapter 04 addresses. `https://genai.owasp.org/llmrisk/llm06-excessive-agency/`. <!-- needs-research: confirm current entry code and URL — has been renumbered across revisions. -->
- **OWASP LLM04 — Data and Model Poisoning** — adjacent entry the "hostile retrieval into tool call" injection path touches. `https://genai.owasp.org/llmrisk/llm04-data-and-model-poisoning/`. <!-- needs-research: confirm current entry code. -->
- **OWASP LLM10 — Unbounded Consumption** — the entry the cost / rate caps address on the "resource-exhaustion" side of tool abuse. `https://genai.owasp.org/llmrisk/llm10-unbounded-consumption/`. <!-- needs-research: confirm current entry code and URL. -->
- **OWASP — Agentic AI Threats and Mitigations** (re-cited) — the companion taxonomy that decomposes tool-catalogue, memory-poisoning, and cascading-tool-failure risks the four guards address. `https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/`.

### MCP and tool-catalogue security

- **Model Context Protocol — Specification** — the transport / boundary the tool-boundary isolation discipline operates on. `https://modelcontextprotocol.io/specification`.
- **Model Context Protocol — Security best practices** — the MCP-native guidance for tool exposure, credential scoping, and confused-deputy avoidance. `https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices`. <!-- needs-research: confirm URL and current spec version. -->
- **Anthropic — Introducing the Model Context Protocol** — the launch post with the tool-boundary framing. `https://www.anthropic.com/news/model-context-protocol`.
- **Simon Willison — MCP tag** — practitioner field notes on MCP-server security incidents. `https://simonwillison.net/tags/mcp/`.

### Rate limiting, cost caps, and quota substrate

- **Cloudflare — Rate limiting rules** — reference implementation of per-key / per-tenant windowed rate limits. `https://developers.cloudflare.com/waf/rate-limiting-rules/`.
- **Google SRE Workbook — Managing overload** — the canonical framing of quota, backpressure, and typed 429 responses that chapter 04's "typed cap errors" discipline derives from. `https://sre.google/workbook/managing-load/`.
- **Stripe — Rate limits** — reference for how a mature product-API surface documents and returns typed rate-limit errors. `https://docs.stripe.com/rate-limits`.
- **OpenAI — Rate limits guide** — the model-provider-side reference for the "typed 429 with `retry_after`" pattern the chapter mirrors. `https://platform.openai.com/docs/guides/rate-limits`.

### Identity, least privilege, and the auth boundary

- **OAuth 2.1 (IETF draft)** — the consolidated OAuth 2.0 profile and the substrate the tool-credential rotation discipline sits on. `https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1`. <!-- needs-research: confirm current draft revision. -->
- **NIST SP 800-53 — AC-6 Least Privilege** — the standards-body statement of the least-privilege discipline the chapter's Guard 4 assumes. `https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final`.
- **AWS — IAM best practices** — the reference least-privilege catalogue for tool credentials on AWS-hosted infrastructure. `https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html`.
- **Google Cloud — IAM best practices** — the parallel on GCP. `https://cloud.google.com/iam/docs/using-iam-securely`.
- **OWASP — Confused deputy problem** — the classic auth-boundary failure the "tool credentials are not the model's credentials" discipline defends against. `https://owasp.org/www-community/vulnerabilities/Confused_Deputy_Problem`. <!-- needs-research: confirm URL. -->

### Kill switches and feature flags

- **LaunchDarkly — Kill switch documentation** — canonical vendor reference for the propagate-in-seconds pattern chapter 04 requires. `https://docs.launchdarkly.com/home/getting-started/use-cases#feature-management-with-kill-switches`. <!-- needs-research: confirm URL. -->
- **Statsig — Feature gate documentation** — an alternative substrate with fast global propagation. `https://docs.statsig.com/feature-flags/overview`. <!-- needs-research: confirm URL. -->
- **Unleash — Feature toggles** — OSS alternative with self-hosted control. `https://docs.getunleash.io/`.
- **Martin Fowler — *Feature Toggles (aka Feature Flags)*** — the canonical practitioner article on flag design, including the kill-switch class. `https://martinfowler.com/articles/feature-toggles.html`.

### Human-in-the-loop UX for confirmation gates

- **Nielsen Norman Group — Preventing user errors (confirmation dialogs)** — the design literature the "render the actual arguments, not 'Send email?'" discipline derives from. `https://www.nngroup.com/articles/confirmation-dialog/`. <!-- needs-research: confirm URL. -->
- **Anthropic — Human-in-the-loop in agents** — provider-side guidance on the pause-and-approve pattern for destructive tool calls. `https://www.anthropic.com/engineering/building-effective-agents#human-in-the-loop`. <!-- needs-research: confirm section anchor. -->
- **OpenAI Agents SDK — Human-in-the-loop** — the SDK-side primitive for interruptible tool calls. `https://openai.github.io/openai-agents-python/human_in_the_loop/`. <!-- needs-research: confirm URL — SDK docs are stabilising. -->

## Chapter 05 — Refusal calibration and the OWASP mapping runbook

### The written safety-policy shape

- **Anthropic — Usage policies** (re-cited) — a reference example of the "what the surface must never do" list at scale. `https://www.anthropic.com/legal/aup`.
- **OpenAI — Usage policies** — the OpenAI-side parallel. `https://openai.com/policies/usage-policies/`.
- **Google — Generative AI prohibited use policy** — the Google-side parallel. `https://policies.google.com/terms/generative-ai/use-policy`.
- **Anthropic — Responsible Scaling Policy** — an example of a versioned safety-policy artefact with review cadence and named owners; useful shape for the surface-level equivalent. `https://www.anthropic.com/rsp`. <!-- needs-research: confirm current URL. -->

### Over-refusal and under-refusal evaluation

- **Röttger et al. — *XSTest: A Test Suite for Identifying Exaggerated Safety Behaviours in Large Language Models*** — the canonical academic reference for over-refusal evaluation, and a working fixture set to seed the "obvious yes" panel from. `https://arxiv.org/abs/2308.01263`, `https://github.com/paul-rottger/xstest`.
- **Cui et al. — *OR-Bench: An Over-Refusal Benchmark for Large Language Models*** — a broader over-refusal benchmark with per-category coverage. `https://arxiv.org/abs/2405.20947`, `https://huggingface.co/datasets/bench-llm/or-bench`. <!-- needs-research: confirm URL. -->
- **AI2 — WildGuardMix / WildGuard** (re-cited) — includes benign-adversarial pairs useful for the "injection-shaped-but-legitimate" panel. `https://huggingface.co/datasets/allenai/wildguardmix`.
- **Anthropic — Sycophancy and refusal-calibration research** — provider-side writeups on model-side refusal behaviour that surfaces the applied engineer calibrates around. `https://www.anthropic.com/research`. <!-- needs-research: pin specific posts on refusal / over-refusal calibration. -->

### OWASP mapping runbook substrate

- **OWASP — Top 10 for LLM Applications** (re-cited) — the primary taxonomy the mapping runbook connects findings into. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **OWASP — Agentic AI Threats and Mitigations** (re-cited) — the companion taxonomy for agent-specific findings. `https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/`.
- **OWASP — LLM AI Cybersecurity & Governance Checklist** — a governance-facing companion to the Top 10, useful for the reviewer-facing side of the runbook. `https://genai.owasp.org/resource/llm-ai-cybersecurity-and-governance-checklist/`. <!-- needs-research: confirm URL. -->
- **MITRE ATLAS** (re-cited) — cross-mapping target for findings that decompose better into ATLAS techniques than OWASP entries. `https://atlas.mitre.org/`.
- **NIST — AI 600-1** (re-cited) — the compliance-adjacent frame the runbook is often expected to roll up into. `https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf`.

### Drift monitoring and incident retrospectives

- **Google SRE Book — Postmortem culture** — the discipline the OWASP-mapped findings' review cadence sits inside. `https://sre.google/sre-book/postmortem-culture/`.
- **Google SRE Workbook — Alerting on SLOs** — the framing the refusal-rate drift alert is a specialisation of. `https://sre.google/workbook/alerting-on-slos/`.
- **PagerDuty — Incident response documentation** — the operational substrate the kill-switch and OWASP-mapped remediation tickets feed into. `https://response.pagerduty.com/`.
- **Charity Majors — On observability, incidents, and operational maturity** — practitioner writing on the alert-design and postmortem discipline that transfers to LLM safety drift. `https://charity.wtf/tag/incidents/`.

### Peer-role handoff substrates

- **`ai-risk-engineer` role curriculum** — the harm-modelling, red-team programme, and cross-product risk-register work the applied engineer's OWASP mapping feeds. (Internal role curriculum; the module's chapter 05 names the handoff explicitly.)
- **`ai-safety-engineer` role curriculum** — the model-side safety-tuning and jailbreak-corpus work the applied engineer consumes. (Internal role curriculum.)
- **`ai-security-engineer` / `application-security-engineer` role curricula** — the deeper security posture the applied engineer's Guard 4 is downstream of. (Internal role curriculum.)
- **`privacy-engineer` / `data-security-engineer` role curricula** — the DPIA / regulatory foundation the chapter-03 scrubbing runs inside. (Internal role curriculum.)

## Standards and topics previewed here, owned in other modules

- **Prompt discipline and structured outputs** consumed by [chapter 02](02-prompt-injection-defence-at-the-surface.md)'s Layer 3 are developed in **mod-102**.
- **RAG retrieval and tenant-scoped ACLs** referenced by [chapter 02](02-prompt-injection-defence-at-the-surface.md)'s Layer 4 and [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md)'s retrieval-side leakage discussion are developed in **mod-103**.
- **Tool schemas, argument validation, and MCP transport** the tool-boundary discipline in [chapter 02](02-prompt-injection-defence-at-the-surface.md) and [chapter 04](04-tool-abuse-and-destructive-action-gates.md) depends on are developed in **mod-104**.
- **Human-in-the-loop UX and long-lived agent memory** the [chapter 04](04-tool-abuse-and-destructive-action-gates.md) confirmation gates and [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md) DSAR extension compose with are developed in **mod-105**.
- **Offline / online eval and CI-gated pre-registration** the [chapter 01](01-layered-moderation-with-classifier-and-provider.md) FP/FN report and [chapter 05](05-refusal-calibration-and-owasp-mapping.md) over-refusal eval plug into are developed in **mod-106**.
- **Model routing and per-provider safety-behaviour differences** the [chapter 01](01-layered-moderation-with-classifier-and-provider.md) provider-moderation composition connects to are developed in **mod-108**.
- **Cost and latency SLOs** the [chapter 04](04-tool-abuse-and-destructive-action-gates.md) per-tool caps roll into are developed in **mod-109**.
- **Trace collection, span attributes, and dashboard substrate** the [chapter 03](03-pii-and-secret-scrubbing-across-the-stack.md) span-processor plugs into — and the [chapter 05](05-refusal-calibration-and-owasp-mapping.md) refusal-drift dashboard queries — are developed in **mod-110**.
- **Feature-flag, kill-switch, and rollout / rollback discipline** the [chapter 04](04-tool-abuse-and-destructive-action-gates.md) kill switch previews are developed in **mod-111**.
- **End-to-end ownership of a shipped feature** with this module's guardrails wired in is the capstone in **mod-112**.
