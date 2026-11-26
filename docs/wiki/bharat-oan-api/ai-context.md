# AI Context: bharat-oan-api

Single entry document for AI agents working in this repository. Dense by design; every claim names a file, class or config key. Deeper detail is linked from each section. Read this, then [README](README.md), then [Architecture Overview](architecture/architecture-overview.md).

## Service Summary

`bharat-oan-api` (app title "Bharatvistaar AI API", `app/config.py`) is a FastAPI backend that answers Indian farmers' questions in 13 languages. Each chat turn runs a moderation LLM, then a tool-using main LLM agent (pydantic-ai) that calls government-scheme, mandi, weather, fertilizer, seed, search and crop-image tools, and streams the answer as plain text. It also exposes speech-to-text, text-to-speech, image upload, telemetry relay and token-issuance endpoints. Entry point: `main.py` (`app = FastAPI(...)`, routers under `/api`).

## Business Purpose

Give farmers a conversational assistant for agriculture advice and for finding, checking and raising grievances on schemes (PM-KISAN, PMFBY, SHC, AIF, SMAM and others), mostly through the Vistaar/Beckn network via a BAP gateway. See [Business Overview](business/business-overview.md), [Workflows](business/workflows.md), [Business Rules](business/business-rules.md).

## Major Capabilities

- Streaming chat with per-session history in Redis: `GET /api/chat/` (`app/routers/chat.py`, `app/services/chat.py`).
- 36 agent tools in `agents/tools/__init__.py` (`TOOLS`). See [AI/LLM](integrations/ai-llm.md#6-tools-registry).
- Moderation of every message (`agents/moderation.py`, 8 categories, prompt-enforced).
- Crop image diagnosis via the NPSS tool (`agents/tools/npss.py`, `app/core/npss_geocodes.py`, `app/services/npss_response.py`).
- ASR/TTS (`app/routers/transcribe.py`, `app/routers/tts.py`, `helpers/transcription.py`, `helpers/tts.py`) and translation helpers (`helpers/translation.py`).
- Optional A/B routing between two model routes `gpt41` and `gemma` with failover (`app/services/agrinet_routing.py`, `AGRINET_ROUTING_ENABLED`).
- Backend-owned telemetry and Langfuse tracing (`helpers/telemetry.py`, `helpers/langfuse_*.py`, `app/routers/telemetry.py`).

## Architecture Summary

Stateless async API process (uvicorn, 8 workers under supervisord) plus Redis. No SQL database. External dependencies: OpenAI / Azure OpenAI / vLLM, the BAP gateway (Beckn), Bhashini, Marqo, Qdrant, Photon geocoder, NPSS service, SATHI master API, Langfuse, a telemetry service. Diagrams: [Components](architecture/component-diagram.md), [Sequences & flows](architecture/sequence-diagrams.md).

Chat turn in one line: JWT check -> normalize query / resolve image id -> select model route -> load history from Redis -> moderation agent -> `agrinet_agent` with tools, streaming chunks -> NPSS post-processing (image turns) -> save history, send telemetry, flush Langfuse (see [W2](architecture/sequence-diagrams.md#w2-streaming-chat-turn)).

## Repository Structure

| Path | Role |
|------|------|
| `main.py` | App factory, lifespan validation, CORS, router mounting |
| `app/config.py` | `Settings` (pydantic-settings) |
| `app/routers/` | HTTP layer: `chat`, `file`, `health`, `image`, `suggestions` (not mounted), `telemetry`, `token`, `transcribe`, `tts`, `chat_query_utils` |
| `app/services/` | `chat.py` (orchestration), `agrinet_routing.py`, `npss_response.py`, `chat_turn_map.py` |
| `app/core/` | `cache.py` (Redis/aiocache), `image_storage.py`, `npss_geocodes.py`, `npss_followup.py` |
| `app/auth/jwt_auth.py` | RS256 JWT validation (`get_current_user`) |
| `agents/` | `agrinet.py`, `moderation.py`, `suggestions.py` (disabled), `models.py`, `deps.py`, `tools/*` |
| `helpers/` | Translation, ASR/TTS, Qdrant search, telemetry, Langfuse, master catalog, utils |
| `assets/` | `prompts/agrinet_<lang>.md`, `moderation_system.md`, JSON reference data |
| `tests/` | 6 offline pytest files, 5 live-LLM runner scripts, committed `results_*.json` |
| `Dockerfile`, `docker-compose.yml`, `supervisord.conf`, `.github/workflows/build.yml` | Build and runtime |

Full map: [Repository Map](repository-map.md).

## Technology Stack

Python 3.13 (`Dockerfile`: `python:3.13-slim`), FastAPI, uvicorn (8 workers via `supervisord.conf`), pydantic v2, `pydantic-ai==1.2.1`, `openai`, Redis (aiocache + `redis`), `httpx`, `PyJWT`/`cryptography`, `rapidfuzz`, `sentence-transformers` (multilingual-e5-large baked into the image), `qdrant-client`, `marqo`, `langfuse>=3`, `python-decouple`. Unpinned: most dependencies, no lock file (`requirements.txt`).

## Critical Business Rules

- 13 answer languages: `en hi as bn gu kn mai ml mr or pa ta te`; `target_lang` picks the prompt `assets/prompts/agrinet_<lang>.md`, default `hi`.
- Moderation runs on every turn; enforcement is prompt-based, the main agent still runs for refused categories.
- Tool calls are in English; answers use ASCII digits; one language per answer.
- OTP-based status checks (PM-KISAN 4-digit, PMFBY/AIF 6-digit) are two-step `init` then `status` tools.
- History: last 3 message pairs in the prompt prefix, `trim_history` to 64k tokens, 2 h TTL.

Details and sources: [Business Rules](business/business-rules.md).

## Important APIs

| Method | Path | Handler | Auth |
|--------|------|---------|------|
| GET | `/api/chat/` | `chat.py:chat_endpoint` | JWT |
| POST | `/api/chat/analyze-image` | `chat.py:chat_analyze_image_endpoint` | JWT (deprecated) |
| POST | `/api/image/upload`, GET `/api/image/{id}` | `image.py` | JWT / none |
| POST | `/api/transcribe/`, `/api/tts/` | `transcribe.py`, `tts.py` | JWT |
| POST | `/api/token`, `/api/token/api-key`, `/api/token/play-integrity` | `token.py` | none / API key / Play Integrity |
| POST | `/api/telemetry/feedback`, `/events`, `/error` | `telemetry.py` | JWT |
| GET | `/api/health/`, `/live`, `/ready`, `/api/file/{hash}`, `/` | various | none |

Full table: [API Specification](application/api-specification.md). `API_DOCUMENTATION.md` is partly outdated (see its reconciliation section).

## Important Data Stores

No relational database or ORM. Redis keys (prefix `REDIS_KEY_PREFIX`, default `sva-cache-`): `<session_id>__SVA` (history), `<session_id>_AGRINET_ROUTE`, qid-to-trace map, `html_file:<hash>`, token/nonce keys. Qdrant (video search), Marqo (documents, pests/diseases), local disk for temporary images. See [Database](application/database.md), [Caching](integrations/caching.md).

## Core Domain Models

`ChatRequest` and other request models in `app/models/requests.py`; `app/models/responses.py` (largely unused); `FarmerContext` in `agents/deps.py` (query, `lang_code`, `session_id`, `question_id`, NPSS state); moderation output types in `agents/moderation.py`.

## Core Services

`stream_chat_messages` in `app/services/chat.py`; `resolve_agrinet_route` in `app/services/agrinet_routing.py`; `post_process_npss_response` in `app/services/npss_response.py`; `write_chat_turn_map` in `app/services/chat_turn_map.py`; `send_telemetry` in `helpers/telemetry.py`.

## AI/LLM Components

Providers via `LLM_PROVIDER` (`openai` default, `azure-openai`, `vllm`) in `agents/models.py`; two models built at import (agrinet, moderation). Prompts are Jinja markdown in `assets/prompts/`. Instrumentation uses Langfuse `@observe`, not pydantic-ai OTel. Timeouts: `AGRINET_MODEL_TIMEOUT_SECONDS` (45), `LLM_MODERATION_TIMEOUT_SECONDS` (20). Details: [AI/LLM](integrations/ai-llm.md).

## Environment Variables

Most important (full list in [Configuration](architecture/configuration.md)): `LLM_PROVIDER`, `OPENAI_API_KEY` or `AZURE_OPENAI_*` or `VLLM_AGRINET_MODEL_URL`, `LLM_AGRINET_MODEL_NAME`, `LLM_MODERATION_MODEL_NAME`, `REDIS_HOST/PORT/DB`, `JWT_PUBLIC_KEY_PATH`, `JWT_PRIVATE_KEY_PATH`, `JWT_EXPIRY_MINUTES`, `API_KEY_AUTH_TOKEN_<CLIENT>`, `BAP_*` and `BAP_ENDPOINT`, `MARQO_ENDPOINT_URL`, `QDRANT_URL`, `PHOTON_HOST`, `BASE_URL`, `ALLOWED_ORIGINS`, `AGRINET_ROUTING_ENABLED`, Langfuse keys. A missing `jwt_public_key.pem` or invalid `LLM_PROVIDER` aborts startup at import.

## Build Commands

```bash
docker network create svanetwork            # once; Redis must be on it (docs/setup/redis.md)
docker compose up --build --force-recreate --detach
python -m pip install -r requirements.txt   # local, Python 3.13
uvicorn main:app --host 0.0.0.0 --port 8000 # local run (needs .env, jwt_public_key.pem)
```

## Test Commands

```bash
pytest tests/test_languages.py tests/test_chat_query_utils.py tests/test_npss_geocodes.py \
       tests/test_audit_npss_empty_villages.py tests/test_backend_owned_telemetry.py
```

Select files explicitly; bare `pytest tests/` imports live-LLM scripts. `tests/test_chat_stream_contract.py` is stale. No CI test job exists. See [Testing](architecture/testing.md).

## Deployment Commands

`docker compose down --remove-orphans`; logs `docker logs -f sva_app`; health `curl localhost:8000/api/health/ready`. CI (`.github/workflows/build.yml`) builds and pushes an image to GCP Artifact Registry on pushes to `main` and `bh-dev`; rollout is outside the repo. See [Deployment](operations/deployment.md).

## Troubleshooting Quick Guide

| Symptom | Check |
|---------|-------|
| App exits at start | `jwt_public_key.pem` present; `LLM_PROVIDER` and its variables set |
| 503 on `/api/health/ready` | Redis reachable (`REDIS_HOST` = container name on `svanetwork`) |
| 401 on chat | Token expired (default 15 min) or not RS256 from this key pair |
| Image analysis falls back to Delhi location | Known issue: coordinates are not passed to NPSS ([TD-03](knowledge/technical-debt.md#td-03--npss-location-always-falls-back-to-the-default)) |
| Template-not-found on chat | Unsupported `target_lang` (no prompt file) |

More: [Troubleshooting](operations/troubleshooting.md).

## Technical Debt Summary

Top items: open guest token endpoint and no role checks; NPSS location never fed from chat; blocking sync calls in async handlers; image uploads never cleaned up; stale tests and API docs; farmer data in logs; unpinned dependencies; exposed credential in `docs/setup/portainer.md`. Register: [Technical Debt](knowledge/technical-debt.md).

## AI Agent Working Instructions

1. Verify against code before changing behaviour; this wiki is derived from code at generation time and `API_DOCUMENTATION.md` is known to be stale.
2. Cursor rules for backend work live in `.cursor/rules/`; skills in `.cursor/skills/`; subagents in `.cursor/agents/`.
3. Adding a language: add `assets/prompts/agrinet_<lang>.md`, update `SUPPORTED_LANGS` in `agents/tools/terms.py` and NPSS label tables in `app/services/npss_response.py`, run `tests/test_languages.py`. See [Repository Conventions](architecture/repository-conventions.md).
4. Adding a tool: write it in `agents/tools/`, wrap with `@observe(as_type="tool")`, register in `TOOLS` in `agents/tools/__init__.py`, describe when to use it in the prompts.
5. Do not print or log farmer identifiers, OTPs or secrets; do not copy values from `.env` or `docs/setup/portainer.md` into docs.
6. Keep the wiki in sync: update the relevant page and its [index](wiki-index.md) when behaviour changes.
