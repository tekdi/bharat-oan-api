---
name: fullstack-developer
description: >
  End-to-end feature development skill, tailored to bharat-oan-api: a Python 3.13 FastAPI backend
  (`main.py`, routers under `app/routers/`, services under `app/services/`) that runs pydantic-ai
  agents (`agents/`) with tools in `agents/tools/`, backed by Redis (aiocache), Qdrant, Marqo and a
  Beckn/BAP gateway, with Langfuse tracing. There is no SQL database, no ORM and no frontend in this
  repo. Trigger this skill whenever the user asks to "add a feature", "add a route/endpoint/API",
  "add an agent tool", "add a language", "add a config option", "add a Redis-backed value", "change
  the chat/generation pipeline", or describes work that spans the router, service, agent/tool and
  client-contract layers of this repo. Follows *this codebase's* actual conventions (recorded in the
  phases below, sourced from `docs/wiki/bharat-oan-api/`) rather than generic framework advice. Does
  not apply to single-layer bug fixes with no route/tool/prompt surface — use targeted
  investigation for those. Also triggers on "update this skill to suit the tech stack of this
  codebase" — see **Tailoring this skill to a specific codebase** at the bottom.
---

# Full-Stack Development (bharat-oan-api)

A guided workflow for building a feature across this repo's actual stack: FastAPI routers ->
services -> pydantic-ai agents/tools and `helpers/` -> Redis / Qdrant / Marqo / external HTTP APIs,
consumed by clients that live outside this repository. There is **no relational DB/ORM, no
migrations, no frontend and no role-based access control** here (wiki:
`application/database.md`, `application/frontend.md`, `application/backend.md` section 5).

**Golden rule: ground every phase in what this repo actually does before writing code.** The wiki
at `docs/wiki/bharat-oan-api/` (start with `ai-context.md`) was derived from code at generation time;
`API_DOCUMENTATION.md` is known to be partly stale. Verify against code before changing behaviour.
Honour the wiki's **Note** callouts and "Recommended (not yet implemented)" labels: those describe
things that are *not* true of the code today, so do not build on them as if they were.

---

## Phase 0 — Stack facts (already discovered; re-verify only what your change touches)

- **Language/runtime**: Python 3.13 (`Dockerfile`: `python:3.13-slim`); `pip` + `requirements.txt`
  (mostly unpinned, no lock file — TD-21). `pytest` is not even listed in it.
- **Framework**: FastAPI + uvicorn (8 workers via `supervisord.conf`), pydantic v2,
  `pydantic-settings` (`app/config.py`: `Settings`/`settings`). Entry point `main.py`; routers are
  mounted with `app.include_router(<mod>.router, prefix=settings.api_prefix)` (`/api`).
- **Agents**: `pydantic-ai==1.2.1`. `agrinet_agent` (`agents/agrinet.py`) and `moderation_agent`
  (`agents/moderation.py`); models built at import in `agents/models.py` per `LLM_PROVIDER`
  (`openai` | `azure-openai` | `vllm`). Per-turn context is `FarmerContext` (`agents/deps.py`).
  `suggestions_agent` is disabled and broken (TD-07) — do not extend it.
- **Data stores** (no SQL): Redis via `aiocache` (`app/core/cache.py`, helpers `get_cache` /
  `set_cache` in `app/utils.py`, prefix `REDIS_KEY_PREFIX` default `sva-cache-`); Marqo and Qdrant
  (external, read-only from here); temp image files on local disk (`app/core/image_storage.py`).
- **Auth**: per-route FastAPI dependency `get_current_user` (`app/auth/jwt_auth.py`), RS256 JWT.
  Tokens are issued by `app/routers/token.py`. No role enforcement exists (see Phase 3).
- **External integrations**: Beckn/BAP gateway (`BAP_*` env vars) used by most tools; Bhashini
  (ASR/TTS/translation, `helpers/`); Photon geocoder; NPSS; SATHI master API; Langfuse; a telemetry
  service. See wiki `integrations/third-party-integrations.md`.
- **Frontend**: none. Clients are external; this repo defines only the HTTP contract (Phase 5).
- **Tests**: 6 offline pytest files + 5 live-LLM runner scripts in `tests/`; no CI test job
  (Phase 6).
- **Docs to read**: `docs/wiki/bharat-oan-api/ai-context.md` first, then only pages matching your
  layers: `application/api-specification.md`, `application/backend.md`, `application/database.md`,
  `integrations/ai-llm.md`, `integrations/caching.md`, `architecture/coding-patterns.md`,
  `architecture/repository-conventions.md` (has recipes for adding a language, tool and router),
  `architecture/testing.md`, `knowledge/technical-debt.md`. Cursor rules live in
  `.cursor/rules/backend-*.mdc` (generic Tekdi guidance; the code deviates in places — see
  `coding-patterns.md`, "Comparison with `.cursor/rules`").

---

## Phase 1 — Scope the request

Before touching any file, pin down which layers the feature needs:

- **State**: new Redis key, new cached value, or new on-disk artifact? -> Phase 2.
- **Route/API**: new or changed endpoint under `app/routers/`? -> Phase 3.
- **Agent/tool/prompt**: new or changed tool in `agents/tools/`, agent behaviour, system prompt in
  `assets/prompts/`, or a new configurable option on the chat pipeline (model routing, language)?
  -> Phase 4.
- **Client contract**: does a response field, header, query param or streaming behaviour change for
  external clients? -> Phase 5.

Read only the wiki pages that match those layers. If scope is ambiguous (e.g. "add a new option"
with no shape/behaviour), ask one focused question before planning — don't guess request/response
shape, Redis key shape or prompt wording. Adding a tool or language touches several files at once
(see Phase 4), so a wrong guess is costly.

---

## Phase 2 — Data layer (Redis / Qdrant / Marqo / disk — no SQL, no migrations)

1. **There is no schema source of truth and no migration mechanism.** Do not add SQLAlchemy,
   Alembic or another database for one feature. Durable business data lives in external systems;
   this service's own state is Redis keys (all with TTLs) and short-lived files.
2. **Redis**: use the existing access path — `app/core/cache.py` (`cache`, aiocache,
   `JsonSerializer`) and the `app/utils.py` wrappers (`get_cache`/`set_cache`, always with an
   explicit TTL; default 2 h). Follow the key naming in `application/database.md` section 2 and
   `architecture/coding-patterns.md` ("Naming"): `{session_id}__SVA` (history), `{session_id}_AGRINET_ROUTE`,
   `chat_turn:<qid>`, `html_file:<hash>`, etc. Every new key needs a TTL, a documented writer/reader
   and a row in the key catalog. Values must be JSON-serialisable.
3. **Sync vs async Redis**: `helpers/master_catalog.py` uses a raw synchronous `redis.Redis` client
   and fails open (returns `None`) by design; it reads a snapshot written by an *external*
   docs-pipeline. Don't write to that key from here.
4. **Qdrant/Marqo** are external and populated by pipelines outside this repo (index creation,
   re-indexing, backup are not here). Only add reads against fields documented in
   `application/database.md` section 3 / `agents/tools/search.py`; confirm collection/index names via
   env (`QDRANT_VIDEO_COLLECTION_NAME`, `MARQO_INDEX_NAME`, ...).
5. **Local files**: use `app/core/image_storage.py` (`_safe_child_path`, `mark_processed`,
   `cleanup_image`) rather than raw file APIs. Note: uploaded-but-unanalysed images are never
   cleaned up and `cleanup_expired_images()` has no caller (TD-05); the upload dir must be shared
   storage with multiple replicas. `boto3`/AWS settings exist but nothing reads or writes S3.
6. **Sibling structures**: a Redis-persisted per-session value usually needs wiring into the same
   lifecycle as its siblings (TTL refreshed in `_finalize_chat_turn` in `app/services/chat.py`, e.g.
   route TTL refresh). A key that is written but never read/refreshed is a silent no-op.
7. **Do not add a second Redis pattern** (a new client, a pool, a new serializer) for one feature.
   Be aware that a Redis outage fails chat and the nonce check is non-atomic (TD-16).

---

## Phase 3 — Route / API / service layer

1. **Routers**: one file per resource in `app/routers/<name>.py` with
   `router = APIRouter(prefix="/<name>", tags=["<name>"])`; filename equals URL prefix. Register in
   `main.py` (import in `from app.routers import ...` and `app.include_router(<name>.router,
   prefix=settings.api_prefix)`). `app/routers/__init__.py` is not required for registration. URLs
   are `/api/<name>/...` with **no version segment** (deliberately unlike `.cursor/rules/backend-2`).
   `app/routers/suggestions.py` is not mounted — don't copy it.
2. **Thin handlers**: parse the request model, call a service/helper, return a `StreamingResponse`
   or dict. Business flow goes in `app/services/*.py` (`chat.py` orchestrates moderation -> agent ->
   history/telemetry; `agrinet_routing.py`, `npss_response.py`, `chat_turn_map.py`). `app/routers/token.py`
   is a known exception that holds substantial logic — don't treat that as the pattern.
3. **Request models**: Pydantic `BaseModel` with `Field(..., description=...)` in
   `app/models/requests.py`; `GET /api/chat/` binds `ChatRequest` via `Depends()` as query params.
   `Literal` for enumerated choices. Note `app/models/responses.py` models are largely unused by
   routers (routes return dicts / `JSONResponse` / streams); follow the surrounding router's actual
   response shape rather than assuming those models are enforced.
4. **Auth**: protect endpoints with `current_user: dict = Depends(get_current_user)` from
   `app/auth/jwt_auth.py` unless intentionally public (health, file/image download, token).
   The returned dict carries `sub`, `mobile`, `role`, `channel`, `is_guest_user`, `telemetry_context`
   etc. **No role/permission checks exist** — `role` is carried but never enforced. Don't invent an
   RBAC layer for one feature; if a feature needs authorization, raise it as a design question.
   `DISABLE_JWT*` env vars populate `settings.disable_jwt` but nothing reads it — there is no
   env-based auth bypass; don't add one.
5. **Never extend or "fix" the open guest endpoint `POST /api/token`** (caller-chosen `mobile`/`role`/
   `metadata`, no credential — TD-02), or the unauthenticated `GET /api/image/{id}` /
   `GET /api/file/{hash}` routes, as a pattern to copy. Flag them instead; treat only an explicit
   hardening task as licence to change them.
6. **Errors**: routers raise `HTTPException` with explicit status codes; tools raise `ModelRetry` or
   return user-safe strings; there is no global exception handler or custom exception hierarchy,
   and error shapes are mixed (`{"detail": ...}` vs `{"status":"error","message":...}`). Match the
   router you are editing; don't add a second convention. Once the chat stream has started, errors
   truncate the stream (status line already sent).
7. **Invariant-owning modules**: route writes through the module that owns them instead of writing
   Redis directly from a router — chat history through `update_message_history` (`app/utils.py`),
   qid->trace map through `write_chat_turn_map`, model route through `app/services/agrinet_routing.py`,
   telemetry through `send_telemetry` (`helpers/telemetry.py`, retried by `tenacity` in
   `app/tasks/telemetry.py`).
8. **Background work**: post-response work uses FastAPI `BackgroundTasks`. Note that background tasks
   do not run once a streaming client disconnects; the chat path handles this with a detached
   finalizer (`_finalize_after_client_disconnect` in `app/services/chat.py`). Don't add a job queue
   or worker (none exists; `integrations/async-processing.md`).
9. **Async correctness**: handlers are `async def`. Don't add synchronous `httpx.post` / blocking
   clients in an async path — that is existing debt (TD-04: `weather_forecast`, `get_mandi_prices`,
   `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py`) and the wiki's
   *recommended* direction is `httpx.AsyncClient` with `get_default_httpx_timeout()`; that migration
   itself is not yet done, so do not assume the existing code does it. Plain `def` pydantic-ai tools
   run in a worker thread, so sync calls there do not block the loop.
10. **Trace new fields through every response path**: the chat stream header (`X-QID`,
    `Access-Control-Expose-Headers`), JSON endpoints, telemetry builders in `helpers/telemetry.py`,
    and Langfuse metadata. A field present in one shape but absent from a sibling is a silent gap.
11. **Logging**: `logger = get_logger(__name__)` from `helpers/utils.py`. Do not log farmer
    identifiers, OTPs, full queries or secrets (TD-08), and don't copy values from `.env` or
    `docs/setup/portainer.md` into code or docs (TD-01).
12. **Settings/config**: add new settings to `Settings` in `app/config.py` (many modules still use
    `os.getenv` directly — TD-25; prefer `Settings`). Update `.env.example` and
    `architecture/configuration.md` in the same change.

---

## Phase 4 — Agent / tool / integration layer (the "integration layer" of this repo)

**Adding a tool** (recipe from `architecture/repository-conventions.md`):

1. Create `agents/tools/<name>.py` (named after the scheme/integration; `_scheme_status` /
   `_grievance` suffixes where applicable). Decorate with `@observe(name="tool:<function_name>",
   as_type="tool")` from `langfuse`; write a clear docstring (pydantic-ai exposes it to the model);
   take `ctx: RunContext[FarmerContext]` only when you need session/language/coordinates.
2. Call external services with `httpx` and `app.config.get_default_httpx_timeout()` (5 s connect /
   10 s read). Newer tools use `async with httpx.AsyncClient(...)` (`maha_vistaar.py`,
   `sathi_seed.py`) — prefer that for new tools; many older tools use sync `httpx.post`. There are
   no shared client factories or a shared retry helper: per-call clients, with retry semantics coming
   from pydantic-ai (`retries=3` on `agrinet_agent`) via `raise ModelRetry(...)`; catch
   `httpx.TimeoutException`/`httpx.RequestError`; return a plain "temporarily unavailable" string
   when retries are exhausted (see `weather.py`, `mandi.py`). Don't add a new retry framework.
3. Beckn/BAP tools build an envelope (`domain` `schemes:vistaar` unless overridden, `bap_*`/`bpp_*`
   from env, UUID `transaction_id`/`message_id`, `ttl: PT10M`) using payload models defined next to
   the tool. `BAP_ENDPOINT` handling is inconsistent across tools and can raise `AttributeError` in
   four of them (TD-11) — check how the sibling tool you copy handles it.
4. Update Langfuse metadata via `helpers/langfuse_tracing.lf_update_current_observation` as sibling
   tools do. Validate inputs (IDs, OTPs; `to_ascii_digits` from `helpers/utils.py`) and raise
   `ModelRetry` on malformed input. Never echo or log OTPs.
5. **Register** the function in `agents/tools/__init__.py`: import it and append
   `Tool(<fn>, takes_ctx=<True|False>, strict=False)` to `TOOLS` (36 tools today).
6. **Prompts**: tool-selection rules live in the system prompts, not in code. Document when to use
   the new tool in all `assets/prompts/agrinet_*.md` languages (13). Only `en`, `mai`, `or`, `pa`
   use `{{ vector_* }}` template placeholders; the other nine hard-code scheme lists — TD-09.
7. Add new env vars to `.env.example` / `architecture/configuration.md`; add the tool name to the
   leakage-check list in `tests/check_streaming_leakage.py` if it must never appear in answers.

**Adding a language** (recipe from `repository-conventions.md`; verified by `tests/test_languages.py`):
add `assets/prompts/agrinet_<code>.md` keeping all Jinja tokens; update the supported-language text
in every existing prompt; update `SUPPORTED_LANGS`, `Language` enum and `TermPair` in
`agents/tools/terms.py` (+ `assets/glossary_terms.json`); update `LABELS`, `SOURCE_LINES`,
`LANGUAGE_SCRIPT_RANGES`, `SOURCE_PREFIXES` in `app/services/npss_response.py`; confirm Bhashini
support in `helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py`. Odia is `or` here.
Note `target_lang` is **not validated** against available prompt files (TD-10) — an unsupported code
fails with a template-not-found error.

**Changing agent behaviour / config options**:

- Provider/model wiring is in `agents/models.py` (`LLM_PROVIDER`, validated at import; invalid values
  raise `ValueError`). The A/B route resolution chain — `resolve_agrinet_route` in
  `app/services/agrinet_routing.py` (sources: `routing_disabled`, `redis`, `session_start_weighted`,
  `state_repair`, `failover`, `capacity_deflect`), enabled by `AGRINET_ROUTING_ENABLED`, validated at
  startup by `validate_agrinet_routing_config()` — is the existing pattern for "which model"; extend
  it rather than adding a bespoke conditional. Model names are env-driven; the `gpt41` label doesn't
  enforce GPT-4.1 (TD-13).
- Moderation (`agents/moderation.py`, `assets/prompts/moderation_system.md`, 8 categories) is
  **prompt-enforced only**: the main agent still runs for refused categories, and a moderation
  exception fails the whole turn (TD-06). A hard code gate is *recommended, not implemented*.
- Timeouts: `AGRINET_MODEL_TIMEOUT_SECONDS` (45 s, whole streamed run), model HTTP timeout 120 s,
  `LLM_MODERATION_TIMEOUT_SECONDS` (20 s). Failover to the alternate route happens only when routing
  is enabled and no chunk has been emitted yet.
- **Response parsing**: main agent `output_type=str` (streamed `TextPartDelta`); moderation uses
  `PromptedOutput(QueryModerationResult)`. Follow the existing pattern per call site.
- **Sync vs async**: the chat turn is one blocking-in-spirit streamed request (no queue); the
  `asyncio.Queue`/`create_task`/`asyncio.shield` pattern in `app/services/chat.py` is delicate — read
  `sequence-diagrams.md` (W2) before touching it. History: last 3 message pairs in the prompt,
  `trim_history(..., 64_000)` tokens, 2 h TTL; thinking parts are stripped before storage.
- NPSS (crop-image diagnosis) post-processing is deterministic in `app/services/npss_response.py`.
  Coordinates are currently *not* passed from the chat request into `FarmerContext`, so NPSS falls
  back to the default Delhi KVK location (TD-03) — don't assume location plumbing works.
- **Standalone scripts**: `scripts/` (e.g. `audit_npss_empty_villages.py`) and `tests/` runner
  scripts reuse production modules without being the deployed path. Confirm which one a request
  means; a fix to one does not apply to the other.
- Instrumentation is Langfuse `@observe` (agents: `Agent.instrument_all(False)`, `instrument=False`),
  not pydantic-ai OpenTelemetry. Use span names from `helpers/langfuse_trace_schema.py`.

---

## Phase 5 — Client contract (there is no frontend in this repo)

This repo has no templates, no `StaticFiles`, no JS build. Phase "frontend" reduces to keeping the
HTTP contract that external clients (web chat, Android app) rely on, and — where relevant — telling
the user the client-side change must happen elsewhere.

1. **Chat stream**: `GET /api/chat/` returns `text/event-stream` but the body is **raw answer text**
   chunks, not `data:`-framed SSE (`application/api-specification.md` section 5; TD-18). Don't change
   framing without flagging it as a breaking client change. True SSE framing is *recommended*, not
   implemented.
2. **Headers/CORS**: `X-QID` on chat responses, exposed via `Access-Control-Expose-Headers`;
   `TimingAllowOriginMiddleware` is plain ASGI on purpose (not `BaseHTTPMiddleware`) so the stream
   isn't re-pumped. CORS defaults to `*` with credentials (dev-oriented).
3. **Auth contract**: clients fetch a JWT from `/api/token*` and send `Authorization: Bearer`.
   Default expiry is 15 min and there is no refresh endpoint (a refresh endpoint is *recommended*,
   not implemented).
4. **Session continuity**: a server-generated `session_id` is not returned to the client when
   omitted (returning it is *recommended*, not implemented).
5. **Image flow**: `POST /api/image/upload` -> `image_id` -> sent as chat `query`; `[IMAGE_URL: ...]`
   normalisation lives in `app/routers/chat_query_utils.py`.
6. **Document the contract change**: update `application/api-specification.md` (and note that
   `API_DOCUMENTATION.md` is stale — TD-12). Close the loop for any new field: the response, telemetry
   events (`/api/telemetry/*`), and the docs. Pass-through of client UI work: say it is out of scope
   for this repo rather than inventing UI code.
7. Design tokens, orphan-component checks and brand styling do not apply here (no UI).

---

## Phase 6 — Wire together and verify

**Do not claim "tests pass" without saying which category of test you ran.** There is no CI test job
(`.github/workflows/build.yml` only builds and pushes an image), no `pytest.ini`, no `conftest.py`,
no coverage config and no linter config (`architecture/testing.md`).

- **Offline / mocked (safe to run)** — select files explicitly from the repo root (bare `pytest tests/`
  imports the live-LLM scripts and may fail at collection):

  ```bash
  pytest tests/test_languages.py tests/test_chat_query_utils.py tests/test_npss_geocodes.py \
         tests/test_audit_npss_empty_villages.py tests/test_backend_owned_telemetry.py
  ```

  Prerequisites: `jwt_public_key.pem` present (read at import by `app/auth/jwt_auth.py`), and `LLM_PROVIDER`
  + its variables set for anything importing `agents.*` / `app/services/chat.py` (`agents/models.py` builds
  clients at import). `pytest` itself must be installed separately.
- **Known stale**: `tests/test_chat_stream_contract.py` references non-existent symbols
  (`_await_with_sse_keepalives`, `SSE_KEEPALIVE`, `chat_router._get_message_history`) and expects the
  server to ignore a client `qid`; expect failures and don't treat them as caused by your change
  without checking (TD-14).
- **Live runners — do not run casually** (they call real LLMs and real tools via a populated `.env`,
  cost money, and need `logfire`, which is commented out of `requirements.txt`):
  `tests/test_agrinet_single_turn.py`, `tests/test_moderation_single_turn.py`,
  `tests/test_moderation_multi_turn.py`, `tests/check_json_in_responses.py`,
  `tests/check_streaming_leakage.py`. Ask the user before running them, never point them at
  production credentials, and don't commit new `tests/results_*.json` outputs.
- **New tests**: follow the existing style — `monkeypatch` for external calls, `TestClient` with
  `app.dependency_overrides[get_current_user]` for routes (`tests/test_backend_owned_telemetry.py`),
  `tests/test_<area>.py` naming. Adding a language must pass `tests/test_languages.py`.
- **Manual verification**: run the app per the repo docs (`docker compose up --build`, Redis must be
  on the `svanetwork` network; or `uvicorn main:app --host 0.0.0.0 --port 8000` locally with `.env` and
  `jwt_public_key.pem`). Obtain a token from `POST /api/token` (local/dev only; do not extend that
  endpoint) or the API-key route, then exercise the route with `Authorization: Bearer`. Check
  `GET /api/health/ready` (503 means Redis is unreachable). Watch app logs (`docker logs -f sva_app`);
  there is no separate worker log because there is no worker.
- **Verify persisted state directly**: inspect the Redis keys (with the `sva-cache-` prefix) or the
  upload directory rather than trusting a response alone; for agent changes, check the Langfuse trace
  (tool spans under `agent.vistaar`) since pydantic-ai OTel is off.

---

## Phase 7 — Documentation

If the change alters behaviour the wiki describes, update the corresponding page(s) in
`docs/wiki/bharat-oan-api/` and `wiki-index.md` in the same change (the wiki is maintained with the
`llm-wiki` skill; label claims **Implemented (verified)** vs **Recommended (not yet implemented)**).
Debt/bugs go only in `knowledge/technical-debt.md`. Add new env vars to `.env.example` and
`architecture/configuration.md` as they are introduced. New/changed endpoints go in
`application/api-specification.md`. Don't add standalone planning docs unless asked.

---

## Behavioral Rules

- **Trust code over docs**: `API_DOCUMENTATION.md` and parts of the README lag the code; the wiki is a
  snapshot of the code at generation time.
- **Know which path you edit**: deployed app (`main.py`, `app/`, `agents/`, `helpers/`) vs `scripts/` and
  `tests/` runners that reuse the same modules; disabled code (`suggestions` router/agent/task, the
  non-streaming `_run_agrinet_with_failover`/`_run_agrinet_once`) is not a pattern to extend.
- **Never extend or "fix"** the open guest `POST /api/token`, or the unauthenticated image/file routes,
  as part of an unrelated feature — flag them (TD-02).
- **No second pattern for one feature**: no second Redis client/serializer, retry framework, auth path,
  error-handling convention, logging approach or global state module when an existing pattern covers it.
- **No privacy leaks**: never log or print farmer identifiers, OTPs, queries or secrets; never copy values
  from `.env` or `docs/setup/portainer.md`.
- **Never claim tests pass** without naming the exact files run and confirming they are offline/mocked,
  or stating that you verified manually. Don't run the live-LLM runners without asking.
- **Don't build on aspirational items**: anything the wiki labels "Recommended (not yet implemented)" or
  a **Note** about missing behaviour (rate limiting, role checks, SSE framing, moderation gate, async
  tool clients, cleanup job, CI) is not current behaviour.
- **Ask before assuming** route/request/response shape, Redis key shape, tool arguments or prompt wording
  when the request doesn't specify it.
- **Gaps left unconfirmed**: this skill was tailored from the wiki plus spot-checks of `main.py`,
  `agents/tools/__init__.py`, `app/config.py` and `tests/`; claims about individual tool modules,
  `.env.example` contents (6 lines) and CI behaviour rest on the wiki. Re-check the module you edit.

---

## Tailoring this skill to a specific codebase

Trigger: the user says something like *"Update this skill to suit the tech stack of this
codebase"* (or names this skill directly while asking for the same).

**Status: already run against bharat-oan-api** (Python 3.13 / FastAPI / pydantic-ai 1.2.1 / Redis
via aiocache / Qdrant / Marqo / Beckn BAP / Langfuse; no SQL DB, no frontend, no RBAC), using the wiki
in `docs/wiki/bharat-oan-api/` as the primary source. Re-run if the stack changes (a new data layer, a
frontend, an auth/role model, a CI test job, a framework migration).

When this happens:

1. **Reuse discovery work already done.** If another skill/subagent in this repo's `.cursor/skills/` or
   `.cursor/agents/` (e.g. `brd-task-creator`, `code-reviewer`, `qa-tester`) has already been tailored
   to this codebase, read it first — its stack findings apply here too.
2. **Prefer the wiki** (`docs/wiki/bharat-oan-api/ai-context.md`, then relevant pages) over re-deriving
   from raw code; otherwise inspect manifests, routing/entry structure, data layer, auth, integrations
   and test setup directly. Honour Note callouts and "Recommended (not yet implemented)" labels.
3. **Rewrite Phases 1–7 in place** with concrete frameworks, file paths, helper names and test commands,
   with citations; drop layers that genuinely don't apply (as the SQL/migration and frontend layers were
   reworded here) but say so explicitly rather than leaving a generic placeholder.
4. **Update the frontmatter `description`** to name the actual stack while keeping the trigger phrasing
   generic enough to match "add a feature" requests.
5. **Keep this section** at the bottom, noting only that it has been run.
6. **Never fabricate** specifics that don't survive inspection; say "check this before assuming".
