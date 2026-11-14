# Resources for mod-105 — Shipping Product-Embedded Agents

Curated primary sources for the topics covered across the five chapters. Provider documentation, standards, canonical open-source projects, and the public writeups from the four case-study companies come first; secondary references are named as such. Agent product surfaces, provider memory features, and public benchmarks all move on a monthly cadence — consult the current live docs before pinning behaviour, and add an accessed-on date to any citation you carry into a design review.

## Cross-cutting framing (all chapters)

- **Anthropic — Building effective AI agents** (engineering blog) — the "workflow vs. agent" framing that shapes when a shipped surface should be an agent at all. `https://www.anthropic.com/engineering/building-effective-agents`.
- **Anthropic — Engineering blog index** — recurring essays on tool use, MCP, Claude Agent SDK, and Claude Code. `https://www.anthropic.com/engineering`.
- **OpenAI — A practical guide to building agents** (whitepaper) — OpenAI's own applied-vs-research framing. `https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf` (also linked from the web version at `https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/`).
- **OpenAI — Building agents (cookbook / guides)** — worked examples across function calling, Responses API, and the Agents SDK. `https://cookbook.openai.com/`.
- **Simon Willison — LLMs and agents tags** — practitioner reference for shipped-agent failure modes, provider changes, and prompt-injection field notes. `https://simonwillison.net/tags/llms/`, `https://simonwillison.net/tags/agents/`.
- **Chip Huyen — *AI Engineering* (book)** — engineering-side framing for applied LLM systems. `https://huyenchip.com/`.
- **Eugene Yan — Applied-LLM writeups** — practitioner posts covering agent patterns, evaluation, and rollout. `https://eugeneyan.com/writing/`.

## Chapter 01 — Agent shape for a real product surface

### Shape A (coding co-pilot)

- **Cursor — Documentation** — canonical entry point for Cursor's feature set (Tab, Chat, Composer, Plan Mode, Agent, Cloud Agents). `https://docs.cursor.com/` and `https://cursor.com/features`.
- **Cursor — Changelog** — the surface-level record of what shipped when, referenced when you need to date a design claim. `https://cursor.com/changelog`.
- **GitHub — Copilot documentation** — canonical entry point for Copilot's inline / chat / workspace surfaces. `https://docs.github.com/en/copilot`.
- **GitHub — Copilot Workspace** — the original GitHub Next technical-preview page for the task-decomposition surface, whose approach has since been folded into the mainline Copilot coding agent. `https://githubnext.com/projects/copilot-workspace/`.
- **GitHub — Copilot coding agent (docs)** — the newer autonomous-tier surface layered above Workspace. `https://docs.github.com/copilot/concepts/agents/coding-agent/about-coding-agent`.
- **GitHub — Engineering blog on Copilot** — long-form writeups on Copilot's design decisions. `https://github.blog/ai-and-ml/github-copilot/`.

### Shape B (customer-experience agent)

- **Sierra — Product overview** — canonical marketing + docs surface for the CX agent platform. `https://sierra.ai/`.
- **Sierra — Agent OS / product documentation** — the customer-facing docs for building a CX agent inside Sierra. `https://sierra.ai/product` (with sub-sections for Agent Studio, Channels, Context Engine, Horizon, Insights, Explorer, and Trust and Reliability).
- **Sierra — Blog and case studies** — Sonos, ADT, WW, SiriusXM, and other named deployments. `https://sierra.ai/blog`.
- **Klarna — AI assistant announcement (Feb 2024)** — the "handles two-thirds of customer service chats" claim that anchors the shape's savings case. `https://www.klarna.com/international/press/klarna-ai-assistant-handles-two-thirds-of-customer-service-chats-in-its-first-month/`. Treat the figure as a point-in-time announcement, not a current SLA — Klarna has since publicly discussed rebalancing toward human agents.

### Shape C (retrieval-with-actions)

- **Notion — AI documentation** — Notion's assistant surface embedded in the workspace, with retrieval + write tools. `https://www.notion.so/help/notion-ai`.
- **Glean — Agents documentation** — enterprise workspace-retrieval-with-actions positioning. `https://www.glean.com/product/agents`.
- **Salesforce — Agentforce** — the CRM-embedded retrieval-with-actions surface. `https://www.salesforce.com/agentforce/`.
- **Microsoft — Copilot for Microsoft 365 documentation** — workspace-scoped retrieval + actions across the Office suite. `https://learn.microsoft.com/en-us/copilot/microsoft-365/`.

### Shape D (browsing agent)

- **OpenAI — Operator (announcement)** — the initial browsing-agent product; its capabilities have since been consolidated into ChatGPT Agent (see next entry). `https://openai.com/index/introducing-operator/`.
- **OpenAI — ChatGPT agent** — the successor / consolidation of Operator into the main ChatGPT surface. `https://openai.com/index/introducing-chatgpt-agent/`.
- **Anthropic — Computer use tool** — the API-side capability for driving screens / DOM. `https://platform.claude.com/docs/en/build-with-claude/computer-use`.
- **Anthropic — Piloting Claude in Chrome (research preview)** — the browser-extension surface for Claude computer-use. `https://claude.com/blog/claude-for-chrome`.

### Shape E (sales / support / ops assistant)

- **Harvey — Product documentation and blog** — professional-user surface for legal work. `https://www.harvey.ai/`, `https://www.harvey.ai/blog`.
- **Gong — Product documentation** — sales-side agent-assist reference. `https://www.gong.io/`.
- **Zendesk — AI agents and Copilot** — CX-side agent-assist for support reps. `https://www.zendesk.com/service/ai/`.
- **Datadog — Bits AI** — engineering-ops agent-assist reference. `https://www.datadoghq.com/product/bits-ai/`.

### Applied vs. agentic split framing

- **Chip Huyen — *Agents* essay** — the framing on what applied engineering owns vs. what agent-engine work owns. `https://huyenchip.com/2025/01/07/agents.html`.
- **Anthropic — Building effective AI agents** — the "workflow vs. agent" distinction underneath the applied/agentic split. `https://www.anthropic.com/engineering/building-effective-agents`.

## Chapter 02 — Memory system with retention and PII policy

### Provider-side memory features

- **OpenAI — Memory in ChatGPT (Memory FAQ)** — user-facing memory surface and controls. `https://help.openai.com/en/articles/8590148-memory-faq`.
- **OpenAI — Data controls in the OpenAI platform** — the platform-level retention posture for API traffic. `https://developers.openai.com/api/docs/guides/your-data`.
- **Anthropic — Trust Center** — Anthropic's published stance on Console vs. API data; browse to the data-handling resources for the retention specifics. `https://trust.anthropic.com/`.
- **Anthropic — Projects in Claude.ai** — the consumer product's project-scoped memory surface. `https://support.claude.com/en/articles/9519177-how-can-i-create-and-manage-projects`.
- **Google — Gemini Apps Privacy Hub** — consumer product retention posture and memory controls. `https://support.google.com/gemini/answer/13594961`.

### Data-subject rights (regulatory floor)

- **GDPR — Regulation (EU) 2016/679** — the source text with the four data-subject rights (Articles 15, 16, 17, 20) and the special-category-data list (Article 9). `https://eur-lex.europa.eu/eli/reg/2016/679/oj`.
- **European Data Protection Board — Guidelines 01/2022 on data subject rights – Right of access (v2.1, final)** — the operational read of Article 15. `https://edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-012022-data-subject-rights-right-access_en`.
- **California — CCPA / CPRA** — the California statute and its regulator's guidance. `https://oag.ca.gov/privacy/ccpa`.
- **LGPD (Brazil) — Lei Geral de Proteção de Dados** — Brazil's general data-protection law; the IAPP maintains an English translation reachable from its resource library at `https://iapp.org/resources/`.
- **HIPAA — HHS overview** — the regulatory frame most relevant if the memory holds health data. `https://www.hhs.gov/hipaa/index.html`.

### Storage substrates and typed schemas

- **Pydantic — Documentation** — Python source-of-truth types for the memory schema. `https://docs.pydantic.dev/latest/`.
- **Zod — Documentation** — TypeScript equivalent. `https://zod.dev/`.
- **SQLite — Documentation** — the lightweight persistent store used in the chapter examples. `https://sqlite.org/docs.html`.
- **PostgreSQL — Documentation** — the more common production-grade choice. `https://www.postgresql.org/docs/`.
- **`cryptography` (Python) — Fernet** — reference for encrypting session content at rest. `https://cryptography.io/en/latest/fernet/`.
- **`libsodium` — Documentation** — the cross-language crypto library referenced for encryption-at-rest. `https://doc.libsodium.org/`.

### Retrieval on memory

- **LangChain / LangGraph — Memory overview** — the framework's canonical patterns for short- and long-term agent memory. `https://docs.langchain.com/oss/python/concepts/memory`.
- **LangGraph — Checkpointers reference** — the session-layer store. `https://langchain-ai.github.io/langgraph/reference/checkpoints/`.
- **OpenAI Agents SDK — Sessions** — the framework's session-layer primitive. `https://openai.github.io/openai-agents-python/sessions/`.
- **Mem0** — an open-source memory library specifically for LLM apps. `https://docs.mem0.ai/`.
- **Zep** — a memory + retrieval store for LLM apps. `https://help.getzep.com/`.

### Vector stores (context, for the retrieval side of memory)

- **FAISS** — Meta's vector index, common in local memory setups. `https://github.com/facebookresearch/faiss`.
- **Chroma** — open-source embedding database. `https://docs.trychroma.com/`.
- **LanceDB** — embedded vector store. `https://lancedb.github.io/lancedb/`.
- **pgvector** — Postgres extension for embeddings. `https://github.com/pgvector/pgvector`.

## Chapter 03 — Human-in-the-loop, kill switches, and steering UX

### Runtime signals and human-in-the-loop primitives

- **LangGraph — Human-in-the-loop docs** — the canonical `interrupt` / `Command` pattern for approval / rejection. `https://langchain-ai.github.io/langgraph/how-tos/#human-in-the-loop`.
- **LangGraph — Streaming** — how the runtime exposes step events to the UI. `https://docs.langchain.com/oss/python/langgraph/streaming`.
- **OpenAI Agents SDK — Handoffs and Guardrails** — the framework's HITL primitives. `https://openai.github.io/openai-agents-python/handoffs/`, `https://openai.github.io/openai-agents-python/guardrails/`.
- **Anthropic — Claude Agent SDK Hooks** — `PreToolUse` / `PostToolUse` as the confirmation-gate surface. `https://code.claude.com/docs/en/agent-sdk/hooks`.
- **Anthropic — Building effective AI agents (checkpoint / interrupt sections)** — the framing on when to pause an agent. `https://www.anthropic.com/engineering/building-effective-agents`.

### Progress, cancel, undo, resume UX

- **Nielsen Norman Group — Response times: the 3 important limits** — the classic UX reference on progress and latency. `https://www.nngroup.com/articles/response-times-3-important-limits/`.
- **Nielsen Norman Group — Progress Indicators Make a Slow System Less Insufferable** — when and how to show progress. `https://www.nngroup.com/articles/progress-indicators/`.
- **W3C — Understanding WCAG 2.2, SC 4.1.3 Status Messages** — accessibility side of long-running operations, i.e. announcing progress/wait states to assistive technology without moving focus. `https://www.w3.org/WAI/WCAG22/Understanding/status-messages.html`.

### Kill switch / operational safety

- **Google SRE Book — Managing incidents** — the discipline behind runbooks. `https://sre.google/sre-book/managing-incidents/`.
- **Google SRE Book — Postmortem culture** — the loop that turns a kill-switch flip into a lesson. `https://sre.google/sre-book/postmortem-culture/`.
- **Google SRE Workbook — Incident response** — the runbook chapter. `https://sre.google/workbook/incident-response/`.
- **Charity Majors — On observability, incidents, and operational maturity** — practitioner writing on kill-switch discipline. `https://charity.wtf/tag/incidents/`.
- **LaunchDarkly — Kill switches** — vendor docs for a feature-flag-driven kill switch. `https://docs.launchdarkly.com/`.
- **OpenFeature — Specification** — the open standard for feature flags underneath kill-switch tooling. `https://openfeature.dev/specification/`.

### Confirmation UX for high-blast-radius writes

- **Anthropic — Mitigate jailbreaks and prompt injections** — the "confirm before execute" framing in the provider docs, including guidance on untrusted tool content. `https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks`.
- **Stripe — Confirmation intents (Payment Intents)** — a real production example of "propose → confirm → execute" for a high-blast-radius action. `https://docs.stripe.com/payments/payment-intents`.

## Chapter 04 — Eval suites from benchmark shapes

### τ-bench

- **Sierra Research — *τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains*** — the original paper (Yao et al., 2024). `https://arxiv.org/abs/2406.12045`.
- **`sierra-research/tau-bench`** — the reference implementation, `retail` and `airline` domains, user simulator, and scoring. `https://github.com/sierra-research/tau-bench`.
- **Sierra — τ²-bench: evaluating conversational agents in a dual-control environment** — the follow-on benchmark iterating on τ-bench. `https://sierra.ai/resources/research/tau-squared-bench`.

### SWE-bench and SWE-bench Verified

- **Jimenez et al. — *SWE-bench: Can Language Models Resolve Real-World GitHub Issues?*** — the original SWE-bench paper. `https://arxiv.org/abs/2310.06770`.
- **SWE-bench — Project site and leaderboard** — canonical entry point. `https://www.swebench.com/`.
- **OpenAI — Introducing SWE-bench Verified** — the OpenAI + SWE-bench-team announcement of the Verified subset. `https://openai.com/index/introducing-swe-bench-verified/`.
- **`princeton-nlp/SWE-bench`** — the reference repository with harness code and evaluation containers. `https://github.com/princeton-nlp/SWE-bench`.
- **SWE-bench Multimodal / SWE-bench Live** — the follow-on variants. `https://www.swebench.com/multimodal.html`, `https://www.swebench.com/SWE-bench-Live/`. <!-- needs-research: confirm URLs. -->

### WebArena and VisualWebArena

- **Zhou et al. — *WebArena: A Realistic Web Environment for Building Autonomous Agents*** — the original paper. `https://arxiv.org/abs/2307.13854`.
- **WebArena — Project site** — canonical entry point. `https://webarena.dev/`.
- **`web-arena-x/webarena`** — the reference repository with the self-hosted realistic apps. `https://github.com/web-arena-x/webarena`.
- **VisualWebArena — Koh et al.** — the multimodal extension. `https://arxiv.org/abs/2401.13649`, `https://jykoh.com/vwa`.
- **WorkArena — ServiceNow research** — a related enterprise-web benchmark, useful comparison. `https://arxiv.org/abs/2403.07718`. <!-- needs-research: confirm current paper / URL. -->

### AgentDojo

- **Debenedetti et al. — *AgentDojo: A Dynamic Environment to Evaluate Attacks and Defenses for LLM Agents*** — the original paper. `https://arxiv.org/abs/2406.13352`.
- **`ethz-spylab/agentdojo`** — the reference implementation. `https://github.com/ethz-spylab/agentdojo`.
- **AgentDojo — Project site** — canonical entry point. `https://agentdojo.spylab.ai/`. <!-- needs-research: confirm URL. -->
- **SPY Lab (ETH Zurich)** — the group that maintains the benchmark, plus adjacent work on agent robustness. `https://spylab.ai/`.

### Adjacent benchmarks worth knowing

- **GAIA — General AI Assistants benchmark** — a related eval covering tool-using assistants. `https://arxiv.org/abs/2311.12983`, `https://huggingface.co/gaia-benchmark`.
- **AgentBench** — a multi-task agent benchmark. `https://arxiv.org/abs/2308.03688`, `https://github.com/THUDM/AgentBench`.
- **MLE-Bench** — OpenAI's benchmark for ML engineering agents. `https://arxiv.org/abs/2410.07095`, `https://github.com/openai/mle-bench`. <!-- needs-research: confirm current URL. -->
- **Terminal-Bench / Mind2Web / VisualAgentBench** — additional context on browsing / terminal-agent shapes. `https://osu-nlp-group.github.io/Mind2Web/`. <!-- needs-research: pin URLs for each. -->

### Evaluation harnesses and LLM-as-judge discipline

- **Zheng et al. — *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*** — foundational reference on LLM-as-judge calibration. `https://arxiv.org/abs/2306.05685`.
- **LangSmith — Evaluations documentation** — hosted eval + trace surface. `https://docs.smith.langchain.com/evaluation`.
- **Braintrust — Docs** — hosted eval framework. `https://www.braintrust.dev/docs/start/`.
- **Weights & Biases Weave — Docs** — cross-framework eval surface. `https://weave-docs.wandb.ai/`.
- **Ragas** — RAG-focused eval library, useful as reference for structured eval harness design. `https://docs.ragas.io/`.

## Chapter 05 — Public applied-agent patterns

### Cursor Composer

- **Cursor — Documentation** — canonical entry to the product surface. `https://docs.cursor.com/`.
- **Cursor — Blog / changelog** — engineering-side writeups on Composer and Background Agents. `https://cursor.com/blog`, `https://cursor.com/changelog`.
- **Aman Sanger, Michael Truell interviews** — founder-side product framing (Lex Fridman, Latent Space, etc.). Search current podcasts / talks for the most recent. <!-- needs-research: pin one or two current talks with URLs. -->
- **Cursor — Enterprise / security posture** — where the tenancy and data controls live. `https://cursor.com/security`. <!-- needs-research: confirm URL. -->

### GitHub Copilot Workspace and Coding Agent

- **GitHub Next — Copilot Workspace** — the preview home for the task-decomposition surface. `https://githubnext.com/projects/copilot-workspace/`.
- **GitHub — Copilot documentation** — the mainline docs surface. `https://docs.github.com/en/copilot`.
- **GitHub — Copilot coding agent** — the newer autonomous-tier product. `https://docs.github.com/en/copilot/using-github-copilot/coding-agent`. <!-- needs-research: confirm URL and product name. -->
- **GitHub — Copilot engineering blog** — long-form design writeups. `https://github.blog/ai-and-ml/github-copilot/`.
- **GitHub — Copilot Business / Enterprise data controls** — the admin surface referenced in the chapter. `https://docs.github.com/en/enterprise-cloud@latest/copilot`. <!-- needs-research: confirm URL. -->

### Sierra

- **Sierra — Product site** — canonical entry point. `https://sierra.ai/`.
- **Sierra — Blog and case studies** — Sonos, ADT, Weight Watchers, SiriusXM, Chubbies deployments. `https://sierra.ai/blog`.
- **Sierra — Voice AI product** — the voice-mode CX agent surface. `https://sierra.ai/voice`. <!-- needs-research: confirm URL and product name. -->
- **Sierra Research** — the research arm that published τ-bench and τ²-bench. `https://sierra.ai/research`. <!-- needs-research: confirm URL. -->
- **Bret Taylor / Clay Bavor talks and interviews** — founder-side framing for Sierra's product bet. Search current podcasts / conference talks. <!-- needs-research: pin one or two current talks with URLs. -->

### Harvey

- **Harvey — Product site and blog** — canonical entry. `https://www.harvey.ai/`, `https://www.harvey.ai/blog`.
- **Harvey — Engineering / research writeups** — model-routing and evaluation posts. `https://www.harvey.ai/blog/how-harvey-uses-multi-agent-systems`. <!-- needs-research: confirm URL — Harvey has renamed / republished a few posts. -->
- **OpenAI — Harvey partnership announcement** — the initial public partnership. `https://openai.com/index/harvey/`. <!-- needs-research: confirm URL. -->
- **Winston Weinberg interviews** — founder-side framing for the professional-user shape. Search current podcasts / conference talks. <!-- needs-research: pin one or two current talks with URLs. -->

### Other product-agent surfaces worth studying (context, not case-studied above)

- **Perplexity — Product docs and blog** — retrieval-heavy assistant surface. `https://www.perplexity.ai/`.
- **Devin (Cognition AI)** — long-running autonomous coding-agent surface; a Shape A "workspace tier" competitor. `https://www.cognition.ai/blog/introducing-devin`. <!-- needs-research: confirm URL. -->
- **Replit Agent** — cloud-IDE-native coding agent. `https://docs.replit.com/replitai/agent`. <!-- needs-research: confirm URL. -->
- **Ada — Product site** — CX-agent platform competitor to Sierra. `https://www.ada.cx/`.
- **Intercom — Fin AI Agent** — CX-agent product embedded in Intercom. `https://www.intercom.com/fin`. <!-- needs-research: confirm URL. -->
- **Decagon — Product site** — CX-agent platform, useful comparative. `https://decagon.ai/`.
- **Amazon — Q Business / Q Developer** — enterprise assistant + coding agent from AWS. `https://aws.amazon.com/q/`.

## Security perimeter around product-embedded agents (previewed here, owned in mod-107)

- **OWASP Top 10 for LLM Applications** — `LLM01: Prompt Injection`, `LLM06: Sensitive Information Disclosure`, `LLM08: Excessive Agency` are the most relevant to the shapes in this module. `https://owasp.org/www-project-top-10-for-large-language-model-applications/`.
- **NIST AI RMF 1.0** — the Measure / Manage functions this module's eval and steering signals feed. `https://www.nist.gov/itl/ai-risk-management-framework`.
- **NIST AI 600-1 — Generative AI Profile** — GenAI-specific companion to the RMF. `https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf`.
- **MITRE ATLAS** — adversarial-technique catalog for ML systems, including LLM-integrated ones. `https://atlas.mitre.org/`.
- **Simon Willison — Prompt injection tag** — the field's practitioner reference for injection failure modes on tool-using agents. `https://simonwillison.net/tags/prompt-injection/`.

## Standards and topics previewed here, owned in other modules

- **Structured outputs, extraction pipelines** for the memory extractor in [chapter 02](02-memory-system-with-retention-and-pii-policy.md) are developed in **mod-102**.
- **Permissions-aware retrieval** for the Shape C surfaces in [chapter 01](01-agent-shape-for-a-real-product-surface.md) and the Layer 3 store in [chapter 02](02-memory-system-with-retention-and-pii-policy.md) is developed in **mod-103**.
- **Tool schemas, idempotency, MCP, and the underlying orchestration** consumed by every shape are developed in **mod-104**.
- **Eval-loop harnessing, LLM-as-judge calibration, CI gating** for the internal suites in [chapter 04](04-eval-suites-from-benchmark-shapes.md) are developed in **mod-106**.
- **Guardrails and prompt-injection defence** for the AgentDojo-shape mode in [chapter 04](04-eval-suites-from-benchmark-shapes.md) are developed in **mod-107**.
- **Model routing and provider selection** (e.g., Harvey's multi-provider stack in [chapter 05](05-public-applied-agent-patterns.md)) is developed in **mod-108**.
- **Per-turn cost and latency SLOs** referenced in [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-109**.
- **Trace / feedback / incident surfaces** for the steering signals in [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-110**.
- **Feature flags, canary rollouts, kill-switch discipline** for the runbook in [chapter 03](03-human-in-the-loop-and-steering-ux.md) are developed in **mod-111**.
- **End-to-end ownership of a shipped product-agent surface** using the shape and eval choices from this module is the capstone in **mod-112**.
