# Multi-Modal Routing: Vision, Audio, Image / Video, and Realtime Voice

## Why the router owns modality

The most common failure shape for a multi-modal feature is not the model — it is that the choice of modality is *hard-coded* in feature code. `openai.audio.transcriptions.create(...)` is in a route handler; `google.vision.detect(...)` is in another; `elevenlabs.text_to_speech(...)` is in a third. When the product team asks "can we swap the STT model to reduce latency by 200ms on the mobile client?" the answer is "we have to grep across three services."

The discipline this chapter builds: **modality is a routing decision**, made at the same seam as the text-model routing decisions in chapter 02. A voice conversation is a stream of routing decisions — STT tier, LLM tier, TTS tier — each of which has a primary, fallbacks, quality gates, and cost/latency SLOs. A vision feature is a routing decision that chooses between "vision-native LLM" and "OCR-then-text-LLM." An image-generation feature is a routing decision that chooses between provider A's model, provider B's model, and a fine-tuned in-house model. When any of these lives in feature code, you have lost the composability that made the router worth building.

Two conditions the applied engineer holds the router to:

1. **Every modality has at least one route with at least one alternative.** No single-provider-per-modality lock-in.
2. **The router speaks modality-aware inputs and outputs.** A "give me a completion" call carries the modality of its input (or `text`) and expects the shape appropriate for the modality it asked for. Feature code does not care which provider served it.

## A modality-aware route table

Chapter 02's route table extended to modalities:

```yaml
# routing/modality-routes.yaml
routes:
  text.cheap: { primary: openai/gpt-4o-mini, fallbacks: [anthropic/claude-haiku] }
  text.frontier: { primary: anthropic/claude-opus, fallbacks: [openai/gpt-4o] }

  vision.doc_ocr:
    primary: azure/document-intelligence
    fallbacks: [google/document-ai, self-hosted/paddleocr]
    output_shape: ocr_v1
  vision.multimodal_chat:
    primary: openai/gpt-4o
    fallbacks: [anthropic/claude-opus, google/gemini-pro]
    output_shape: chat_v1

  audio.stt.short:                 # < 30s utterances, chat surface
    primary: openai/whisper-cloud
    fallbacks: [deepgram/nova, self-hosted/whisper-large-v3]
    max_p95_ms: 800
  audio.stt.long:                  # multi-minute recordings, background
    primary: assemblyai/universal
    fallbacks: [openai/whisper-cloud]
    max_p95_ms: 60000

  audio.tts.chat:
    primary: elevenlabs/turbo
    fallbacks: [openai/tts-1-hd, google/wavenet]
    output_shape: audio_mp3

  image.gen.marketing:
    primary: openai/gpt-image
    fallbacks: [stability/sdxl, black-forest/flux]

  video.gen.short:
    primary: runway/gen-3
    fallbacks: [google/veo, luma/dream-machine]

  realtime.voice.conversation:
    primary: openai/realtime
    fallbacks:
      - composed: { stt: audio.stt.short, llm: text.frontier, tts: audio.tts.chat }
    max_ttft_ms: 900
    max_e2e_p95_ms: 2000
```

<!-- needs-research: the model / product names above are illustrative of the current shape of each modality market; verify exact SKUs and availability at author time before shipping this file to learners. -->

Three properties. First, every route names an **output shape** so the fallback and the primary are commensurable (chapter 05 formalises this as a contract test). Second, `realtime.voice.conversation`'s fallback is a **composed route** — if the native realtime API is unavailable, the router builds an equivalent by chaining STT → text-LLM → TTS. The realtime experience degrades (higher end-to-end latency, no barge-in, no vocal-affect passthrough) but the feature still functions. Third, the **latency SLO is per-route** and *per-modality* — realtime voice's p95 is far tighter than long-form STT's.

## Modality-specific gotchas

Each modality has a set of things that trip teams new to it. What to know when routing across them.

### Vision

- **"Vision-native LLM" vs "OCR-then-text-LLM"** are two different tools. Vision-native LLMs (GPT-4o, Claude with vision, Gemini) can *reason about* images, count objects, describe scenes, and answer open questions. Dedicated OCR / document-AI stacks (Azure Document Intelligence, Google Document AI, AWS Textract, self-hosted PaddleOCR / MinerU) *extract structured text* from documents with higher recall and legible bounding boxes. Route the request to the tool that fits the job — a "extract line items from this invoice PDF" prompt sent to a general vision LLM leaves recall on the table; a "does this photo contain a person?" prompt sent to a document-AI stack returns nothing useful.<!-- needs-research: confirm currently supported document-AI product names; MinerU is a valid recent OSS document parser. -->
- **Image tokenisation is per-provider.** OpenAI, Anthropic, and Google each count image tokens differently, and each has resolution-preset knobs (`detail: low` / `high` on OpenAI, image-resize behaviours on Anthropic, thumbnail preprocessing on Gemini). A route that ignores the preset is either spending frontier tokens on preview thumbnails or feeding a full-resolution 24MP photo to a route that expected 1024×1024. Normalise resolution at the router seam.<!-- needs-research: current tokenisation semantics for images across the three providers. -->
- **Video is not "images"**. A 30-second clip at 30fps is 900 frames; sending all of them is prohibitive and unnecessary. The router should sample frames according to a policy (uniform, or shot-detection-driven) before handing to a vision LLM. Video-native models (Gemini's video input, dedicated video-understanding APIs) accept the media stream directly.

### Audio input (STT)

- **Latency profiles diverge on short vs long clips.** Whisper-class models optimise for accuracy on multi-minute clips; realtime STT (Deepgram, Speechmatics, OpenAI realtime STT, Azure Speech) optimises for sub-second first-partial on streaming input. Choose the route to match the surface.<!-- needs-research: verify current product names / SKUs. -->
- **Diarization, timestamps, punctuation, and profanity-filter behaviour vary per provider.** The output shape a downstream consumer expects (e.g. `[{start, end, speaker, text}]`) has to be produced by every route on the fallback list or the fallback breaks the consumer.
- **Language / accent coverage is uneven.** Whisper is strong on many languages; some providers are English-only or English-dominant. The route should pick by input language; the input language is often known from user metadata, not detected.
- **PII in speech is a scrubbing problem, not a moderation problem.** Transcribed speech may contain the caller's phone number, address, or credit-card number spoken aloud. The scrubbing pipeline in mod-107 chapter 03 runs on transcript output before it lands in the LLM prompt or in traces.

### Audio output (TTS)

- **Voice choice is a product decision, not a router default.** Cloning a specific voice, using a branded voice, using a language-specific voice — the router exposes the voice as a parameter but does not decide it silently.
- **Chunking for streaming TTS.** Long-form generation should stream audio chunks as tokens arrive rather than waiting for the full text. The router keeps the chunk-boundary policy (sentence, punctuation, N-words) legible.
- **Provider-specific SSML / prompting.** OpenAI, ElevenLabs, Google, and Amazon Polly each accept slightly different markup for pauses, emphasis, and prosody. If the primary route accepts SSML and the fallback does not, the fallback ships flat-affect audio.

### Image generation

- **Prompt shape and safety filters vary sharply.** OpenAI's image model, Stability's SDXL, Black Forest Labs' Flux, Google's Imagen, and Amazon Titan / Nova each have different prompt idioms and different content-policy blocklists. A prompt that works on one may refuse on another; a prompt that produces X on one may produce Y on another.<!-- needs-research: verify current model SKUs at author time. -->
- **Aspect ratio, resolution, seed, and negative-prompt support are uneven.** A fallback that ignores your requested aspect ratio silently reshapes the output. Contract tests (chapter 05) check this.
- **Cost per image dwarfs cost per text call.** A single high-resolution image is often the cost of thousands of text tokens. A fallback that generates 4 candidates instead of 1 is a 4x cost hit.

### Video generation

- **Long tail latency measured in *minutes*, not seconds.** Multi-second video generation is a background job, not an interactive call. Route these through your background-job substrate (queue, worker, callback / webhook), not through the chat request path.<!-- needs-research: this remains true across current video models. -->
- **Content policies are strict and provider-specific.** Face generation, likeness handling, and copyrighted-character prohibitions differ across providers; the router does not paper over policy differences with a silent fallback.

### Realtime voice

Realtime voice — OpenAI's Realtime API, Hume's Empathic Voice Interface (EVI), ElevenLabs' conversational AI, Vapi, Retell — collapses STT + LLM + TTS into a single duplex WebSocket / WebRTC stream with sub-second turn-taking, interruption (barge-in), and often vocal-affect passthrough.<!-- needs-research: verify current realtime-voice product landscape and their duplex-transport semantics. -->

- **The latency budget is unforgiving.** A conversational surface degrades sharply past ~1s of turn-taking latency. The route's `max_e2e_p95_ms` should be enforced, not aspirational.
- **The fallback path is a *composed* route.** As shown in the yaml above, when the native realtime API is unavailable, the router falls back to STT → text-LLM → TTS — accepting the latency hit and the loss of barge-in as a graceful degradation, not a full outage.
- **Transport is different.** Realtime APIs use WebSocket, WebRTC, or duplex HTTP/2 rather than request-response HTTP. The gateway (chapter 01) has to *proxy* these transports, not merely rewrite JSON. LiteLLM / OpenRouter / Portkey all vary in how well they support duplex modalities; if realtime voice is a required surface, verify duplex support before picking the gateway.
- **Session state is model-side.** A realtime session carries its own conversation state on the provider's side; killing and restarting a session is not free. The router's outage-handling has to reason about session boundaries, not just per-turn calls.
- **Model + voice are entangled.** In many realtime APIs the voice is not a separate call from the LLM; changing the voice changes the model configuration. The route table names both.

## Cross-modal composition patterns

Two composition patterns worth naming explicitly, because they are the most common multi-modal features you will ship.

### The "read this document" pattern

User uploads a PDF or image. Feature answers a question about it.

```
upload
  ├─► route: vision.doc_ocr        (extract structured text + spans)
  │
  ├─► route: text.frontier          (LLM answers the user's question using OCR text + spans)
  │
  └─► response
```

The router owns both hops. The OCR route's output shape (`ocr_v1`) is a contract the text route consumes. Swapping the OCR provider is a route-table edit; the text route does not care.

An alternative single-hop shape sends the raw image straight to a vision-native LLM. Which shape wins depends on document complexity (multi-page structured forms benefit from the two-hop shape; a single photo of a receipt is fine for the one-hop shape) and cost profile (OCR-then-text is often cheaper for repeated queries against the same document because the OCR result can be cached).

### The "voice conversation" pattern

Realtime as the primary; STT → text-LLM → TTS as the composed fallback.

```
websocket in ────┬─► [ primary: realtime.voice.conversation ]
                 │        │
                 │        └─► ok → websocket out
                 │
                 └─► [ fallback: composed { STT → text-LLM → TTS } ]
                          │
                          └─► out (higher latency, no barge-in, still functional)
```

The router owns the switch. The fallback path uses the same STT and TTS routes the rest of the product uses — no separate "emergency" STT stack to maintain.

## Cost, latency, and SLO shapes per modality

A quick reference — orders of magnitude, not benchmarks. Actual numbers depend on provider, model, resolution / duration, and traffic profile; treat this as a mental model, not a lookup table.

| Modality              | Cost driver       | Interactive-SLO shape       | Notes                                          |
| --------------------- | ----------------- | --------------------------- | ---------------------------------------------- |
| Text                  | Tokens            | TTFT / TPOT in ms           | Baseline everything else compares to.          |
| Vision (photo)        | Image tokens      | TTFT in ms                  | Resolution presets dominate cost.              |
| Vision (long doc)     | Pages × tokens    | Batched / async             | OCR-first often cheaper than vision-native.    |
| STT (short)           | Audio seconds     | First-partial in ms         | Streaming providers only.                      |
| STT (long-form)       | Audio minutes     | Batched (seconds → minutes) | Not interactive; queue-and-callback.           |
| TTS                   | Text chars        | TTFT audio chunk in ms      | Stream chunks; do not wait for full audio.     |
| Image gen             | Per image         | Seconds                     | 1 image ≈ thousands of text tokens.            |
| Video gen             | Per second/clip   | Minutes                     | Background job; not in chat path.              |
| Realtime voice        | Session-minutes   | End-to-end < 1s             | Duplex transport; composed fallback.           |

<!-- needs-research: this table is qualitative; before quoting any number to a user, cite a current provider price page or an internal baseline. -->

Two consequences for the router:

1. **The per-route SLO has to fit the modality.** A `max_p95_ms` of 500 for a video-gen route is nonsensical; a `max_p95_ms` of 60,000 for a chat-turn STT is a bug you will not detect until a mobile user tries the feature. The route SLO is authored per modality, then enforced.
2. **Async modalities need an async substrate.** Video gen, long-form STT, and any batch generation belong on a queue with a webhook / callback, not in the request path. The router's job on those routes is to enqueue, not to block.

## Anti-patterns

- **Hard-coded modality calls in feature code.** `openai.audio.transcriptions.create(...)` in the request handler, not behind the router. First provider-outage on that endpoint takes the surface down.
- **The single-hop fallback for realtime voice.** No composed fallback. When the realtime API rate-limits, the voice feature dies. The composed STT-LLM-TTS fallback is worth the engineering cost.
- **Ignoring image-resolution presets.** Every user photo is sent full-resolution because "the model handles it." The bill is 3x what it should be, the P95 latency is off, and the quality gain over the `detail: low` preset is often negligible.
- **STT without diarization contract.** Downstream code expects `[{start, end, speaker, text}]` from provider A. Fallback provider B returns `{transcript: "..."}` with no speaker labels. When the fallback fires, the feature silently loses speaker attribution and every call gets attributed to "speaker 1."
- **Blocking the request path on video gen.** A "generate a 5-second clip of…" call blocks the HTTP handler for two minutes because no one queued it. The mobile client's HTTP timeout fires first. The router has to know which modalities are async and enforce it.
- **The gateway that cannot proxy duplex.** The team picked a gateway six months ago. Now they need realtime voice and the gateway does not proxy WebSocket streams. The trade-off note (chapter 01) should have flagged the duplex requirement.
- **Speech PII that reaches the trace store.** Transcripts of user speech contain SSNs, card numbers, and addresses that the user spoke aloud. Without scrubbing at the STT-output seam, all of that lands in the observability store. Mod-107 chapter 03's scrubbing pipeline runs here.

## Summary

- Modality is a **routing decision**, not a hard-coded call. Every modality has a route with at least one alternative, and every route has an output shape so fallbacks are commensurable.
- Vision splits into **vision-native LLMs** (reason about images) and **document-AI / OCR** (extract structured text). Route by task, not by "which model do we have."
- Audio splits into **short-clip / streaming STT** (interactive) and **long-form STT** (batched); TTS routes name the voice and the chunk-streaming policy.
- Image and video generation have far larger cost-per-call than text; video is async and lives on a queue.
- **Realtime voice** (Realtime API / EVI / ElevenLabs / Vapi / Retell) collapses STT + LLM + TTS into a duplex stream with a strict end-to-end latency budget. The router's fallback is a **composed** STT → text-LLM → TTS pipeline that degrades gracefully.
- The gateway (chapter 01) has to proxy the transports each modality uses — the trade-off note flags duplex / streaming requirements up front.
- Speech PII, image resolution normalisation, and output-shape parity are cross-cutting concerns owned at the router seam.

[Chapter 05](05-cross-provider-contract-testing.md) formalises the "commensurable fallback" discipline that has run through chapters 02 and 04: cross-provider contract tests, per-provider golden traces, and the machinery that keeps a router swap from silently regressing your feature.
