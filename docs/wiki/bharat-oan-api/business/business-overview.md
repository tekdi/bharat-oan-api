# Business Overview

Related pages: [Users and Personas](./users-and-personas.md) | [Workflows](./workflows.md) | [Business Rules](./business-rules.md) | [AI Context](../ai-context.md) | [Architecture Overview](../architecture/architecture-overview.md) | [AI / LLM Integrations](../integrations/ai-llm.md) | [Third-Party Integrations](../integrations/third-party-integrations.md) | [Technical Debt](../knowledge/technical-debt.md)

---

## 1. Purpose and Problem Statement

`bharat-oan-api` (application title "Bharatvistaar AI API", `app/config.py::Settings.app_name`) is the backend of **Bharat Vistaar**, an AI-powered agricultural advisory and farmer-services assistant for Indian farmers. The FastAPI description in `main.py` reads "AI-powered Voice Assistant API for Agricultural Support", and the system prompts (`assets/prompts/agrinet_en.md`) describe the product as "your digital farming assistant - built by the Ministry of Agriculture and Farmers Welfare, India, as part of the Bharat Vistaar Grid", powered by AI and Digital Public Infrastructure (DPI).

The problem it addresses, inferred from the prompts and tools:

- Farmers need trustworthy, source-cited answers on crops, pests, weather, mandi prices and government schemes, in their own language and, often, by voice.
- Scheme information and status data are spread across many government portals and provider systems. The service puts them behind one conversational interface by calling a Beckn-based Vistaar network gateway (see [Third-Party Integrations](../integrations/third-party-integrations.md#2-vistaar-network-bap--beckn-gateway)).
- General-purpose chatbots may hallucinate agricultural advice. The service enforces a **zero-fabrication, tool-grounded** policy: every factual statement must come from a tool result and carry a translated source citation (system prompt "Information Integrity" section).

The same codebase also serves partner channels. The JWT `channel` / `client_code` claim (`app/routers/token.py`, `app/auth/jwt_auth.py`) and prompt/tool coverage for MahaVistaar (Maharashtra) and AmulVistaar (dairy unions) indicate a multi-network deployment under the "Bharat OAN" / AgriNet umbrella (the agent is named "Vistaar Agent" and the code uses the term `agrinet` throughout).

> **Note:** The browser/mobile front end is not part of this repository. This service exposes HTTP APIs (chat streaming, speech, image upload, telemetry, token issuance) that Bharat Vistaar clients call. See [Frontend](../application/frontend.md).

---

## 2. Scope (Implemented Today)

Each row is traced to code. Technical depth is in the linked pages.

| # | Capability | What it does | Evidence in code |
|---|-----------|--------------|------------------|
| 1 | Conversational agricultural advice (streaming) | Answers farmer questions in the selected language, streaming text | `app/routers/chat.py` (`GET /api/chat/`), `app/services/chat.py::stream_chat_messages`, `agents/agrinet.py` |
| 2 | Multilingual responses (13 languages) | One system prompt per language: English, Hindi, Assamese, Bengali, Gujarati, Kannada, Maithili, Malayalam, Marathi, Odia, Punjabi, Tamil, Telugu | `assets/prompts/agrinet_{en,hi,as,bn,gu,kn,mai,ml,mr,or,pa,ta,te}.md`, `agents/tools/terms.py::SUPPORTED_LANGS`, `tests/test_languages.py` |
| 3 | Query moderation | Classifies each message into 8 categories; non-agricultural, unsafe, political, prompt-injection and similar queries get a fixed refusal | `agents/moderation.py`, `assets/prompts/moderation_system.md`, `app/services/chat.py::_run_moderation` |
| 4 | Government scheme information | 16 legacy schemes (`get_scheme_info`), vector-indexed schemes from a live catalog (`search_schemes`), MahaVistaar and AmulVistaar cross-network schemes | `agents/tools/scheme_info.py`, `search.py`, `maha_vistaar.py`, `amul_vistaar.py`, `helpers/master_catalog.py` |
| 5 | Real-time scheme status (OTP-verified) | PM-KISAN, PMFBY (policy / claim), Soil Health Card, SMAM, AIF (loan and grievance tracking) | `pmkisan_scheme_status.py`, `pmfby_scheme_status.py`, `shc_scheme_status.py`, `smam_scheme_status.py`, `aif.py` |
| 6 | Grievance filing and tracking | File and track PM-KISAN and PMFBY grievances; track AIF grievances (no filing) | `pmkisan_grievance.py`, `pmfby_grievance.py`, `aif.py` |
| 7 | Weather forecast | Forecast by coordinates (India Meteorological Department data via the network) | `agents/tools/weather.py` |
| 8 | Mandi (market) prices | Commodity prices by place, commodity and date | `agents/tools/mandi.py`, `commodity.py`, `assets/commodity_codes.json` (543 commodities) |
| 9 | Official fertilizer recommendation (GFR) | Government fertilizer quantities per crop using Soil Health Card data | `agents/tools/gfr.py` |
| 10 | Seed availability and dealers (SATHI) | Certified seed stock and dealers (Maharashtra districts per prompt) | `agents/tools/sathi_seed.py` |
| 11 | Crop, livestock and pest/disease knowledge | Hybrid vector search over Package-of-Practices style documents; separate index for crop pests/diseases | `agents/tools/search.py::search_documents`, `search_pests_diseases` |
| 12 | Agricultural term lookup | Fuzzy glossary across 14 language fields (5,140 terms) | `agents/tools/terms.py`, `assets/glossary_terms.json` |
| 13 | Training videos | eNAM / training video search (Qdrant) | `agents/tools/search.py::search_video`, `helpers/video_qdrant_search.py` |
| 14 | Crop image pest/disease analysis | Upload a photo; NPSS identifies pest/crop/cause; result localized with deterministic source line | `app/routers/image.py`, `app/core/image_storage.py`, `agents/tools/npss.py`, `app/services/npss_response.py` |
| 15 | Geocoding | Place name to coordinates and back (India) | `agents/tools/maps.py`, `app/core/npss_geocodes.py` |
| 16 | Speech-to-text | Bhashini ASR or OpenAI Whisper | `app/routers/transcribe.py`, `helpers/transcription.py` |
| 17 | Text-to-speech | Bhashini TTS | `app/routers/tts.py`, `helpers/tts.py` |
| 18 | Authentication token issuance | Guest JWT, API-key JWT for partner clients, Google Play Integrity JWT | `app/routers/token.py`, `app/auth/jwt_auth.py` |
| 19 | Telemetry and feedback | Question / answer / error / ASR / TTS events to an observability service; user feedback relayed and scored in Langfuse | `helpers/telemetry.py`, `app/routers/telemetry.py`, `app/tasks/telemetry.py`, `helpers/langfuse_scores.py` |
| 20 | LLM observability | Langfuse traces with model, token usage, route and moderation tags | `helpers/langfuse_*.py`, `app/services/chat.py` |
| 21 | Optional model A/B routing and failover | Weighted split between a primary model and a Gemma/vLLM model with capacity gate and failover | `app/services/agrinet_routing.py`, `agents/models.py` |
| 22 | Health probes | Liveness, readiness (Redis check), health | `app/routers/health.py` |
| 23 | Conversation memory | Per-session history kept in Redis for 2 hours | `app/utils.py` (`HISTORY_SUFFIX`, `DEFAULT_CACHE_TTL`) |

Workflow-level detail is in [Workflows](./workflows.md); rule-level detail is in [Business Rules](./business-rules.md).

---

## 3. Non-Scope

Verified absent from the code (or explicitly excluded by prompt/tool design):

| Item | Evidence |
|------|----------|
| Persistent database or user accounts / farmer profiles | No ORM, models or migrations; state is Redis (history, caches) and a temp image folder. Identity is whatever the JWT claims carry (`app/auth/jwt_auth.py`) |
| Chat history beyond 2 hours | `DEFAULT_CACHE_TTL = 60*60*2` in `app/utils.py` |
| Filing grievances for schemes other than PM-KISAN and PMFBY | Prompt "Grievance Management"; AIF is tracking-only (`agents/tools/aif.py` has no submit tool) |
| Offering status checks for KCC, PMKSY, SATHI, PMASHA, PDMC, FFS, NBHM | Prompt "When to offer status checks" |
| Handling non-agricultural, political, unsafe or prompt-injection requests | Moderation policy and refusal templates |
| Answers from model memory / general advice | Prompt "Always use tools" and "Zero fabrication policy" |
| Pointing farmers to external websites or apps when a tool fails | Prompt "Graceful tool failures" |
| SATHI seed data outside Maharashtra | Prompt "Geographic scope" |
| Payments, subsidy disbursement, applications submission to schemes | No tools exist; scheme tools are read-only information / status / grievance |
| Suggested follow-up questions endpoint | Agent, router and task exist but are disabled (`main.py`, `app/routers/__init__.py`, `app/tasks/__init__.py`) |
| ElevenLabs TTS and Google Cloud Translation | Declared in schema/helpers but not callable (see [Third-Party Integrations](../integrations/third-party-integrations.md#6-declared-but-unused-integrations)) |
| Rate limiting | `rate_limit_requests_per_minute` is declared in `app/config.py` but not applied anywhere |
| Front-end application, admin console, CMS for prompts | Not in repository; prompts are files in `assets/prompts/` |
| Content ingestion / document pipeline | Performed by an external "docs-pipeline" that writes the master catalog snapshot to Redis (`helpers/master_catalog.py`) |

> **Note:** Production infrastructure (load balancers, Kubernetes, backups) is not documented in this wiki; see [Infrastructure](../operations/infrastructure.md) and [Deployment](../operations/deployment.md) for what the repository does contain (Dockerfile, Docker Compose, supervisord, a GitHub Actions image-build workflow).

---

## 4. Reconciliation with Raw Business Documents

> **Note:** The repository has no `docs/raw_documents/` folder. The only product-style artifacts are `API_DOCUMENTATION.md`, `README.md` (Docker commands only) and `docs/data/archive/CROP ADVISORY SOP.docx`. These are reconciled below; the wiki always documents what the code does today.

### 4.1 `docs/data/archive/CROP ADVISORY SOP.docx`

A Word file kept in the archive folder. The text that can be extracted without a Word viewer begins with a Marathi-language directory of officials and contact numbers of a state agricultural university (Marathwada region). It was not used as a source for any behaviour documented in this wiki, and no code references it. It is listed in [References](../knowledge/references.md) and [Legacy Artifacts](../knowledge/legacy-artifacts.md) for completeness.

### 4.2 `API_DOCUMENTATION.md` versus code

| Statement in `API_DOCUMENTATION.md` | Verified implementation |
|------------------------------------|-------------------------|
| JWT authentication "currently disabled for testing" | `get_current_user` (`app/auth/jwt_auth.py`) always validates the RS256 signature. `DISABLE_JWT` exists in `app/config.py` but is not used by any code |
| `POST /api/token` token "expires in 30 days" (`expires_in: 2592000`) | Guest token expiry is `JWT_EXPIRY_MINUTES` (default 15 minutes) - `_build_guest_auth_payload` in `app/routers/token.py` |
| `POST /api/token` body fields `mobile`, `name`, `role`, `metadata` (defaults `1111111111`, `guest`, `public`) | Request model also accepts `fingerprint_id`; `mobile` is optional and not defaulted; `name` defaults to `guest`, `role` to `public`. Two further endpoints exist: `/api/token/api-key`, `/api/token/play-integrity` |
| `GET /api/suggest/` returns suggestions | The router is not included in `main.py` (commented out); the endpoint is not callable |
| `POST /api/tts/` supports `bhashini` and `eleven_labs` | Router accepts only `bhashini`; `eleven_labs` returns HTTP 400 |
| `GET /api/chat/` parameters `query`, `session_id`, `source_lang`, `target_lang`, `user_id` | Also accepts `qid`, `latitude`, `longitude`; there is an additional `POST /api/chat/analyze-image` (deprecated) and `/api/image/*` upload endpoints |
| Rate limit of 1000 requests per minute | Setting is declared but not enforced by middleware or dependencies |
| Languages: `en`, `hi`, `mr` "and others" | 13 languages have system prompts (see Scope row 2) |
| Health endpoints `/api/health/`, `/live`, `/ready` | Matches `app/routers/health.py` |
| `GET /api/file/{file_hash}` | Matches `app/routers/file.py`; content comes from the SHC report cache (10 minute TTL) |

The authoritative endpoint list is [API Specification](../application/api-specification.md).

---

## 5. Business Context Summary

| Dimension | Summary |
|-----------|---------|
| Owner / program | Ministry of Agriculture and Farmers Welfare, India - Bharat Vistaar Grid (per system prompts); CI builds the image named `bharat-oan-api` (`.github/workflows/build.yml`) |
| Channels | Public web/app (`BharatVistaar` default channel), partner clients via API-key `client_code`, Android clients via Play Integrity |
| Value | Single multilingual, voice-capable entry point to advisory, scheme and service-status information with official sources |
| Trust model | Tool-grounded answers, mandatory source citation, moderation of out-of-scope content |
| Key dependencies | Vistaar BAP gateway, LLM provider, Bhashini, Redis, Marqo/Qdrant, NPSS, Langfuse (see [Third-Party Integrations](../integrations/third-party-integrations.md)) |

## Source Files

`main.py`, `app/config.py`, `app/routers/*.py`, `app/services/chat.py`, `app/auth/jwt_auth.py`, `agents/**`, `assets/prompts/*.md`, `helpers/**`, `API_DOCUMENTATION.md`, `README.md`, `docs/data/archive/CROP ADVISORY SOP.docx`.
