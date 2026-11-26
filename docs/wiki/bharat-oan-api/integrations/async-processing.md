# Async Processing

The service has no message broker, job queue or scheduler. Asynchronous work is done with Python `asyncio` inside each uvicorn worker process, with FastAPI `BackgroundTasks` for post-response telemetry. This page documents every asynchronous mechanism that exists in the code.

Related pages: [Backend](../application/backend.md) | [Caching](caching.md) | [Third-party integrations](third-party-integrations.md) | [AI/LLM](ai-llm.md) | [Sequence diagrams](../architecture/sequence-diagrams.md) | [Infrastructure](../operations/infrastructure.md) | [Monitoring](../operations/monitoring.md) | [Technical debt](../knowledge/technical-debt.md)

> **Note:** No Celery, RQ, Dramatiq, Arq or cron-style scheduler is used by this service. The `Dockerfile` comment "manage both FastAPI and Celery" and a commented-out `send_telemetry.s(...).apply_async()` fragment in `helpers/telemetry.py` are leftovers from an earlier design; `supervisord.conf` defines only a `fastapi` program and `requirements.txt` has no Celery entry.

---

## 1. Process and Worker Model

| Item | Value | Source |
|------|-------|--------|
| Process supervisor | `supervisord` runs in the foreground (`nodaemon=true`) | `supervisord.conf`, `Dockerfile` (`CMD`) |
| App server | `uvicorn main:app --host 0.0.0.0 --port 8000 --workers 8` (single `[program:fastapi]`, `autorestart=true`, `startretries=3`, `stopsignal=TERM`) | `supervisord.conf` |
| Worker count | Fixed at 8 in `supervisord.conf`; `settings.uvicorn_workers` (`os.cpu_count()`) is defined but unused | `app/config.py` |
| Gunicorn | `gunicorn` and `uvicorn-worker` appear in `requirements.txt`, but no gunicorn command or config file exists | `requirements.txt` |
| Event loop | One asyncio loop per uvicorn worker; all handlers are `async def` | `app/routers/*.py` |
| State shared across workers | Redis and the local filesystem only. In-process dicts/sets (`_upload_registry`, `_detached_finalizers`, `_play_integrity_credentials`, embedder caches) are per worker | see [database.md](../application/database.md) |

Because each worker is a separate process, startup work in `main.lifespan` (including the scheme-embedder warm-up in a thread via `asyncio.to_thread(warm_scheme_search)`) runs once per worker.

---

## 2. FastAPI BackgroundTasks (post-response work)

`BackgroundTasks` run after the HTTP response has finished. All usages enqueue `send_telemetry` (and one Langfuse score call).

| Where | Task enqueued | Trigger |
|-------|---------------|---------|
| `app/services/chat.py` (`stream_chat_messages`) | `send_telemetry` with a chat **question** event | Start of every chat turn |
| `app/services/chat.py` (`_queue_telemetry_event` from `_finalize_chat_turn`) | `send_telemetry` with a chat **answer** event | End of a normal (connected) turn |
| `app/services/chat.py` (`except Exception` in `stream_chat_messages`) | `send_telemetry` with a chat **error** event | Unhandled error during a turn |
| `app/routers/transcribe.py` | `send_telemetry` with an ASR event (success flag, latency, language) | After a successful transcription |
| `app/routers/tts.py` | `send_telemetry` with a TTS event | After a successful synthesis |
| `app/routers/telemetry.py` `/feedback` | `send_telemetry` (feedback item) and, when the qid maps to a trace, `helpers.langfuse_scores.create_user_feedback_score` | Feedback submission |
| `app/routers/telemetry.py` `/events`, `/error` | `send_telemetry` | Client-reported UI events / errors |

How `BackgroundTasks` is wired for streaming chat (`app/routers/chat.py`): the route declares `background_tasks: BackgroundTasks`, passes it into `stream_chat_messages(...)` (which appends tasks while streaming) and also passes it as `StreamingResponse(..., background=background_tasks)`. The queued tasks therefore execute once the stream ends normally.

Caveats (verified from code and Starlette semantics):

- If the client disconnects, response background tasks are not run. `app/services/chat.py` compensates for the *answer* event by sending it inline from the detached finalizer (`_send_telemetry_event`). The *question* event queued at the start of the turn is, however, still only a background task.
- In `transcribe` and `tts`, failures re-raise inside `try/except/finally`, so the exception response replaces the normal response and the telemetry event built in `finally` is not delivered for failed calls (only successful calls produce telemetry).
- Background tasks share the worker's event loop; they are not retried after a process crash and are lost on restart.

---

## 3. The Telemetry Task (`app/tasks/telemetry.py`)

`send_telemetry(telemetry_data: dict) -> dict` is the only active task module.

| Aspect | Behaviour |
|--------|-----------|
| Destination | `settings.telemetry_api_url` (env `TELEMETRY_API_URL`, default `https://dev-vistaar.da.gov.in/observability-service/action/data/v3/telemetry`) |
| Payload | `TelemetryRequest(events=[...]).model_dump()` built by `helpers/telemetry.py` (`create_chat_*_event`, `create_tts_event`, `create_asr_event`, `create_ui_interact_event`, `create_frontend_compatible_item_batch`) |
| HTTP client | New `httpx.AsyncClient` per call; timeout `httpx.Timeout(30.0, read=60.0)` |
| Auth | `TELEMETRY_AUTH_TOKEN` if set, otherwise an HS256 JWT built from `TELEMETRY_AUTH_KEY` + `TELEMETRY_AUTH_SECRET` (`iat`/`exp` are `None`); no header and a warning if neither is configured. Optional `TELEMETRY_ORIGIN` and `TELEMETRY_REFERER` headers |
| Retry | `tenacity`: up to 3 attempts, 1 s fixed wait, only for `httpx.TimeoutException` / `httpx.RequestError`, `reraise=True` |
| Non-2xx response | Logged at WARNING with the first 500 characters of the body; not retried; the function returns `{"status_code", "response"}` |
| Return value | Ignored by callers |

Flow: [W6 - Telemetry and feedback](../architecture/sequence-diagrams.md#w6-telemetry-and-feedback).

Operational note: because failures after retries propagate as exceptions out of a background task, they surface only in application logs (Starlette logs the traceback); there is no dead-letter store.

---

## 4. Task Modules Present but Not Active

| Module | State |
|--------|-------|
| `app/tasks/suggestions.py` (`create_suggestions(session_id, target_lang)`) | Disabled. Builds a prompt from the last 3 message pairs, runs `agents.suggestions.suggestions_agent`, and would cache the list under `suggestions_<session_id>_<lang>` for `SUGGESTIONS_CACHE_TTL = 60*60` (the adjacent comment says 30 minutes). Its import in `app/tasks/__init__.py`, the router include in `main.py`, and any scheduling call in the chat service are commented out or removed. The module also appears stale: `agents/suggestions.py` imports `LLM_AGRINET_MODEL` which is not defined in `agents/models.py`, and `create_suggestions` calls `trim_history(..., include_tool_calls=..., include_system_prompts=...)` with keyword arguments the current `app/utils.py:trim_history` signature does not accept |
| `app/routers/suggestions.py` | Reads `suggestions_<session_id>_<target_lang>` from Redis; not mounted |
| `app/tasks/logging.py` | Only imports; contains no task |

The suggestions workflow is documented for reference in [W7](../architecture/sequence-diagrams.md#w7-conversation-suggestions-disabled).

---

## 5. Long-Lived asyncio Tasks Inside a Chat Request

`app/services/chat.py` uses `asyncio.create_task` for two purposes:

| Task | Created by | Purpose | Lifetime / cancellation |
|------|-----------|---------|-------------------------|
| `agrinet_task` (`_run_agrinet_with_failover_streaming`) | `stream_chat_messages` | Runs the agent in the background while the generator drains an `asyncio.Queue` of text chunks and yields them to the client | Awaited with `asyncio.shield(...)` so cancelling the request does not cancel the agent run. If the generator hits an exception the task is cancelled explicitly. Per-attempt timeout `AGRINET_MODEL_TIMEOUT_SECONDS` (45 s) via `asyncio.timeout` |
| Detached finalizer (`_spawn_detached(_finalize_after_client_disconnect(...))`) | `stream_chat_messages` on `CancelledError` / `GeneratorExit` | After the client hangs up, waits for the still-running agent task (up to `_DISCONNECT_FINALIZE_TIMEOUT_SECONDS = 180`), then saves history, writes the route TTL, sends answer telemetry inline, and flushes Langfuse | Strong reference kept in the module-level set `_detached_finalizers` (removed by a done callback) so the task cannot be garbage collected; timeouts and errors are logged and the turn is dropped |

Why this exists (from code comments): response-level background tasks never run once the client is gone, so work that must survive a disconnect (saving the turn to Redis) is moved out of the request's cancel scope. Sequence: [W2](../architecture/sequence-diagrams.md#w2-streaming-chat-turn).

Failover is also asynchronous but sequential: if the first model attempt fails before any chunk is emitted (`_StreamChunkSink.emitted` is false) and routing is enabled, a second attempt starts on the alternate route ([W8](../architecture/sequence-diagrams.md#w8-model-routing-and-failover)).

---

## 6. Thread Offloading and Blocking Calls

| Call | Mechanism | Source |
|------|-----------|--------|
| Scheme embedder warm-up at startup | `await asyncio.to_thread(warm_scheme_search)` | `main.py` |
| Writing uploaded image bytes | `await asyncio.to_thread(file_path.write_bytes, ...)` | `app/core/image_storage.py` |
| Google service-account token refresh | `await asyncio.to_thread(credentials.refresh, GoogleAuthRequest())` | `app/routers/token.py` |
| Qdrant video search | `await asyncio.to_thread(qdrant_search_videos, ...)` | `agents/tools/search.py` (`search_video`, around line 553) |
| Bhashini ASR / Whisper / TTS | **Not** offloaded: synchronous `httpx` calls made directly inside `async def` handlers | `app/routers/transcribe.py` -> `helpers/transcription.py`; `app/routers/tts.py` -> `helpers/tts.py` |
| NPSS Bhashini translation | **Not** offloaded: synchronous `BhashiniTranslator.translate_texts` called from `post_process_npss_response` inside the streaming generator | `app/services/npss_response.py`, `helpers/translation.py` |
| Master catalog Redis read | Synchronous `redis.Redis` call on the event loop | `helpers/master_catalog.py` |
| Image and metadata file I/O | Synchronous `Path.read_text` / `write_text` / `unlink` in request handlers | `app/core/image_storage.py` |

The synchronous calls block the worker's event loop for the duration of the external call (timeouts up to 30-60 s for TTS), which reduces concurrency for all requests on that worker. This is recorded as technical debt in [technical-debt.md](../knowledge/technical-debt.md).

---

## 7. Scheduled / Periodic Work

> **Note:** There are no scheduled or periodic jobs in this service. Cache expiry relies on Redis TTLs. Cleanup of uploaded images after processing happens inline inside the NPSS tool (`cleanup_image`), and the periodic helper `cleanup_expired_images()` in `app/core/image_storage.py` is not invoked by any scheduler or startup hook in the repository, so retention of unprocessed uploads is managed by the hosting environment.

---

## 8. Recommended (not yet implemented)

- Offload the blocking Bhashini/Whisper/TTS/translation calls with `asyncio.to_thread` or switch to `httpx.AsyncClient`.
- Add a lifespan-managed periodic task (or an external cron) calling `cleanup_expired_images()` and expiring unprocessed uploads.
- For durable telemetry (survive restarts and disconnects), publish events to a queue instead of relying on in-process background tasks.
- Align `supervisord.conf` worker count with `settings.uvicorn_workers` or remove the unused setting.

---

## Source Files

`main.py`, `supervisord.conf`, `Dockerfile`, `requirements.txt`, `app/tasks/{__init__,telemetry,suggestions,logging}.py`, `app/services/chat.py`, `app/routers/{chat,transcribe,tts,telemetry}.py`, `app/core/image_storage.py`, `app/services/npss_response.py`, `helpers/telemetry.py`, `helpers/transcription.py`, `helpers/tts.py`, `helpers/langfuse_scores.py`.
