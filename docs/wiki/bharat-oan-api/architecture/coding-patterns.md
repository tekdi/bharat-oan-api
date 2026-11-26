# Coding Patterns

Patterns observed in the source tree (**Implemented, verified**), followed by observations where the code departs from the Cursor rules in `.cursor/rules/backend-*.mdc` and a short recommended list.

Related: [Repository Conventions](repository-conventions.md) | [Testing](testing.md) | [Configuration](configuration.md) | [Backend](../application/backend.md) | [Repository Map](../repository-map.md) | [Technical Debt](../knowledge/technical-debt.md)

## Layering

| Layer | Location | Pattern |
|-------|----------|---------|
| HTTP / controllers | `app/routers/*.py` | One `APIRouter(prefix="/<name>", tags=["<name>"])` per file. Handlers stay thin: parse request model, call a service or helper, return `StreamingResponse` or a dict. Example: `app/routers/chat.py` delegates to `app.services.chat.stream_chat_messages`. |
| Services / orchestration | `app/services/*.py` | Business flow (moderation then agent, routing, SSE, telemetry). `app/services/chat.py` is the largest module (887 lines). |
| Agents and tools | `agents/` | pydantic-ai `Agent` objects (`agrinet_agent`, `moderation_agent`) and `Tool` wrappers in `agents/tools/__init__.py`. |
| Helpers | `helpers/*.py`, `app/utils.py`, `app/core/*.py` | Utilities, third-party clients, Redis-backed history and cache helpers. |
| Config | `app/config.py` | Single `Settings` object plus many direct `os.getenv` reads (see [Configuration](configuration.md)). |

## Async style

- Routers are `async def` (tools are a mix of `async def` and `def`, see the observation below); the chat endpoint returns `StreamingResponse(..., media_type="text/event-stream")` over an async generator (`app/routers/chat.py`).
- Post-response work uses FastAPI `BackgroundTasks` (for example `background_tasks.add_task(send_telemetry, ...)` in `app/routers/telemetry.py` and `app/services/chat.py`).
- Long agent runs use `asyncio.create_task` wrapped in `asyncio.shield`, with a detached finalizer on client disconnect (`app/services/chat.py`, `_spawn_detached`, `_finalize_after_client_disconnect`).
- Blocking work is pushed to threads where noticed: `await asyncio.to_thread(warm_scheme_search)` in `main.py` lifespan.
- Middleware is deliberately plain ASGI (`TimingAllowOriginMiddleware` in `main.py`) rather than `BaseHTTPMiddleware`, with the reason documented in its docstring.
- Redis access goes through `aiocache` (`app/core/cache.py`, `cache = Cache(Cache.REDIS, ...)`) with thin wrappers `get_cache` / `set_cache` in `app/utils.py`.

Observation: tool functions are a mix of styles. Most are plain `def` tools that use synchronous `httpx.post(...)` (`agents/tools/gfr.py`, `scheme_info.py`, `pmkisan_scheme_status.py`, `pmfby_scheme_status.py`, `pmfby_grievance.py`, `aif.py`); pydantic-ai runs sync tool functions in a worker thread, so these do not block the event loop. Two tools are `async def` but still call synchronous `httpx.post(...)` inside (`weather_forecast` in `agents/tools/weather.py`, `get_mandi_prices` in `agents/tools/mandi.py`), which blocks the worker's event loop for the duration of the call. `helpers/transcription.py`, `helpers/tts.py` and `helpers/translation.py` also use sync httpx clients. Newer tools use `async with httpx.AsyncClient(...)` (for example `agents/tools/maha_vistaar.py`, `sathi_seed.py`).

## Pydantic models

- Request models are `BaseModel` with `Field(..., description=...)` in `app/models/requests.py`; `GET /api/chat/` binds `ChatRequest` through `Depends()` so fields arrive as query parameters.
- `Literal` constrains choices (`TranscribeRequest.service_type`, `QueryModerationResult.category`); `field_validator` is used in `GenericTelemetryEventRequest`.
- Agent dependencies are a model: `FarmerContext` in `agents/deps.py` builds the user message (`get_user_message`) and carries per-turn NPSS state.
- Tool modules define their own Beckn payload models (`Descriptor`, `Location`, ...) next to the tool (for example `agents/tools/mandi.py`).
- Structured LLM output: `output_type=PromptedOutput(QueryModerationResult)` in `agents/moderation.py`.
- `app/models/responses.py` defines `BaseResponse`, `TranscribeResponse`, `ErrorResponse` and others; most routers return plain dicts or streams instead, and `/api/token` uses inline `AuthResponse`/`StaticAuthResponse`.

## Logging

- Standard pattern: `logger = get_logger(__name__)` from `helpers/utils.py` (about 41 uses). `get_logger` sets the logger level to `DEBUG` and adds no handler, so records propagate to the root logger configured in `main.py` (`logging.basicConfig(level=settings.log_level, format=settings.log_format, force=True)`).
- A few modules call `logging.getLogger(__name__)` directly (for example `app/tasks/telemetry.py`).
- Message style is mixed: f-strings dominate (`logger.info(f"Chat request received - session_id: ...")`), newer code uses lazy `%s` formatting (`agents/agrinet.py`, `app/routers/telemetry.py`).
- `logger.exception(...)` is used in background and finalizer paths (`app/services/chat.py`, `helpers/langfuse_scores.py`).
- `print(...)` is used for startup/shutdown banners in `main.py` and in `helpers/inject_pdf_header.py`.
- Observation: `app/routers/chat.py` logs the full user `query` and the authenticated mobile at INFO level.

## Error handling

| Pattern | Example |
|---------|---------|
| Routers raise `HTTPException` with explicit status codes | `app/routers/token.py`, `app/auth/jwt_auth.py` (401 with `WWW-Authenticate`), `app/routers/health.py` (503 when Redis is unhealthy) |
| Tools translate failures into model-visible retries or messages | `raise ModelRetry(...)` after catching `httpx.TimeoutException` / `httpx.RequestError` in `agents/tools/weather.py`; user-facing constants such as `UNAVAILABLE` in `agents/tools/aif.py` |
| Fail-open helpers | `helpers/master_catalog.py` returns `None` on any Redis problem, documented in the module docstring |
| Retry with backoff | `tenacity` `@retry(stop_after_attempt(3), wait_fixed(1), ...)` on `send_telemetry` in `app/tasks/telemetry.py` |
| Route failover | primary-route failure retried on the alternate agrinet route, but not after partial output was streamed (`app/services/chat.py`) |
| Import-time and startup validation | `validate_multi_provider_auth_config()` and `validate_agrinet_routing_config()` in `main.py` lifespan; `ValueError` for bad `LLM_PROVIDER` in `agents/models.py` |
| Defensive path handling | `_safe_child_path` in `app/core/image_storage.py`; `_validated_output_dir` in `scripts/audit_npss_empty_villages.py` |

There is no global exception handler and no custom exception hierarchy.

## Observability decorators

Tools are wrapped with Langfuse `@observe(name="tool:<name>", as_type="tool")` (for example `weather_forecast`), agents with `@observe(name=AGENT_VISTAAR, as_type="agent")` in `app/services/chat.py`; names come from `helpers/langfuse_trace_schema.py`. Metadata updates go through `helpers/langfuse_tracing.py`. See [Monitoring](../operations/monitoring.md).

## Naming and file layout

- Python: `snake_case` modules, functions, variables; `PascalCase` classes; `UPPER_SNAKE` constants (`HISTORY_SUFFIX`, `TOOLS`, `SUPPORTED_LANGS`). Private helpers use a leading underscore (`_get_message_history`, `_validated_photon_base_url`).
- Tool modules are named after the scheme or integration, with `_scheme_status`, `_grievance` suffixes (`pmkisan_scheme_status.py`, `pmfby_grievance.py`); Tool function names are the model-facing names (for example `initiate_pm_kisan_status_check`).
- Prompts: `assets/prompts/agrinet_<iso-639-code>.md` (Odia is `or`, noted in `agents/tools/terms.py`).
- Redis key conventions: `{session_id}__SVA` (history; `HISTORY_SUFFIX = "_SVA"` joined with `_`), `..._SVA_MODERATION`, `suggestions_{session_id}_{lang}`, `html_file:{hash}`, `AGRINET_ROUTE` suffix, `gemma_concurrency`, all under the `REDIS_KEY_PREFIX` (`sva-cache-`).
- Docstrings: Google-style `Args:` / `Returns:` in `app/utils.py` and `helpers/utils.py`; module docstrings explain intent in the newer modules.
- Type hints are present in most signatures, mixing `typing.Optional[...]` and `X | None`.

## Import-time side effects

Several modules do real work at import: `app/auth/jwt_auth.py` reads the public key file; `app/routers/token.py` loads the private key; `agents/models.py` builds model clients and may raise; `app/core/image_storage.py` creates the temp directory; `helpers/langfuse_helper.py` writes Langfuse variables into `os.environ`; many modules call `load_dotenv()`. This affects test setup (see [Testing](testing.md)).

## Comparison with `.cursor/rules/backend-*.mdc` (observations)

These rule files are generic Tekdi guidance (untracked in git). Differences below are observations, not defects of the rules.

| Rule topic | Rule says | Observed in code |
|------------|-----------|------------------|
| Layered architecture (`backend-1`) | Thin controllers, logic in services | Matches for chat flow. The `app/utils.py` module mixes cache access and message formatting, and `app/routers/token.py` holds substantial logic (595 lines). |
| API design (`backend-2`) | Versioned URLs (`/api/v1/`), consistent success/message/data envelope, 201 for create | Prefix is `/api` with no version segment; responses use mixed shapes (streams, bare dicts, `{status: ...}`); `POST /api/image/upload` returns the default 200. |
| Code quality (`backend-3`) | Small functions (< 20 lines) | Several large modules (`app/services/chat.py` 887 lines, `agents/tools/pmkisan_grievance.py` 984 lines, `helpers/telemetry.py` 918 lines). |
| Error handling (`backend-5`) | Custom exceptions, global handlers, structured error body with `error_type`, `request_id` | Uses FastAPI defaults (`{"detail": ...}`); no global handler. `X-QID` header provides per-turn correlation for chat. |
| Performance (`backend-6`) | Async I/O, connection pooling, caching with TTL | Redis cache with pooled aiocache and TTLs is in place; sync httpx calls inside two async tools are an exception (see above). |
| Security (`backend-7`) | RBAC, rate limiting, secure defaults, no sensitive logging | JWT RS256 verification exists; no rate limiting is implemented (`rate_limit_requests_per_minute` is declared but unused); CORS defaults to `*` with credentials enabled; no role checks beyond token validity. |
| Testing (`backend-8`) | >80% coverage, unit/integration folders, tests in pipeline | Flat `tests/` folder, no coverage config, no CI test job (see [Testing](testing.md)). |

## Recommended (not yet implemented)

- Replace synchronous `httpx.post` inside the async tools (`weather_forecast`, `get_mandi_prices`) with `httpx.AsyncClient` using `get_default_httpx_timeout()`, and standardise the other tools on one style.
- Consolidate environment reads behind `Settings` so one table (see [Configuration](configuration.md)) is authoritative.
- Replace `print` banners with `logger` calls; avoid logging full queries and phone numbers.
- Add a global exception handler that includes the chat `qid` for correlation.

## Source Files

`main.py`, `app/routers/chat.py`, `app/services/chat.py`, `app/utils.py`, `app/core/cache.py`, `helpers/utils.py`, `agents/deps.py`, `agents/moderation.py`, `agents/tools/weather.py`, `app/tasks/telemetry.py`, `.cursor/rules/backend-*.mdc`.
