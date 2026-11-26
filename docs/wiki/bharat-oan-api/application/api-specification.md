# API Specification

All routes below were verified from the router decorators in `app/routers/*.py` and `main.py`. Every router is mounted with `prefix=settings.api_prefix` (default `/api`) except the root route. The app is a standard FastAPI instance, so the machine-generated OpenAPI schema is also available from the running service at `/openapi.json` (interactive docs at `/docs`).

Related pages: [Backend](backend.md) | [Frontend / client contract](frontend.md) | [Sequence diagrams](../architecture/sequence-diagrams.md) | [Business workflows](../business/workflows.md) | [Database / data stores](database.md) | [Technical debt](../knowledge/technical-debt.md)

Legacy hand-written documentation: `API_DOCUMENTATION.md` (see [Reconciliation](#reconciliation-with-api_documentationmd) - it is partly outdated).

---

## 1. Endpoint Summary

Auth column: **JWT** = `Authorization: Bearer <token>` validated by `get_current_user` (`app/auth/jwt_auth.py`); **None** = unauthenticated.

| # | Method | Path | Handler (file:function) | Auth | Purpose | Workflow |
|---|--------|------|-------------------------|------|---------|----------|
| 1 | GET | `/` | `main.py:root` | None | App name, environment, debug flag, API prefix | - |
| 2 | GET | `/api/chat/` | `app/routers/chat.py:chat_endpoint` | JWT | Stream an AI chat answer for one farmer question | [W2](../architecture/sequence-diagrams.md#w2-streaming-chat-turn) |
| 3 | POST | `/api/chat/analyze-image` | `app/routers/chat.py:chat_analyze_image_endpoint` | JWT | Deprecated multipart variant: upload an image and stream the diagnosis in one call | [W3](../architecture/sequence-diagrams.md#w3-crop-image-upload-and-analysis) |
| 4 | POST | `/api/transcribe/` | `app/routers/transcribe.py:transcribe` | JWT | Speech-to-text (Bhashini or Whisper) | [W4](../architecture/sequence-diagrams.md#w4-voice-transcription-asr) |
| 5 | POST | `/api/tts/` | `app/routers/tts.py:tts` | JWT | Text-to-speech (Bhashini) | [W5](../architecture/sequence-diagrams.md#w5-text-to-speech-tts) |
| 6 | POST | `/api/image/upload` | `app/routers/image.py:upload_image` | JWT | Store a crop image temporarily, return `image_id` | [W3](../architecture/sequence-diagrams.md#w3-crop-image-upload-and-analysis) |
| 7 | GET | `/api/image/{image_id}` | `app/routers/image.py:serve_image` | None | Serve an uploaded image by UUID (used by the NPSS tool to fetch the bytes) | [W3](../architecture/sequence-diagrams.md#w3-crop-image-upload-and-analysis) |
| 8 | GET | `/api/file/{file_hash}` | `app/routers/file.py:serve_html_file` | None | Serve a cached HTML document produced by a tool (Redis key `html_file:<hash>`) | [W9](../architecture/sequence-diagrams.md#w9-cached-html-document-serving) |
| 9 | POST | `/api/token` | `app/routers/token.py:create_auth_token` | None | Issue a guest JWT (optional body) | [W1](../architecture/sequence-diagrams.md#w1-authentication-and-token-issuance) |
| 10 | POST | `/api/token/api-key` | `app/routers/token.py:create_auth_token_with_api_key` | `X-API-Key` header | Issue a JWT for a trusted client identified by `client_code` | [W1](../architecture/sequence-diagrams.md#w1-authentication-and-token-issuance) |
| 11 | POST | `/api/token/play-integrity` | `app/routers/token.py:create_auth_token_with_play_integrity` | Play Integrity verdict | Issue a JWT to a verified Android app/device | [W1](../architecture/sequence-diagrams.md#w1-authentication-and-token-issuance) |
| 12 | POST | `/api/telemetry/feedback` | `app/routers/telemetry.py:relay_feedback_telemetry` | JWT | Relay like/dislike feedback to telemetry and attach a Langfuse score | [W6](../architecture/sequence-diagrams.md#w6-telemetry-and-feedback) |
| 13 | POST | `/api/telemetry/events` | `app/routers/telemetry.py:relay_generic_telemetry_events` | JWT | Relay a batch of generic client UI events | [W6](../architecture/sequence-diagrams.md#w6-telemetry-and-feedback) |
| 14 | POST | `/api/telemetry/error` | `app/routers/telemetry.py:relay_error_telemetry` | JWT | Relay a client-reported chat error | [W6](../architecture/sequence-diagrams.md#w6-telemetry-and-feedback) |
| 15 | GET | `/api/health/` | `app/routers/health.py:health_check` | None | App metadata, uptime, Redis dependency check (503 when unhealthy) | [W10](../architecture/sequence-diagrams.md#w10-health-probes) |
| 16 | GET | `/api/health/live` | `app/routers/health.py:liveness` | None | Liveness probe (`{"status":"alive"}`) | [W10](../architecture/sequence-diagrams.md#w10-health-probes) |
| 17 | GET | `/api/health/ready` | `app/routers/health.py:readiness` | None | Readiness probe (Redis round trip; 503 if not healthy) | [W10](../architecture/sequence-diagrams.md#w10-health-probes) |

Trailing slashes matter for routers declared with `"/"` (`/api/chat/`, `/api/transcribe/`, `/api/tts/`, `/api/health/`); FastAPI's default `redirect_slashes` redirects the slash-less form. `POST /api/token` is declared with an empty path, so its canonical URL has no trailing slash.

> **Note:** Role-based restrictions are not part of this service today: all valid tokens may call every JWT-protected route above. Image download, HTML file download, health, root and token-issuance routes require no bearer token.

---

## 2. Authentication Requirements

- Header: `Authorization: Bearer <jwt>` (RS256). Failure responses are HTTP 401 with `{"detail": "..."}` and `WWW-Authenticate: Bearer`.
- Obtain a token through the three `/api/token*` routes ([W1](../architecture/sequence-diagrams.md#w1-authentication-and-token-issuance)). Default lifetime: `JWT_EXPIRY_MINUTES` = 15 minutes (`settings.jwt_expiry_minutes`). Claim handling is described in [backend.md](backend.md#5-authentication-and-authorization).

---

## 3. Request Models

Defined in `app/models/requests.py` unless noted.

### `ChatRequest` (query parameters of `GET /api/chat/`, bound with `Depends()`)

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `query` | string | required | User question. May also carry an uploaded image id (see [Image conventions](#image-conventions)) |
| `session_id` | string | none | A UUID is generated if omitted (not returned to the client) |
| `qid` | string | none | Question id for telemetry correlation; server generates a UUID when missing |
| `source_lang` | string | `hi` | |
| `target_lang` | string | `hi` | Language of the answer; selects the `agrinet_<lang>` system prompt |
| `user_id` | string | `anonymous` | Used as the Langfuse user id and logged; not authoritative identity |
| `latitude` | string | none | Browser coordinates for image diagnosis |
| `longitude` | string | none | |

### `POST /api/chat/analyze-image` (multipart form, `app/routers/chat.py`)

`image` (file, required), `query` (default `"Analyze this crop image"`), `session_id`, `source_lang` (`hi`), `target_lang` (`hi`), `user_id` (`anonymous`), `latitude` (float), `longitude` (float). Deprecated in the handler docstring in favour of `POST /api/image/upload` + chat.

### `TranscribeRequest` (JSON body)

| Field | Type | Default |
|-------|------|---------|
| `audio_content` | string (base64) | required |
| `lang_code` | string | required |
| `service_type` | `"bhashini"` or `"whisper"` | `bhashini` |
| `session_id` | string | none |
| `qid` | string | none (`asr_<session_id>` used in telemetry) |

### `TTSRequest` (JSON body)

| Field | Type | Default |
|-------|------|---------|
| `text` | string | required |
| `target_lang` | string | `hi` |
| `session_id` | string | none |
| `service_type` | `"bhashini"` or `"eleven_labs"` | `bhashini` (only `bhashini` is accepted by the handler) |
| `qid` | string | none (`tts_<session_id>` used in telemetry) |

### Token requests (`app/routers/token.py`)

| Model | Fields |
|-------|--------|
| `AuthRequest` (optional body of `POST /api/token`) | `mobile`, `name` (default `guest`), `role` (default `public`), `fingerprint_id`, `metadata` (dict, or JSON string parsed into a dict, otherwise `{"raw": ...}`) |
| `ApiKeyAuthRequest` + header `X-API-Key` | `client_code` (required), `claims` (optional dict merged into the JWT payload) |
| `PlayIntegrityAuthRequest` | `integrity_token`, `nonce`, `client_code` (all required), `mobile` (optional; defaults to `verified_device`) |

### Telemetry requests (`app/models/requests.py`)

| Model | Fields |
|-------|--------|
| `TelemetryFeedbackRequest` | `qid`, `session_id`, `feedback_type` (required); `message_id`, `feedback_text`, `question_text`, `answer_text` (optional) |
| `TelemetryErrorRequest` | `qid`, `session_id`, `error_text` (required); `question_text`, `message_id` (optional) |
| `GenericTelemetryEventRequest` (body is a **JSON array** of these) | `event_name`, `category` (non-empty after strip, validated by `validate_required_text`), `time` (client timestamp string), `metadata` (dict) |

### Image upload

`POST /api/image/upload`: multipart form with file field `image`. Accepted content types: `image/jpeg`, `image/jpg`, `image/png`.

---

## 4. Response Models and Status Codes

| Endpoint | Success response | Error responses |
|----------|------------------|-----------------|
| `GET /` | `{"app", "environment", "debug", "api_prefix"}` | - |
| `GET /api/chat/` | Streaming text (see below); headers `X-QID`, `Access-Control-Expose-Headers: X-QID` | 401; 422 for missing `query`; mid-stream failures abort the stream |
| `POST /api/chat/analyze-image` | Streaming text (no `X-QID` header) | 401; 413 if the file exceeds the size limit (raised by `save_uploaded_image`) |
| `POST /api/transcribe/` | `{"status":"success","text","lang_code","session_id"}` | 400 (`Invalid service type`), 401, 422, 500 on provider failure |
| `POST /api/tts/` | `{"status":"success","audio_data" (base64),"session_id"}` | 400 (`service_type` not `bhashini`), 401, 422, 500 on provider failure |
| `POST /api/image/upload` | `ImageUploadResponse` `{"image_id": "<uuid>"}` | 400 (unsupported format), 413 (too large), 401 |
| `GET /api/image/{image_id}` | Image bytes with stored mimetype | 404 (`Image not found or expired`), 422 (not a UUID) |
| `GET /api/file/{file_hash}` | HTML body with `Cache-Control: public, max-age=600` | 404, 500 |
| `POST /api/token` | `AuthResponse` `{"token","expires_in"}` | 500 (no private key / encode failure) |
| `POST /api/token/api-key` | `StaticAuthResponse` `{"token","expires_in"}` | 400 (missing `client_code`), 401 (`Invalid API key.`), 500 (missing per-client key config or private key) |
| `POST /api/token/play-integrity` | `AuthResponse` | 400, 401 (verdict/package/nonce/age/device checks, replayed nonce), 502 (Google API failure), 503 (replay store down), 500 (config) |
| `POST /api/telemetry/feedback` | `{"status":"accepted","langfuse_score":"queued" \| "skipped"}` | 401, 422 |
| `POST /api/telemetry/events` | `{"status":"accepted","count":<n>}` | 401, 422 |
| `POST /api/telemetry/error` | `{"status":"accepted"}` | 401, 422 |
| `GET /api/health/` | `{"app":{name,environment,uptime_seconds},"dependencies":{"cache":{status,latency_ms}}}` | 503 with the same structure under `detail` |
| `GET /api/health/live` | `{"status":"alive"}` | - |
| `GET /api/health/ready` | `{"status":"ready","cache":{...}}` | 503 `{"detail":{"status":"not ready","cache":{...}}}` |

`latency_ms` in the health payload is always `0` (a code `TODO` in `app/routers/health.py`).

---

## 5. Chat Streaming Contract

Canonical sequence: [W2 - Streaming chat turn](../architecture/sequence-diagrams.md#w2-streaming-chat-turn).

- **Transport:** HTTP GET, response is a `StreamingResponse` with `media_type="text/event-stream"` (`app/routers/chat.py`).
- **Framing (as implemented):** the generator `stream_chat_messages` yields plain text fragments (`str`). There is no `data:` prefix, no `event:` names, no blank-line delimiters and no keep-alive comments. Clients should treat the body as a progressively delivered UTF-8 text stream and concatenate chunks. `tests/test_chat_stream_contract.py` confirms the body equals the yielded text verbatim.
- **Chunk content:** only text deltas from the final answer part of the model run are forwarded (`PartDeltaEvent` / `TextPartDelta` after `FinalResultEvent`); deltas are forwarded only after `FinalResultEvent`, and the flag is reset after every tool result (`function_tool_result`), so intermediate text between tool calls is not streamed; `<think>...</think>` content is filtered from the stream.
- **Image-analysis turns:** when the query carries an image id/URL, or a previous NPSS location request is pending, the stream is buffered and the post-processed NPSS answer is sent as a single chunk (`defer_npss_output`).
- **Headers:** `X-QID: <qid>` and `Access-Control-Expose-Headers: X-QID` on `GET /api/chat/`. The deprecated `analyze-image` route generates a qid internally but does not return it.
- **History:** the conversation history is loaded before streaming (`get_session_history`) and saved after the model run completes (also after a client disconnect, via a detached finalizer with a 180 s timeout). Redis key and TTL: see [caching.md](../integrations/caching.md).
- **Timeouts:** each model attempt is limited to `AGRINET_MODEL_TIMEOUT_SECONDS` (default 45 s). If routing is enabled and no chunk has been sent, one retry on the alternate model route occurs ([W8](../architecture/sequence-diagrams.md#w8-model-routing-and-failover)).
- **Errors:** after the first byte is sent the status line is already 200; an exception ends the stream abnormally. An error telemetry event is queued server-side.
- **Disconnects:** closing the connection triggers `_finalize_after_client_disconnect`, which still saves history and sends answer telemetry.

### Image conventions

A chat `query` is treated as an image analysis request when `normalize_chat_query` (`app/routers/chat_query_utils.py`) finds an image reference: a bare UUID, a UUID inside the text, a URL containing `/api/image/<uuid>`, or a tag `[IMAGE_ID: <uuid>]`. The query is rewritten into `"<prompt>\n\n[IMAGE_URL: <BASE_URL>/api/image/<uuid>]"` using env `BASE_URL` (default `http://localhost:8000`) and the default prompt `"Analyze this crop image for pests or diseases."`. A query that merely contains `/api/image/` without a parsable id is also flagged as image analysis.

---

## 6. Rate Limiting, Versioning, Pagination

| Topic | Status |
|-------|--------|
| Rate limiting | `settings.rate_limit_requests_per_minute = 1000` is declared but never enforced in code; no limiter middleware exists. Any limiting is expected from the hosting environment |
| API versioning | None; routes are prefixed only with `/api` |
| Pagination | Not applicable; no list endpoints |
| Idempotency | `POST /api/telemetry/feedback` is idempotent per `qid` at the Langfuse score level (`score_id = user-feedback:<qid>`) |

---

## Reconciliation with `API_DOCUMENTATION.md`

| Topic | `API_DOCUMENTATION.md` says | Code today |
|-------|----------------------------|------------|
| Suggestions endpoint | `GET /api/suggest/` documented | Router exists in `app/routers/suggestions.py` but is not mounted in `main.py` ("suggestion agent disabled"); the running app does not expose it |
| Authentication | "currently disabled for testing" | JWT is always enforced on protected routes; no bypass |
| Token lifetime | 30 days (`expires_in: 2592000`) | `JWT_EXPIRY_MINUTES`, default 15 minutes |
| Token request defaults | `mobile` defaults to `"1111111111"` | `mobile` is omitted from the token unless supplied; guest tokens carry `sub=guest:<fingerprint_id>` |
| TTS `eleven_labs` | Listed as an option | Accepted by the request model but rejected by the handler with HTTP 400 |
| Chat parameters | `query`, `session_id`, `source_lang`, `target_lang`, `user_id` | Also `qid`, `latitude`, `longitude`; response header `X-QID` |
| Chat framing | "Server-Sent Events (SSE) format" | Content type is `text/event-stream` but chunks are raw text (see section 5) |
| Error body | `{"status":"error","message":...}` | Mixed: FastAPI `{"detail": ...}` for `HTTPException`, `{"status":"error","message":...}` only for the explicit 400s in transcribe/TTS |
| Endpoints not documented | - | `/api/chat/analyze-image`, `/api/image/*`, `/api/token/api-key`, `/api/token/play-integrity`, `/api/telemetry/*` |
| Rate limit | "1000 requests per minute" | Setting exists; no enforcement |
| Suggestions payload | `[{"suggestion": ...}]` | n/a (not exposed); the disabled router returns a list of strings |
| "All timestamps in UTC" | - | Token `iat`/`exp` are UTC; telemetry timestamps follow `helpers/telemetry.py` |

---

## Source Files

`main.py`, `app/routers/chat.py`, `app/routers/chat_query_utils.py`, `app/routers/transcribe.py`, `app/routers/tts.py`, `app/routers/image.py`, `app/routers/file.py`, `app/routers/token.py`, `app/routers/telemetry.py`, `app/routers/health.py`, `app/models/requests.py`, `app/auth/jwt_auth.py`, `app/services/chat.py`, `tests/test_chat_stream_contract.py`, `API_DOCUMENTATION.md`.
