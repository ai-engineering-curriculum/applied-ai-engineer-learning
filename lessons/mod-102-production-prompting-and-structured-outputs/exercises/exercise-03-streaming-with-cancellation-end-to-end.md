# exercise-03: Streaming With Cancellation End To End

**Estimated effort:** 2 hours

## Objective

Build a minimal end-to-end streaming surface — provider → backend → SSE → browser — that renders tokens as they arrive, exposes a cancel button that stops both the UI *and* the upstream provider call, and proves the upstream cancel happened. Assert the property that most teams get wrong: **that clicking cancel stops the provider from billing further tokens.**

## Learning outcomes

By the end you should be able to:

- Wire a provider streaming API (Anthropic Messages stream, OpenAI Chat Completions / Responses stream, or an equivalent) into a backend SSE endpoint that respects client disconnects.
- Render partial tokens to a browser DOM (or a simple TUI) with the three visual states from [chapter 03](../03-streaming-and-cancellation.md): still streaming, ended cleanly, errored / cancelled.
- Propagate a UI abort all the way to `stream.close()` on the provider SDK, and *prove* it in a test.
- Measure client-side TTFT, cancel latency (click → UI stop), and cancel round-trip (click → provider close).
- Explain the three failure modes streaming introduces: partial output before an error, markdown reflow mid-stream, and orphaned upstream generation on cancel.

## Prerequisites

- You've read [chapter 03](../03-streaming-and-cancellation.md).
- You are comfortable with async Python (`asyncio`, an ASGI framework like FastAPI or Starlette) *or* Node (async iterators, Express / Fastify). Pick one.
- You have access to a provider that supports streaming (Anthropic, OpenAI, Gemini, or a self-hosted vLLM endpoint with streaming enabled).
- Comfortable with `EventSource` / `fetch` + `ReadableStream` on the browser side, or an equivalent CLI SSE consumer.

## Problem statement

Build a small app with three components:

1. **Backend** — an HTTP endpoint `POST /stream` that accepts `{ "prompt": "..." }`, opens an upstream streaming request to your chosen provider, and re-emits the stream to the client as SSE (`Content-Type: text/event-stream`). Must respect client disconnect and call the provider SDK's stream close on disconnect.
2. **Client** — a minimal single-page HTML app (no framework needed) with a text input, a submit button, a streaming output area, and a stop button. Or a small CLI client if you prefer terminal — either is fine, but the requirements below are written assuming a browser.
3. **Test harness** — an automated integration test that opens a stream, cancels after N milliseconds, and asserts the upstream provider stream was closed within another M milliseconds.

## Requirements

### Part A — Backend

- On receipt of a `POST /stream`, open the provider streaming request (Anthropic `client.messages.stream(...)` or OpenAI `client.chat.completions.create(stream=True)` or `client.responses.create(stream=True)`).
- Emit each provider event as an SSE event to the client. At minimum, forward token deltas as `event: token\ndata: {"text": "..."}\n\n` and the final done event as `event: done\ndata: {}\n\n`. If an upstream error occurs, emit `event: error\ndata: {"message": "..."}\n\n` and close.
- On every yield boundary, check whether the client has disconnected (Starlette: `await request.is_disconnected()`; Node: subscribe to the `close` event on `req`). On disconnect, **call the provider SDK's `stream.close()`** (or `aclose()`), then exit the generator.
- Emit a `keepalive` comment (`: ping\n\n`) every 15 seconds so intermediaries don't idle-drop the connection.
- Log to structured stdout: request id, provider request id, "streaming started", token count at N-second intervals, "streaming ended (reason: completed | cancelled | errored)", total tokens streamed.

### Part B — Client

- On submit, open a streaming request. Use `fetch` with a `ReadableStream` reader rather than `EventSource` if you want to keep POST semantics (EventSource is GET-only), or wire an `EventSource` off a session-scoped GET id if you prefer that shape — both are acceptable.
- Append each `token` event to the output area as it arrives.
- Distinguish three visual states:
  - **Streaming**: caret / typing indicator visible.
  - **Ended cleanly**: caret hidden; a "copy" or "retry" affordance appears.
  - **Errored / cancelled**: partial text remains visible; a clear "generation was interrupted" label appears with a "retry" button.
- The stop button aborts the request via `AbortController.abort()`. On abort, the UI transitions to the "errored / cancelled" state within 100 ms of the click.
- Render tokens as monospace plain text for this exercise. (Streaming-safe markdown rendering is a stretch goal.)

### Part C — Test harness

- Automated test using pytest + `httpx.AsyncClient` (Python) or Jest + `undici` (Node), or the framework of your choice.
- Test 1 — **happy path**: open a stream, consume tokens, assert the `done` event arrives, assert total tokens > 0.
- Test 2 — **cancel path**: open a stream, cancel after 500 ms of streaming, assert:
  - The client abort completes within 100 ms.
  - The backend log records "streaming ended (reason: cancelled)".
  - The provider SDK stream had `close()` called on it. Mock the provider SDK for this test — assert `close.assert_called_once()` on the mock.
- Test 3 — **error path**: mock the provider SDK to raise an exception after 100 ms of streaming, assert the client receives an `event: error` and the UI would enter the errored state.
- Test 4 — **backpressure / idle-timeout path**: open a stream, don't read from it for 30 seconds, assert that either the connection stays alive (because of keepalives) or the backend gracefully closes the upstream. Which behaviour you pick is a design choice; document your choice.

### Part D — Measurements

Run the client against a real provider on a short prompt (target ≈ 200 tokens generated). Record:

- **TTFT (client-side)** — from `submit` click to first token rendered.
- **TPOT (client-side)** — mean inter-token latency after the first.
- **End-to-end** — from submit to `done` event.
- **Cancel-click → UI-state-change latency** (from a mid-stream cancel).
- **Cancel-click → upstream-close latency** — how quickly the backend log shows "streaming ended (reason: cancelled)" after the client abort.

Report as median + p95 over 10 runs each.

## Starter guidance

- **Do not buffer the entire response before flushing.** The classic bug is a middleware or reverse proxy that buffers `text/event-stream`. Explicitly set `X-Accel-Buffering: no` for Nginx and confirm your local dev server doesn't buffer. Test with the browser DevTools Network tab — you should see the response stay pending with bytes arriving over time.
- **On the Anthropic SDK**, use the async streaming context manager: `async with client.messages.stream(...) as stream: async for event in stream: ...`. `stream.close()` (implicit on `__aexit__`) is what stops upstream generation; the explicit call inside a disconnect check is how you stop it *early*.
- **On the OpenAI SDK**, the streaming response supports iteration; cancel via closing the response context or by using an `AbortController`-shaped mechanism from the SDK. Confirm which the current SDK exposes.
- **Use a partial-JSON parser only if you're rendering structured output.** For this exercise the output is plain text — no parser needed. If you extend to tool calls or JSON extraction, chapter 03's partial-parser section applies.
- **Test the mock stream first.** Before testing against a real provider, run the whole flow against a mock stream that yields "hello world" one character at a time with `asyncio.sleep(0.05)`. That eliminates provider costs from the debug loop and gives you a predictable stream to assert against.
- **Verify upstream-close semantics.** The most common way to prove this without touching the wire is: mock the provider SDK, wrap the mock stream's `close()` method with a `Mock`, and assert it was called. If you want to prove it against a real provider, watch the token count in the provider's usage logs and confirm it stops climbing shortly after the cancel.

## Acceptance criteria

- [ ] Backend forwards streaming tokens as SSE events with distinct `token`, `done`, and `error` event types.
- [ ] Backend polls for client disconnect and calls provider `stream.close()` on disconnect.
- [ ] Client renders partial tokens as they arrive with a visible typing indicator.
- [ ] Client distinguishes streaming / ended / cancelled visual states.
- [ ] Client cancel button aborts within 100 ms and transitions to cancelled state.
- [ ] Test 1 (happy path) passes.
- [ ] Test 2 (cancel path) passes and includes an assertion on `stream.close()` being called on the provider mock.
- [ ] Test 3 (error path) passes.
- [ ] Test 4 (backpressure / keepalive) is documented — either passing with keepalives or with a documented graceful timeout.
- [ ] Measurements section reports TTFT median + p95, TPOT median + p95, end-to-end median + p95, cancel-to-UI and cancel-to-upstream-close latencies.
- [ ] The report includes a written explanation of *why* the upstream-close assertion in Test 2 is the load-bearing test — the one that catches the most common streaming bug.

## Stretch goals

Pick any two:

- **Mid-stream tool call.** Extend the backend to handle a provider stream that includes a tool-call event mid-stream. Accumulate the tool arguments (Anthropic `input_json_delta` or OpenAI accumulating chunks) keyed by tool-call id; only parse and dispatch on the tool-call-done event; emit `event: tool_call` to the client so it can render a "running tool: …" indicator.
- **WebSocket variant.** Reimplement the backend + client using WebSocket instead of SSE. Show the extra work: heartbeat / ping frames, explicit cancel-message shape, connection-level reconnect handling.
- **Streaming-safe markdown.** Render the streaming output as markdown that updates in place. Use a permissive parser (e.g. `marked` in JS, `mistune` in Python) that handles unclosed constructs; measure and report the layout-thrash you observed and how you damped it.
- **Cancel-during-tool-call.** Design and test cancellation semantics when the user cancels *while a tool call is in flight*. Which tools are safe to cancel and which must be allowed to complete? Document a policy and implement it for one example tool (e.g. a `send_email` tool must complete or roll back; a `search_docs` tool is safe to abort).
- **Auto-scroll politeness.** Implement the auto-scroll rule from chapter 03: only auto-scroll if the user is at the bottom; if they scrolled up, pause auto-scroll and show a "jump to latest" affordance.

## Deliverable

Submit as `mod-102/exercise-03/` in the paired `applied-ai-engineer-solutions` repo:

- `server.py` (or `server.ts`) — the streaming backend.
- `index.html` + `client.js` (or `client.ts`) — the browser client.
- `tests/` — the four integration tests.
- `measurements.md` — the latency + cancel table from Part D.
- `report.md` — a short (≤ 1 page) discussion of the upstream-close bug and how the test catches it.
