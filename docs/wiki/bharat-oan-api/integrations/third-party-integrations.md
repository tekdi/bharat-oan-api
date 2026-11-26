# Third-Party Integrations

Every external API, SDK and data service that `bharat-oan-api` talks to, with the environment variables that configure it, how it is called and how the service behaves when it is unavailable. All claims were traced to source files.

Related pages: [AI / LLM Integrations](./ai-llm.md) | [Caching](./caching.md) | [Async Processing](./async-processing.md) | [Configuration](../architecture/configuration.md) | [Component Diagram](../architecture/component-diagram.md) | [Sequence Diagrams](../architecture/sequence-diagrams.md) | [Technical Debt](../knowledge/technical-debt.md)

---

## 1. Integration Inventory

| # | Integration | Type | Used by (code) | Role in the product |
|---|-------------|------|----------------|---------------------|
| 1 | Vistaar network BAP (Beckn) | HTTP/JSON gateway | `agents/tools/*.py` (scheme info, status, grievance, weather, mandi, GFR, SATHI, AIF, SMAM, MahaVistaar, AmulVistaar, scheme search) | Single gateway to government and partner data providers |
| 2 | LLM provider (OpenAI / Azure OpenAI / vLLM) | SDK (`openai`, `pydantic-ai`) | `agents/models.py` | Main agent, moderation, optional Gemma route - see [AI / LLM](./ai-llm.md) |
| 3 | Bhashini (Dhruva) ASR / TTS / translation | HTTP | `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py` | Speech-to-text, text-to-speech, NPSS-response translation |
| 4 | OpenAI Whisper | SDK | `helpers/transcription.py::transcribe_whisper` | Alternative speech-to-text |
| 5 | Marqo | SDK (`marqo`) | `agents/tools/search.py` | Hybrid vector search for documents and pests/diseases |
| 6 | Qdrant (+ `sentence-transformers`) | SDK (`qdrant-client`) | `helpers/video_qdrant_search.py`, `helpers/scheme_qdrant_search.py` | eNAM/training video search; scheme embedder warm-up |
| 7 | Redis | SDK (`aiocache`, `redis`) | `app/core/cache.py`, `helpers/master_catalog.py` | Session history, route pins, caches, master catalog snapshot |
| 8 | Photon geocoder | HTTP | `agents/tools/maps.py`, `app/core/npss_geocodes.py` | Forward/reverse geocoding for India |
| 9 | NPSS (National Pest Surveillance System) | HTTP | `agents/tools/npss.py`, `app/core/npss_geocodes.py` | Crop image pest/disease identification |
| 10 | SATHI master API | HTTP | `agents/tools/sathi_seed.py` | Seed crop-group / crop catalogue |
| 11 | Telemetry service (Sunbird-style observability) | HTTP | `app/tasks/telemetry.py`, `helpers/telemetry.py` | Question/answer/feedback/error/ASR/TTS events |
| 12 | Langfuse | SDK (`langfuse>=3.0.0`) | `helpers/langfuse_*.py`, `app/services/chat.py` | LLM tracing and user-feedback scoring |
| 13 | Google Play Integrity API | HTTP + `google-auth` | `app/routers/token.py` | Optional device attestation to issue JWTs |
| 14 | Google Cloud Storage (mounted volume) | Filesystem mount | `app/core/image_storage.py`, `agents/tools/npss.py` | Optional archive of processed crop images |
| 15 | docs-pipeline master catalog | Redis snapshot | `helpers/master_catalog.py` | Live list of vector-indexed schemes |

---

## 2. Vistaar Network (BAP / Beckn) Gateway

Most farmer-facing data flows through one Beckn Application Platform (BAP) client endpoint. The service acts as a BAP; providers (BPPs) sit behind the gateway.

### Environment variables

| Variable | Used in | Purpose |
|----------|---------|---------|
| `BAP_ENDPOINT` | all Beckn tools | Base URL; tools append `/search`, `/init` or `/status` |
| `BAP_ID`, `BAP_URI` | all Beckn tools | Sent in the Beckn `context` |
| `BPP_ID`, `BPP_URI` | most tools (not sent by `get_scheme_info`, `call_maha_vistaar_network`, `call_amul_vistaar_network`) | Target provider identity |
| `SCHEME_DOMAIN` | `scheme_info.py`, `maha_vistaar.py` | Beckn domain, default `schemes:vistaar` |
| `AMUL_SCHEME_DOMAIN` | `amul_vistaar.py` | Default `schemes:amul-union` |
| `AMUL_SEARCH_TTL` | `amul_vistaar.py` | Beckn TTL, default `PT30S` |
| `DEFAULT_API_TIMEOUT`, `DEFAULT_API_READ_TIMEOUT` | `app/config.py` | Connect/read timeout (defaults 5 s / 10 s) |
| `API_BASE_URL` | `agents/tools/shc_scheme_status.py` | Public base URL used to build `/api/file/{hash}` links for Soil Health Card reports |
| `GRIEVANCE_TYPES_PATH` | `pmkisan_grievance.py` | Optional override of `assets/grievance_types.json` |

### Beckn endpoints used (by tool)

| Tool(s) | Beckn action | Category / provider |
|---------|--------------|---------------------|
| `get_scheme_info` | `/search` | `schemes-agri` |
| `search_schemes` | `/search` | `scheme-agri-qdrant` |
| `weather_forecast` | `/search` | `WFC` (Weather-Forecast-Mausamgram) |
| `get_mandi_prices` | `/search` | `price-discovery` |
| `gfr_*` | `/search` | GFR crop registry / recommendation |
| `search_sathi_seed_availability` | `/search` | provider `sathi-seed`, item `seed_availability` |
| `check_smam_scheme_status` | `/search` | SMAM status |
| `initiate_pm_kisan_status_check` / `check_pm_kisan_status_with_otp`, `initiate_pmfby_status_check` / `check_pmfby_status_with_otp` | `/init`, `/status` | PM-KISAN, PMFBY status |
| `check_shc_status` | `/init` | Soil Health Card |
| `initiate_aif_otp`, `verify_aif_otp`, `check_aif_*` | `/init`, `/status` | provider id from `agents/tools/aif.py` (`PROVIDER_ID`), item `aif-status` |
| `pmfby_*` grievance tools | `/init`, `/status` | `pmfby-grievance` |
| `pmkisan_*` grievance tools | `/init`, `/status` | PM-KISAN grievance |
| `call_maha_vistaar_network`, `call_amul_vistaar_network` | `/search` | cross-network (MahaVistaar / AmulVistaar) |

Transaction ids for OTP flows are deterministic: `uuid5(NAMESPACE_DNS, session_id + identifier)` (`generate_transaction_id` in `pmfby_scheme_status.py`, `pmkisan_grievance.py`), so the `/init` and `/status` calls of one session match.

> **Note:** Several status tools log the full Beckn response body at INFO level (for example `[PM KISAN STATUS] Response Payload` in `pmkisan_scheme_status.py`, `[AIF ...] Response Payload` in `aif.py`). PMFBY has a masking helper (`mask_pii_value` in `pmfby_scheme_status.py`), but the other tools do not mask farmer data in logs. This is recorded in [Technical Debt](../knowledge/technical-debt.md).

---

## 3. Speech and Translation: Bhashini and Whisper

### Bhashini (Dhruva) - `https://dhruva-api.bhashini.gov.in/services/inference/pipeline` (hard-coded in code)

| Capability | Function | Details |
|------------|----------|---------|
| ASR | `helpers/transcription.py::transcribe_bhashini` | `serviceId`: `bhashini/ai4bharat/conformer-multilingual-asr` for non-English, `ai4bharat/whisper-medium-en--gpu--t4` for `en`; WAV, 16 kHz, VAD pre-processor. Shared `httpx.Client` (connect 60 s, read 120 s) |
| TTS | `helpers/tts.py::text_to_speech_bhashini` | Empty `serviceId` (language-based routing), gender `female`, sampling rate 8000 (router fixes these), timeout 30 s / read 60 s; returns base64 audio |
| Translation | `helpers/translation.py::BhashiniTranslator` | `serviceId` `bhashini/ai4bharat/indictrans-v3`, timeout 10 s / read 30 s; only called from `app/services/npss_response.py` |

Auth: `MEITY_API_KEY_VALUE` (sent as the `Authorization` header value). The request/curl command is logged (with the key replaced by a placeholder).

### Whisper

`transcribe_whisper` uses `openai.OpenAI(api_key=OPENAI_API_KEY)` with model `whisper-1` and `verbose_json`, mapping the detected language through `langcodes`. Selected via `service_type: "whisper"` on `POST /api/transcribe/`.

### API surface

| Endpoint | Handler | Services allowed |
|----------|---------|------------------|
| `POST /api/transcribe/` | `app/routers/transcribe.py` | `bhashini`, `whisper` |
| `POST /api/tts/` | `app/routers/tts.py` | `bhashini` only (the request model also lists `eleven_labs`, but the router rejects it with HTTP 400) |

See [API Specification](../application/api-specification.md) for request/response shapes.

### Environment variables

`MEITY_API_KEY_VALUE` (Bhashini), `OPENAI_API_KEY` (Whisper). `BHASHINI_API_KEY`, `BHASHINI_API_URL` and `ELEVEN_LABS_API_KEY` exist in `app/config.py` but are not read by runtime code.

---

## 4. Search and Retrieval Backends

| Service | Env vars | Behaviour |
|---------|----------|-----------|
| Marqo | `MARQO_ENDPOINT_URL`, `MARQO_INDEX_NAME` (default `sunbird-va-index`), `MARQO_PESTS_DISEASES_INDEX_NAME` | `marqo.Client(url=...)`, `client.config.timeout = 10`; hybrid RRF search with `top_k` default 10 |
| Qdrant | `QDRANT_URL`, `QDRANT_API_KEY` (required when the URL is not localhost), `QDRANT_VIDEO_COLLECTION_NAME` (default `video_data_collection`), `QDRANT_COLLECTION_NAME` (setting default `schemes-index`) | Video search via `query_points` with a `type=video` filter; embedder `intfloat/multilingual-e5-large` runs locally |
| Hugging Face | `HF_TOKEN` (Docker build arg only) | Used while baking the embedding model into the image |

`search_schemes` itself does **not** call Qdrant: it posts a Beckn `/search` (category `scheme-agri-qdrant`) to the BAP and formats the returned chunks. The local Qdrant/embedder code is used for videos and for the startup warm-up.

---

## 5. Other Services

### Redis
Configured by `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`, `REDIS_KEY_PREFIX` (default `sva-cache-`), `REDIS_SOCKET_CONNECT_TIMEOUT`, `REDIS_SOCKET_TIMEOUT`, `REDIS_MAX_CONNECTIONS`, `REDIS_RETRY_ON_TIMEOUT` (`app/config.py`). The `aiocache` instance is in `app/core/cache.py`; `helpers/master_catalog.py` uses a raw `redis.Redis` client with no key prefix. Details in [Caching](./caching.md). Redis setup notes: `docs/setup/redis.md`.

### Photon geocoder
`PHOTON_HOST`. `agents/tools/maps.py` builds `http://<host>:<port>` from the URL (falling back to a hard-coded internal host/port when the variable is not parseable) and keeps a shared `httpx.AsyncClient` (10 s timeout). `app/core/npss_geocodes.py` requires an **HTTPS** `PHOTON_HOST` (`_validated_photon_base_url`) and otherwise disables NPSS geocoding. `.env.example` documents `PHOTON_HOST` with the HTTPS requirement.

### NPSS
| Variable | Default | Purpose |
|----------|---------|---------|
| `NPSS_BASE_URL` | `https://npss.dac.gov.in/api3.0` | API base |
| `NPSS_USERNAME`, `NPSS_PASSWORD` | empty (required at call time) | Token login at `/api/Vistaar/token` |
| `BASE_URL` | none | Resolves relative `/api/image/{id}` URLs for download (required for uploaded images) |
| `TEMP_UPLOAD_DIR`, `IMAGE_TTL_MINUTES`, `MAX_IMAGE_UPLOAD_SIZE_MB` | `.oan-uploads`, 60, 10 | Temporary image storage (`app/core/image_storage.py`) |
| `GCS_MOUNT_PATH`, `GCS_MOVE_AFTER_PROCESS` | empty | Optional move of processed images to a mounted GCS bucket instead of deleting |

Flow (`agents/tools/npss.py`): fetch bearer token (cached in Redis key `npss:bearer_token` for 25 minutes), resolve location hierarchy IDs, download the image, validate size/MIME (JPEG/PNG; WebP rejected), `POST /api/Vistaar/analyze-image` (60 s timeout), cleanup. Location resolution uses browser coordinates when present, else a fixed fallback (Krishi Vigyan Kendra, Ujwa, Delhi - `NPSS_FALLBACK_LOCATION`).

> **Note:** In `app/services/chat.py` the `FarmerContext` for a turn is built without `latitude`/`longitude` (they are only used to word the image-analysis instruction), and `analyze_crop_image` reads coordinates from `ctx.deps`. As written, the NPSS location therefore always resolves to the Delhi fallback. Recorded in [Technical Debt](../knowledge/technical-debt.md).

### SATHI master API
`SATHI_MASTER_BASE_URL`, `SATHI_MASTER_API_KEY`. Endpoints `/get-crop-group` and `/get-crops-list` (the latter passes the key as the `apiKey` query parameter). Crop groups are cached in Redis for 7 days (namespace `sathi`). Dealer stock comes from the BAP.

### Telemetry service
`TELEMETRY_API_URL` (default in `app/config.py` is a hard-coded `https://dev-vistaar.da.gov.in/observability-service/action/data/v3/telemetry`), auth via `TELEMETRY_AUTH_TOKEN` or an HS256 token generated from `TELEMETRY_AUTH_KEY` + `TELEMETRY_AUTH_SECRET`, optional `TELEMETRY_ORIGIN`, `TELEMETRY_REFERER`, `TELEMETRY_CHANNEL`. `send_telemetry` (`app/tasks/telemetry.py`) retries 3 times, 1 s apart, on timeouts/request errors. Events are queued with FastAPI `BackgroundTasks`.

### Langfuse
`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` / `LANGFUSE_BASE_URL`, `LANGFUSE_TRACING_ENVIRONMENT`, `LANGFUSE_TRACE_NAME`, `LANGFUSE_TRACE_ROOT_NAME`. See [AI / LLM - Langfuse](./ai-llm.md#9-langfuse-tracing).

### Google Play Integrity
`PLAY_INTEGRITY_PACKAGE_NAME`, `PLAY_INTEGRITY_PACKAGE_NAME_<CLIENT>`, `PLAY_INTEGRITY_SERVICE_ACCOUNT_FILE`, `PLAY_INTEGRITY_SERVICE_ACCOUNT_BASE_PATH`, `PLAY_INTEGRITY_PRIVATE_KEY_<CLIENT>`, `PLAY_INTEGRITY_FRESHNESS_SECONDS` (120), `PLAY_INTEGRITY_DEFAULT_NAME/ROLE/METADATA`. Used only by `POST /api/token/play-integrity` (`app/routers/token.py`); it calls `playintegrity.googleapis.com/v1/{package}:decodeIntegrityToken`, validates package, nonce (replay-protected through Redis `integrity_nonce:<nonce>`), freshness, `PLAY_RECOGNIZED` and `MEETS_DEVICE_INTEGRITY`. See [Users and Personas](../business/users-and-personas.md).

### Master catalog (docs-pipeline)
`MASTER_CATALOG_REDIS_KEY` (optional exact key), `ENVIRONMENT` (selects `dev` tier for `local/dev/development`, otherwise `live`). Pull-based read; see [AI / LLM - Retrieval](./ai-llm.md#10-retrieval-and-embeddings).

---

## 6. Declared but Unused Integrations

The following appear in `app/config.py`, `requirements.txt` or helper modules but are not called from any runtime path (verified by grep):

| Item | Where declared | Observation |
|------|----------------|-------------|
| Google Cloud Translation (`GoogleTranslator`) | `helpers/translation.py`; `google-cloud-translate` in `requirements.txt` | Class is never instantiated; expects `cloud-translate-credentials.json` two directories above `helpers/` |
| ElevenLabs TTS | `TTSRequest.service_type`, `ELEVEN_LABS_API_KEY` | Router rejects `eleven_labs` |
| Settings keys `SARVAM_API_KEY`, `GEMINI_API_KEY`, `MAPBOX_API_TOKEN`, `INFERENCE_*`, `OLLAMA_ENDPOINT_URL`, `LOGFIRE_TOKEN`, AWS keys, `MARQO_*` settings fields | `app/config.py` | No code reads these `settings.*` attributes. `boto3` is listed in `requirements.txt` and imported in `helpers/utils.py`, but no S3 call exists |
| `helpers/encryption.py` (AES-GCM helpers for PM-KISAN grievance) | module only | Not imported anywhere |
| `word_mapping_reduced_1000.json` | `helpers/translation.py` | File is absent from `assets/`; the loader catches the error and uses an empty list |
| Suggestions agent | `agents/suggestions.py` | Disabled, see [AI / LLM](./ai-llm.md) |

---

## 7. Fallback Behavior

"Hard fail" means the user turn or request errors out. "Soft degrade" means the feature returns a polite message or reduced output and the chat continues. All tool failure strings below are returned to the LLM as the tool result, and the prompt instructs the model to tell the farmer plainly and not fill the gap from memory.

| Integration | Failure mode | Behaviour | Type | Source |
|-------------|--------------|-----------|------|--------|
| LLM provider (routing off) | Error or 45 s timeout | Exception propagates, chat error telemetry queued, stream errors | **Hard fail** | `app/services/chat.py` |
| LLM provider (routing on) | Primary route error before first chunk | One retry on the alternate route (`gpt41` <-> `gemma`); session pin updated | Redirect (single retry) | `_run_agrinet_with_failover_streaming` |
| LLM provider (routing on) | Error after streaming started | No retry, error propagates | **Hard fail** | same |
| Moderation LLM | Error / 20 s timeout | Not caught; turn errors | **Hard fail** | `_run_moderation` |
| Gemma metrics endpoint | Unreachable | `get_gemma_concurrency` returns `None` -> capacity gate treats as "not under limit" and deflects to `gpt41` for the turn | Soft degrade | `agrinet_routing.py` |
| Redis (history, route, caches) | Down | `get_cache`/`set_cache` exceptions propagate; `/api/health/ready` returns 503. Route-metrics cache read/write errors are swallowed | **Hard fail** for chat; soft for Gemma metrics cache | `app/utils.py`, `app/routers/health.py` |
| Master catalog snapshot | Missing / Redis error / bad JSON | Returns `None`; scheme list is empty and prompt block shows zero vector schemes; `search_schemes` reports scheme unavailable | Soft degrade | `helpers/master_catalog.py` |
| BAP gateway (most tools) | Non-200 | Returns a "service unavailable, try later" string to the LLM (e.g. `weather.py`, `mandi.py`, `search.py`) | Soft degrade | `agents/tools/*` |
| BAP gateway | Timeout / request error | Timeout string, or `request failed: <error>` string | Soft degrade | same |
| BAP gateway | `BAP_ENDPOINT` unset | Most tools return "not configured"; `get_scheme_info`, PM-KISAN/PMFBY/SHC status and similar call `os.getenv("BAP_ENDPOINT").rstrip(...)` directly and raise `AttributeError`, which is converted to `ModelRetry` or a generic error | Mixed (soft for some, retry loop for others) | e.g. `scheme_info.py`, `pmkisan_scheme_status.py` |
| BAP gateway (GFR) | Any failure | `ModelRetry` (model retries up to the agent's limit) | Retry, then error | `gfr.py` |
| BAP gateway (AIF) | Any failure | Raised internally as `_AifUnavailable` with farmer-facing wording | Soft degrade | `aif.py` |
| Marqo | Error / missing env | `ModelRetry("Error searching documents, please try again")`; after retries the agent errors | Retry then hard fail | `search.py` |
| Qdrant (video) | Error / `QDRANT_URL` unset | `ModelRetry` | Retry then hard fail | `search.py` |
| Bhashini ASR | 429/5xx, connect/read timeout | 3 attempts with exponential back-off (2-20 s); then exception -> HTTP 500; telemetry event still recorded | **Hard fail** (after retries) | `helpers/transcription.py`, `app/routers/transcribe.py` |
| Bhashini TTS | Any error | Exception -> HTTP 500; no retry | **Hard fail** | `helpers/tts.py`, `app/routers/tts.py` |
| Bhashini translation (NPSS only) | Any error | Logs a warning, returns untranslated English text | Soft degrade | `app/services/npss_response.py` |
| Whisper | Any error | Exception -> HTTP 500 | **Hard fail** | `helpers/transcription.py` |
| Photon (tools) | Timeout / connect / HTTP error | Forward: "Unable to find location..." string; reverse: `Location(place_name="Unknown Location")` | Soft degrade | `maps.py` |
| Photon (NPSS geocode) | Not HTTPS / unreachable | Geocoding disabled/empty -> fall back to the Delhi KVK location | Soft degrade | `npss_geocodes.py`, `npss.py` |
| NPSS token / analyze | Token error, 401, 5xx | `ModelRetry` (401 clears cached token); unexpected error returns "pest analysis service encountered an unexpected error" | Retry, then soft message | `agents/tools/npss.py` |
| NPSS image problems | Too large / unsupported / WebP | Friendly message returned, no NPSS call | Soft degrade | `npss.py::_prepare_image` |
| SATHI master API | Error | "Could not load SATHI crop groups/crops" string | Soft degrade | `sathi_seed.py` |
| Telemetry service | Timeout / request error | 3 retries, then exception inside a background task (logged, no user impact); non-2xx is logged only | Soft degrade | `app/tasks/telemetry.py` |
| Langfuse | Write failure | Feedback score failure is caught and logged; other SDK errors are not wrapped in the chat path | Soft (scores), unverified elsewhere | `helpers/langfuse_scores.py` |
| Play Integrity | Google API error / bad verdict | HTTP 502 or 401 on `/api/token/play-integrity`; Redis failure -> 503 "Replay protection is unavailable" | **Hard fail** (no token issued) | `app/routers/token.py` |
| Qdrant embedder warm-up | Error at startup | Logged, app continues; first `search_schemes`-adjacent call may be slow | Soft degrade | `helpers/scheme_qdrant_search.py::warm_scheme_search` |
| GCS mount | Move fails | Falls through to delete the file | Soft degrade | `app/core/image_storage.py` |

No LLM-provider fallback exists when `AGRINET_ROUTING_ENABLED` is false, and there is no fallback for the moderation model.

---

## Source Files

`agents/tools/*.py`, `agents/models.py`, `app/config.py`, `app/core/cache.py`, `app/core/image_storage.py`, `app/core/npss_geocodes.py`, `app/routers/token.py`, `app/routers/transcribe.py`, `app/routers/tts.py`, `app/tasks/telemetry.py`, `app/services/npss_response.py`, `app/services/agrinet_routing.py`, `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py`, `helpers/master_catalog.py`, `helpers/scheme_qdrant_search.py`, `helpers/video_qdrant_search.py`, `helpers/telemetry.py`, `.env.example`, `requirements.txt`, `docs/setup/redis.md`.
