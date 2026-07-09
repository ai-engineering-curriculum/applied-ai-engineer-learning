# Streaming and Cancellation, End to End

## Why streaming is a product decision, not an optimisation

A non-streamed model response makes the user wait for the whole answer before seeing anything. On a 300-token completion at 60 tokens per second, that's a five-second stare at a spinner. On a 3,000-token completion, it's fifty seconds. Users leave.

Streaming — emitting tokens as they are generated — replaces that stare with a **time to first token (TTFT)** the user actually feels. A well-shaped streaming surface renders the first token in under a second, keeps drawing at a readable pace, and gives the user a **cancel** button that stops both the UI *and* the upstream provider call. Get any of those three parts wrong and streaming is worse than not streaming: a stalled half-answer with a cancel button that does nothing is more frustrating than a five-second wait for a clean result.

Streaming introduces new failure modes: partial output that renders before an error, a schema whose validity is only defined at end-of-stream, tool calls whose arguments arrive incrementally, and cancellation semantics that must be honoured all the way to the provider. This chapter walks the end-to-end shape — provider → backend → transport → UI → cancel — with the failure modes named.

## The four hops of a streaming response

```
[Provider inference server]
        │  provider SSE / event stream (varies by provider)
        ▼
[Application backend]
        │  transport: SSE or WebSocket
        ▼
[Client transport layer]
        │  DOM update or React state update
        ▼
[UI render surface]
```

Cancellation must propagate the other direction, all the way from a UI click back to the provider. Each hop can silently break the pipe.

### Hop 1 — Provider event stream

Each managed provider emits a distinct stream shape:

- **OpenAI Chat Completions** and **Responses API** stream deltas over SSE-shaped chunks. Chat Completions emits `choices[].delta.content` (and `delta.tool_calls[]` for tool arguments); Responses emits a richer event stream (`response.output_text.delta`, `response.output_item.done`, `response.completed`, etc.) with explicit lifecycle events for each output item.
- **Anthropic Messages streaming** emits a typed event stream: `message_start`, `content_block_start`, `content_block_delta` (with `input_json_delta` for tool arguments), `content_block_stop`, `message_delta`, `message_stop`. The typed events make partial-tool-call state explicit.
- **Google Gemini** streams via `generateContentStream` returning candidate deltas with parts (text, function-call arguments).
- **AWS Bedrock** wraps the underlying model's stream in `InvokeModelWithResponseStream` / `ConverseStream`, exposing chunks whose shape depends on the model.

**You do not code against the raw event shape directly** in most cases; the provider SDK exposes it as an async iterator. But you should understand three things about each stream:

1. **What lifecycle events exist** — message start, content-block start/stop, message stop. Rendering logic hangs off these.
2. **How incomplete tool-call arguments are conveyed** — as an accumulating JSON delta (Anthropic's `input_json_delta`), or as accumulating string chunks per tool call (OpenAI). You must accumulate correctly before parsing.
3. **What "cancellation" means to that SDK** — dropping the HTTP connection, calling an explicit `close()` on the stream, or aborting via a `CancelToken`.

### Hop 2 — Application backend

The backend receives the provider event stream and translates it into an event stream for its own client — typically SSE, sometimes WebSocket. In async Python (FastAPI + `httpx.AsyncClient` + `starlette.responses.StreamingResponse` or `sse_starlette.EventSourceResponse`) or Node (Express / Fastify + async iterators + `res.write` on a `text/event-stream` response), the shape is:

```python
async def stream_response(req):
    async def event_stream():
        async with client.messages.stream(**params) as stream:
            async for event in stream:
                if await req.is_disconnected():
                    await stream.close()   # tell the provider to stop generating
                    return
                yield format_sse(event)
    return EventSourceResponse(event_stream())
```

Three properties of a healthy backend stream:

- **Backpressure aware.** If the client stops reading (a paused download, a suspended tab), the backend blocks on its socket write, which back-propagates and eventually cancels the upstream provider stream. Do not buffer the entire response and then flush at the end; that defeats the point.
- **Disconnect aware.** Poll `req.is_disconnected()` (Starlette) or listen for the equivalent event (Node) at every yield boundary, and close the upstream stream when it fires. This is the cancel path when the user closes the tab.
- **Error framed inside the stream.** When the upstream fails mid-stream, do not silently close — emit an `error` SSE event with a structured payload so the UI can render "Something went wrong" instead of leaving a spinner spinning. Chapter 04 covers the mid-stream repair loop.

### Hop 3 — Transport: SSE vs WebSocket

Both work; pick based on the flow, not the vibe.

**Server-Sent Events (SSE)** — one-way server-to-client stream over HTTP. Standard `text/event-stream` content type, browser-native `EventSource`, works through most proxies, uses the same TCP connection as normal HTTP, benefits from HTTP/2 multiplexing. Cancellation is by closing the connection (JavaScript `EventSource.close()` or aborting a `fetch` with an `AbortController`).

- **Fits when**: streaming a response back for a single user turn. Almost every LLM chat surface fits here.
- **Trade-offs**: only server-to-client. If you want to send interim client events (e.g. "the user is still watching, keep going"), you need a separate channel.

**WebSocket** — full-duplex, persistent connection. Both sides can send at any time. Useful when the client sends events *during* generation — e.g. voice input, live document editing that updates the model's context, agent-shape mid-run tool approval.

- **Fits when**: agent surfaces with interactive tool approval, multi-user co-editing surfaces, voice agents.
- **Trade-offs**: connection management is heavier (heartbeats, reconnects, backoff), and some proxies / load balancers mishandle long-lived upgrades.

**Cancellation shape differs.** SSE cancel = close the HTTP connection. WebSocket cancel = send a well-defined cancel message *and* close, so the server knows the user cancelled vs the socket dropped.

Both transports should ship with a heartbeat / keepalive so intermediaries don't idle-drop the connection during a slow generation.

### Hop 4 — UI render surface

The UI does three things in a streaming render:

1. **Renders each token as it arrives.** Not each byte; SSE and provider chunks are token-level. Concatenate to the running buffer and re-render.
2. **Shows a stop button that actually stops.** The stop button aborts the client-side `fetch` / `EventSource`, which drops the HTTP connection, which the backend notices and translates into an upstream cancel.
3. **Distinguishes "still streaming" from "stream ended" from "stream errored".** Three visual states, not one. A stream that ended cleanly renders the final answer with any downstream affordances (copy, retry, feedback). A stream that errored mid-way shows the partial output plus a clear "generation was interrupted" affordance and a retry button. A stream that is still streaming shows a caret / typing indicator so the user knows generation is ongoing.

Partial-render pitfalls:

- **Markdown that renders differently mid-stream.** A `**bold**` token that shows one `*` then two `**` briefly renders "**" as literal characters. Render markdown incrementally with a parser that handles unclosed constructs, or defer markdown parsing to end-of-stream and render as monospace-then-format.
- **Code blocks that reflow the layout.** Streaming into a `<pre>` block should reserve minimum height to avoid a layout thrash.
- **Auto-scroll fighting the user.** The classic bug: user scrolls up to read a prior message, streaming continues, auto-scroll yanks them back to the bottom. Detect user scroll-up and pause auto-scroll until they return to the bottom.

## Streaming a schema-safe output

Chapter 02 landed the schema-safe output as a *complete* Pydantic model. Streaming reintroduces the "what does partial mean" question.

Two shapes to know:

### Partial JSON parsing

For a JSON extraction schema, most of a partial stream is not valid JSON. `{"vendor_name": "Ac` is not valid until it becomes `{"vendor_name": "Acme"}` and eventually the object closes. To render partial fields in the UI (progressive rendering), use an **incremental / permissive JSON parser** that accepts prefixes and returns "best-guess" partial trees:

- [partial-json-parser](https://github.com/promplate/partial-json-parser) (Python and JS) — parses partial JSON, returning the deepest complete subtree.
- Instructor exposes `client.chat.completions.create_partial(response_model=Model, stream=True)` which yields incrementally-populated Pydantic instances (from the partial-json parser under the hood).

The rule for the UI: **treat partial fields as tentative and never commit them to storage until the stream ends cleanly.** A partial `total_amount: 12` that becomes `total_amount: 1200` when the next token arrives will corrupt any store that persists on every delta.

### Mid-stream tool calls

Function / tool calls arrive as accumulating argument deltas. The provider emits `input_json_delta` (Anthropic) or growing `arguments` string chunks (OpenAI). Two failure modes:

- **Parsing before end.** Do not `json.loads` the growing arguments string. Only parse when the content-block-stop / tool-call-done event fires. Every provider emits an explicit "this tool call is complete" event; use it.
- **Multiple concurrent tool calls in one turn.** Both OpenAI and Anthropic can emit multiple tool calls per turn, interleaved. Track them by their id (Anthropic: `content_block_start.index`; OpenAI: `delta.tool_calls[].index`) and accumulate arguments per id. Do not concatenate all tool-call arguments into a single buffer.

The tool-call flow becomes: stream in → accumulate per-id → on content-block-stop parse and validate → dispatch (with authorisation!) → capture result → resume streaming with the result as a new tool-result message.

## Cancellation, all the way through

An abort in the UI must:

1. Cancel the client fetch / EventSource (`AbortController.abort()` or `EventSource.close()`).
2. The backend must detect the closed client connection at the next stream yield.
3. The backend must call `stream.close()` (or equivalent) on the provider SDK to stop upstream generation and stop billing.
4. Any in-flight tool call started by the model must be handled — either allowed to complete (if it has side effects that can be safely rolled back or are idempotent) or explicitly cancelled if the tool supports it.
5. The partial output must remain readable in the UI (do not blank the screen on cancel) with a clear "cancelled" affordance.

The most common bug is step 3. Backends that translate provider events into an outbound stream often forget to close the *upstream* SDK stream when the *downstream* client goes away. Result: the user cancels, the UI stops, and the model keeps generating for another 20 seconds while you keep paying. The bug looks like nothing on the UI and shows up on the bill.

Test it: in an integration test, subscribe to the stream, cancel after 500 ms, and assert that the provider mock saw a `close()` call within another 100 ms. Chapter 04 turns this into a fixtured regression test.

## Latency SLOs for a streaming surface

Recall the SLO shape from mod-101 chapter 02 — every promise gets a number.

- **TTFT p95** — time from user submit to first rendered token. Measured **client-side**, because the network hop is what the user feels. Target based on the surface: a chat surface should aim under a second; a co-pilot inline suggestion needs less.
- **TPOT / ITL p95** — inter-token latency after the first. A model with fast TTFT but slow TPOT feels good and then drags; measure both.
- **End-to-end p95** — from user submit to end of stream. Includes tool round-trips for surfaces that emit tool calls.
- **Cancel latency** — time from user click to (a) UI update, (b) provider stream closed. Both should be under 100 ms client-side, under a second all-the-way-through.

Report percentiles, not averages, and exclude user-cancelled runs from end-to-end (they are not failures, they are successful cancels).

## Summary

- Streaming is a product decision. A streaming surface must render early, render honestly, and honour cancellation all the way to the provider, or it is worse than a plain wait.
- The end-to-end flow has four hops — provider event stream, application backend, transport (SSE or WebSocket), UI render — with a cancellation path that runs the other direction.
- SSE fits almost every LLM streaming surface; WebSocket fits agent and voice surfaces with duplex traffic.
- Partial JSON is not valid JSON; use a permissive parser (Instructor's `create_partial` or `partial-json-parser`) for progressive rendering and never persist partial fields.
- Tool calls arrive as accumulating deltas keyed by id; only parse and dispatch when the provider emits the tool-call-done event.
- Cancellation must propagate all the way — UI abort → backend disconnect detection → upstream `stream.close()`. Forgetting step 3 is the classic silent-cost bug.
- Streaming SLOs are TTFT, TPOT, end-to-end, and cancel latency, all measured client-side and reported as percentiles.

Chapter 04 covers what to do when the response arrives but fails validation — the retry-and-repair loop with bounded budgets, and how to build one that does not become a runaway cost multiplier.
