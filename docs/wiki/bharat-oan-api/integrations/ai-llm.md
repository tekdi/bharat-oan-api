# AI / LLM Integrations

This page documents every LLM-facing component of `bharat-oan-api`: the agents, models, prompts, guardrails, tools, retry/timeout behaviour, response handling and Langfuse tracing. All claims below were traced to source files in this repository.

Related pages: [AI Context](../ai-context.md) | [Architecture Overview](../architecture/architecture-overview.md) | [Sequence & Workflow Diagrams](../architecture/sequence-diagrams.md) | [Third-Party Integrations](./third-party-integrations.md) | [Caching](./caching.md) | [Business Rules](../business/business-rules.md) | [Technical Debt](../knowledge/technical-debt.md)

---

## 1. Overview

The service uses [pydantic-ai](https://ai.pydantic.dev/) (`pydantic-ai==1.2.1`, see `requirements.txt`) to run two agents per chat turn and, historically, a third agent for suggestions.

| Agent | Defined in | Purpose | Status |
|-------|-----------|---------|--------|
| `agrinet_agent` ("Vistaar Agent") | `agents/agrinet.py` | Main tool-using farmer assistant; produces the streamed answer | Implemented (verified) - used by `app/services/chat.py` |
| `moderation_agent` ("Moderation Agent") | `agents/moderation.py` | Classifies each user message into one of 8 moderation categories | Implemented (verified) - run before every chat turn |
| `suggestions_agent` ("Suggestions Agent") | `agents/suggestions.py` | Generates 3-5 follow-up questions | Disabled - router and task are commented out in `main.py`, `app/routers/__init__.py`, `app/tasks/__init__.py`; the module also imports `LLM_AGRINET_MODEL`, which is not defined in `agents/models.py` (see [Technical Debt](../knowledge/technical-debt.md)) |

`agents/__init__.py` calls `Agent.instrument_all(False)` so pydantic-ai's own OpenTelemetry instrumentation is off; tracing is done with Langfuse `@observe` decorators instead (see [Langfuse Tracing](#9-langfuse-tracing)). Both active agents also pass `instrument=False`.

> **Note:** No translation LLM or MT call sits between the user and the agent in the chat path. The user's selected language (`target_lang`) is injected into the user message and into the choice of system prompt, and the main model answers directly in that language. `helpers/translation.py` (Bhashini / Google translators) is only used for the deterministic NPSS post-processing step (see [Response Handling](#8-response-handling)).

---

## 2. Model, Parameters & API-Key Management

### 2.1 Provider selection (`agents/models.py`)

`agents/models.py` builds two `OpenAIChatModel` objects at import time - `AGRINET_MODEL` and `MODERATION_MODEL` - according to the `LLM_PROVIDER` env var (default `openai`, lower-cased). Any other value raises `ValueError` at import.

| `LLM_PROVIDER` | Required env vars | Optional env vars | Notes |
|----------------|-------------------|-------------------|-------|
| `openai` (default) | `OPENAI_API_KEY` (read with `os.getenv`; not validated at startup) | `LLM_AGRINET_MODEL_NAME`, `LLM_MODERATION_MODEL_NAME` (both default to `gpt-3.5-turbo` in code) | Uses `OpenAIProvider(api_key=...)` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_API_VERSION`, `AZURE_OPENAI_DEPLOYMENT_NAME` (each missing value raises `ValueError`) | `LLM_AGRINET_MODEL_NAME`, `LLM_MODERATION_MODEL_NAME` (default to the deployment name) | One cached `AsyncAzureOpenAI` client (`_get_default_azure_client`, `lru_cache`) shared by both models |
| `vllm` | `VLLM_AGRINET_MODEL_URL` (raises `ValueError` if missing) | `VLLM_MODERATION_MODEL_URL` (defaults to the agrinet URL), `LLM_AGRINET_MODEL_NAME` (default `agrinet-model`), `LLM_MODERATION_MODEL_NAME` (default `moderation-model`) | API key is hard-coded to the placeholder `"not-needed"` |

The model **names** are not pinned in code: the actual model used in a deployment is whatever `LLM_AGRINET_MODEL_NAME` / `LLM_MODERATION_MODEL_NAME` (or the Azure deployment name) resolve to. The code-level default of `gpt-3.5-turbo` only applies when those variables are unset on the `openai` provider.

> **Note:** The route labels `gpt41` / `gemma` (see below) are identifiers used in Redis, logs and Langfuse. The label `gpt41` implies GPT-4.1, but the code does not enforce that; the real model name for the `gpt41` route is the value of `LLM_AGRINET_MODEL_NAME` / Azure deployment configured in the environment.

### 2.2 Optional A/B routing between two agrinet models (`agents/models.py`, `app/services/agrinet_routing.py`)

Implemented (verified). Disabled unless `AGRINET_ROUTING_ENABLED` is truthy (`1/true/yes/on`).

| Env var (`app/config.py`) | Default | Meaning |
|---------------------------|---------|---------|
| `AGRINET_ROUTING_ENABLED` | `false` | Master switch. When false, every turn uses route `gpt41` with source `routing_disabled` |
| `AGRINET_ROUTE_GPT41_WEIGHT` / `AGRINET_ROUTE_GEMMA_WEIGHT` | `70` / `30` | Session-start weighted split; must be non-negative and sum to 100 (`validate_agrinet_routing_config`, called from the FastAPI lifespan in `main.py`) |
| `AGRINET_ROUTE_TTL_SECONDS` | `7200` | Redis TTL of the per-session route pin |
| `AGRINET_MODEL_TIMEOUT_SECONDS` | `45` | Hard wall-clock timeout for one agrinet run (see [Retries & Timeouts](#7-retries-timeouts-and-failover)) |
| `AGRINET_GEMMA_PROVIDER` | `vllm` | Only `vllm` is supported (`_build_gemma_agrinet_model`) |
| `AGRINET_GEMMA_MODEL_NAME`, `AGRINET_GEMMA_BASE_URL` | none | Required when routing is enabled (validated at startup by `validate_agrinet_routing_config`) |
| `AGRINET_GEMMA_API_KEY` | `not-needed` | Optional bearer for the vLLM endpoint |
| `AGRINET_GEMMA_MAX_CONCURRENCY` | `10` | Capacity gate: Gemma turns are deflected to `gpt41` while (running + waiting) requests >= this value |
| `AGRINET_GEMMA_METRICS_URL` | derived from base URL (`/v1` stripped, `/metrics` appended) | vLLM Prometheus endpoint parsed for `vllm:num_requests_running` and `vllm:num_requests_waiting` |
| `AGRINET_GEMMA_METRICS_CACHE_TTL` | `2` | Seconds the concurrency reading is cached in Redis (`gemma_concurrency` key) |

Route decision sources (`AgrinetRouteSource` in `app/services/agrinet_routing.py`): `redis` (pinned route found), `session_start_weighted` (new session, random roll), `state_repair` (session has history but no stored route -> pinned to `gpt41`), `failover`, `capacity_deflect`, `routing_disabled`. The Redis key is `<session_id>_AGRINET_ROUTE`.

### 2.3 Generation parameters

| Agent | temperature | top_p | other | Source |
|-------|-------------|-------|-------|--------|
| `agrinet_agent` | 0.7 | 0.95 | `max_tokens=4096`, `timeout=120` (model-level HTTP timeout), `parallel_tool_calls=True` | `agents/agrinet.py` (`OpenAIChatModelSettings`) |
| `moderation_agent` | 0.0 | 1.0 | `timeout=float(LLM_MODERATION_TIMEOUT_SECONDS)` default `20` s | `agents/moderation.py` |
| `suggestions_agent` (disabled) | 1.0 | 1.0 | `top_k=0`, `openai_reasoning_effort='medium'`, `parallel_tool_calls=False` | `agents/suggestions.py` |

Agent-level behaviour for `agrinet_agent`: `retries=3`, `end_strategy='exhaustive'` (all tool calls requested in a step are executed), `output_type=str`, `deps_type=FarmerContext`. Moderation: `retries=2`, `output_type=PromptedOutput(QueryModerationResult)`.

### 2.4 API-key storage, rotation and required vs optional keys

| Aspect | Implemented behaviour (verified) |
|--------|----------------------------------|
| Storage | Environment variables, loaded from `.env` by `python-dotenv` (`load_dotenv()` in `agents/models.py`, `app/config.py`) and by Docker Compose `env_file: .env` (`docker-compose.yml`). `.env` is git-ignored (`.gitignore`); only `.env.example` (NPSS/Photon placeholders, no LLM keys) is committed |
| Secret manager / vault | Not integrated in code |
| Rotation | No rotation mechanism. Keys are read once at import (`OpenAIProvider(api_key=...)`, cached Azure client), so rotating a key requires a process restart / redeploy |
| Startup validation | Azure and vLLM provider variables are validated at import (raises `ValueError`). `OPENAI_API_KEY` is **not** validated - a missing key surfaces on the first LLM call |
| Logging of keys | Not logged by the LLM modules. The vLLM and Gemma paths use the placeholder `not-needed` |
| Langfuse keys | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` (or `LANGFUSE_BASE_URL`) via `app/config.py` -> exported to `os.environ` by `helpers/langfuse_helper.py`; optional (empty string when unset) |

Required keys depend on provider: `OPENAI_API_KEY` (openai) or the four `AZURE_OPENAI_*` variables (azure-openai) or `VLLM_AGRINET_MODEL_URL` (vllm). Optional LLM-adjacent keys: `OPENAI_API_KEY` is also used by `helpers/transcription.py` for Whisper; `AGRINET_GEMMA_*` only when A/B routing is on.

> **Note:** `app/config.py` also declares fields for `SARVAM_API_KEY`, `GEMINI_API_KEY`, `LOGFIRE_TOKEN`, `INFERENCE_API_KEY`, `MAPBOX_API_TOKEN`, `OLLAMA_ENDPOINT_URL`, `ELEVEN_LABS_API_KEY` and similar. No runtime code under `agents/`, `helpers/` or `app/` reads these values, so they do not currently affect behaviour (see [Third-Party Integrations](./third-party-integrations.md#6-declared-but-unused-integrations)).

---

## 3. Prompts

### 3.1 Prompt locations and loading

Prompts are Markdown/Jinja2 templates in `assets/prompts/`, loaded by `helpers/utils.py::get_prompt(prompt_file, context, prompt_dir="assets/prompts")` using a Jinja2 `FileSystemLoader` (`autoescape=False`). The path is relative to the process working directory.

| Prompt file | Used by | Notes |
|-------------|---------|-------|
| `agrinet_{lang}.md` for `en, hi, as, bn, gu, kn, mai, ml, mr, or, pa, ta, te` (13 files) | `agents/agrinet.py::get_system_prompt` (`@agrinet_agent.system_prompt(dynamic=True)`) | Rendered **fresh every turn** with `today_date`, `crop_season`, `last_weekday_table` and the vector-scheme block |
| `moderation_system.md` | `agents/moderation.py` (loaded once at import) | English-only classifier policy |
| `suggestions_system.md` | `agents/suggestions.py` (disabled agent) | Describes English/Hindi/Marathi only |
| `assets/prompts_backup_20260922_164719/` | Not referenced by code | Older snapshot of prompts (no `agrinet_mai/or/pa.md`) |

The language file is chosen by `get_prompt(f'agrinet_{lang_code}')`, where `lang_code` is the request's `target_lang` (default `hi`, `FarmerContext.lang_code`). `ChatRequest.target_lang` (`app/models/requests.py`) is a free-form string; the code does not validate it against the 13 supported prompt files (see [Technical Debt](../knowledge/technical-debt.md)).

### 3.2 Template variables

`get_system_prompt` builds the context from `helpers/utils.py` and `helpers/scheme_qdrant_search.py`:

| Variable | Source | Used in |
|----------|--------|---------|
| `today_date` | `get_today_date_str()` (IST, e.g. "Thursday, 26 November 2026") | all prompts |
| `crop_season` | `get_crop_season()` - Kharif Jun-Sep, Rabi Oct-Feb, Zaid Mar-May | all prompts |
| `last_weekday_table` | `get_last_weekday_table()` | passed to the template but not referenced by any file in `assets/prompts/` (verified by grep) |
| `vector_scheme_count`, `vector_schemes_bullets`, `vector_schemes_identifiers` | `format_vector_schemes_prompt_block()` reading the docs-pipeline Redis snapshot (`helpers/master_catalog.py`) | only `agrinet_en.md`, `agrinet_mai.md`, `agrinet_or.md`, `agrinet_pa.md` |

> **Note:** The other nine language prompts (`as, bn, gu, hi, kn, ml, mr, ta, te`) do not contain the `{{ vector_* }}` placeholders; they name the vector-indexed schemes statically (for example the Hindi prompt lists ten schemes). Dynamically promoted schemes therefore appear in the live scheme list only for the four template-driven languages. `tests/test_languages.py` asserts that `or`, `pa`, `mai` keep the same Jinja tokens as English.

### 3.3 What the main prompt enforces (traced to `assets/prompts/agrinet_en.md`)

The English prompt is the reference; other languages are translations of the same structure (some are shorter - 314 to 406 lines).

- **Scope**: central government schemes, real-time scheme status (PM-KISAN, PMFBY, Soil Health Card, SMAM, AIF), grievances (PM-KISAN, PMFBY filing; AIF tracking only), weather, soil health / GFR, crop and pest advisory, mandi prices, SATHI seed availability, Amul union schemes, MahaVistaar cross-network schemes.
- **Response rules**: short answers (2-4 sentences simple, max 10), answer first, then a bold translated `**Source: ...**` line, then one follow-up question; single language only; Roman numerals 0-9 only; no LaTeX.
- **Grounding**: "Always use tools" - every factual statement must come from a tool result; no tool data means say so plainly; never fabricate; mandatory source citation translated to the response language.
- **Moderation compliance**: only `Valid Agricultural` is processed; every other category returns a fixed, friendly refusal template (table in the prompt, "Moderation Categories").
- **Tool selection guide**: a table mapping query types to tools and source labels (see [Tools Registry](#6-tools-registry)).
- **OTP safety**: never echo OTP digits; never invent masked mobile numbers; AIF flow requires `initiate_aif_otp` and `verify_aif_otp` tool results before claiming anything.
- **Mandi rule**: date intent must be confirmed before any tool call; location must be at least district level.

> **Note:** The English prompt hard-codes a PM-KISAN 23rd instalment release rule keyed to the dates 20 and 21 June 2026, mirroring `helpers/pmkisan_installment_release.py` (`PM_KISAN_23RD_INSTALLMENT_RELEASE_DATE = date(2026, 6, 20)`). This is time-bound content that will need a content update for later instalments.

### 3.4 Per-turn user message construction (`app/services/chat.py`)

The user turn is not just the raw query:

1. `FarmerContext.get_user_message()` (`agents/deps.py`) joins `**User:** "<query>"`, `**Selected Language:** <display name>` (via `langcodes`) and, after moderation, the `**Moderation Compliance:** <action> (<Category>)` line.
2. If prior history exists, the last 3 user/assistant pairs (`format_message_pairs(history, 3)` in `app/utils.py`) are prepended as `**Conversation** ... ---`.
3. For image-analysis turns (`[IMAGE_URL: ...]` / UUID / `/api/image/<id>`), `_wrap_image_analysis_message` appends detailed instructions: must call `analyze_crop_image`, do not call `search_pests_diseases`, present a Pest / Crop / Cause card, no treatment advice, no location questions.

---

## 4. Guardrails and Moderation

### 4.1 Moderation agent (`agents/moderation.py`, `assets/prompts/moderation_system.md`)

Implemented (verified). `_run_moderation` in `app/services/chat.py` calls `moderation_agent.run(user_message)` before the main agent on every turn, using the same composed message (conversation prefix + query + language). Output is typed by `QueryModerationResult`:

| `category` (Literal) | Standard action text in the prompt |
|----------------------|------------------------------------|
| `valid_agricultural` | Proceed with the query |
| `invalid_non_agricultural` | Decline with standard non-agri response |
| `invalid_external_reference` | Decline with external reference response |
| `invalid_compound_mixed` | Decline with mixed content response |
| `invalid_language` | Decline with language policy response |
| `unsafe_illegal` | Decline with safety policy response |
| `political_controversial` | Decline with political neutrality response |
| `role_obfuscation` | Decline with agricultural-only response |

The policy file defines priority order (unsafe_illegal > role_obfuscation > political_controversial > invalid_compound_mixed > invalid_external_reference > invalid_non_agricultural > invalid_language > valid_agricultural), says "when in doubt, decline", lists scheme acronyms that must be treated as valid agricultural, and gives 14 boundary examples. Details and business interpretation are in [Business Rules](../business/business-rules.md#moderation-rules).

### 4.2 How moderation is enforced

- The moderation result is stringified (`QueryModerationResult.__str__`) and **injected into the main agent's user message**; the main system prompt then tells the model to respond with the matching refusal template and to never override the decision. Enforcement is therefore prompt-level, not a hard code gate: the code does not skip the main agent when the category is not `valid_agricultural`.
- The category is attached to the Langfuse trace as a tag (`propagate_attributes(tags=[moderation_data.category])`) and stored in observation metadata (`moderation_category`).
- Moderation failures are not caught in `stream_chat_messages`: an exception from `_run_moderation` goes to the outer `except`, which emits a chat error telemetry event and re-raises (stream error). There is no "fail-open" or "fail-closed" classification fallback.
- `create_moderation_event` exists in `helpers/telemetry.py` but is not called anywhere (verified by grep), so moderation outcomes are visible in Langfuse but not emitted as telemetry events.

### 4.3 Other guardrails

| Guardrail | Where | Behaviour |
|-----------|-------|-----------|
| Think-block stripping | `_strip_streaming_thinking_chunk`, `_sanitize_streamed_output` in `app/services/chat.py` | Removes `<think>...</think>` from streamed and final text; `filter_thinking_from_history` (`app/utils.py`) strips `ThinkingPart` from stored/replayed history (documented reason: vLLM errors such as "Unknown role: final") |
| OTP privacy | `assets/prompts/agrinet_*.md`, `agents/tools/aif.py::_loggable` | Prompt forbids echoing OTP; AIF tool masks the OTP in logged payloads |
| Digit normalisation | `helpers/utils.py::to_ascii_digits` | Native-script digits converted to ASCII before OTP/phone/ID tool calls |
| Image upload limits | `app/core/image_storage.py`, `agents/tools/npss.py` | JPEG/PNG only for upload (`/api/image/upload`), `MAX_IMAGE_UPLOAD_SIZE_MB` default 10, WebP rejected by NPSS tool |
| NPSS deterministic attribution | `app/services/npss_response.py::post_process_npss_response` | Removes model-written source lines/trailing question and appends a fixed localized `Source: NPSS` line |
| Tool-argument validation | e.g. `agents/tools/gfr.py`, `pmkisan_scheme_status.py`, `aif.py` | Raise `ModelRetry` for malformed IDs/OTPs so the model asks the farmer again |

---

## 5. System Prompt Context Freshness

`get_system_prompt` is registered with `dynamic=True`, so date, season and the vector-scheme list are re-rendered every turn. By contrast, the `search_schemes` tool docstring (which is part of the tool schema sent to the model) is patched once at import time with scheme codes (`format_qdrant_scheme_codes_for_doc` at the bottom of `agents/tools/search.py`) and is only refreshed on restart. A scheme added to the Redis snapshot after process start is routable immediately but is not named in that docstring until the next restart (documented in the source comments).

---

## 6. Tools Registry

All tools are registered in `agents/tools/__init__.py` as `TOOLS`, a list of 36 `pydantic_ai.Tool(...)` entries (all `strict=False`). `takes_ctx=True` means the tool receives `RunContext[FarmerContext]` (session id, question id, language, NPSS state).

| # | Tool function | Module | Takes ctx | Backend / purpose |
|---|---------------|--------|-----------|-------------------|
| 1 | `get_scheme_info` | `scheme_info.py` | yes | Beckn `/search` via BAP, category `schemes-agri`; 16 legacy scheme codes (`kcc, pmkisan, pmfby, shc, pmksy, sathi, pmasha, aif, smam, pdmc, pkvy, nfsm, rad, ffs, nbm, nbhm`); appends the PM-KISAN 23rd instalment section for `pmkisan` |
| 2 | `call_maha_vistaar_network` | `maha_vistaar.py` | yes | BAP `/search`, cross-network MahaVistaar schemes (`ndksp-drip-irrigation`, `ndksp-farm-pond-lining`, `aif`) |
| 3 | `call_amul_vistaar_network` | `amul_vistaar.py` | yes | BAP `/search`, domain `schemes:amul-union`, optional `union` (`banas, kutch, sumul, surendranagar`) / `provider_id` |
| 4-5 | `initiate_pm_kisan_status_check`, `check_pm_kisan_status_with_otp` | `pmkisan_scheme_status.py` | yes | BAP `/init` then `/status`; 4-digit OTP |
| 6-7 | `initiate_pmfby_status_check`, `check_pmfby_status_with_otp` | `pmfby_scheme_status.py` | yes | BAP `/init` then `/status`; 6-digit OTP; PII masking helper |
| 8-11 | `initiate_pmfby_grievance_otp`, `check_pmfby_grievance_otp`, `pmfby_grievance_status`, `pmfby_submit_grievance` | `pmfby_grievance.py` | yes | PMFBY grievance flow via BAP |
| 12 | `check_shc_status` | `shc_scheme_status.py` | no | BAP `/init`; converts returned HTML report to cached file URL (`/api/file/{hash}`) |
| 13-15 | `pmkisan_grievance_send_otp`, `pmkisan_submit_grievance`, `pmkisan_grievance_status` | `pmkisan_grievance.py` | yes | PM-KISAN grievance flow via BAP; grievance label->code map from `assets/grievance_types.json` (10 entries) |
| 16 | `search_terms` | `terms.py` | no | Local fuzzy glossary lookup (`assets/glossary_terms.json`, 5,140 terms, `rapidfuzz`) across 14 language fields |
| 17 | `search_documents` | `search.py` | no | Marqo hybrid search (index `MARQO_INDEX_NAME`, default `sunbird-va-index`, filter `type:document`) |
| 18 | `search_schemes` | `search.py` | yes | BAP `/search`, category `scheme-agri-qdrant`; scheme resolved locally against the Redis catalog |
| 19 | `search_video` | `search.py` | no | Qdrant video collection (`QDRANT_VIDEO_COLLECTION_NAME`, default `video_data_collection`) via `helpers/video_qdrant_search.py` |
| 20 | `search_pests_diseases` | `search.py` | no | Marqo index `MARQO_PESTS_DISEASES_INDEX_NAME` |
| 21 | `weather_forecast` | `weather.py` | no | BAP `/search`, category code `WFC` (IMD Mausamgram) |
| 22 | `get_mandi_prices` | `mandi.py` | yes | BAP `/search`, category `price-discovery` |
| 23 | `search_commodity` | `commodity.py` | no | Local fuzzy lookup in `assets/commodity_codes.json` (543 commodities) |
| 24-25 | `forward_geocode`, `reverse_geocode` | `maps.py` | no | Photon geocoder (`PHOTON_HOST`), India bbox `[68, 6, 98, 36]` |
| 26-27 | `gfr_get_recommendations`, `gfr_get_crop_registries` | `gfr.py` | no | Government fertilizer recommendations via BAP `/search` |
| 28-30 | `get_sathi_crop_groups`, `list_sathi_crops_in_group`, `search_sathi_seed_availability` | `sathi_seed.py` | no | SATHI master API (`SATHI_MASTER_BASE_URL`, `SATHI_MASTER_API_KEY`) + BAP `/search` for dealer stock |
| 31 | `check_smam_scheme_status` | `smam_scheme_status.py` | no | BAP `/search`; lookup by `mobile` or `application_no` |
| 32 | `analyze_crop_image` | `npss.py` | yes | NPSS image analysis (see [Third-Party Integrations](./third-party-integrations.md#npss)) |
| 33-36 | `initiate_aif_otp`, `verify_aif_otp`, `check_aif_loan_status`, `check_aif_grievance_status` | `aif.py` | yes | AIF portal via BAP `/init` and `/status`; 6-digit OTP |

Shared conventions across tools:

- Each tool is wrapped with `@observe(name="tool:<name>", as_type="tool")` (Langfuse) and writes metadata (transaction id, HTTP status, error type) via `lf_update_current_observation`.
- Beckn tools build a request envelope with `domain` (`schemes:vistaar` unless overridden by `SCHEME_DOMAIN` / `AMUL_SCHEME_DOMAIN`), `bap_id`/`bap_uri`/`bpp_id`/`bpp_uri` from env, `transaction_id`/`message_id` UUIDs and `ttl: PT10M`.
- Tool selection rules are expressed in the system prompt, not in code (see [Business Rules - tool selection](../business/business-rules.md#tool-selection-rules)). Code only enforces input validity.
- `search_terms` and the glossary: `agents/tools/terms.py` also exposes `normalize_text_with_glossary`, which appends `[translated term]` (default Hindi) after matched English terms in Marqo results (`SearchHit.processed_text`) and scheme chunks.

---

## 7. Retries, Timeouts and Failover

| Layer | Mechanism | Value | Source |
|-------|-----------|-------|--------|
| Main agent | pydantic-ai `retries` | 3 (tool/validation retries) | `agents/agrinet.py` |
| Moderation agent | pydantic-ai `retries` | 2 | `agents/moderation.py` |
| Tool errors | `raise ModelRetry(...)` | Asks the model to retry/re-collect input; used by most tools for unexpected exceptions and invalid input | `agents/tools/*` |
| Retry exhaustion | `UnexpectedModelBehavior` caught inside tools | Returns a "temporarily unavailable" string | e.g. `weather.py`, `mandi.py`, `scheme_info.py` |
| Whole-run timeout | `asyncio.timeout(settings.agrinet_model_timeout_seconds)` | 45 s default, wraps the entire streamed run including tool calls | `_run_agrinet_once_streaming` in `app/services/chat.py` |
| Model HTTP timeout | `OpenAIChatModelSettings.timeout` | 120 s | `agents/agrinet.py` |
| Moderation timeout | `LLM_MODERATION_TIMEOUT_SECONDS` | 20 s | `agents/moderation.py` |
| Outbound tool HTTP | `DEFAULT_API_TIMEOUT` / `DEFAULT_API_READ_TIMEOUT` | 5 s connect / 10 s read (`app/config.py::get_default_httpx_timeout`) | most BAP tools |
| Failover | `_run_agrinet_with_failover_streaming` | Only when `AGRINET_ROUTING_ENABLED`; on any exception retry once on the alternate route (`get_alternate_agrinet_route`) **only if no chunk has reached the client** (`_StreamChunkSink.emitted`); on success the session route is re-pinned in Redis | `app/services/chat.py` |
| Capacity deflection | `_apply_capacity_gate` | Gemma sessions are deflected to `gpt41` for one turn when vLLM concurrency is at/over `AGRINET_GEMMA_MAX_CONCURRENCY`; stored route is left unchanged | `app/services/agrinet_routing.py` |
| Client disconnect | `_finalize_after_client_disconnect` | The agent task is shielded; a detached task (timeout 180 s) finishes the run and still saves history and telemetry | `app/services/chat.py` |

When routing is disabled there is no automatic LLM fallback: a model error or the 45 s timeout propagates, a chat error telemetry event is queued and the stream errors.

Conversation memory: history is stored in Redis (`<session_id>__SVA`, TTL 2 hours - `app/utils.py`), trimmed to ~64,000 tokens by `trim_history` before each run (first conversation always kept; newest conversations added while the token sum - from recorded usage - fits).

> **Note:** `_run_agrinet_with_failover` and `_run_agrinet_once` (the non-streaming variants) in `app/services/chat.py` have no callers; the streaming variants are the live path.

---

## 8. Response Handling

### 8.1 Parse strategy

- **Main agent**: `output_type=str`; no JSON parsing. The final text is assembled from streamed `TextPartDelta` events.
- **Moderation**: `PromptedOutput(QueryModerationResult)` - the model is prompted to return a compact JSON object which pydantic validates against the `Literal` categories (`retries=2`). The returned object (not a tuple) is used directly (`.category`, `str()`).
- **Suggestions** (disabled): `PromptedOutput(List[str])`.

### 8.2 Streaming algorithm (`_run_agrinet_once_streaming`)

1. `agrinet_agent.run_stream_events(...)` is iterated inside the timeout.
2. A `FinalResultEvent` marks that subsequent text deltas are final-answer text; text emitted before a tool call is ignored. A `function_tool_result` event resets that flag, so text between tool rounds is not streamed.
3. Each visible delta is stripped of `<think>` content (`_strip_streaming_thinking_chunk`) and pushed to an `asyncio.Queue`; `stream_chat_messages` yields chunks as raw text from the queue (the response is declared `text/event-stream` in `app/routers/chat.py`, and chunks are yielded as plain strings rather than `data:`-framed events).
4. On completion, `AgentRunResultEvent` provides the result; output text = sanitized streamed text, falling back to `result.output`.
5. Usage tokens (`result.usage()`) are written to the Langfuse generation (`input_tokens`, `output_tokens`).
6. If no `AgentRunResultEvent` arrives, `RuntimeError("Agrinet stream finished without a final result")` is raised.

### 8.3 Return shape and batch-level behaviour

- Internal return: `_AgrinetCompletedRun(result, output_text)` plus the final `AgrinetRouteDecision` and a `fallback_used` flag.
- The HTTP response is a single streaming body (`StreamingResponse`) with header `X-QID` exposing the question id.
- **Image-analysis turns** (`defer_npss_output`): chunks are not streamed live; after the run, `post_process_npss_response` rewrites the text (translates body lines via `BhashiniTranslator`, localizes the Pest/Crop/Cause labels, strips model-written source/follow-up, appends a fixed `Source: NPSS` line) and the whole result is yielded at once. If the Bhashini translation call fails, the untranslated text is returned (soft degrade, logged as a warning in `app/services/npss_response.py`).
- Stored history: `new_messages()` from the run, thinking parts removed, last text part replaced with the exact text the farmer saw (`_replace_last_text_output`), written back with `update_message_history` and the route TTL refreshed.
- There is no batching of multiple user queries per request; one request is one turn.

### 8.4 Moderation response shape

`QueryModerationResult.__str__` renders `**Moderation Compliance:** <action> (<Category Title Case>)`, which is the exact string appended to the next user message.

---

## 9. Langfuse Tracing

Implemented (verified). Files: `helpers/langfuse_helper.py`, `helpers/langfuse_tracing.py`, `helpers/langfuse_trace_schema.py`, `helpers/langfuse_scores.py`, `app/services/chat_turn_map.py`.

| Concern | Implementation |
|---------|----------------|
| Client init | `helpers/langfuse_helper.py` sets `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` (falls back to `LANGFUSE_BASE_URL`) and `LANGFUSE_TRACING_ENVIRONMENT` (defaults to `ENVIRONMENT`) in `os.environ`, then creates `Langfuse()` |
| Trace structure | `chain.chat` (`@observe`, root, name overridable by `LANGFUSE_TRACE_NAME` / `LANGFUSE_TRACE_ROOT_NAME`, default `bharat-vistaar-chat` for the trace) -> `agent.moderation` -> `agent.vistaar` -> `tool:*` (constants in `langfuse_trace_schema.py`) |
| Propagated attributes | `user_id`, `session_id`, metadata (`source_lang`, `target_lang`, `environment`, `channel`, `query` truncated to 200 chars, `qid`, `agrinet_route`, `agrinet_model_name`, `route_source`, `fallback_used`), tags `env:`, `channel:`, `model:`, `moderation:` plus the moderation category |
| Generation fields | `lf_update_current_observation` sets `model`, `usage_details` (prompt/completion/total tokens) so Langfuse cost/model dashboards work |
| Feedback scores | `POST /api/telemetry/feedback` (`app/routers/telemetry.py`) looks up the qid -> trace mapping and writes a categorical score `user-feedback` with idempotent `score_id=user-feedback:<qid>` (`helpers/langfuse_scores.py`) |
| qid -> trace map | `write_chat_turn_map` stores `chat_turn:<qid>` in Redis (TTL 2 h) with trace id, session id, model name, route, channel (`app/services/chat_turn_map.py`). Feedback arriving after expiry is skipped (`langfuse_score: "skipped"`) |
| Flush | `get_client().flush()` at the end of `_finalize_chat_turn` |
| Disabling | Not documented in code; empty keys are exported as empty strings. Behaviour of the Langfuse SDK with empty credentials is not verified in this repository |

Note that user query text and model inputs/outputs (full user message including conversation prefix) are sent to Langfuse as observation input/output.

---

## 10. Retrieval and Embeddings

| Component | File | Details |
|-----------|------|---------|
| Marqo hybrid search | `agents/tools/search.py` | `search_method=hybrid`, `retrievalMethod=disjunction`, `rankingMethod=rrf`, `alpha=0.5`, `rrfK=60`, client timeout 10 s |
| Scheme chunk search (network) | `agents/tools/search.py::search_schemes`, `helpers/scheme_qdrant_search.py` | Query goes to the BAP; local helpers resolve scheme code, classify intent (eligibility / application / support / exclusion), and format chunks |
| Embedding model | `helpers/scheme_qdrant_search.py` | `intfloat/multilingual-e5-large` (`DEFAULT_EMBEDDING_MODEL`), loaded with `sentence-transformers`; baked into the Docker image (`Dockerfile`, `HF_HOME=/opt/hf-cache`); pre-warmed at startup when `QDRANT_URL` is set (`main.py` lifespan -> `warm_scheme_search`) |
| Qdrant | `helpers/scheme_qdrant_search.py`, `helpers/video_qdrant_search.py` | Used directly for eNAM/training video search (`type=video` filter); remote Qdrant requires `QDRANT_API_KEY` |
| Master catalog | `helpers/master_catalog.py` | Redis snapshot `master-catalog:<dev|live>:snapshot` (or `MASTER_CATALOG_REDIS_KEY`) written by an external docs-pipeline; fails open (returns `None` -> empty scheme list) |

---

## Recommended (not yet implemented)

- Validate `target_lang` against the available `assets/prompts/agrinet_*.md` files and fall back to a default prompt.
- Add a hard code gate for non-`valid_agricultural` moderation results (skip tools) in addition to prompt-level enforcement.
- Extract the PM-KISAN instalment rule into tool data rather than prompt text, and render the vector-scheme block in all language prompts.
- Add a key-rotation path (secret manager and periodic client refresh) for LLM and Langfuse credentials.

## Source Files

`agents/agrinet.py`, `agents/deps.py`, `agents/models.py`, `agents/moderation.py`, `agents/suggestions.py`, `agents/tools/__init__.py`, `agents/tools/*.py`, `app/services/chat.py`, `app/services/agrinet_routing.py`, `app/services/chat_turn_map.py`, `app/services/npss_response.py`, `app/utils.py`, `app/config.py`, `helpers/utils.py`, `helpers/langfuse_*.py`, `helpers/master_catalog.py`, `helpers/scheme_qdrant_search.py`, `helpers/video_qdrant_search.py`, `assets/prompts/*.md`, `main.py`.
