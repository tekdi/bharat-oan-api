# Repository Map

Navigational map of the `bharat-oan-api` repository (FastAPI + pydantic-ai agricultural advisory chat API). Every path below is repo-relative and was verified against the working tree.

Related pages: [ai-context](ai-context.md) | [Architecture Overview](architecture/architecture-overview.md) | [Configuration](architecture/configuration.md) | [Coding Patterns](architecture/coding-patterns.md) | [Repository Conventions](architecture/repository-conventions.md) | [Legacy Artifacts](knowledge/legacy-artifacts.md)

## Folder Hierarchy

```
bharat-oan-api/
├── main.py                     # FastAPI app factory, middleware, lifespan, router registration
├── Dockerfile                  # python:3.13-slim image, supervisord entrypoint
├── docker-compose.yml          # single service `app` (container sva_app), external network svanetwork
├── supervisord.conf            # runs `uvicorn main:app --workers 8` as program `fastapi`
├── requirements.txt            # dependencies (mostly unpinned)
├── .env.example                # only NPSS_* / PHOTON_HOST documented
├── .dockerignore / .gitignore
├── README.md                   # short Docker/Redis command cheat-sheet
├── API_DOCUMENTATION.md        # hand-written endpoint notes (partly outdated, see below)
├── LICENSE                     # MIT, COSS / OpenAgriNet
├── .github/workflows/build.yml # only CI: build + push image to GCP Artifact Registry
├── agents/                     # pydantic-ai agents, model wiring, tool implementations
│   ├── agrinet.py              # main "Vistaar Agent" (agrinet_agent)
│   ├── moderation.py           # moderation agent + QueryModerationResult
│   ├── suggestions.py          # suggestions agent (disabled / broken import, see legacy-artifacts)
│   ├── models.py               # LLM provider selection + Gemma route model
│   ├── deps.py                 # FarmerContext (agent dependency / user-message builder)
│   └── tools/                  # 19 tool modules + __init__.py registering TOOLS
├── app/
│   ├── config.py               # Settings (pydantic-settings) + default httpx timeout
│   ├── utils.py                # Redis history helpers, message-pair formatting, trim_history
│   ├── auth/jwt_auth.py        # get_current_user (RS256 JWT verification)
│   ├── core/                   # cache.py, image_storage.py, npss_followup.py, npss_geocodes.py
│   ├── models/                 # requests.py, responses.py (pydantic request/response models)
│   ├── routers/                # chat, transcribe, tts, health, file, token, telemetry, image (+ suggestions, disabled)
│   ├── services/               # chat.py (orchestration), agrinet_routing.py, chat_turn_map.py, npss_response.py
│   └── tasks/                  # telemetry.py (active), suggestions.py / logging.py (inactive)
├── helpers/                    # langfuse_*, telemetry, translation, transcription, tts, qdrant search, master_catalog, utils
├── assets/
│   ├── prompts/                # agrinet_<lang>.md (13 langs), moderation_system.md, suggestions_system.md
│   ├── prompts_backup_20260922_164719/   # tracked backup of older prompts (10 langs)
│   ├── glossary_terms.json     # multilingual term pairs (~4 MB)
│   ├── commodity_codes.json    # AGMKT commodity codes + search terms
│   ├── grievance_types.json    # PM-KISAN grievance types
│   └── new_marqo_settings.json # Marqo index settings (not referenced by code)
├── docs/
│   ├── setup/portainer.md      # Portainer install notes (contains committed credentials, see Note)
│   ├── setup/redis.md          # Redis Stack production setup
│   ├── data/archive/CROP ADVISORY SOP.docx
│   └── wiki/bharat-oan-api/    # this wiki
├── scripts/audit_npss_empty_villages.py   # read-only NPSS master-data audit CLI
├── tests/                      # pytest tests, manual LLM runners, fixtures, committed results_*.json
└── .cursor/                    # (untracked) Cursor rules, agents, skills incl. llm-wiki
```

> **Note:** No `CODEOWNERS` file or ownership document is part of this repository. Folder ownership described below is inferred from the application structure and from commit history (for example NPSS work by the OD-2127 branch authors, language and prompt work by the OpenAgriNet contributors), not from a formal assignment.

## Key Entry Points

| Entry point | Path | What it does |
|-------------|------|--------------|
| ASGI app | `main.py` (`app`) | Builds `FastAPI`, CORS + `TimingAllowOriginMiddleware`, lifespan validation, registers routers under `/api` |
| Process manager | `supervisord.conf` | Starts `uvicorn main:app --host 0.0.0.0 --port 8000 --workers 8` |
| Container CMD | `Dockerfile` | `/usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf` |
| Chat orchestration | `app/services/chat.py` (`stream_chat_messages`) | Moderation then agrinet agent, SSE streaming, telemetry, Langfuse |
| Main agent | `agents/agrinet.py` (`agrinet_agent`) | pydantic-ai `Agent` with `TOOLS`, dynamic system prompt per language |
| Tool registry | `agents/tools/__init__.py` (`TOOLS`) | List of `pydantic_ai.Tool` objects exposed to the agent |
| CLI script | `scripts/audit_npss_empty_villages.py` | NPSS master data audit (JSON/CSV reports) |

## Configuration Locations

| What | Path |
|------|------|
| Central settings class | `app/config.py` (`Settings`, `settings`, `get_default_httpx_timeout`, `DEFAULT_HTTP_TIMEOUT`) |
| Env var template | `.env.example` (partial) |
| Scattered `os.getenv` reads | `agents/models.py`, `agents/tools/*.py`, `app/routers/token.py`, `app/core/image_storage.py`, `app/tasks/telemetry.py`, `helpers/*.py` |
| Container/runtime config | `Dockerfile`, `docker-compose.yml`, `supervisord.conf` |
| CI config | `.github/workflows/build.yml` |
| Redis tuning guidance | `docs/setup/redis.md` |

Full variable inventory: [Configuration](architecture/configuration.md).

## Prompt Locations

| Prompt | Path | Loaded by |
|--------|------|-----------|
| Agrinet system prompt per language (`as bn en gu hi kn mai ml mr or pa ta te`) | `assets/prompts/agrinet_<lang>.md` | `agents/agrinet.py` via `helpers.utils.get_prompt` (Jinja2) |
| Moderation prompt | `assets/prompts/moderation_system.md` | `agents/moderation.py` |
| Suggestions prompt | `assets/prompts/suggestions_system.md` | `agents/suggestions.py` (agent currently disabled) |
| Older prompt snapshot | `assets/prompts_backup_20260922_164719/` | Not loaded by code |

See [AI/LLM integration](integrations/ai-llm.md) for prompt behaviour.

## API Locations

| Area | Path |
|------|------|
| Router modules | `app/routers/` (`chat.py`, `transcribe.py`, `tts.py`, `health.py`, `file.py`, `token.py`, `telemetry.py`, `image.py`) |
| Router registration | `main.py` (`app.include_router(..., prefix=settings.api_prefix)` with `api_prefix="/api"`) |
| Request/response models | `app/models/requests.py`, `app/models/responses.py`; inline models in `app/routers/token.py`, `app/routers/image.py` |
| Hand-written API notes | `API_DOCUMENTATION.md` |

Endpoint catalogue: [API Specification](application/api-specification.md).

## Data Models

| Kind | Location |
|------|----------|
| API request models (`ChatRequest`, `TelemetryFeedbackRequest`, `TTSRequest`, ...) | `app/models/requests.py` |
| API response models | `app/models/responses.py` |
| Agent dependency `FarmerContext` | `agents/deps.py` |
| Moderation output `QueryModerationResult` | `agents/moderation.py` |
| Tool-local Beckn/response models | inside each `agents/tools/*.py` (for example `mandi.py`, `shc_scheme_status.py`) |
| Telemetry models (`TelemetryRequest`) | `helpers/telemetry.py` |
| Glossary `TermPair` | `agents/tools/terms.py` |

> **Note:** This service has no relational database, ORM, or migration files. State lives in Redis (chat history, routing decisions, cached files) and in an in-process image registry plus temp files; see [Database](application/database.md) and [Caching](integrations/caching.md).

## Infrastructure Files

| File | Purpose | Details |
|------|---------|---------|
| `Dockerfile` | Image build (CPU torch, baked `intfloat/multilingual-e5-large`) | [Infrastructure](operations/infrastructure.md) |
| `docker-compose.yml` | Runs service `app` on `svanetwork` | [Infrastructure](operations/infrastructure.md) |
| `supervisord.conf` | Process supervision | [Infrastructure](operations/infrastructure.md) |
| `.github/workflows/build.yml` | Build and push to GCP Artifact Registry | [Deployment](operations/deployment.md) |
| `docs/setup/*.md` | Portainer and Redis host setup | [Deployment](operations/deployment.md), [Disaster Recovery](operations/disaster-recovery.md) |

## Per-Folder Detail

### `agents/`
- **Purpose:** LLM agents, model/provider wiring, and every tool the agent can call.
- **Dependencies:** `pydantic-ai`, `openai`, `langfuse`, `httpx`, `app.config`, `helpers/*`, `assets/*`.
- **Important files:** `agrinet.py`, `moderation.py`, `models.py`, `deps.py`, `tools/__init__.py`.
- **Typical modifications:** add or edit a tool module and register it in `TOOLS`; change model parameters in `agrinet.py`; add a routing option in `models.py`. See [Repository Conventions](architecture/repository-conventions.md#how-to-add-a-tool).

### `agents/tools/`
- **Purpose:** One module per integration (schemes, weather, mandi, NPSS, SATHI, grievances, search, glossary, geocoding).
- **Dependencies:** `httpx`, `pydantic`, `langfuse.observe`, `helpers.langfuse_tracing`, `agents.deps.FarmerContext`, `app.config.DEFAULT_HTTP_TIMEOUT`, Redis `cache` for some (`npss.py`, `sathi_seed.py`, `shc_scheme_status.py`).
- **Important files:** `__init__.py` (registry), `search.py` (Marqo + Qdrant `search_schemes`), `scheme_info.py`, `npss.py`, `mandi.py`, `weather.py`.
- **Typical modifications:** payload changes for Beckn BAP calls (`BAP_*`/`BPP_*` env vars), new scheme codes, retry/`ModelRetry` messages. See [Third-party integrations](integrations/third-party-integrations.md).

### `app/`
- **Purpose:** HTTP layer (routers), orchestration (services), config, auth, Redis cache, image storage.
- **Dependencies:** `fastapi`, `aiocache`, `PyJWT`, `cryptography`, `agents/*`, `helpers/*`.
- **Important files:** `config.py`, `services/chat.py` (887 lines, largest orchestration file), `routers/token.py` (595 lines, three auth flows), `utils.py`.
- **Typical modifications:** new endpoint in `routers/` plus registration in `main.py`; auth changes in `routers/token.py` and `auth/jwt_auth.py`; routing weights in `services/agrinet_routing.py`.

### `helpers/`
- **Purpose:** Cross-cutting utilities: Langfuse client/scores/trace naming, telemetry event builders, Bhashini transcription/TTS/translation, Qdrant scheme/video search, master catalog (Redis snapshot reader), prompt rendering (`utils.get_prompt`).
- **Dependencies:** `langfuse`, `qdrant-client`, `sentence-transformers`, `redis`, `httpx`, `boto3`, `jinja2`.
- **Important files:** `utils.py`, `telemetry.py` (918 lines), `scheme_qdrant_search.py` (840 lines), `master_catalog.py`.
- **Typical modifications:** telemetry event shapes, Qdrant search tuning, Langfuse metadata.

### `assets/`
- **Purpose:** Data and prompt content loaded at runtime (relative paths such as `assets/prompts` and `assets/glossary_terms.json`, so the process working directory must be the repo root, `/app` in the container).
- **Typical modifications:** prompt edits per language; glossary additions (large JSON diffs); commodity codes updates.

### `tests/`
- **Purpose:** pytest unit tests plus manual live-LLM runner scripts and committed result artifacts.
- **Important files:** see [Testing](architecture/testing.md).

### `scripts/`
- **Purpose:** Operational one-off tooling (`audit_npss_empty_villages.py`, writes to `.local-dev-logs/`).
- **Dependencies:** `httpx`, `python-dotenv`, NPSS credentials.

### `docs/`
- **Purpose:** Setup notes (`docs/setup/`), archived business document (`docs/data/archive/`), and this wiki (`docs/wiki/bharat-oan-api/`).

### `.cursor/`
- **Purpose:** Cursor IDE assets (`rules/backend-*.mdc`, `agents/`, `skills/llm-wiki`). Currently untracked in git.

## Source Files

`main.py`, `Dockerfile`, `docker-compose.yml`, `supervisord.conf`, `app/config.py`, `agents/tools/__init__.py`, `app/routers/__init__.py`, `.github/workflows/build.yml`, `.gitignore`, `.dockerignore`.
