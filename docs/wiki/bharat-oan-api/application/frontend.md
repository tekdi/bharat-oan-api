# Frontend

> **Note:** This repository contains only the backend API. There is no web or mobile user interface, no JavaScript/TypeScript build, and no static asset serving for an application UI in this service. The client applications (for example the BharatVistaar web chat and an Android app using Play Integrity) are developed and hosted outside this repository. This page documents the contract those clients are expected to follow, as implemented by the API.

Related pages: [API specification](api-specification.md) | [Backend](backend.md) | [Sequence diagrams](../architecture/sequence-diagrams.md) | [Users and personas](../business/users-and-personas.md)

---

## What the API Serves

| Served content | Route | Source |
|----------------|-------|--------|
| JSON endpoints and a streaming chat response | See [API specification](api-specification.md) | `app/routers/*.py` |
| Cached, server-generated HTML documents (for example status/report pages produced by tools) | `GET /api/file/{file_hash}` | `app/routers/file.py` |
| Uploaded crop images | `GET /api/image/{image_id}` | `app/routers/image.py` |
| Auto-generated OpenAPI docs | FastAPI defaults (`/docs`, `/openapi.json`, `/redoc`) - no custom configuration in `main.py` | `main.py` |

No templates, no `StaticFiles` mount and no SPA fallback are configured in `main.py`.

## Expected Client Behaviours (Derived from Backend Code)

| Concern | Contract implemented by the backend |
|---------|-------------------------------------|
| Sign-in | Client first obtains a JWT: `POST /api/token` (guest, optional `fingerprint_id`, `mobile`, `name`, `role`, `metadata`), `POST /api/token/api-key`, or `POST /api/token/play-integrity`. It then sends `Authorization: Bearer <jwt>` on every protected call. Default token lifetime is `JWT_EXPIRY_MINUTES` = 15 minutes, so clients must re-request tokens (the API provides no refresh endpoint) |
| Chat streaming | `GET /api/chat/` with query parameters. The response uses `Content-Type: text/event-stream` but the body is the **raw answer text** emitted in incremental chunks, not `data:`-framed SSE events - clients must read the body stream and append chunks (`tests/test_chat_stream_contract.py` asserts `response.text == "assistant output"` for a single chunk). Details in [api-specification.md](api-specification.md#5-chat-streaming-contract) |
| Correlation id | Response header `X-QID` carries the question id (client-supplied `qid` or a server-generated UUID). It is exposed through `Access-Control-Expose-Headers: X-QID` so browser JavaScript can read it, and the client reuses it for `POST /api/telemetry/feedback` and `/api/telemetry/error` |
| Session continuity | Client keeps `session_id` (server generates a UUID when omitted; note that the generated id is **not** returned in a header or body for `GET /api/chat/`, so clients that need continuity should send their own `session_id`) |
| Voice input | `POST /api/transcribe/` with base64 audio and `lang_code` |
| Voice output | `POST /api/tts/` returns base64 audio in the `audio_data` field |
| Image diagnosis | `POST /api/image/upload` returns an `image_id`; the client then sends that id (or `/api/image/<id>` URL) as the `query` of a normal chat request. `latitude` / `longitude` may be supplied as query parameters |
| Client telemetry | `POST /api/telemetry/events` (generic UI events), `/feedback`, `/error` - the backend relays them to the observability service |
| Language | `source_lang` and `target_lang` language codes (`hi` default). Prompt files exist for as, bn, en, gu, hi, kn, mai, ml, mr, or, pa, ta, te (`assets/prompts/agrinet_<lang>.md`) |

## CORS Configuration

Implemented in `main.py` via `CORSMiddleware`:

| Setting | Source | Default |
|---------|--------|---------|
| `allow_origins` | `settings.allowed_origins` from env `ALLOWED_ORIGINS` (comma-separated; values are not trimmed) | `["*"]` |
| `allow_credentials` | `settings.allowed_credentials` | `True` |
| `allow_methods` | `settings.allowed_methods` | `["*"]` |
| `allow_headers` | `settings.allowed_headers` | `["*"]` |
| Exposed header (chat only) | Set on the streaming response: `Access-Control-Expose-Headers: X-QID` | - |
| `Timing-Allow-Origin` | `TimingAllowOriginMiddleware` (all responses) | `*` when origins include `*`, otherwise the single configured origin |

> **Note:** The default `ALLOWED_ORIGINS=*` is permissive and combined with `allow_credentials=True` is intended for development. Production deployments should set `ALLOWED_ORIGINS` explicitly to the front-end origins. Browsers do not honour a wildcard origin together with credentials.

## Recommended (not yet implemented)

- A token-refresh endpoint or longer-lived session token so web clients do not need to re-authenticate every 15 minutes.
- Returning the generated `session_id` to the client (for example as a response header next to `X-QID`).
- True SSE framing (`data:` lines plus keep-alive comments) if browser `EventSource` support is desired.

## Source Files

`main.py`, `app/config.py`, `app/routers/chat.py`, `app/routers/file.py`, `app/routers/image.py`, `app/routers/token.py`, `assets/prompts/`.
