# Testing

Related: [Repository Map](../repository-map.md) | [Coding Patterns](coding-patterns.md) | [Deployment](../operations/deployment.md) | [Legacy Artifacts](../knowledge/legacy-artifacts.md) | [Technical Debt](../knowledge/technical-debt.md)

## Current State (verified in repository)

| Attribute | Status |
|-----------|--------|
| Test directory | Yes, `tests/` (with `tests/__init__.py`, `tests/fixtures/`) |
| Framework | `pytest` (used via `import pytest`, `monkeypatch`, `tmp_path`, `fastapi.testclient.TestClient`). **Not listed in `requirements.txt`** |
| Pytest configuration (`pytest.ini`, `pyproject.toml`, `setup.cfg`, `tox.ini`) | **Not present** |
| `conftest.py` / shared fixtures module | **Not present** |
| Coverage config (`.coveragerc`) or coverage tooling | **Not present** (`.gitignore` ignores `.coverage`, `htmlcov/` only) |
| CI test job | **Not present** — `.github/workflows/build.yml` only builds and pushes a Docker image |
| Pre-commit hooks / linters (ruff, flake8, mypy config) | **Not present** (`.gitignore` ignores `.ruff_cache/`, `.mypy_cache/` only) |
| Unit tests (offline, mocked) | Yes — 6 files (see below) |
| Live-LLM manual runners | Yes — 5 scripts that call real models and tools |
| Committed result artifacts | Yes — 12 `tests/results_*.json` files |
| Test of agents/tools against live Beckn network | **Not present** as automated tests |

> **Note:** There is no automated test gate in this repository today. Tests are run by developers locally, and the only GitHub Actions workflow builds the container image without running them. Quality of the LLM answers is evaluated with the manual runner scripts described below plus Langfuse traces (see [Monitoring](../operations/monitoring.md)).

### Offline pytest suites

| File | Tests | What it covers |
|------|-------|----------------|
| `tests/test_languages.py` | 4 | Every `assets/prompts/agrinet_<lang>.md` language is wired into `SUPPORTED_LANGS` (`agents/tools/terms.py`) and NPSS `LABELS` / `SOURCE_LINES` / `LANGUAGE_SCRIPT_RANGES` (`app/services/npss_response.py`); `or`/`pa`/`mai` prompts keep the same Jinja tokens as `agrinet_en.md`; every language prompt renders via `helpers.utils.get_prompt`; `TermPair.get_term` round-trips `or`, `as`, `pa`, `mai`. Uses a relative `Path("assets/prompts")`, so run from the repo root. |
| `tests/test_chat_query_utils.py` | 2 | `extract_image_id` and `normalize_chat_query` (embedded UUID becomes `[IMAGE_URL: .../api/image/<id>]`) in `app/routers/chat_query_utils.py`. |
| `tests/test_npss_geocodes.py` | 18 | NPSS location resolution in `app/core/npss_geocodes.py` (HTTPS-only `PHOTON_HOST`, forward/reverse geocoding, browser coordinates priority, KVK fallback, zero-village urban hierarchy), pending-image discovery in `app/core/npss_followup.py`, image wrapping in `app/services/chat.py`, and `agents/tools/npss.py` behaviour. Uses `monkeypatch` to stub master-data and Photon calls. |
| `tests/test_audit_npss_empty_villages.py` | 4 (one parametrized) | `_validated_output_dir` in `scripts/audit_npss_empty_villages.py` rejects traversal, absolute paths outside the report root, and symlink escapes. |
| `tests/test_backend_owned_telemetry.py` | 6 | Telemetry identity resolution and event builders in `helpers/telemetry.py`, the `/api/telemetry/events` route (via `TestClient`, `send_telemetry` monkeypatched), and guest token fingerprint context from `app/routers/token.py`. |
| `tests/test_chat_stream_contract.py` | 2 | Intended to cover the SSE keepalive helper and the `X-QID` response header of `GET /api/chat/`. **Appears stale:** it references `chat_service._await_with_sse_keepalives`, `chat_service.SSE_KEEPALIVE` and `chat_router._get_message_history`, none of which exist in `app/services/chat.py` or `app/routers/chat.py` (verified by grep), and it expects the server to ignore a client-supplied `qid` whereas `app/routers/chat.py` uses `request.qid or str(uuid.uuid4())`. Expect failures until it is updated. |

### Manual live-LLM runners (network and credential dependent)

These files live in `tests/`, import the production agents, and call real LLM endpoints and real tools. They are scripts with a `main()` and argparse, not assertion-based pytest tests.

| File | Purpose | Fixture | Key flags |
|------|---------|---------|-----------|
| `tests/test_agrinet_single_turn.py` | Runs questions through moderation then `agrinet_agent`; reports latency, errors, tool usage per category | `tests/fixtures/agrinet_single_turn.json` | `--sample`, `--concurrency`, `--out`, `--category` |
| `tests/test_moderation_single_turn.py` | Compares `moderation_agent` category against expected | `tests/fixtures/moderation_single_turn.json` | `--sample`, `--concurrency`, `--out` |
| `tests/test_moderation_multi_turn.py` | Same with conversation history formatted like `app/services/chat.py` | `tests/fixtures/moderation_multi_turn.json` | same |
| `tests/check_json_in_responses.py` | Detects JSON, tool-parameter keys, `<think>` traces, code blocks leaking into answers | `tests/fixtures/agrinet_single_turn.json` | `--sample` |
| `tests/check_streaming_leakage.py` | Mirrors the production streaming path and checks every streamed chunk for leakage | `tests/fixtures/agrinet_single_turn.json` | `--sample`, `--concurrency` |

Requirements for these runners (from imports): a populated `.env` (LLM provider keys, `BAP_*`, Marqo/Qdrant, etc.), network access to those services, and the `logfire` package, which is imported at the top of each file (`logfire.configure(scrubbing=False)`) but is commented out in `requirements.txt` (see [Legacy Artifacts](../knowledge/legacy-artifacts.md)).

### Fixtures and committed result artifacts

| Path | Kind | Notes |
|------|------|-------|
| `tests/fixtures/agrinet_single_turn.json` | Input questions (`question`, `category`, `language`, `tools`) | Used by agrinet runner and both leakage checkers |
| `tests/fixtures/moderation_single_turn.json`, `moderation_multi_turn.json` | Input with expected categories | Moderation runners |
| `tests/fixtures/review_sample.json` | Small sample (question, category, language) | Not referenced by any script in the repo |
| `tests/results_agrinet_sample.json`, `results_single_turn.json`, `results_multi_turn.json`, `results_final.json`, `results_after_changes.json`, `results_review.json`, `results_full_review.json`, `results_json_check.json`, `results_json_check_full.json`, `results_streaming_check.json`, `results_streaming_high.json`, `results_streaming_quick.json` | **Committed output of past manual runs** | Added 2026-02-09/10 (git history). Roughly 560 KB in total. They are snapshots, not test inputs, and are not read by any code. |

> **Note:** The `results_*.json` files are historical run outputs kept in version control. They are useful as examples of the runner output shape (`question`, `category`, `language`, `tools`, `response`, `time_s`, `error`) but do not represent current model behaviour.

## Commands

The commands below are derived from the runner docstrings and the test imports. They were not executed during wiki generation (the local environment used for documentation did not have the project dependencies installed), so treat them as the documented intent.

```bash
# From the repository root, with dependencies installed and a valid .env
python -m pip install -r requirements.txt pytest

# Offline unit tests, run selectively
pytest tests/test_languages.py tests/test_chat_query_utils.py \
       tests/test_npss_geocodes.py tests/test_audit_npss_empty_villages.py \
       tests/test_backend_owned_telemetry.py

# Manual live-LLM runners (docstrings use .venv/bin/python3 -m ...)
python -m tests.test_agrinet_single_turn --sample 10 --concurrency 4 --out results.json
python -m tests.test_moderation_single_turn --sample 10
python -m tests.test_moderation_multi_turn --sample 5
python -m tests.check_json_in_responses --sample 20
python -m tests.check_streaming_leakage --sample 10 --concurrency 8
```

Environment caveats verified from source:

- `app/auth/jwt_auth.py` reads `jwt_public_key.pem` at import time, so any test importing it (`test_chat_stream_contract.py`, `test_backend_owned_telemetry.py`) needs that file present (see [Configuration](configuration.md)).
- Importing `app/services/chat.py` or `agents.*` imports `agents/models.py`, which builds LLM clients at import and raises for missing provider variables (for example `VLLM_AGRINET_MODEL_URL`, `AZURE_OPENAI_*`). Set `LLM_PROVIDER` and related variables before running tests that import them.
- A bare `pytest tests/` also imports the manual runner modules during collection; because they call `logfire.configure(...)` and load production agents at import time, collection may fail without `logfire` and a full environment. Select files explicitly instead.

## Recommended improvements (not yet implemented)

- Add `pytest` (and `pytest-asyncio` if async tests are introduced) to a `requirements-dev.txt`, and a `pytest.ini` with `testpaths = tests` and a marker such as `live` for runner scripts.
- Move the manual runners out of `test_*.py` names (for example into `scripts/eval/`) so pytest does not collect them; stop committing `results_*.json` outputs (write to an ignored directory instead).
- Repair `tests/test_chat_stream_contract.py` to match the current `chat_router` and `chat_service` API.
- Add a CI job to `.github/workflows/build.yml` (or a new workflow) running the offline suite on pull requests, with a placeholder `jwt_public_key.pem` and stubbed LLM environment.
- Add mocked tests for the auth flows in `app/routers/token.py`, the health router, `app/services/agrinet_routing.py` weighting and capacity gate, and for tool payload builders in `agents/tools/`.
- Add coverage reporting (`pytest-cov`) with an agreed baseline.

## Source Files

`tests/*.py`, `tests/fixtures/*.json`, `tests/results_*.json`, `scripts/audit_npss_empty_villages.py`, `requirements.txt`, `.github/workflows/build.yml`, `.gitignore`.
