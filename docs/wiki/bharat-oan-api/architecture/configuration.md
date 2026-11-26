# Configuration

All configuration is supplied through environment variables (optionally via a `.env` file) and a few files on disk (JWT keys, prompts, glossary). This page inventories every variable found in the source.

Related: [Repository Map](../repository-map.md) | [Deployment](../operations/deployment.md) | [Infrastructure](../operations/infrastructure.md) | [AI/LLM](../integrations/ai-llm.md) | [Third-party integrations](../integrations/third-party-integrations.md) | [Caching](../integrations/caching.md)

## How configuration is loaded (Implemented, verified)

| Mechanism | Where | Notes |
|-----------|-------|-------|
| `Settings(BaseSettings)` | `app/config.py` | `pydantic-settings`; `class Config: env_file = ".env"`, `extra = 'ignore'`. Instance exported as `settings`. |
| `load_dotenv()` | `app/config.py`, `main.py`, `agents/*.py`, `helpers/utils.py`, tool modules | Loads `.env` into `os.environ` at import time. |
| Direct `os.getenv(...)` | `agents/models.py`, `agents/tools/*.py`, `app/routers/token.py`, `app/core/image_storage.py`, `app/tasks/telemetry.py`, `helpers/*.py` | Many variables are read outside `Settings`, so the table below lists the real consumer. |
| `docker-compose.yml` `env_file: .env` | `docker-compose.yml` | The container receives the host `.env`. `HF_TOKEN` is also a build arg. |

Two styles coexist: fields declared with `os.getenv(...)` as the default (value read once at import) and plain annotated fields (`openai_api_key: Optional[str] = None`), which pydantic-settings fills from an env var named after the field (case-insensitive). Several of the plain fields are declared but never read by code (see "Declared but unused" below).

> **Note:** Only the NPSS/Photon variables are documented in `.env.example`. Every other variable in this page is discovered from source. A complete production `.env` is managed outside the repository by the deploying team.

> **Note (unverified here):** `allowed_origins` is a `List[str]` field whose default is built from `ALLOWED_ORIGINS.split(",")`. pydantic-settings normally parses list-typed environment values as JSON, so confirm in your environment how a comma-separated `ALLOWED_ORIGINS` is accepted before relying on it. This was not executed during wiki generation.

## Core application and server

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `ENVIRONMENT` | `production` | No | Environment label; also drives Langfuse env tag and master-catalog tier (`local`/`dev`/`development` map to `dev`, everything else to `live`) | `app/config.py`, `helpers/master_catalog.py`, `helpers/langfuse_helper.py` |
| `SECRET_KEY` | `your-secret-key-change-in-production` (placeholder) | No (declared, unused) | Declared in `Settings.secret_key`; no code reads it. **Secret** | `app/config.py` |
| `ALLOWED_ORIGINS` | `*` | No | CORS origins, comma-split | `app/config.py`, `main.py` (`CORSMiddleware`, `TimingAllowOriginMiddleware`) |
| `LOG_LEVEL` (field `log_level`) | `INFO` | No | Root log level in `main.py` | `main.py` |
| `DEBUG` (field `debug`) | `False` | No | FastAPI `debug` flag and root endpoint output | `main.py` |
| `DEFAULT_API_TIMEOUT` | `5.0` | No | httpx connect timeout (seconds) for outbound calls | `app/config.py` (`get_default_httpx_timeout`) |
| `DEFAULT_API_READ_TIMEOUT` | `10.0` | No | httpx read timeout (seconds) | `app/config.py` |
| `BASE_URL` | `""` (image storage) / `http://localhost:8000` (chat query utils) | Yes for image analysis in deployed environments | Public base URL used to build `/api/image/<id>` URLs that the NPSS tool downloads | `app/core/image_storage.py`, `app/routers/chat_query_utils.py`, `agents/tools/npss.py` |

Hard-coded in `Settings` (no env override unless a same-named env var is set): `app_name="Bharatvistaar AI API"`, `host="0.0.0.0"`, `port=8000`, `api_prefix="/api"`, `timezone="Asia/Kolkata"`, `default_cache_ttl=86400`, `suggestions_cache_ttl=1800`, `allowed_credentials=True`, `allowed_methods=["*"]`, `allowed_headers=["*"]`.

## Authentication

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `JWT_PUBLIC_KEY_PATH` | `jwt_public_key.pem` | **Yes** (import-time file read) | Path, relative to repo root, of the RS256 public key. The app fails at import if the file is missing | `app/config.py`, `app/auth/jwt_auth.py` |
| `JWT_PRIVATE_KEY_PATH` | unset (falls back to `private_key.pem` in repo root) | Yes for token issuance | RS256 private key for signing tokens. **Secret (file)** | `app/config.py`, `app/routers/token.py` |
| `JWT_EXPIRY_MINUTES` | `15` | No | Token lifetime | `app/config.py`, `app/routers/token.py` |
| `DISABLE_JWT`, `DISABLE_JWT_FOR_LOCAL` | off | No | Parsed into `settings.disable_jwt`; nothing reads it (`get_current_user` states there is no env bypass) | `app/config.py` |
| `API_KEY_AUTH_TOKEN` | unset | No | Single static API key setting | `app/config.py` |
| `API_KEY_AUTH_TOKEN_PREFIX` | `API_KEY_AUTH_TOKEN_` | No | Prefix for per-client keys | `app/config.py`, `app/routers/token.py` |
| `API_KEY_AUTH_TOKEN_<CLIENT>` | none | Yes per API-key client | Per-client static API key (client code upper-cased, `-`/space to `_`). **Secret.** Empty values abort startup | `app/routers/token.py` (`_resolve_api_key`, `validate_multi_provider_auth_config`) |
| `PLAY_INTEGRITY_PACKAGE_NAME` | unset | Only for default (no client code) flow | Android package name | `app/config.py` |
| `PLAY_INTEGRITY_PACKAGE_NAME_PREFIX` | `PLAY_INTEGRITY_PACKAGE_NAME_` | No | Prefix for per-client package names | `app/config.py` |
| `PLAY_INTEGRITY_PACKAGE_NAME_<CLIENT>` | none | Yes per Play Integrity client | Per-client package name | `app/routers/token.py` |
| `PLAY_INTEGRITY_PRIVATE_KEY_PREFIX` | `PLAY_INTEGRITY_PRIVATE_KEY_` | No | Prefix for per-client service-account private key overrides | `app/config.py` |
| `PLAY_INTEGRITY_PRIVATE_KEY_<CLIENT>` | none | Optional | Overrides `private_key` of the service-account JSON (`\n` sequences expanded). **Secret** | `app/routers/token.py` |
| `PLAY_INTEGRITY_SERVICE_ACCOUNT_FILE` | unset | Yes for Play Integrity | Service account JSON path. **Secret (file)** | `app/config.py`, `app/routers/token.py` |
| `PLAY_INTEGRITY_SERVICE_ACCOUNT_BASE_PATH` | unset | No | Base path for per-client service account files | `app/config.py` |
| `PLAY_INTEGRITY_FRESHNESS_SECONDS` | `120` | No | Max token age | `app/config.py`, `app/routers/token.py` |
| `PLAY_INTEGRITY_DEFAULT_NAME` / `_ROLE` / `_METADATA` | `android_user` / `public` / `""` | No | Claims placed in tokens issued after Play Integrity verification | `app/config.py`, `app/routers/token.py` |

## Redis and caching

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `REDIS_HOST` | `localhost` | **Yes** in containers | Redis host (use the Redis container name on `svanetwork`) | `app/core/cache.py`, `helpers/master_catalog.py` |
| `REDIS_PORT` | `6379` | No | Port | same |
| `REDIS_DB` | `0` | No | DB index | same |
| `REDIS_KEY_PREFIX` | `sva-cache-` | No | aiocache key prefix | `app/core/cache.py` |
| `REDIS_SOCKET_CONNECT_TIMEOUT` | `10` | No | Connect timeout of the raw master-catalog client | `helpers/master_catalog.py` |
| `REDIS_SOCKET_TIMEOUT` | `10` | No | Socket timeout; also aiocache `timeout` | `app/core/cache.py`, `helpers/master_catalog.py` |
| `REDIS_MAX_CONNECTIONS` | `100` | No | aiocache `pool_max_size` | `app/core/cache.py` |
| `REDIS_RETRY_ON_TIMEOUT` | `true` | No | Declared only; not read | `app/config.py` |
| `MASTER_CATALOG_REDIS_KEY` | `""` (then `master-catalog:<tier>:snapshot`) | No | Exact Redis key for the master catalog snapshot written by the docs-pipeline | `app/config.py`, `helpers/master_catalog.py` |

## LLM providers and routing

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `LLM_PROVIDER` | `openai` | No | One of `vllm`, `openai`, `azure-openai`; anything else raises at import | `agents/models.py` |
| `LLM_AGRINET_MODEL_NAME` | `gpt-3.5-turbo` (openai), `agrinet-model` (vllm), Azure deployment name (azure) | Recommended | Agrinet agent model | `agents/models.py` |
| `LLM_MODERATION_MODEL_NAME` | same pattern | Recommended | Moderation model | `agents/models.py` |
| `LLM_MODERATION_TIMEOUT_SECONDS` | `20` | No | Moderation agent timeout | `agents/moderation.py` |
| `OPENAI_API_KEY` | unset | Yes for `openai`; also Whisper transcription | **Secret** | `agents/models.py`, `helpers/transcription.py` |
| `VLLM_AGRINET_MODEL_URL` | unset | Yes for `vllm` | OpenAI-compatible base URL | `agents/models.py` |
| `VLLM_MODERATION_MODEL_URL` | `VLLM_AGRINET_MODEL_URL` | No | Moderation endpoint | `agents/models.py` |
| `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_API_VERSION`, `AZURE_OPENAI_DEPLOYMENT_NAME` | unset | Yes for `azure-openai` (all four) | Azure client. API key is a **secret** | `agents/models.py` |
| `AGRINET_ROUTING_ENABLED` | `false` | No | Enables weighted GPT-4.1 / Gemma routing | `app/config.py`, `agents/models.py` |
| `AGRINET_ROUTE_GPT41_WEIGHT` / `AGRINET_ROUTE_GEMMA_WEIGHT` | `70` / `30` | No | Must sum to 100 when routing is enabled (validated at startup) | `app/config.py`, `agents/models.py` |
| `AGRINET_ROUTE_TTL_SECONDS` | `7200` | No | Session route stickiness; must be positive | `app/config.py`, `app/services/agrinet_routing.py` |
| `AGRINET_MODEL_TIMEOUT_SECONDS` | `45` | No | Per-route model timeout | `app/config.py` |
| `AGRINET_GEMMA_PROVIDER` | `vllm` | No | Only `vllm` supported | `agents/models.py` |
| `AGRINET_GEMMA_MODEL_NAME`, `AGRINET_GEMMA_BASE_URL` | unset | Yes when routing enabled | Gemma vLLM model and endpoint | `agents/models.py` |
| `AGRINET_GEMMA_API_KEY` | `not-needed` | No | **Secret** if the endpoint is protected | `agents/models.py` |
| `AGRINET_GEMMA_MAX_CONCURRENCY` | `10` | No | Capacity gate threshold | `app/services/agrinet_routing.py` |
| `AGRINET_GEMMA_METRICS_URL` | unset | No | vLLM Prometheus metrics URL for the gate | `app/services/agrinet_routing.py` |
| `AGRINET_GEMMA_METRICS_CACHE_TTL` | `2` | No | Seconds to cache concurrency reading | `app/services/agrinet_routing.py` |

## Search, schemes and Beckn network

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `QDRANT_URL` | unset (`http://localhost:6333` fallback inside the client builder) | Enables scheme/video vector search; startup pre-warm only when set | Qdrant endpoint | `app/config.py`, `main.py`, `helpers/scheme_qdrant_search.py`, `agents/tools/search.py` |
| `QDRANT_API_KEY` | unset | If Qdrant requires it. **Secret** | Qdrant key | `helpers/scheme_qdrant_search.py` |
| `QDRANT_COLLECTION_NAME` | `schemes-index` | No | Declared in `Settings` only; no `settings.qdrant_collection_name` reader was found | `app/config.py` |
| `QDRANT_VIDEO_COLLECTION_NAME` | `video_data_collection` | No | Video search collection | `agents/tools/search.py` |
| `MARQO_ENDPOINT_URL` | unset | Yes for `search_documents`/`search_pests_diseases` | Marqo endpoint | `agents/tools/search.py` |
| `MARQO_INDEX_NAME` | `sunbird-va-index` | No | Document index | `agents/tools/search.py` |
| `MARQO_PESTS_DISEASES_INDEX_NAME` | unset | Yes for `search_pests_diseases` | Pest/disease index | `agents/tools/search.py` |
| `BAP_ENDPOINT` | unset | **Yes** for all Beckn scheme/mandi/weather/GFR/SATHI tools | Beckn BAP base URL | most files in `agents/tools/` |
| `BAP_ID`, `BAP_URI`, `BPP_ID`, `BPP_URI` | unset | Yes for Beckn contexts | Beckn context identifiers | `agents/tools/*.py` |
| `SCHEME_DOMAIN` | `schemes:vistaar` | No | Beckn domain for MahaVistaar / scheme search | `agents/tools/maha_vistaar.py`, `agents/tools/scheme_info.py` |
| `AMUL_SCHEME_DOMAIN` | `schemes:amul-union` | No | Beckn domain for AmulVistaar | `agents/tools/amul_vistaar.py` |
| `AMUL_SEARCH_TTL` | `PT30S` | No | Beckn TTL string | `agents/tools/amul_vistaar.py` |
| `SATHI_MASTER_BASE_URL`, `SATHI_MASTER_API_KEY` | unset | For SATHI master lookups. Key is a **secret** | SATHI seed tool | `agents/tools/sathi_seed.py` |
| `API_BASE_URL` | unset | For SHC HTML file links | Base for cached HTML file URLs | `agents/tools/shc_scheme_status.py` |
| `GRIEVANCE_TYPES_PATH` | `assets/grievance_types.json` | No | PM-KISAN grievance types file | `agents/tools/pmkisan_grievance.py` |
| `MH_BPP_ID` | n/a | n/a | Referenced only in commented-out code | `agents/tools/maha_vistaar.py` |

## NPSS, geocoding and image handling

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `NPSS_BASE_URL` | `https://npss.dac.gov.in/api3.0` | No | NPSS API base | `.env.example`, `app/config.py`, `agents/tools/npss.py`, `app/core/npss_geocodes.py`, `scripts/audit_npss_empty_villages.py` |
| `NPSS_USERNAME`, `NPSS_PASSWORD` | `""` | Yes for image analysis. **Secrets** | NPSS credentials | `.env.example`, `agents/tools/npss.py`, `scripts/audit_npss_empty_villages.py` |
| `PHOTON_HOST` | unset | No (geocoding disabled if empty); **HTTPS required** (validated in `_validated_photon_base_url`) | Photon geocoder | `.env.example`, `agents/tools/maps.py`, `app/core/npss_geocodes.py` |
| `TEMP_UPLOAD_DIR` | `<repo>/.oan-uploads` | No | Temp image directory | `app/core/image_storage.py` |
| `GCS_MOUNT_PATH` | `""` | No | Mounted bucket to move processed images into | `app/core/image_storage.py` |
| `GCS_MOVE_AFTER_PROCESS` | off | No | Move image to `GCS_MOUNT_PATH` after NPSS | `agents/tools/npss.py` |
| `IMAGE_TTL_MINUTES` | `60` | No | Uploaded image retention | `app/core/image_storage.py` |
| `MAX_IMAGE_UPLOAD_SIZE_MB` | `10` | No | Upload limit | `app/core/image_storage.py` |

## Speech, translation, telemetry and observability

| Variable | Default | Required | Purpose | Used in |
|----------|---------|----------|---------|---------|
| `MEITY_API_KEY_VALUE` | unset | Yes for Bhashini transcribe/TTS/translation. **Secret** | Bhashini (Dhruva) `Authorization` header | `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py` |
| `TELEMETRY_API_URL` | `https://dev-vistaar.da.gov.in/observability-service/action/data/v3/telemetry` | Set for production | Telemetry sink | `app/config.py`, `app/tasks/telemetry.py` |
| `TELEMETRY_AUTH_TOKEN` | unset | No. **Secret** | Bearer token | `app/tasks/telemetry.py` |
| `TELEMETRY_AUTH_KEY`, `TELEMETRY_AUTH_SECRET` | unset | No. Secret is a **secret** | Used to sign an HS256 JWT when no token is given | `app/tasks/telemetry.py` |
| `TELEMETRY_ORIGIN`, `TELEMETRY_REFERER` | unset | No | Extra headers | `app/tasks/telemetry.py` |
| `TELEMETRY_CHANNEL` | unset (falls back to token claims, then `BharatVistaar`) | No | Channel override | `helpers/telemetry.py` |
| `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` | unset | Needed for traces. Secret key is a **secret** | Langfuse credentials | `app/config.py`, `helpers/langfuse_helper.py` |
| `LANGFUSE_HOST` (alt: `LANGFUSE_BASE_URL`) | unset | Needed for traces | Langfuse endpoint | `app/config.py`, `helpers/langfuse_helper.py` |
| `LANGFUSE_TRACING_ENVIRONMENT` | value of `ENVIRONMENT` | No | Langfuse environment badge | `app/config.py`, `helpers/langfuse_helper.py` |
| `LANGFUSE_TRACE_NAME` (alt: `LANGFUSE_TRACE_ROOT_NAME`) | `bharat-vistaar-chat` | No | Root trace name | `app/services/chat.py` |
| `HF_TOKEN` | empty (build arg) | No | HuggingFace token during image build; persisted as `ENV HF_TOKEN` in the image | `Dockerfile`, `docker-compose.yml` |

## Declared but unused in `Settings` (observations)

`SECRET_KEY`, `DISABLE_JWT*`, `rate_limit_requests_per_minute`, `uvicorn_workers`, `host`/`port`, `timezone`, `bhashini_api_url`, `bhashini_api_key`, `ollama_endpoint_url`, `marqo_endpoint_url`, `inference_endpoint_url`, `inference_api_key`, `openai_api_key`, `sarvam_api_key`, `meity_api_key_value`, `logfire_token`, `eleven_labs_api_key`, `gemini_api_key`, `mapbox_api_token`, `aws_*`, `llm_provider`, `llm_model_name`, `marqo_index_name`, `qdrant_api_key`, `qdrant_collection_name`, `redis_retry_on_timeout`, `suggestions_cache_ttl`, `telemetry_*` extras. No code path was found that reads these via `settings.<name>` (verified by grep); the runtime values come from the direct `os.getenv` calls listed above. See [Technical Debt](../knowledge/technical-debt.md) and [Legacy Artifacts](../knowledge/legacy-artifacts.md).

## Files expected on disk (not in git)

| File | Purpose | Gitignore rule |
|------|---------|----------------|
| `jwt_public_key.pem` | Required at import by `app/auth/jwt_auth.py` | `*jwt_public_key.pem` |
| `private_key.pem` | Token signing | `*private_key.pem` |
| `cloud-translate-credentials.json` | Only for `GoogleTranslator` (not used by active code) | `*credentials.json` |
| Play Integrity service account JSON | Path from `PLAY_INTEGRITY_SERVICE_ACCOUNT_FILE` | not covered by a rule |

With `docker-compose.yml` mounting `.:/app`, these files must exist in the host checkout next to `main.py`.

## Secrets summary

Secrets or secret-bearing variables: `OPENAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AGRINET_GEMMA_API_KEY`, `QDRANT_API_KEY`, `SATHI_MASTER_API_KEY`, `NPSS_USERNAME`/`NPSS_PASSWORD`, `MEITY_API_KEY_VALUE`, `TELEMETRY_AUTH_TOKEN`/`_KEY`/`_SECRET`, `LANGFUSE_SECRET_KEY`, `API_KEY_AUTH_TOKEN_<CLIENT>`, `PLAY_INTEGRITY_PRIVATE_KEY_<CLIENT>`, `HF_TOKEN`, the PEM/JSON key files, and the placeholder `SECRET_KEY`. Keep them in the host `.env` (gitignored via `.env`) and never in compose files or images.

> **Note:** `HF_TOKEN` is passed as a Docker build arg and re-exported with `ENV HF_TOKEN=${HF_TOKEN}`, so a non-empty value remains visible in image metadata. Whether production builds pass a token is not documented in this wiki.

## Source Files

`app/config.py`, `.env.example`, `docker-compose.yml`, `Dockerfile`, `agents/models.py`, `app/routers/token.py`, `app/core/image_storage.py`, `app/tasks/telemetry.py`, `helpers/langfuse_helper.py`, `helpers/master_catalog.py`, `agents/tools/*.py`.
