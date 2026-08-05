# exercise-04: Multi-Modal Routing — Vision, Audio, Realtime

**Estimated effort:** 2 hours

## Objective

Extend the route table from exercise-02 with **at least two additional modalities** (from: vision-native chat, document-AI / OCR, streaming STT, TTS, image generation, video generation, realtime voice), and — for realtime voice specifically — implement the **composed fallback** (STT → text-LLM → TTS) that keeps the feature functional when the native realtime API is unavailable. Prove the router picks the correct modality per request and degrades gracefully under a simulated realtime outage.

## Learning outcomes

By the end you should be able to:

- Extend a route table to modality-aware routes with per-route output shapes.
- Distinguish **vision-native LLM** routes from **OCR / document-AI** routes and pick per request.
- Handle the streaming / async modality distinction (chat-STT is streaming; long-form STT and video-gen are async / queued).
- Implement a **composed fallback** for realtime voice — STT → text-LLM → TTS — that runs when the primary realtime API is down.
- Normalise **image resolution presets** and **STT output shape** at the router seam so downstream code does not care which provider served the request.

## Prerequisites

- You have read [chapter 04](../04-multi-modal-routing-vision-audio-realtime.md).
- Exercise-02's router — the routes, classifier, and quality gates it produced are the substrate this exercise extends.
- Access to at least two providers per additional modality (e.g. for STT: OpenAI Whisper + one of Deepgram / Speechmatics / AssemblyAI; for realtime voice: OpenAI Realtime + a text-LLM + a TTS provider for the composed fallback).<!-- needs-research: verify current API availability for these products at author time. -->
- The gateway pick from exercise-01 has to proxy the transports the modalities need. If it does not, note the gap and use a direct-to-provider path for this exercise.

## Problem statement

The route table you shipped in exercise-02 is text-only. The product team wants to add two features: (1) users can attach a document to the chat and ask questions about it, and (2) users can hold a voice conversation with the same surface. Both features have been prototyped by three different engineers, and each of them wrote a hard-coded provider call in feature code. Your job is to pull those calls back to the router seam and to make the realtime voice feature survive an outage of the native realtime API.

In two hours, ship the modality-aware router and prove the composed realtime fallback works.

## Requirements

### Deliverable 1 — Modality choice

Under `docs/modality-choices.md`, one page:

- Which two (or more) modalities you are adding beyond text. Pick from: vision-native chat, document-AI / OCR, streaming STT, TTS, image gen, video gen, realtime voice.
- **Realtime voice must be one of the two** — the composed-fallback requirement in Deliverable 4 exercises it directly.
- For each chosen modality, one paragraph: what surface / feature uses it, what the interactive SLO shape is (streaming / async / target latency), and which two providers are the primary and fallback.

### Deliverable 2 — The modality-aware route table

Extend `routing/routes.yaml` (from exercise-02) with the new routes in the shape of [chapter 04](../04-multi-modal-routing-vision-audio-realtime.md):

- Every route names an **output shape** (e.g. `ocr_v1`, `chat_v1`, `audio_mp3`, `stt_words_v1`) that both the primary and every fallback conform to.
- Every route has a **modality-appropriate SLO** — a realtime-voice route has an end-to-end p95 in the sub-second range; a video-gen route has a p95 in minutes and is marked async.
- The route for realtime voice includes a **composed fallback**: `{ stt: <stt route>, llm: <text route>, tts: <tts route> }`.

### Deliverable 3 — The router extension

Extend `routing/router.py` (from exercise-02) to dispatch per modality:

- The router accepts the request modality as an explicit parameter (or infers it from the request payload — image attached, audio-bytes attached, etc.) and picks the corresponding route.
- **Image resolution normalisation** at the router seam — if the route has a preset (`low` / `high` / a specific resolution), the router resizes / re-encodes the image before dispatch.
- **STT output normalisation** — the router transforms the provider's response into the route's `output_shape` before returning (e.g. `[{start, end, speaker, text}]`) so downstream code is provider-agnostic. Fallbacks with weaker native support get an explicit *lossy* normalisation (e.g. no diarization → single-speaker) with a warning in the response envelope, not a silent field-drop.
- **Async modalities** (video gen, long-form STT) are enqueued to a background substrate (a job queue is fine; even a simple in-process asyncio queue is acceptable for the exercise) and return a job handle, not a completion.

### Deliverable 4 — The composed realtime-voice fallback

Under `routing/realtime.py`:

- The **primary** realtime path opens the native realtime WebSocket / duplex stream and forwards user audio bytes to it.
- The **composed fallback** — activated by the same circuit-breaker discipline from exercise-02 — chains STT → text-LLM → TTS:
  - User audio chunks are streamed to the STT route as they arrive.
  - Once a natural boundary (end-of-utterance detected by the STT, or a silence threshold) is hit, the text-LLM route is called with the transcript.
  - The text-LLM response is streamed to the TTS route.
  - TTS audio chunks are streamed back to the client.
- The response envelope carries a field indicating whether the primary or the composed fallback served the request, and — if the composed path — the end-to-end latency broken out per stage.

The fallback does not have to match the primary's turn-taking latency or barge-in behaviour. It has to *function* — the user gets a spoken response — and the router has to *tell the client* the experience is degraded.

### Deliverable 5 — Two proofs

Under `proofs/`:

- `modality-picked.md` — show three requests (a text message with an image attached, a document-upload for extraction, and a voice audio-blob for STT) and the routing-decision log for each. Prove the router picked the correct modality route and the correct output shape.
- `realtime-degraded.md` — simulate a failure of the native realtime API (kill the API key, use a mock that raises, or hit a rate-limit). Run a short voice conversation. Show the routing-decision log confirming: (a) the composed fallback fired, (b) the client received a spoken response, (c) the response envelope indicated degraded mode, and (d) the end-to-end latency per stage.

## Starter guidance

- **Do not attempt every modality.** Two is the acceptance floor for a reason. Vision-native chat + realtime voice is a very defensible pair; document-AI + streaming STT is another.
- **Image resolution normalisation before the provider call is the single biggest cost lever.** A prod feature that sends full-resolution phone photos to a vision LLM is often paying 3× what it should. Land this even if you skip other optimisations.
- **STT diarization is where the fallback most often silently breaks.** Pick a primary that supports diarization and a fallback that either supports it or explicitly drops to single-speaker with a warning field. Do not drop the field.
- **Async modalities need an actual queue.** A background asyncio task is fine for the exercise; do not block the HTTP handler on a video-gen call. Return a job handle immediately; poll or webhook back on completion.
- **The composed realtime fallback does not need to be beautiful.** A working STT → text → TTS chain that produces a legible spoken response counts as a success. The point of the exercise is that it *exists*, not that it feels indistinguishable from the primary.
- **The gateway audit from exercise-01 is load-bearing here.** If the gateway you picked cannot proxy WebSocket / duplex streams, the primary realtime path bypasses the gateway; note the exception in your write-up.

## Acceptance criteria

- [ ] `docs/modality-choices.md` names the two (or more) modalities and includes realtime voice.
- [ ] `routing/routes.yaml` has modality-appropriate routes with `output_shape` and modality-appropriate SLOs.
- [ ] Realtime voice route names a `composed:` fallback with `stt`, `llm`, `tts` references to other routes on the table.
- [ ] `routing/router.py` dispatches per modality, normalises image resolution at the seam, normalises STT output to the declared shape (with an explicit warning field for lossy fallbacks), and enqueues async modalities.
- [ ] `routing/realtime.py` implements the composed fallback and returns a response envelope with `mode: primary | composed_fallback` and per-stage latency.
- [ ] `proofs/modality-picked.md` shows correct modality dispatch on three different-modality requests.
- [ ] `proofs/realtime-degraded.md` demonstrates the composed fallback serving a full voice turn during a simulated primary outage, with the degradation flagged in the client response.
- [ ] No feature-code file imports a modality provider SDK directly; every modality call goes through the router.

## Stretch goals

Pick any one:

- **Speech-PII scrubbing.** Wire the mod-107 chapter-03 scrubbing pipeline on STT output before it lands in the LLM prompt or in traces. Show a `before / after` on a transcript that contained a spoken card number.
- **Vision-native vs OCR bake-off.** Add both routes for the document-question feature and A/B them on a fixed set of documents. Report recall, latency, and cost. Recommend a default with a rationale.
- **Realtime session-boundary handling.** Extend the composed fallback to handle mid-session failure — the native realtime API drops mid-conversation; the composed path picks up the next user turn without dropping the session state. This is the harder version of the composed fallback and worth the write-up.
- **Async substrate wire-up.** For the video-gen (or long-form STT) route, ship the queue + worker + webhook path end-to-end, not the in-process asyncio version. This is more infra work but it is the honest shape.
- **Cost per turn dashboard.** For the composed realtime fallback, produce a per-turn cost breakdown (STT + LLM + TTS) and show how it compares to the primary realtime API's session-minute pricing. The pricing shape often surprises people.

## Deliverable

Submit as `mod-108/exercise-04/` in the paired `applied-ai-engineer-solutions` repo:

- `docs/modality-choices.md`.
- `routing/routes.yaml` (updated) with the modality-aware routes.
- `routing/router.py` (updated) with per-modality dispatch and normalisation.
- `routing/realtime.py` with the primary + composed-fallback implementation.
- `proofs/modality-picked.md` and `proofs/realtime-degraded.md`.
- Any stretch-goal artefacts named alongside.
- A short `README.md` naming the modalities and how to run the proofs.
