# References

Internal documents, external endpoints referenced by the code, and the dependency list of `bharat-oan-api`.

Related: [Glossary](glossary.md) | [Legacy Artifacts](legacy-artifacts.md) | [Configuration](../architecture/configuration.md) | [Third-party integrations](../integrations/third-party-integrations.md) | [Repository Map](../repository-map.md)

## Internal documents in the repository

| Document | Path | Notes |
|----------|------|-------|
| README | `README.md` | Docker and Redis command cheat-sheet; names `oan-network` and `oan_app` that differ from compose (`svanetwork`, `sva_app`) |
| API notes | `API_DOCUMENTATION.md` | Hand-written endpoint overview (root, chat, transcribe, TTS, suggestions ...); partly outdated relative to the code (see [API Specification](../application/api-specification.md)) |
| Redis setup | `docs/setup/redis.md` | Redis Stack production configuration and commands |
| Portainer setup | `docs/setup/portainer.md` | Container UI install notes (contains a clear-text credential, see [Deployment](../operations/deployment.md)) |
| Archived SOP | `docs/data/archive/CROP ADVISORY SOP.docx` | Marathi-language Word document; not referenced by code (see [Legacy Artifacts](legacy-artifacts.md)) |
| License | `LICENSE` | MIT, Centre for Open Societal Systems (COSS), OpenAgriNet initiative |
| CI workflow | `.github/workflows/build.yml` | Image build and push |
| Cursor rules | `.cursor/rules/backend-1-architectural-patterns.mdc` ... `backend-8-testing-standards.mdc`, `common-products.md`, `tekdi-values-as-rules.md` | Generic engineering guidance (untracked in git) |
| Cursor skills and agents | `.cursor/skills/*`, `.cursor/agents/*` | Includes the `llm-wiki` skill used to generate this wiki |
| Prompts | `assets/prompts/` | Runtime LLM prompts (see [AI/LLM integration](../integrations/ai-llm.md)) |
| Wiki | `docs/wiki/bharat-oan-api/` | This documentation set; entry points [ai-context](../ai-context.md) and [README](../README.md) |

> **Note:** No `docs/raw_documents/` folder, architecture decision records, or product requirement documents are part of this repository, so no raw-document reconciliation was needed for this wiki.

## External endpoints and services referenced in code

| Reference | Where | Purpose |
|-----------|-------|---------|
| `https://npss.dac.gov.in/api3.0` | `.env.example`, `agents/tools/npss.py`, `app/core/npss_geocodes.py`, `scripts/audit_npss_empty_villages.py` | Default NPSS API base (`NPSS_BASE_URL`) |
| `https://dhruva-api.bhashini.gov.in/services/inference/pipeline` | `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py` | Bhashini / Dhruva speech and translation pipeline |
| `https://dev-vistaar.da.gov.in/observability-service/action/data/v3/telemetry` | `app/config.py` (`telemetry_api_url` default) | Telemetry sink default (dev host) |
| `https://playintegrity.googleapis.com/v1/{package_name}:decodeIntegrityToken` | `app/routers/token.py` | Google Play Integrity verdict decoding |
| `https://www.googleapis.com/auth/playintegrity`, `https://www.googleapis.com/auth/cloud-platform` | `app/routers/token.py`, `helpers/translation.py` | OAuth scopes |
| `https://download.pytorch.org/whl/cpu`, `https://pypi.org/simple` | `Dockerfile` | Package indexes used at build time |
| `<region>-docker.pkg.dev/<project>/<repo>` | `.github/workflows/build.yml` | GCP Artifact Registry image destination (values from GitHub secrets) |
| `intfloat/multilingual-e5-large` | `Dockerfile`, `helpers/scheme_qdrant_search.py` | HuggingFace embedding model baked into the image |
| `BAP_ENDPOINT`, `QDRANT_URL`, `MARQO_ENDPOINT_URL`, `PHOTON_HOST`, `SATHI_MASTER_BASE_URL`, `LANGFUSE_HOST`, `VLLM_AGRINET_MODEL_URL`, `AZURE_OPENAI_ENDPOINT` | environment variables | Deployment-specific endpoints, see [Configuration](../architecture/configuration.md) |

Documentation links used by this wiki: pydantic-ai <https://ai.pydantic.dev/>.

## Dependencies (`requirements.txt`)

Pinned versions are shown; everything else is unpinned (latest at build time). Usage was checked by searching for imports in the repository.

| Package | Version | Role | Used in code |
|---------|---------|------|--------------|
| `fastapi[standard]`, `starlette`, `uvicorn[standard]` | unpinned, unpinned, `==0.34.3` | Web framework and ASGI server | Yes (`main.py`, routers) |
| `pydantic`, `pydantic-settings` | unpinned, `==2.9.1` | Models and settings | Yes |
| `pydantic-ai` | `==1.2.1` | Agents and tools | Yes |
| `openai` | unpinned | `AsyncAzureOpenAI`, Whisper client | Yes (`agents/models.py`, `helpers/transcription.py`) |
| `httpx` | unpinned | HTTP client | Yes |
| `redis`, `aiocache` | unpinned, `==0.12.3` | Cache and Redis client | Yes |
| `PyJWT`, `cryptography` | unpinned | JWT and key loading, AES-GCM | Yes (`import jwt`, `app/auth/jwt_auth.py`) |
| `langfuse` | `>=3.0.0` | LLM tracing | Yes |
| `opentelemetry-api` | `>=1.28.0,<1.44` | Langfuse/OTel compatibility bound | Indirect |
| `qdrant-client`, `sentence-transformers` | unpinned | Vector search and embeddings | Yes (`helpers/scheme_qdrant_search.py`, `helpers/video_qdrant_search.py`) |
| `marqo` | unpinned | Marqo client | Yes (`agents/tools/search.py`) |
| `rapidfuzz` | unpinned | Fuzzy term matching | Yes (`agents/tools/terms.py`, `commodity.py`) |
| `tenacity` | unpinned | Retry | Yes (`app/tasks/telemetry.py`, `helpers/transcription.py`) |
| `Jinja2` | `>=3.1.0` | Prompt templating | Yes (`helpers/utils.py`) |
| `python-dotenv` | unpinned | `.env` loading | Yes |
| `langcodes[data]` | `==3.4.0` | Language display names | Yes |
| `boto3` | unpinned | AWS SDK | Imported in `helpers/utils.py`; no AWS call sites were found |
| `google-cloud-translate` | unpinned | Google Translate client | Imported by `helpers/translation.py` (`GoogleTranslator`, not used by active flows); `google.auth` is also used by `app/routers/token.py` |
| `simplejson`, `pytz`, `python-dateutil` | unpinned | JSON, timezone, date parsing | Yes |
| `markdownify` | unpinned | HTML to markdown | Yes (`agents/tools/shc_scheme_status.py`) |
| `transformers`, `huggingface-hub` | unpinned | Model hub support | Transitive via `sentence-transformers`; no direct import |
| `wheel`, `setuptools` | unpinned | Build tooling | n/a |
| `gunicorn`, `uvicorn-worker`, `gevent`, `httpx_oauth`, `pandas`, `openpyxl`, `python-decouple`, `humanize`, `pycryptodome`, `asgiref` | unpinned | Listed dependencies | No import found (see [Legacy Artifacts](legacy-artifacts.md)) |

`torch` is not in `requirements.txt`; the `Dockerfile` installs the CPU build separately. `logfire` and `hiredis-py` appear only as comments.

## Source Files

`README.md`, `API_DOCUMENTATION.md`, `docs/setup/*.md`, `requirements.txt`, `Dockerfile`, `.github/workflows/build.yml`, `app/config.py`, `app/routers/token.py`, `helpers/*.py`.
