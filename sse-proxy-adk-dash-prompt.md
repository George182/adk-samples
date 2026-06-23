# VS Code Agent Prompt — SSE Proxy for Two-Process ADK + Dash Monorepo

I have a monorepo with two processes in one container (managed by supervisord):

- `apps/api` — ADK FastAPI app via `get_fast_api_app()`, uvicorn on :8000, INTERNAL only.
  It exposes ADK's `POST /run_sse` (Server-Sent Events). Session service is
  `VertexAiSessionService` (`agentengine://` URI).
- `apps/dashboard` — Plotly Dash whose underlying server is a FastAPI app, uvicorn
  on :8080, this is the Cloud Run external port.

Only :8080 is reachable from the browser. I need the agent chat tab to stream
tokens from the API process's `/run_sse` to the browser. Implement an SSE proxy on
the dashboard FastAPI app plus a Dash clientside fetch-stream handler.

## First, inspect my code and tell me before writing anything:

1. How `apps/dashboard/main.py` exposes the FastAPI server object (the `server`
   attr of the Dash app) so I can attach a route to it.
2. Whether the agent tab currently uses `EventSource` or `fetch` for streaming
   (if it exists at all).
3. The exact request body `/run_sse` expects (`app_name`, `user_id`, `session_id`,
   `new_message`) and how sessions are created in my code.

## Then implement:

### A) A streaming proxy route on the dashboard's FastAPI server, e.g. `POST /api/agent/stream`:

- Fully async. Use `httpx.AsyncClient.stream()` to POST to
  `http://localhost:8000/run_sse` and relay frames.
- Return `StreamingResponse(media_type="text/event-stream")` yielding from an
  async generator; flush each frame immediately, never accumulate the body.
- httpx timeout MUST be `httpx.Timeout(connect=5.0, read=None)` so long agent
  turns don't get cut.
- Set response headers `Cache-Control: no-cache` and `X-Accel-Buffering: no`.
- If any compression/GZip middleware is on the dashboard app, exclude
  `text/event-stream` from it (it buffers and breaks SSE).
- Check `await request.is_disconnected()` and tear down the upstream httpx
  stream when the client disconnects, so the agent run on :8000 doesn't leak.
- SECURITY: do NOT accept `user_id` from the client. Inject `user_id` and the
  per-tab `session_id` server-side from the dashboard's authenticated session.
  The client sends only the message text and which tab/dataset it's scoped to.
  (If auth isn't wired yet, stub `user_id` from a single hardcoded value for now,
  but keep the injection point server-side so the security boundary exists.)

### B) Client side in the agent tab:

- `/run_sse` is POST, so `EventSource` won't work (it's GET-only). Use `fetch()`
  with a streaming `ReadableStream` body reader.
- A Dash clientside callback fires on the send button, POSTs the message to
  `/api/agent/stream`, reads the stream, parses SSE frames, and appends tokens
  to the chat output via `dash_clientside.set_props` as they arrive. No
  server-side Dash callback in the token loop.

## Constraints:

- Do NOT migrate to `Dash(server=...)` native FastAPI backend. Keep the two-process
  `get_fast_api_app()` model.
- Keep agent code isolated in `packages/agent_core` and `apps/api`; the dashboard
  must not import agent internals — it only talks to the API over HTTP.
- Match my existing code style and the session-creation pattern already in the repo.

## Output:

Show me the proxy route and the clientside handler, and tell me exactly which
files to add/edit.
