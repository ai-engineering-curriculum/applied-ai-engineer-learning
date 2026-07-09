# Resources for mod-101 — Applied AI Foundations

Curated primary sources for the topics covered in the chapters. Standards and primary documentation first; secondary references are named as such.

## Standards and frameworks

- **ISO/IEC 25059:2023 — *Software engineering — Systems and software Quality Requirements and Evaluation (SQuaRE) — Quality model for AI systems*** — the AI-specific extension of the ISO/IEC 25010 quality model used as the checklist in [chapter 04](04-one-page-feature-architecture-doc.md). Available for purchase from ISO (`https://www.iso.org/standard/80655.html`). The overview and outline are viewable without purchase.
- **ISO/IEC 25010:2011 — *Systems and software Quality Requirements and Evaluation (SQuaRE) — System and software quality models*** — the base quality model that 25059 extends. `https://www.iso.org/standard/35733.html`.
- **ISO/IEC 42001:2023 — *Information technology — Artificial intelligence — Management system*** — the AI management system standard governance owners will reference; useful context for the Section 5 / Section 6 sign-off framing. `https://www.iso.org/standard/81230.html`.
- **NIST AI Risk Management Framework (AI RMF 1.0)** — Govern / Map / Measure / Manage functions and their subcategories; the crosswalk pattern in the stretch goal of exercise-04. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **NIST AI RMF Generative AI Profile (NIST AI 600-1)** — GenAI-specific companion to the RMF; useful for the safety-envelope shape and the risk-category vocabulary. `https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook`. <!-- needs-research: confirm the current canonical NIST-hosted URL for the GenAI Profile (NIST AI 600-1) and the Playbook; the NIST AIRC domain has restructured multiple times. -->

## SLO and reliability foundations

- **Google SRE Book — *Site Reliability Engineering*, Chapters 3 (Embracing Risk) and 4 (Service Level Objectives)** — the canonical framing for SLI / SLO / SLA and error budgets; the shape [chapter 02](02-slos-from-spec-to-metrics.md) borrows from. Free online: `https://sre.google/sre-book/table-of-contents/`.
- **Google SRE Workbook — *The Site Reliability Workbook*, Chapters 2 (Implementing SLOs) and 5 (Alerting on SLOs)** — practical follow-on with worked SLO examples and burn-rate alert design. Free online: `https://sre.google/workbook/table-of-contents/`.
- **Nielsen Norman Group — Response Time Limits (Nielsen, 1993; still cited)** — the 100 ms / 1 s / 10 s perception thresholds used as the basis for TTFT and end-to-end latency targets. `https://www.nngroup.com/articles/response-times-3-important-limits/`.

## LLM feature-shape references and applied-agent case studies

- **Anthropic — *Building Effective Agents*** — Anthropic's applied engineering post on when to use workflows vs agents; useful for the autonomous-agent-vs-assistant / co-pilot decision in [chapter 01](01-choosing-the-feature-shape.md). `https://www.anthropic.com/engineering/building-effective-agents`.
- **OpenAI — *A practical guide to building agents*** — OpenAI's applied guide to agentic surfaces and their evals. `https://openai.com/index/a-practical-guide-to-building-agents/`. <!-- needs-research: confirm current canonical URL; OpenAI reorganises their guides periodically. -->
- **Cursor documentation — *Composer / Chat / Cmd-K*** — a canonical shipped example of the co-pilot shape with a mode switcher for chat and generation. `https://docs.cursor.com`.
- **GitHub — *Copilot Chat documentation*** — shipped co-pilot + chat blend; a reference point for the "assistant vs co-pilot" distinction. `https://docs.github.com/en/copilot`.
- **Simon Willison — *Prompt injection: what's the worst that can happen?* and follow-ups** — practitioner writeups on the failure surface of assistant / agent shapes; useful reading before the safety-envelope work in mod-107. `https://simonwillison.net/tags/prompt-injection/`.

## Provider documentation for the tier trade-off note

The trade-off note in [chapter 03](03-provider-and-tier-selection.md) and [exercise-03](exercises/exercise-03-provider-tier-trade-off-note.md) draws on public provider docs. Consult the current version of each — pricing, regions, structured-output support, and prompt-caching semantics change often.

- **OpenAI Platform docs** — models, pricing, JSON mode, function calling, Batch API, prompt caching. `https://platform.openai.com/docs`.
- **Anthropic API docs** — models, pricing, tool use, prompt caching (explicit cache breakpoints), regions. `https://docs.anthropic.com`.
- **Google Vertex AI generative-AI docs** — Gemini models, regions, structured output, batch prediction. `https://cloud.google.com/vertex-ai/generative-ai/docs`.
- **AWS Bedrock docs** — model catalogue (Anthropic, Meta, Mistral, Amazon, Cohere, AI21), regions, structured output, batch inference. `https://docs.aws.amazon.com/bedrock/`.
- **Azure OpenAI Service / Azure AI Foundry docs** — model availability, regions, quotas, structured output. `https://learn.microsoft.com/azure/ai-services/openai/`.

## Open-weights model families (for the open-source tier)

- **Meta Llama documentation and licence** — Llama 3, 3.1, 3.2, 3.3 model card; licence terms and use restrictions. `https://llama.meta.com`.
- **Mistral open-weights models** — model cards and licence terms. `https://mistral.ai/technology/#models`.
- **Alibaba Qwen model family** — model cards, licence terms, benchmark reports. `https://qwen.readthedocs.io/en/latest/`.
- **DeepSeek open-weights family** — model cards and licence. `https://huggingface.co/deepseek-ai`.

## Self-hosted inference (for the "self-hosted" tier)

- **vLLM documentation** — model deployment, throughput and latency tuning; consumed here, owned by `ai-infra-performance`. `https://docs.vllm.ai`.
- **Text Generation Inference (TGI)** — Hugging Face's serving stack. `https://huggingface.co/docs/text-generation-inference/`.
- **SGLang** — high-throughput inference engine. `https://github.com/sgl-project/sglang`.

## Structured-output and prompt-caching foundations for portability

Deep coverage is in mod-102; these are the primary sources referenced by the portability accounting in [chapter 03](03-provider-and-tier-selection.md).

- **Instructor** — schema-constrained structured outputs via Pydantic. `https://python.useinstructor.com`.
- **Outlines** — grammar / regex / JSON-schema constrained decoding. `https://dottxt-ai.github.io/outlines/`.
- **Pydantic** — validation library used with Instructor and directly in structured-output pipelines. `https://docs.pydantic.dev`.

## Benchmarks referenced in [chapter 05](05-leaderboard-driven-failure-modes.md)

Referenced only to name the anti-patterns; not endorsements of any of them as evidence for feature design.

- **MMLU / MMLU-Pro** — multi-subject multiple-choice benchmarks. `https://github.com/hendrycks/test` and `https://github.com/TIGER-AI-Lab/MMLU-Pro`.
- **GPQA (Graduate-Level Google-Proof Q&A)** — expert-level science multiple-choice benchmark. `https://arxiv.org/abs/2311.12022`.
- **HumanEval** — Python function-generation benchmark. `https://github.com/openai/human-eval`.
- **SWE-bench and SWE-bench Verified** — multi-file software-engineering benchmark on public GitHub issues. `https://www.swebench.com`.
- **Chatbot Arena / LMArena** — human-preference leaderboard. `https://lmarena.ai`.
- **τ-bench** — multi-turn agentic tool-use benchmark used in mod-105 as a shape template. `https://github.com/sierra-research/tau-bench`.

## Safety and governance references (previewed here, owned in mod-107 / mod-112)

- **OWASP Top 10 for Large Language Model Applications** — surface-side vulnerability taxonomy referenced in Section 6 of the one-pager. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **OWASP Agentic AI Top 10** — agent-shape-specific vulnerability taxonomy. <!-- needs-research: confirm the canonical URL and current publication status of OWASP's Agentic AI Top 10; the OWASP project directory has restructured multiple times and the "Agentic AI" project has moved between working groups. -->
- **EU AI Act (Regulation (EU) 2024/1689)** — obligations for AI systems, risk classifications, transparency notices, human oversight. Official text: `https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=OJ:L_202401689`.

## Secondary reading

- Kohavi, Tang, Xu — *Trustworthy Online Controlled Experiments* — the reference on A/B testing methodology. Deep methodology is owned by `senior-ml-engineer` and `model-evaluation-engineer`; useful context for the "shipping / rollback" chapters (mod-111).
- *Designing Data-Intensive Applications* (Kleppmann) — chapters 1 (Reliable, Scalable, and Maintainable Applications), 3 (Storage and Retrieval), and 10 (Batch Processing) are load-bearing background for the reliability / freshness / batch discussions in later modules.
