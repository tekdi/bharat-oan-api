# Technical Debt

Single register for technical debt, bugs, limitations and architectural risks of `bharat-oan-api`. All entries were traced to source files at wiki generation time. Priorities: P0 critical (act now), P1 high, P2 medium, P3 low.

Related: [AI Context](../ai-context.md) | [Architecture Overview](../architecture/architecture-overview.md) | [Testing](../architecture/testing.md) | [Legacy Artifacts](legacy-artifacts.md) | [API Specification](../application/api-specification.md)

## Summary Register

### P0 — Critical

| ID | Debt | Impact | Remediation |
|----|------|--------|-------------|
| [TD-01](#td-01--administrator-credential-and-host-details-committed-in-docs) | Administrator credential and host details committed in docs | Credential exposure | Rotate, remove from repo and history |

### P1 — High

| ID | Debt | Impact | Remediation |
|----|------|--------|-------------|
| [TD-02](#td-02--open-guest-token-endpoint-and-no-authorization-model) | Open guest token endpoint, no authorization model | Auth is bypassable; caller-chosen claims | Gate token issuance, enforce roles, apply rate limits |
| [TD-03](#td-03--npss-location-always-falls-back-to-the-default) | NPSS location always falls back to the default | Wrong location for every image diagnosis | Pass coordinates into `FarmerContext` |
| [TD-04](#td-04--blocking-synchronous-calls-in-async-handlers) | Blocking synchronous calls in async handlers | Event-loop stalls up to 30-60 s | Use async clients or thread offload |
| [TD-05](#td-05--uploaded-images-never-cleaned-up-and-stored-on-local-disk) | Uploaded images never cleaned up, local disk | Disk growth; breaks with several replicas | Schedule cleanup, use shared storage |
| [TD-06](#td-06--moderation-is-prompt-only-and-fails-the-whole-turn-on-error) | Moderation is prompt-only and fails the turn on error | Weak guardrail, outage amplification | Code-level gate and fallback |

### P2 — Medium

| ID | Debt | Impact | Remediation |
|----|------|--------|-------------|
| [TD-07](#td-07--suggestions-feature-is-disabled-and-broken) | Suggestions feature disabled and broken | Fails at import if re-enabled | Repair or delete |
| [TD-08](#td-08--farmer-data-written-to-logs) | Farmer data written to logs | Privacy risk | Mask or drop payload logging |
| [TD-09](#td-09--prompt-drift-across-languages-and-hard-coded-dates) | Prompt drift and hard-coded dates | Inconsistent answers by language | Shared template, config for dates |
| [TD-10](#td-10--target_lang-is-not-validated) | `target_lang` not validated | 500 on unsupported language | Validate and return 400 |
| [TD-11](#td-11--bap_endpoint-handling-is-inconsistent) | `BAP_ENDPOINT` handling inconsistent | `AttributeError` in four tools | Central helper |
| [TD-12](#td-12--api_documentationmd-is-out-of-date) | `API_DOCUMENTATION.md` out of date | Clients misled | Regenerate from OpenAPI |
| [TD-13](#td-13--model-defaults-and-route-labels-do-not-match-reality) | Model default and route labels misleading | Wrong model in unset env | Require explicit names |
| [TD-14](#td-14--no-automated-test-gate-and-stale-tests) | No automated test gate, stale tests | Regressions unnoticed | CI job, repair tests |
| [TD-15](#td-15--telemetry-lost-on-failure-or-client-disconnect) | Telemetry lost on failure or disconnect | Incomplete analytics | Emit in `finally` without re-raise loss |
| [TD-16](#td-16--redis-outage-fails-chat-and-nonce-check-is-not-atomic) | Redis outage fails chat; nonce check non-atomic | Availability, replay risk | Guard reads, use `SET NX` |
| [TD-17](#td-17--analyze-image-skips-content-type-validation) | `analyze-image` skips content-type check | Unvalidated uploads | Reuse upload validation |
| [TD-18](#td-18--chat-stream-content-type-and-contract-are-misleading) | Stream content type and contract misleading | Client confusion | Align header or implement SSE |
| [TD-21](#td-21--unpinned-dependencies-and-no-lock-file) | Unpinned dependencies, no lock file | Non-reproducible builds | Lock file |
| [TD-23](#td-23--container-hardening-gaps) | Container runs as root, no healthcheck, secrets may be baked in | Security, ops | Non-root user, `.dockerignore`, `HEALTHCHECK` |
| [TD-24](#td-24--compose-bind-mount-and-documentation-naming-mismatch) | Compose bind-mount and naming mismatches | Confusing deploys | Align docs and compose |
| [TD-25](#td-25--configuration-sprawl) | Configuration sprawl | Hard to audit config | Centralise in `Settings` |

### P3 — Low

| ID | Debt | Impact | Remediation |
|----|------|--------|-------------|
| [TD-19](#td-19--dead-code-unused-settings-and-unused-dependencies) | Dead code, unused settings and dependencies | Noise, image size | Remove |
| [TD-20](#td-20--hard-coded-and-missing-configuration) | Hard-coded and missing configuration | Environment leaks | Move to env |
| [TD-22](#td-22--small-inconsistencies) | Small inconsistencies | Confusion | Fix individually |

## Detailed Register

### TD-01 — Administrator credential and host details committed in docs

**Priority:** P0
**Location:** `docs/setup/portainer.md`
The setup document contains an access URL and an administrator credential in clear text. The values are intentionally not repeated in this wiki.
**Risk:** anyone with repository access can reach the management console if it is still live.
**Remediation:** rotate the credential, remove it from the file, rewrite git history if the repository is shared widely, and keep host details in a secret store.

### TD-02 — Open guest token endpoint and no authorization model

**Priority:** P1
**Location:** `app/routers/token.py`, `app/auth/jwt_auth.py`, `app/config.py`, `main.py`
`POST /api/token` issues a valid RS256 JWT with no credential and lets the caller set `mobile`, `role` and `metadata`. The `role` claim is never checked and telemetry trusts the claimed `mobile`. `GET /api/image/{id}` and `GET /api/file/{hash}` need no token. CORS defaults to `*` with credentials allowed and origins are not trimmed. `rate_limit_requests_per_minute` is configured but never applied. A missing `mobile` claim logs a warning but the raise is commented out. Any `session_id` is readable by anyone who knows it.
**Remediation:** decide the guest-access policy, bind claims to verified identity, add rate limiting, restrict CORS, and protect file and image routes. See [Users and Personas](../business/users-and-personas.md#2-access-permissions-verified-in-code).

### TD-03 — NPSS location always falls back to the default

**Priority:** P1
**Location:** `app/services/chat.py`, `agents/tools/npss.py` (`_resolve_effective_location`), `app/core/npss_followup.py`
`FarmerContext` is built without latitude and longitude, but the NPSS tool reads `deps.latitude`. Image diagnosis therefore uses the KVK Delhi fallback. The `[NPSS_LOCATION_REQUIRED]` follow-up path is dormant because the tool no longer emits the marker. The `qid`, `latitude` and `longitude` chat parameters are also undocumented.
**Remediation:** pass request coordinates into `FarmerContext`, add a test, remove the dormant marker code or restore it.

### TD-04 — Blocking synchronous calls in async handlers

**Priority:** P1
**Location:** `app/routers/transcribe.py`, `app/routers/tts.py`, `app/services/npss_response.py`, `helpers/transcription.py`, `helpers/translation.py`, `helpers/tts.py`
Bhashini, Whisper, TTS and translation use synchronous HTTP on the event loop with timeouts up to 30-60 s, stalling other requests on the same worker.
**Remediation:** use `httpx.AsyncClient` or `run_in_threadpool`.

### TD-05 — Uploaded images never cleaned up and stored on local disk

**Priority:** P1
**Location:** `app/core/image_storage.py`, `app/routers/image.py`
`cleanup_expired_images()` has no caller. Unanalysed uploads persist, `GET /api/image/{id}` ignores the TTL, the in-memory registry is unbounded, and files live on local disk so a multi-replica deployment breaks the NPSS image download.
**Remediation:** schedule cleanup, enforce TTL on read, move to shared storage.

### TD-06 — Moderation is prompt-only and fails the whole turn on error

**Priority:** P1
**Location:** `app/services/chat.py`, `agents/moderation.py`
The main agent still runs for refused categories. A moderation LLM error fails the turn with no fallback. `agents/moderation.py` has a `TODO` about adding scheme tools.
**Remediation:** gate refused categories in code, add a fallback path, define timeouts policy.

### TD-07 — Suggestions feature is disabled and broken

**Priority:** P2
**Location:** `agents/suggestions.py`, `app/tasks/suggestions.py`, `app/routers/suggestions.py`
The router and task are not mounted. `agents/suggestions.py` imports `LLM_AGRINET_MODEL`, which `agents/models.py` does not define, and `trim_history` is called with removed keyword arguments. Re-enabling fails at import.
**Remediation:** repair or delete the feature, including `assets/prompts/suggestions_system.md` and the `SUGGESTIONS_CACHE_TTL` setting.

### TD-08 — Farmer data written to logs

**Priority:** P2
**Location:** `agents/tools/pmkisan_scheme_status.py`, `aif.py`, `shc_scheme_status.py`, `pmfby_scheme_status.py`, `scheme_info.py`
Full request and response bodies, including phone numbers, registration numbers and statuses, are logged. Only the PMFBY tool masks selected fields and only the AIF tool masks the OTP. `app/routers/chat.py` logs the user query and mobile number at INFO.
**Remediation:** a shared masking helper and a policy not to log payloads at INFO.

### TD-09 — Prompt drift across languages and hard-coded dates

**Priority:** P2
**Location:** `assets/prompts/agrinet_*.md`, `helpers/pmkisan_installment_release.py`
Only `en`, `mai`, `or`, `pa` render the live vector-scheme list; nine prompts carry a static list. `last_weekday_table` is passed to every prompt but used in none. The PM-KISAN 23rd instalment rule is hard-coded to June 2026. `assets/prompts_backup_20260922_164719/` is a stale copy.
**Remediation:** shared base template, configurable dates, extend `tests/test_languages.py`.

### TD-10 — `target_lang` is not validated

**Priority:** P2
**Location:** `app/models/requests.py`, `helpers/utils.py` (`get_prompt`)
An unsupported value raises a template-not-found error while rendering the prompt, surfacing as a server error.
**Remediation:** validate against `SUPPORTED_LANGS` and return 400.

### TD-11 — `BAP_ENDPOINT` handling is inconsistent

**Priority:** P2
**Location:** `agents/tools/scheme_info.py`, `pmkisan_scheme_status.py`, `pmfby_scheme_status.py`, `shc_scheme_status.py`
They call `os.getenv("BAP_ENDPOINT").rstrip(...)` and raise `AttributeError` when unset, while other tools return a friendly message. `maha_vistaar.py` still returns "Set MH_BPP_ID" though that check is commented out.
**Remediation:** one helper that validates configuration at startup.

### TD-12 — `API_DOCUMENTATION.md` is out of date

**Priority:** P2
**Location:** `API_DOCUMENTATION.md`
It documents an unmounted suggestions endpoint, JWT "disabled", 30-day tokens, a default `mobile`, `eleven_labs` TTS, rate limiting that is not enforced, and omits `/chat/analyze-image`, `/image/*`, `/token/api-key`, `/token/play-integrity` and `/telemetry/*`. Full table in [API Specification](../application/api-specification.md#reconciliation-with-api_documentationmd).
**Remediation:** replace with a link to `/openapi.json` or this wiki.

### TD-13 — Model defaults and route labels do not match reality

**Priority:** P2
**Location:** `agents/models.py`, `app/services/agrinet_routing.py`
With `LLM_PROVIDER=openai` and no model name set, the code falls back to `gpt-3.5-turbo`. The route label `gpt41` is not enforced against the configured model.
**Remediation:** require explicit model names and rename or validate labels.

### TD-14 — No automated test gate and stale tests

**Priority:** P2
**Location:** `tests/`, `.github/workflows/build.yml`
No CI test job, no pytest configuration, `pytest` not in `requirements.txt`. `tests/test_chat_stream_contract.py` references helpers that no longer exist. Five live-LLM scripts are named `test_*`/`check_*` and break bare `pytest tests/` collection. 12 `results_*.json` files are committed. Nothing covers the tools, token flows or health router. See [Testing](../architecture/testing.md).
**Remediation:** see the recommended setup there.

### TD-15 — Telemetry lost on failure or client disconnect

**Priority:** P2
**Location:** `app/routers/transcribe.py`, `app/routers/tts.py`, `app/services/chat.py`, `helpers/telemetry.py`
Handlers re-raise, so failed-call telemetry built in `finally` is never delivered; the chat question event is lost on client disconnect. `helpers/telemetry.py` carries a TODO and a commented Celery fragment.
**Remediation:** emit events before raising or via a detached task.

### TD-16 — Redis outage fails chat and nonce check is not atomic

**Priority:** P2
**Location:** `app/utils.py`, `app/routers/chat_query_utils.py`, `app/routers/token.py`
The history read is unguarded, so a Redis outage fails chat. The Play Integrity nonce check is a get-then-set sequence.
**Remediation:** degrade to empty history with a warning; use `SET NX EX` for nonces.

### TD-17 — `analyze-image` skips content-type validation

**Priority:** P2
**Location:** `app/routers/chat.py` (`chat_analyze_image_endpoint`)
Unlike `/api/image/upload`, it does not validate the mimetype.
**Remediation:** reuse the upload validation.

### TD-18 — Chat stream content type and contract are misleading

**Priority:** P2
**Location:** `app/routers/chat.py`
The response media type is `text/event-stream` but the body is raw text, not `data:` frames. There are no keep-alives and no `session_id` is returned to the client when the server generates it.
**Remediation:** either implement SSE framing or declare `text/plain`; return the session id in a header.

### TD-19 — Dead code, unused settings and unused dependencies

**Priority:** P3
**Location:** `app/services/chat.py` (`_run_agrinet_with_failover`, `_run_agrinet_once`), `helpers/encryption.py`, `helpers/translation.py` (`GoogleTranslator`), `helpers/telemetry.py` (`create_moderation_event`, `create_translation_event`), `helpers/scheme_qdrant_search.py` (`search_schemes` has no caller), `app/config.py` (`secret_key`, `disable_jwt`, `rate_limit_requests_per_minute`, `uvicorn_workers`, `suggestions_cache_ttl`, `redis_retry_on_timeout`, AWS keys), `app/models/responses.py`, `requirements.txt` (`boto3` and packages with no importer such as `gunicorn`, `uvicorn-worker`, `gevent`, `pandas`, `openpyxl`, `httpx_oauth`, `humanize`, `pycryptodome`, `asgiref`; keep `google-cloud-translate` because it provides `google-auth` for `app/routers/token.py`), `app/tasks/logging.py`, `assets/new_marqo_settings.json`, `.gitignore` gaps (`.oan-uploads/`, `.local-dev-logs/`)
See [Legacy Artifacts](legacy-artifacts.md).
**Remediation:** remove after confirming no external use.

### TD-20 — Hard-coded and missing configuration

**Priority:** P3
**Location:** `agents/tools/maps.py` (fallback to an internal host and port), `app/config.py` (hard-coded dev telemetry URL), `app/routers/token.py` and `app/config.py` (placeholder `secret_key`), `supervisord.conf` (`--workers 8`), `helpers/translation.py` (expects `assets/word_mapping_reduced_1000.json`, which does not exist)
**Remediation:** move to environment variables or remove.

### TD-21 — Unpinned dependencies and no lock file

**Priority:** P2
**Location:** `requirements.txt`, `Dockerfile`
Most dependencies are unpinned; images are heavy (CPU `torch`, `sentence-transformers`, a baked embedding model) and CI builds pass no `HF_TOKEN`.
**Remediation:** pip-compile or uv lock, pin by hash, split dev dependencies.

### TD-23 — Container hardening gaps

**Priority:** P2
**Location:** `Dockerfile`, `supervisord.conf`, `.dockerignore`
The container runs as root (non-root user lines are commented out) and defines no `HEALTHCHECK`. `.env`, `*.pem` and credential JSON files are not listed in `.dockerignore`, so `COPY . .` can bake them into the image. `HF_TOKEN` is persisted as an `ENV`.
**Remediation:** run as a non-root user, extend `.dockerignore`, pass the token only as a build secret, add a healthcheck on `/api/health/live`.

### TD-24 — Compose bind-mount and documentation naming mismatch

**Priority:** P2
**Location:** `docker-compose.yml`, `README.md`, `docs/setup/redis.md`
Compose bind-mounts `.:/app`, so the host checkout overrides the image code. `README.md` uses `oan-network` and `oan_app` while compose uses `svanetwork` and `sva_app`. The Redis setup rationale refers to an earlier Django, Gunicorn and Celery stack.
**Remediation:** remove the bind-mount outside development and align the documents with compose.

### TD-25 — Configuration sprawl

**Priority:** P2
**Location:** `app/config.py`, `agents/`, `helpers/`, `.env.example`
Many `os.getenv` reads sit outside `Settings`, many `Settings` fields are never read, and `.env.example` documents only the NPSS and Photon variables. See [Configuration](../architecture/configuration.md).
**Remediation:** one validated `Settings` object and a complete `.env.example`.

### TD-22 — Small inconsistencies

**Priority:** P3
**Location:** `app/routers/health.py` (`latency_ms` always 0, TODO), `app/routers/suggestions.py` (`SUGGESTIONS_CACHE_TTL` 3600 vs "30 minutes" comment), `Dockerfile` (comment mentions Celery), `README.md` (`oan_app` vs compose `sva_app`), `app/core/image_storage.py` (unbounded in-memory registry)
**Remediation:** fix individually.

## Remediation Roadmap

| Priority | Items | Typical effort |
|----------|-------|----------------|
| P0 | TD-01 | Hours |
| P1 | TD-02 to TD-06 | Days each; TD-02 needs a product decision |
| P2 | TD-07 to TD-18, TD-21, TD-23 to TD-25 | Hours to days each |
| P3 | TD-19, TD-20, TD-22 | Hours |

## Related Documentation

[Testing](../architecture/testing.md), [Configuration](../architecture/configuration.md), [Third-Party Integrations](../integrations/third-party-integrations.md), [Operations: Monitoring](../operations/monitoring.md), [Legacy Artifacts](legacy-artifacts.md).

## Source Files

`app/`, `agents/`, `helpers/`, `tests/`, `docs/setup/portainer.md`, `requirements.txt`, `Dockerfile`, `supervisord.conf`, `.github/workflows/build.yml`.
