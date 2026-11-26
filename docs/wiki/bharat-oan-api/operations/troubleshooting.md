# Troubleshooting

Symptom, likely cause, and fix, each traced to the code or documentation that produces it. Start with `docker logs -f sva_app` (see [Monitoring](monitoring.md)) and the health endpoints.

Related: [Deployment](deployment.md) | [Infrastructure](infrastructure.md) | [Configuration](../architecture/configuration.md) | [Monitoring](monitoring.md) | [Disaster Recovery](disaster-recovery.md) | [Technical Debt](../knowledge/technical-debt.md)

## Quick triage

```bash
docker ps --filter name=sva_app
docker logs --tail 200 sva_app
curl -i http://localhost:8000/api/health/live
curl -i http://localhost:8000/api/health/ready      # 503 = Redis problem
docker exec -it redis-stack redis-cli ping           # container name per docs/setup/redis.md
```

## Startup and build

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Container restarts; traceback ends with `FileNotFoundError: ... jwt_public_key.pem` | `app/auth/jwt_auth.py` opens `settings.base_dir / JWT_PUBLIC_KEY_PATH` at import. The file is gitignored (`*jwt_public_key.pem`) and compose bind-mounts the host checkout over `/app` | Put the public key in the host checkout next to `main.py`, or set `JWT_PUBLIC_KEY_PATH` |
| `ValueError: Invalid LLM_PROVIDER: ... Must be one of: 'vllm', 'openai', 'azure-openai'` | `agents/models.py` validates at import | Set `LLM_PROVIDER` to a supported value |
| `ValueError: VLLM_AGRINET_MODEL_URL is required when using vllm provider` or `AZURE_OPENAI_ENDPOINT/_API_KEY/_API_VERSION/_DEPLOYMENT_NAME environment variable is required` | Provider-specific variables missing | Set the variables listed in [Configuration](../architecture/configuration.md) |
| `ValueError: AGRINET routing weights must sum to 100`, `AGRINET_GEMMA_MODEL_NAME environment variable is required`, `AGRINET_GEMMA_BASE_URL environment variable is required` at startup | `AGRINET_ROUTING_ENABLED=true` triggers `validate_agrinet_routing_config()` in the lifespan | Make `AGRINET_ROUTE_GPT41_WEIGHT + AGRINET_ROUTE_GEMMA_WEIGHT = 100` and set the Gemma model and URL, or disable routing |
| `RuntimeError: Invalid multi-provider auth configuration: API_KEY_AUTH_TOKEN_<X> is empty` | `validate_multi_provider_auth_config()` rejects empty per-client variables | Provide a value or remove the variable |
| `network svanetwork declared as external, but could not be found` | `docker-compose.yml` uses `external: true` | `docker network create svanetwork` (see `docs/setup/redis.md`) |
| `docker logs -f oan_app` reports no such container | `README.md` uses an outdated name | Use `sva_app` |
| Build is very slow or fails downloading the model | Image installs CPU `torch` and downloads `intfloat/multilingual-e5-large` during build (`Dockerfile`) | Ensure outbound access to PyPI, download.pytorch.org and HuggingFace; pass `HF_TOKEN` to avoid rate limits |
| Container uses old code after `git pull` | Compose bind-mounts `.:/app`, so the host checkout is what runs; workers keep modules loaded | Restart the container (`docker compose up --force-recreate --detach`) |
| Startup crash with `ImportError` after a refactor | Import-time wiring across `agents/`, `helpers/`, `app/` (history: commit `6a415a9` fixed `get_last_weekday_table` import errors that crashed every worker) | Read the traceback; run `python -c "import main"` in the container |

## Authentication

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `POST /api/token` returns 500 "JWT private key is not configured" | `private_key.pem` (or `JWT_PRIVATE_KEY_PATH`) missing; `app/routers/token.py` logs a warning at import | Provide the private key matching the public key used by `jwt_auth.py` |
| 401 "Could not validate credentials" / "Invalid token: ..." | Token signed with a key that does not match `jwt_public_key.pem`, malformed header, or wrong algorithm (`RS256` expected) | Request a new token from `/api/token`, `/api/token/api-key`, or `/api/token/play-integrity` |
| 401 "Token has expired" | Default lifetime is `JWT_EXPIRY_MINUTES=15` | Refresh the token on the client or raise the setting |
| `/api/token/api-key` returns 401 "Invalid API key." or 500 "Missing API key configuration" | Wrong `X-API-Key`, or `API_KEY_AUTH_TOKEN_<CLIENT_CODE>` not defined (client code is upper-cased, `-` and spaces become `_`) | Define the per-client variable |
| Play Integrity 500 "Missing Play Integrity package configuration" or "service account file not found" | `PLAY_INTEGRITY_PACKAGE_NAME_<CLIENT>` or `PLAY_INTEGRITY_SERVICE_ACCOUNT_FILE` missing | Set them; the JSON file must exist inside the container |
| Play Integrity "Replay protection is unavailable." | Nonce replay check uses Redis (`_ensure_nonce_not_reused`) | Restore Redis connectivity |
| Browser CORS failure | `ALLOWED_ORIGINS` does not include the web origin (default `*`) | Set `ALLOWED_ORIGINS`; remember the list-parsing caveat in [Configuration](../architecture/configuration.md) |

## Redis and sessions

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `/api/health/ready` returns 503 with `cache.status: unhealthy` | Redis unreachable: `REDIS_HOST` still `localhost`, container not on `svanetwork`, or Redis down | Set `REDIS_HOST` to the Redis container name; check `docker ps`, `docker logs redis-stack` |
| Conversation context disappears | History keys expire after `DEFAULT_CACHE_TTL` = 2 hours (`app/utils.py`); Redis uses `allkeys-lru` with 8 GB maxmemory per `docs/setup/redis.md`, and persistence is RDB-only (`appendonly no`) | Expected after 2 h idle or Redis restarts; see [Caching](../integrations/caching.md) |
| Feedback returns `langfuse_score: "skipped"` | `chat_turn:<qid>` expired (2 h) or the session id differs from the original turn (`app/routers/telemetry.py`) | Send feedback sooner; verify the `session_id` matches |
| Scheme list empty, "no vector-indexed schemes known" behaviour | `master-catalog:<tier>:snapshot` missing in Redis (fail-open in `helpers/master_catalog.py`); tier is `dev` only when `ENVIRONMENT` is `local`/`dev`/`development` | Run the docs-pipeline that writes the snapshot, or set `MASTER_CATALOG_REDIS_KEY` |

## Chat, models and tools

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Chat returns an error event or generic failure | Moderation or agent exception; logged with `logger.exception` in `app/services/chat.py` and sent as a telemetry error event | Check logs for the `qid`; verify LLM credentials and endpoint |
| Gemma route failing, requests retried on the other route | Failover logic (`Agrinet route ... failed ... retrying on ...`); capacity gate deflects when `AGRINET_GEMMA_MAX_CONCURRENCY` is reached (needs `AGRINET_GEMMA_METRICS_URL`) | Check the vLLM server and metrics URL; or disable `AGRINET_ROUTING_ENABLED` |
| vLLM error "Unknown role: final" | Thinking parts replayed into history | Already handled by `filter_thinking_from_history` in `app/utils.py`; ensure the current code path is used |
| Answer contains raw JSON, tool names, or `<think>` text | Model leakage | Reproduce with `tests/check_json_in_responses.py` and `tests/check_streaming_leakage.py` (see [Testing](../architecture/testing.md)); adjust prompts in `assets/prompts/` |
| Scheme/mandi/weather tools return errors; log shows `AttributeError: 'NoneType' object has no attribute 'rstrip'` | Several tools call `os.getenv("BAP_ENDPOINT").rstrip("/")` (`scheme_info.py`, `pmfby_scheme_status.py`, `pmkisan_scheme_status.py`, `shc_scheme_status.py`) | Set `BAP_ENDPOINT`, `BAP_ID`, `BAP_URI`, `BPP_ID`, `BPP_URI` |
| `search_documents` / `search_pests_diseases` fail with "Marqo endpoint URL is required" | `MARQO_ENDPOINT_URL` or `MARQO_PESTS_DISEASES_INDEX_NAME` unset | Set them |
| First `search_schemes` call slow or "Scheme search warm-up failed" in logs | `QDRANT_URL` unset (warm-up skipped) or Qdrant unreachable; the embedder is baked in the image at `/opt/hf-cache` | Set `QDRANT_URL`/`QDRANT_API_KEY`; confirm Qdrant reachability |
| Transcribe/TTS errors | `MEITY_API_KEY_VALUE` missing or Bhashini down; Whisper needs `OPENAI_API_KEY` | Set the keys; test the upstream directly |
| Mandi/weather call hangs a worker | Two async tools call synchronous `httpx.post` (see [Coding Patterns](../architecture/coding-patterns.md)); timeouts come from `DEFAULT_API_TIMEOUT` / `DEFAULT_API_READ_TIMEOUT` | Tune timeouts; see [Technical Debt](../knowledge/technical-debt.md) |
| `GET /api/suggest/` returns 404 | Suggestions router is commented out in `main.py` and `app/routers/__init__.py` | Not an outage; feature disabled |

## Images and NPSS

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| NPSS image analysis error "Relative image URL provided but BASE_URL env var is not set" | `agents/tools/npss.py` needs `BASE_URL` to resolve `/api/image/<id>` | Set `BASE_URL` to the externally reachable API URL |
| Upload returns 400 or 413 | Unsupported format, or larger than `MAX_IMAGE_UPLOAD_SIZE_MB` (10) | Use JPEG/PNG within the limit |
| `GET /api/image/<id>` returns 404 | Image older than `IMAGE_TTL_MINUTES` (60) or already cleaned up | Re-upload |
| Location always falls back to KVK or no geocoding | `PHOTON_HOST` empty or not `https://` (`_validated_photon_base_url` returns `None`) | Provide an HTTPS Photon URL |
| NPSS 401 or empty results | `NPSS_USERNAME` / `NPSS_PASSWORD` missing or wrong | Set credentials (see `.env.example`); `scripts/audit_npss_empty_villages.py` can inspect master data |

## Telemetry

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Log "Telemetry send failed - status: 401/403" | Missing or wrong `TELEMETRY_AUTH_TOKEN` / key and secret | Configure auth; warning "Telemetry auth is not configured" appears when none is set |
| Telemetry sent to a dev host | Default `TELEMETRY_API_URL` points at `dev-vistaar.da.gov.in` | Set the production URL |

## Escalation

> **Note:** On-call rotations, severity levels, and support contacts are not documented in this wiki. Use your organisation's escalation process; Langfuse traces (search by `qid` or session) are the fastest way to inspect an LLM turn.

## Source Files

`app/auth/jwt_auth.py`, `app/routers/token.py`, `app/routers/health.py`, `app/routers/telemetry.py`, `app/utils.py`, `agents/models.py`, `main.py`, `agents/tools/npss.py`, `app/core/npss_geocodes.py`, `app/core/image_storage.py`, `helpers/master_catalog.py`, `helpers/scheme_qdrant_search.py`, `docker-compose.yml`, `README.md`, `docs/setup/redis.md`.
