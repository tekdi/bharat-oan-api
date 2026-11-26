# Monitoring

What the service exposes for observability today: health endpoints, application logs, Langfuse LLM tracing, backend-owned telemetry events, and remnants of an earlier Logfire setup.

Related: [Troubleshooting](troubleshooting.md) | [Infrastructure](infrastructure.md) | [Configuration](../architecture/configuration.md) | [AI/LLM integration](../integrations/ai-llm.md) | [API Specification](../application/api-specification.md) | [Async Processing](../integrations/async-processing.md)

## Health endpoints (Implemented, verified: `app/routers/health.py`)

| Endpoint | Auth | Behaviour |
|----------|------|-----------|
| `GET /api/health/live` | none | Returns `{"status": "alive"}`; liveness only |
| `GET /api/health/ready` | none | Writes and reads a Redis test key (`health_check_test`, TTL 5 s) through `app.utils.cache`; returns `{"status": "ready", "cache": {...}}` or HTTP 503 with `{"status": "not ready", "cache": ...}` |
| `GET /api/health/` | none | Returns app name, environment, `uptime_seconds` (since module import) and dependency status; 503 when the cache check fails |
| `GET /` | none | App name, environment, debug flag, API prefix (`main.py`) |

Observations: only Redis is checked; LLM, Qdrant, Marqo, Beckn and Bhashini reachability are not part of health. `latency_ms` is always `0` (a `TODO` in `check_cache_connection`). Neither `Dockerfile` nor `docker-compose.yml` defines a `HEALTHCHECK`, so supervisord (`autorestart=true`) and Docker (`restart: unless-stopped`) react only to process exit, not to an unhealthy `/ready` response.

## Application logging (Implemented, verified)

| Aspect | Detail |
|--------|--------|
| Setup | `main.py`: `logging.basicConfig(level=settings.log_level, format=settings.log_format, force=True)` with format `%(asctime)s - %(name)s - %(levelname)s - %(message)s` |
| Level | `LOG_LEVEL` (default `INFO`) |
| Logger factory | `helpers.utils.get_logger(__name__)`: sets logger level `DEBUG`, no handler, propagates to root |
| Destination | stdout/stderr only. `supervisord.conf` sends both program streams and the supervisor log to `/dev/stdout` / `/dev/stderr` with rotation disabled (`*_maxbytes=0`); use `docker logs -f sva_app` |
| Log dir | `Dockerfile` creates `/app/logs`, but nothing writes there; `.dockerignore` excludes `logs/` |
| Per-request correlation | `X-QID` response header on `GET /api/chat/` and `qid` fields in telemetry/Langfuse metadata (`app/routers/chat.py`) |
| Startup banners | `print(...)` lines in the lifespan hook (environment, debug, CORS origins) |
| Sensitive data | Chat request logs include the user query and authenticated mobile at INFO (`app/routers/chat.py`); consider this when forwarding logs to shared systems |

Useful log markers: `Chat request received`, `Cache configured with Redis at ...` (startup), `System prompt rendered for session=... vector schemes live` (`agents/agrinet.py`), `Agrinet route ... failed ... retrying on ...` (failover), `feedback_langfuse_hit` / `feedback_langfuse_miss` (`app/routers/telemetry.py`), `Telemetry sent successfully` / `Telemetry send failed` (`app/tasks/telemetry.py`), `Failed to write Langfuse user-feedback score`.

## Langfuse tracing (Implemented, verified)

- Client bootstrap: `helpers/langfuse_helper.py` copies `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` (or `LANGFUSE_BASE_URL`) and `LANGFUSE_TRACING_ENVIRONMENT` into `os.environ` and creates `Langfuse()`.
- Trace shape (`helpers/langfuse_trace_schema.py`, `app/services/chat.py`): one root chain `chain.chat` (trace name `bharat-vistaar-chat`, override with `LANGFUSE_TRACE_NAME`), child agents `agent.moderation` and `agent.vistaar`, and tool spans `tool:<name>` via `@observe(..., as_type="tool")`.
- Trace attributes: `user_id`, `session_id`, tags `env:<env>`, `channel:<channel>`, `model:<name>`, `moderation:<model>` plus the moderation category, metadata `source_lang`, `target_lang`, `environment`, `channel`, truncated `query`, `qid`, `agrinet_route`, `agrinet_model_name`, `route_source`, `fallback_used` (`chat_trace_metadata_strings` and `app/services/chat.py`).
- Pydantic-ai's own instrumentation is switched off (`Agent.instrument_all(False)` in `agents/__init__.py`, `instrument=False` on agents) so Langfuse `@observe` is the single source of spans.
- Generation metadata: `helpers/langfuse_tracing.lf_update_current_observation` sets `model` and token usage so Langfuse dashboards can break down by model (`LANGFUSE_AGRINET_MODEL_NAME`, `LANGFUSE_MODERATION_MODEL_NAME` from `agents/models.py`).
- Flush: `get_client().flush()` is called when a chat turn is finalised (`app/services/chat.py`).
- Feedback scoring: `POST /api/telemetry/feedback` looks up `chat_turn:<qid>` in Redis (`app/services/chat_turn_map.py`, TTL `DEFAULT_CACHE_TTL` = 2 hours) to find the trace and writes a categorical score `user-feedback` through `helpers/langfuse_scores.py`. If the turn map expired the API still returns `accepted` with `langfuse_score: "skipped"`.

> **Note:** Langfuse is a hosted or self-hosted service configured by environment variables; its dashboards, alerts, and retention are managed outside this repository.

## Backend-owned telemetry (Implemented, verified)

- Event builders in `helpers/telemetry.py` (`create_chat_question_event`, `create_chat_answer_event`, `create_chat_error_event`, `create_chat_feedback_event`, `create_ui_interact_event`) produce Sunbird/OpenAgriNet-style events (`EventType` values `OE_START`, `OE_ITEM_RESPONSE`, `OE_INTERACT`, and others) with identity resolved from the JWT (`resolve_telemetry_identity`).
- Delivery: `app/tasks/telemetry.py::send_telemetry` POSTs to `TELEMETRY_API_URL` (default `https://dev-vistaar.da.gov.in/observability-service/action/data/v3/telemetry`) with bearer auth from `TELEMETRY_AUTH_TOKEN` or an HS256 token signed with `TELEMETRY_AUTH_KEY`/`TELEMETRY_AUTH_SECRET`, retrying three times (1 s wait) on timeouts and request errors. It runs as a FastAPI background task and also after client disconnects.
- Frontend relay endpoints: `POST /api/telemetry/feedback`, `/events`, `/error` (see [API Specification](../application/api-specification.md)).
- Tests: `tests/test_backend_owned_telemetry.py` (see [Testing](../architecture/testing.md)).

> **Note:** The default `TELEMETRY_API_URL` points to a `dev-vistaar` host. Confirm the production value is set in the deployment `.env`.

## Logfire remnants (Legacy)

Logfire is not part of the running service: `requirements.txt` has `# logfire` commented out and `Settings.logfire_token` is never read. It survives in the manual runner scripts under `tests/` (each calls `logfire.configure(scrubbing=False)`). Installing `logfire` is needed only to run those scripts. See [Legacy Artifacts](../knowledge/legacy-artifacts.md).

## Redis visibility

`docs/setup/redis.md` documents the Redis Stack UI on port `8001` and commands such as `redis-cli INFO memory`, `INFO clients`, `INFO commandstats`, `SLOWLOG GET`, plus maintenance triggers (memory above 80 percent, high CPU).

## Not part of the service

> **Note:** Integration with external monitoring tools (Prometheus metrics endpoint, Sentry, Datadog, uptime alerting) is not documented in this wiki and no such configuration exists in the repository. The service's own signals are the health endpoints, container logs, Langfuse traces, and telemetry events described above. Alerting and dashboards are managed by the operations team in their hosting environment.

Recommended (not yet implemented):

- Add a `HEALTHCHECK` to the Dockerfile that calls `/api/health/ready`.
- Extend readiness to optionally check Qdrant and the configured LLM endpoint, and measure the Redis latency to replace the `0` placeholder.
- Emit structured (JSON) logs with `qid` and `session_id`; mask mobile numbers and query text.
- Expose Prometheus metrics for request latency, tool errors, and agrinet route decisions (`AgrinetRouteDecision.source`).

## Source Files

`app/routers/health.py`, `main.py`, `helpers/utils.py`, `helpers/langfuse_helper.py`, `helpers/langfuse_tracing.py`, `helpers/langfuse_scores.py`, `helpers/langfuse_trace_schema.py`, `app/services/chat.py`, `app/services/chat_turn_map.py`, `helpers/telemetry.py`, `app/tasks/telemetry.py`, `agents/__init__.py`, `supervisord.conf`, `requirements.txt`.
