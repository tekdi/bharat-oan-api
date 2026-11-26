# Backend

The `bharat-oan-api` backend is a single FastAPI application (`main.py`) that exposes a REST + streaming API for a multilingual agricultural voice/chat assistant. All business logic lives in Python modules under `app/` (HTTP layer, services, storage helpers), `agents/` (pydantic-ai agents and tools) and `helpers/` (provider clients, telemetry builders, prompt loading).

Related pages: [API specification](api-specification.md) | [Database / data stores](database.md) | [Caching](../integrations/caching.md) | [Async processing](../integrations/async-processing.md) | [AI/LLM integrations](../integrations/ai-llm.md) | [Component diagram](../architecture/component-diagram.md) | [Sequence diagrams](../architecture/sequence-diagrams.md) | [Configuration](../architecture/configuration.md)

---

## 1. Layered Structure (Implemented, verified)

| Layer | Location | Responsibility |
|-------|----------|----------------|
| Entry point / bootstrap | `main.py` | Creates the `FastAPI` app, installs middleware, runs startup validation (`lifespan`), mounts routers under `settings.api_prefix` (`/api`) |
| Configuration | `app/config.py` (`Settings`, `settings`) | `pydantic_settings.BaseSettings` reading env vars / `.env`; HTTP timeouts (`get_default_httpx_timeout`) |
| Authentication | `app/auth/jwt_auth.py` (`get_current_user`) | RS256 JWT verification dependency shared by every protected route |
| Routers (HTTP layer) | `app/routers/*.py` | Route declaration, request parsing, auth dependency, response shaping. Thin, except `token.py` (auth issuance logic) and `transcribe.py` / `tts.py` (inline telemetry timing) |
| Request/response models | `app/models/requests.py`, `app/models/responses.py` | Pydantic request models (used) and response models (see note below) |
| Services | `app/services/chat.py`, `chat_turn_map.py`, `npss_response.py`, `agrinet_routing.py` | Chat orchestration, qid-to-trace mapping, deterministic NPSS post-processing, model routing/capacity gate |
| Core infrastructure | `app/core/cache.py`, `app/core/image_storage.py`, `app/core/npss_followup.py`, `app/core/npss_geocodes.py` | Redis cache instance, temp image storage, NPSS follow-up detection, geocode lookup tables |
| Background tasks | `app/tasks/telemetry.py` (active), `app/tasks/suggestions.py` (disabled), `app/tasks/logging.py` (empty) | See [Async processing](../integrations/async-processing.md) |
| Shared utilities | `app/utils.py` | Cache get/set wrappers, message-history load/save, history trimming/filtering helpers |
| Agents and tools | `agents/agrinet.py`, `agents/moderation.py`, `agents/models.py`, `agents/deps.py`, `agents/tools/*` | pydantic-ai agents; the chat service calls into these |
| Provider helpers | `helpers/*.py` | Telemetry event builders, Bhashini TTS/ASR/translation, Langfuse tracing, Qdrant search, master-catalog reader, prompt loader |

> **Note:** `app/models/responses.py` defines `BaseResponse`, `TranscribeResponse`, `SuggestionsResponse`, `TTSResponse` and `ErrorResponse`, and `app/models/requests.py` defines `FileRequest`, but no router currently references them. The routes return plain dicts / `JSONResponse` objects instead, so the models document an intended shape rather than enforced behaviour.

### Layer diagram

See the canonical component views in [component-diagram.md](../architecture/component-diagram.md).

---

## 2. Application Bootstrap (`main.py`)

1. `load_dotenv()` and logging setup: log level from `settings.log_level` (default `INFO`), format `settings.log_format`, `force=True`.
2. Routers are imported with `from app.routers import chat, transcribe, tts, health, file, token, telemetry, image`. Importing `app.routers.token` loads the RSA private key (if the file exists); importing `app.auth.jwt_auth` (through the routers) opens the public key file immediately - see [Startup failure modes](#startup-failure-modes).
3. `FastAPI(title=settings.app_name, debug=settings.debug, description=..., lifespan=lifespan)`.
4. Middleware registration (see [section 3](#3-middleware)).
5. `GET /` root handler returning app name, environment, debug flag and `api_prefix`.
6. `app.include_router(<router>, prefix=settings.api_prefix)` for `chat`, `transcribe`, `tts`, `health`, `file`, `token`, `telemetry`, `image`. The `suggestions` router import and `include_router` are commented out ("suggestion agent disabled").

### Lifespan (`main.lifespan`)

Startup, in order:

| Step | Call | Effect |
|------|------|--------|
| 1 | `token.validate_multi_provider_auth_config()` (`app/routers/token.py`) | Scans `os.environ` for per-client keys with prefixes `API_KEY_AUTH_TOKEN_`, `PLAY_INTEGRITY_PACKAGE_NAME_`, `PLAY_INTEGRITY_PRIVATE_KEY_`; raises `RuntimeError` if any such variable is empty, aborting startup |
| 2 | `validate_agrinet_routing_config()` (`agents/models.py`) | No-op unless `AGRINET_ROUTING_ENABLED` is true; otherwise validates weights (non-negative, sum to 100), positive TTLs/concurrency, and eagerly builds the Gemma model (requires `AGRINET_GEMMA_MODEL_NAME`, `AGRINET_GEMMA_BASE_URL`) |
| 3 | `print(...)` banner | Prints app name, environment, debug flag and CORS origins to stdout |
| 4 | `warm_scheme_search()` via `asyncio.to_thread` | Only if `settings.qdrant_url` is set; loads the `intfloat/multilingual-e5-large` embedder and Qdrant client (`helpers/scheme_qdrant_search.py`). Failures are logged, not raised |

Shutdown: prints a shutdown banner only. No explicit closing of Redis, HTTP clients or Langfuse is implemented in the lifespan (Langfuse is flushed per chat turn via `get_client().flush()` in `app/services/chat.py`).

### Startup failure modes

- `app/auth/jwt_auth.py` executes `open(settings.base_dir / settings.jwt_public_key_path, 'rb')` at import time. If the public key PEM (default `jwt_public_key.pem`, env `JWT_PUBLIC_KEY_PATH`) is missing, the whole app fails to import.
- `app/routers/token.py` is tolerant of a missing private key: it logs a warning, and token issuance endpoints later return HTTP 500 (`JWT private key is not configured...`).
- `agents/models.py` raises `ValueError` at import time for an invalid `LLM_PROVIDER` or missing provider variables (`VLLM_AGRINET_MODEL_URL`, `AZURE_OPENAI_*`).
- `app/core/image_storage.py` creates `TEMP_UPLOAD_DIR` (default `<repo>/.oan-uploads`, env `TEMP_UPLOAD_DIR`) at import time.

---

## 3. Middleware

| Order added | Middleware | Source | Behaviour |
|-------------|-----------|--------|-----------|
| 1 | `CORSMiddleware` | `main.py` | Origins from `settings.allowed_origins` (env `ALLOWED_ORIGINS`, comma separated, default `*`); `allow_credentials=True`; all methods and headers (`["*"]`) |
| 2 | `TimingAllowOriginMiddleware` | `main.py` | Pure ASGI middleware that sets `Timing-Allow-Origin` on every `http.response.start` (`*` when origins contain `*` or there is more than one origin, otherwise the single configured origin). Deliberately **not** `BaseHTTPMiddleware` so the streaming chat body is not pumped through an extra task/stream (rationale in the class docstring) |

Because Starlette applies the last-added middleware outermost, `TimingAllowOriginMiddleware` wraps `CORSMiddleware`.

There is **no** rate-limiting, request-ID, gzip or authentication middleware. `settings.rate_limit_requests_per_minute = 1000` exists but is not read anywhere (see [Technical debt](../knowledge/technical-debt.md)). Authentication is per-route via a dependency, not global middleware.

---

## 4. Dependency Injection

FastAPI dependencies used in the codebase:

| Dependency | Defined in | Used by | Notes |
|------------|-----------|---------|-------|
| `get_current_user` | `app/auth/jwt_auth.py` | `chat`, `chat/analyze-image`, `transcribe`, `tts`, `telemetry/*`, `image/upload` | Depends on `OAuth2PasswordBearer(tokenUrl="token")`; returns a normalized `dict` of claims |
| `ChatRequest = Depends()` | `app/models/requests.py` | `GET /api/chat/` | Pydantic model used as query-parameter set |
| `SuggestionsRequest = Depends()` | `app/models/requests.py` | `app/routers/suggestions.py` (not mounted) | |
| `BackgroundTasks` | FastAPI | `chat`, `transcribe`, `tts`, `telemetry` | See [Async processing](../integrations/async-processing.md) |
| `FarmerContext` (`deps_type`) | `agents/deps.py` | pydantic-ai `agrinet_agent` | Per-turn context object passed to tools via `RunContext` - constructed in `app/services/chat.py` |

The normalized user dict returned by `get_current_user` contains: `sub`, `mobile`, `name`, `email`, `role`, `farmer_id`, `unique_id`, `locations` (default `[]`), `channel` (default `"BharatVistaar"`), `client_code`, `auth_source`, `is_guest_user` (bool), `telemetry_context` (dict), `metadata` (dict).

Module-level singletons (not injected, imported directly): `settings` (`app/config.py`), `cache` (`app/core/cache.py`), `private_key` (`app/routers/token.py`), `public_key` (`app/auth/jwt_auth.py`), `agrinet_agent`, `moderation_agent`.

---

## 5. Authentication and Authorization

Implemented in `app/auth/jwt_auth.py` (verification) and `app/routers/token.py` (issuance). Workflow: [W1 in sequence-diagrams.md](../architecture/sequence-diagrams.md#w1-authentication-and-token-issuance).

**Verification (`get_current_user`)**

- Bearer token required; verified with RS256 (`settings.jwt_algorithm`) against the PEM public key.
- `verify_aud` and `verify_iss` are disabled; `exp` is enforced by PyJWT when present.
- 401 responses: `Token has expired`, `Invalid token: ...`, `Token verification failed`, `Could not validate credentials` (all with `WWW-Authenticate: Bearer`).
- A missing `mobile` claim only logs a warning (the check that would raise is commented out).
- The `DISABLE_JWT` / `DISABLE_JWT_FOR_LOCAL` env vars populate `settings.disable_jwt`, but nothing reads that flag; the dependency docstring states there is no environment-based bypass.

**Issuance (`app/routers/token.py`)**

| Route | Credential presented | Token content |
|-------|---------------------|---------------|
| `POST /api/token` | None (open endpoint) | Guest token: `sub=guest:<fingerprint_id>` or `guest:anon`, `is_guest_user=True`, `auth_source=guest_token`, `channel=BharatVistaar`, `telemetry_context`, `metadata` (adds `surface=public_chat`, `fingerprint_id`), expiry `settings.jwt_expiry_minutes` (default 15) |
| `POST /api/token/api-key` | `X-API-Key` header + `client_code` body field; compared with `hmac.compare_digest` against `API_KEY_AUTH_TOKEN_<CLIENT_CODE>` env var | `channel=<client_code>`, optional caller-supplied `claims` merged into the payload |
| `POST /api/token/play-integrity` | Play Integrity token + nonce + `client_code` | Verified via Google Play Integrity API (service-account OAuth, `PLAY_INTEGRITY_SCOPE`), then package name, nonce (`hmac.compare_digest`), freshness (`PLAY_INTEGRITY_FRESHNESS_SECONDS`, default 120 s), `PLAY_RECOGNIZED`, and `MEETS_DEVICE_INTEGRITY` checks; nonce replay guard stored in Redis (`integrity_nonce:<nonce>`) |

> **Note:** Sign-in is required for chat, transcription, TTS, telemetry and image upload, but all authenticated users share the same level of access: no role-based restrictions are enforced by the API even though a `role` claim is carried in tokens. `POST /api/token` is open by design for guest sessions. Image download (`GET /api/image/{image_id}`), HTML file download (`GET /api/file/{file_hash}`) and health endpoints are unauthenticated.

---

## 6. Services

### 6.1 `app/services/chat.py` - chat orchestration

Core entry: `stream_chat_messages(...)`, an async generator decorated with `@observe(name="chain.chat", as_type="chain")` (Langfuse). Steps (diagram: [W2](../architecture/sequence-diagrams.md#w2-streaming-chat-turn)):

1. Queue a telemetry "question" event through `BackgroundTasks` (`create_chat_question_event` -> `send_telemetry`).
2. `resolve_agrinet_route(...)` selects the model route (`gpt41` or `gemma`) - see 6.3.
3. Open a Langfuse `propagate_attributes` context (user id, session id, tags `env:`, `channel:`, `model:`, `moderation:`), write the qid -> trace map (`_record_chat_turn` -> `write_chat_turn_map`).
4. Build `FarmerContext(query, lang_code=target_lang, session_id, question_id)`. Format the last 3 message pairs (`format_message_pairs(history, 3)`) into a `**Conversation**` prefix; detect a pending NPSS image (`find_pending_npss_image_url`).
5. Run `_run_moderation` (moderation agent) on the assembled message; store the result on `deps` and rebuild the user message.
6. `trim_history(history, max_tokens=64_000)` then `filter_thinking_from_history`.
7. Start `_run_agrinet_with_failover_streaming` as an `asyncio.Task` that pushes text chunks into an `asyncio.Queue`; the generator drains the queue and `yield`s each chunk. For image-analysis turns (`defer_npss_output`), chunks are withheld and the post-processed text is emitted once.
8. After the task completes: `post_process_npss_response(...)`, then `_finalize_chat_turn` (answer telemetry event, save history, refresh route TTL, flush Langfuse).
9. On client disconnect (`CancelledError` / `GeneratorExit`) the still-running agent task is handed to `_finalize_after_client_disconnect` via `_spawn_detached` (an `asyncio.create_task` tracked in `_detached_finalizers`) with a 180 s timeout (`_DISCONNECT_FINALIZE_TIMEOUT_SECONDS`). It saves history and sends telemetry inline because response background tasks do not run once the client is gone.
10. On any other exception: queue an error telemetry event, cancel the agent task if still running, re-raise.

Streaming helpers: `_StreamChunkSink` (remembers whether anything was emitted so failover is only attempted before the first chunk), `_strip_streaming_thinking_chunk` / `_sanitize_streamed_output` (remove `<think>...</think>` blocks), `_replace_last_text_output` (history stores the exact streamed text).

Failover: `_run_agrinet_with_failover_streaming` retries once on the alternate route (`get_alternate_agrinet_route`) only if `settings.agrinet_routing_enabled` is true and no chunk has been emitted; success after failover rewrites the stored session route (`set_session_agrinet_route`). Per-attempt timeout: `settings.agrinet_model_timeout_seconds` (default 45).

> **Note:** The non-streaming variants `_run_agrinet_with_failover` and `_run_agrinet_once` in `app/services/chat.py` are not called from anywhere in the repository (only each other), so they act as legacy code paths.

### 6.2 `app/services/chat_turn_map.py`

Redis join map keyed `chat_turn:<qid>` holding `trace_id`, `session_id`, `model_name`, `agrinet_route`, `channel`, `created_at` (TTL `DEFAULT_CACHE_TTL` = 2 h). Written by chat, read by `POST /api/telemetry/feedback` to attach user feedback as a Langfuse score (`helpers/langfuse_scores.py`). Details: [caching.md](../integrations/caching.md).

### 6.3 `app/services/agrinet_routing.py`

Per-session model routing with Redis-persisted stickiness. `resolve_agrinet_route` returns an `AgrinetRouteDecision(route, model_name, source)`. Possible `source` values: `routing_disabled`, `redis`, `state_repair`, `session_start_weighted`, `failover`, `capacity_deflect`. New sessions are assigned by weighted random roll (`AGRINET_ROUTE_GPT41_WEIGHT` / `AGRINET_ROUTE_GEMMA_WEIGHT`, default 70/30); Gemma turns pass a capacity gate that reads vLLM `/metrics` (`vllm:num_requests_running` + `vllm:num_requests_waiting`) and deflects to `gpt41` for that turn only when concurrency >= `AGRINET_GEMMA_MAX_CONCURRENCY`. See [ai-llm.md](../integrations/ai-llm.md) for model details. Flow diagram: [W8](../architecture/sequence-diagrams.md#w8-model-routing-and-failover).

### 6.4 `app/services/npss_response.py`

`post_process_npss_response(text, target_lang, npss_used, translator_factory=None)` makes NPSS (crop-image diagnosis) answers deterministic when `deps.npss_used` is true: strips model-written source lines and a trailing follow-up question, translates English lines to the target language via `BhashiniTranslator` (only lines not already in the target script, using Unicode range heuristics in `LANGUAGE_SCRIPT_RANGES`), localizes `Pest` / `Crop` / `Cause` labels (`LABELS`), and appends a localized `**Source: NPSS**` line (`SOURCE_LINES`). Supported label languages: en, hi, as, bn, gu, kn, ml, mai, mr, or, pa, ta, te. Translation failures degrade to the untranslated text.

### 6.5 Support modules

- `app/core/npss_followup.py` - `find_pending_npss_image_url(history)` scans the message history for a prior tool result containing `[NPSS_LOCATION_REQUIRED]` so the next turn can re-call the tool with the saved image URL.
- `app/routers/chat_query_utils.py` - image id detection from the query text (bare UUID, UUID in text, `/api/image/<uuid>` URL, `[IMAGE_ID: <uuid>]`), `build_internal_image_url` (uses env `BASE_URL`, default `http://localhost:8000`), `normalize_chat_query`, `get_session_history`, `prepare_image_analyze_payload`.
- `app/utils.py` - history persistence under `<session_id>__SVA` (`HISTORY_SUFFIX = "_SVA"` with an extra `_` from the f-string), `trim_history`, `group_convos`, `filter_thinking_from_history`, `format_message_pairs`, `filter_out_tool_calls`.

---

## 7. Error Handling

| Situation | Behaviour (verified) |
|-----------|---------------------|
| Auth failure | `HTTPException` 401 from `get_current_user` (messages above) |
| Missing private key / misconfigured client | `HTTPException` 500 with descriptive `detail` (`token.py`) |
| Invalid / unknown client config, bad API key | 400 (`client_code` missing) / 401 (`Invalid API key.`) / 500 (config env missing) |
| Play Integrity upstream failure | 502 (`Play Integrity verification failed.` / `Play Integrity API unavailable.`); verdict problems 401; Redis replay-store failure 503 |
| Unsupported TTS `service_type` | `JSONResponse` 400 `{"status":"error","message":...}` (only `bhashini` accepted) |
| Invalid transcribe `service_type` | `JSONResponse` 400 (`Invalid service type`); the Pydantic `Literal` already rejects unknown values with 422 first |
| Transcribe / TTS provider failure | Exception is logged and **re-raised** (`raise` in the `except`), producing FastAPI's default 500; the telemetry `finally` block builds an event but the background task is attached to a response that is never sent |
| Image problems | 400 (unsupported mimetype on `/image/upload`), 413 (over `MAX_IMAGE_UPLOAD_SIZE_MB`, default 10), 404 (`Image not found or expired`), 422 (non-UUID `image_id`) |
| HTML file | 404 (`File not found or expired`), 500 (`Internal server error`) |
| Health | 503 when the Redis round-trip fails (`/api/health/`, `/api/health/ready`) |
| Chat stream failure | The generator re-raises after queuing an error telemetry event; because the HTTP status line has already been sent once streaming begins, clients see an aborted/truncated stream rather than a structured error |
| Telemetry send | Retried by `tenacity` (3 attempts, 1 s wait, only on `httpx.TimeoutException` / `httpx.RequestError`); HTTP error statuses are logged, not retried |

There are no global `@app.exception_handler` registrations; the API relies on FastAPI defaults (`{"detail": ...}` for `HTTPException`, 422 validation payloads). Several routes use a custom `{"status": "error", "message": ...}` body instead, so the error shape is not uniform.

---

## 8. Concurrency and Process Model

- Container entrypoint: `supervisord` (`supervisord.conf`) runs `uvicorn main:app --host 0.0.0.0 --port 8000 --workers 8`. `settings.uvicorn_workers` (CPU count) is defined but not used by the supervisor command; the worker count is hard-coded to 8 in `supervisord.conf`.
- `gunicorn` and `uvicorn-worker` are listed in `requirements.txt`, but no gunicorn configuration or command exists in the repository.
- Several `async def` handlers call synchronous, blocking clients on the event loop: `transcribe_bhashini` / `transcribe_whisper` (`helpers/transcription.py`, `httpx.Client`), `text_to_speech_bhashini` (`helpers/tts.py`, `httpx.post`) and `BhashiniTranslator.translate_texts` (`helpers/translation.py`, via `post_process_npss_response`). See [technical debt](../knowledge/technical-debt.md).
- Per-process in-memory state: `_upload_registry` (`app/core/image_storage.py`), `_play_integrity_credentials` + `asyncio.Lock` (`app/routers/token.py`), `_detached_finalizers` (`app/services/chat.py`), embedder/Qdrant client caches (`helpers/scheme_qdrant_search.py`). Cross-process state is held in Redis ([caching.md](../integrations/caching.md)) or on disk ([database.md](database.md)).

---

## 9. Testing Touchpoints

Tests under `tests/` that exercise backend modules: `test_backend_owned_telemetry.py` (telemetry builders, `/api/telemetry/events`, guest token claims), `test_chat_query_utils.py`, `test_chat_stream_contract.py`, `test_npss_geocodes.py`, plus agent/LLM scenario scripts. Full details belong in [testing.md](../architecture/testing.md).

> **Note:** `tests/test_chat_stream_contract.py` references `chat_service._await_with_sse_keepalives`, `chat_service.SSE_KEEPALIVE` and `chat_router._get_message_history`, none of which exist in the current code, so that test module appears out of date with the implementation.

---

## Source Files

`main.py`, `app/config.py`, `app/auth/jwt_auth.py`, `app/routers/{chat,chat_query_utils,health,file,image,suggestions,telemetry,token,transcribe,tts}.py`, `app/services/{chat,chat_turn_map,npss_response,agrinet_routing}.py`, `app/core/{cache,image_storage,npss_followup}.py`, `app/models/{requests,responses}.py`, `app/tasks/*.py`, `app/utils.py`, `supervisord.conf`, `Dockerfile`.
