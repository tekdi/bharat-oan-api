# Legacy Artifacts

Dead, disabled, commented-out, or committed-output items found in the repository. Each item was checked with repository-wide searches for imports and references (state at documentation time). Items are documented here rather than removed; prioritised remediation lives in [Technical Debt](technical-debt.md).

Related: [Repository Map](../repository-map.md) | [Testing](../architecture/testing.md) | [References](references.md) | [Coding Patterns](../architecture/coding-patterns.md)

## Disabled suggestions feature

| Artifact | State |
|----------|-------|
| `app/routers/suggestions.py` (`GET /api/suggest/`) | Not registered: commented out in `main.py` and `app/routers/__init__.py` ("suggestion agent disabled") |
| `app/tasks/suggestions.py` (`create_suggestions`) | Import commented out in `app/tasks/__init__.py`; calls `trim_history(..., include_tool_calls=..., include_system_prompts=...)`, parameters that `app/utils.py::trim_history` does not accept (they are commented out in its signature) |
| `agents/suggestions.py` (`suggestions_agent`) | Imports `LLM_AGRINET_MODEL` from `agents/models.py`, a name that does not exist there; importing the module would raise `ImportError` |
| `assets/prompts/suggestions_system.md` | Prompt for the disabled agent |
| `app/config.py` `suggestions_cache_ttl`, `app/models/requests.py` `SuggestionsRequest`, `app/models/responses.py` `SuggestionsResponse` | Remain for the disabled feature |

Re-enabling the feature requires fixing the model import and the `trim_history` call first.

## Unused modules and functions

| Artifact | Evidence of non-use |
|----------|---------------------|
| `helpers/encryption.py` (AES-GCM helpers for PM-KISAN grievance) | No importer anywhere |
| `app/tasks/logging.py` | Defines a logger only; no importer |
| `helpers/utils.py`: `is_sentence_complete`, `split_text`, `load_json_data` | Defined, never called |
| `app/utils.py`: `filter_out_tool_calls`, `get_message_pairs` (used only through `format_message_pairs`), `group_convos` (used only by `trim_history`) | `filter_out_tool_calls` has no caller |
| `helpers/translation.py`: `GoogleTranslator` | Only `BhashiniTranslator` is used (`app/services/npss_response.py`); `GoogleTranslator` expects `cloud-translate-credentials.json` (gitignored via `*credentials.json`) |
| `helpers/transcription.py` commented block `detect_audio_language_bhashini` | Marked in-file "Unused: not referenced anywhere in the codebase" |
| `helpers/inject_pdf_header.py` | Used by `agents/tools/shc_scheme_status.py`; its docstring suggests running it as `scripts/inject_pdf_header.py`, a path that does not exist |
| `assets/new_marqo_settings.json` | Marqo index settings; referenced by no code or doc |
| `agents/tools/__init__.py` comment `# from agents.tools.search_beckn import search_documents` | Stale import comment |

## Dependencies declared but not imported

Searches for imports found no use of these `requirements.txt` entries (some may be pulled in transitively by other packages, so removal needs a clean-environment check):

| Package | Notes |
|---------|-------|
| `gunicorn`, `uvicorn-worker` | Runtime is plain `uvicorn --workers 8` under supervisord (`supervisord.conf`) |
| `gevent` | No import; Gunicorn/gevent heritage |
| `hiredis-py` | Only a comment ("not sure if needed") |
| `httpx_oauth`, `pandas`, `openpyxl`, `python-decouple`, `humanize`, `pycryptodome`, `asgiref` | No imports found (`python-dotenv` is the loader in use) |
| `boto3` | `import boto3` at the top of `helpers/utils.py` but never used; `aws_*` settings are unused too |
| `transformers`, `huggingface-hub` | Not imported directly; arrive with `sentence-transformers` |
| `google-cloud-translate` | Used only by the unused `GoogleTranslator`, but it also provides `google-auth`, which `app/routers/token.py` imports; do not remove without replacing that dependency |

Not unused (checked): `marqo` is imported by `agents/tools/search.py` (`search_documents`, `search_pests_diseases`), `markdownify` by `agents/tools/shc_scheme_status.py`, `rapidfuzz` by `agents/tools/terms.py` and `commodity.py`, `tenacity` by `app/tasks/telemetry.py` and `helpers/transcription.py`. `logfire` is commented out of `requirements.txt` but imported by the manual runners in `tests/` (see below).

## Logfire remnants

- `requirements.txt`: `# Logfire` / `# logfire` commented out.
- `app/config.py`: `logfire_token` setting never read.
- `tests/test_agrinet_single_turn.py`, `test_moderation_single_turn.py`, `test_moderation_multi_turn.py`, `check_json_in_responses.py`, `check_streaming_leakage.py`: `import logfire` and `logfire.configure(scrubbing=False)`.
- `agents/__init__.py` explicitly calls `Agent.instrument_all(False)` because Langfuse `@observe` replaced pydantic-ai instrumentation.

## Declared-but-unused settings and flags

`Settings` fields never read through `settings.<name>`: `secret_key`, `rate_limit_requests_per_minute` (no rate limiting exists), `uvicorn_workers`, `disable_jwt` (the JWT dependency has no bypass), `redis_retry_on_timeout`, `qdrant_collection_name`, AWS/Gemini/Mapbox/Sarvam/ElevenLabs/Ollama/inference fields (full list in [Configuration](../architecture/configuration.md)). `TTSRequest.service_type` accepts `eleven_labs`, but no ElevenLabs client code exists in the repository.

## Commented-out or stale code

| Location | Description |
|----------|-------------|
| `app/auth/jwt_auth.py` | `# raise credentials_exception` after the "No mobile number found in token" warning; tokens without `mobile` are accepted |
| `agents/tools/maha_vistaar.py` | Commented `MH_BPP_ID` configuration checks (commit `8be970b`) |
| `agents/moderation.py` | `# openai_reasoning_effort='low'` and `# TODO: Add tools from tools/scheme.py` |
| `app/utils.py` | Commented parameters in `trim_history` (`include_system_prompts`, `include_tool_calls`) |
| `Dockerfile` | Commented non-root user lines; comment mentions Celery, which is not part of the repo |
| `docs/setup/redis.md` | Rationale mentions Django cache, Gunicorn, Celery workers from an earlier stack |
| `helpers/transcription.py` | Request header `User-Agent: Thunder Client (https://www.thunderclient.com)` copied from an API testing tool |
| `README.md` | Network `oan-network` and container `oan_app` do not match `docker-compose.yml` (`svanetwork`, `sva_app`) |

## Committed result artifacts and fixtures

`tests/results_*.json` (12 files, about 560 KB, committed 2026-02-09/10) are outputs of past manual runs and are read by nothing. `tests/fixtures/review_sample.json` is referenced by no script. Details in [Testing](../architecture/testing.md). `tests/test_chat_stream_contract.py` targets functions that no longer exist.

## Backup and archive material

| Path | Description |
|------|-------------|
| `assets/prompts_backup_20260922_164719/` | Timestamped copy of ten language prompts plus moderation and suggestions prompts from before later prompt changes. It is listed in `.gitignore` but is tracked in git, and no code loads it |
| `docs/data/archive/CROP ADVISORY SOP.docx` | Archived Marathi-language Word document (a crop advisory SOP that also appears to include a contact directory with personal phone numbers). Not referenced by code; added in commit `8c90ef7` (codebase import) |
| `assets/glossary_terms.json` | Active data, not legacy; noted here because it is a 4 MB single-file diff hotspot (commit `e44d44a` changed ~25,700 lines) |

## Operational leftovers

- `docs/setup/portainer.md` contains a clear-text access URL and administrator credential (not reproduced here).
- Untracked local paths `.oan-uploads/` (default `TEMP_UPLOAD_DIR`) and `.local-dev-logs/` (audit script output) are not covered by `.gitignore` rules, so generated files could be committed by accident.
- Many remote branches (for example `bh-dev-bkp`, `main-bkp`, `feat/ffs_integration_v0` through `v8`, `revert-*`) remain as historical references; see [Repository Conventions](../architecture/repository-conventions.md).

## Source Files

`main.py`, `app/routers/__init__.py`, `app/tasks/*.py`, `agents/suggestions.py`, `agents/models.py`, `app/utils.py`, `helpers/utils.py`, `helpers/encryption.py`, `helpers/translation.py`, `helpers/transcription.py`, `requirements.txt`, `.gitignore`, `tests/`, `assets/`, `docs/`.
