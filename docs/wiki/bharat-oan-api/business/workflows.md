# Workflows

Business workflows of `bharat-oan-api`, identified W1 to W10. This page gives triggers, endpoints and narrative steps. **Diagrams are not repeated here**; the canonical sequence and flow diagrams (including flowcharts F1 chat lifecycle and F2 startup) live in [Sequence & Workflow Diagrams](../architecture/sequence-diagrams.md).

Related pages: [Business Overview](./business-overview.md) | [Users and Personas](./users-and-personas.md) | [Business Rules](./business-rules.md) | [API Specification](../application/api-specification.md) | [AI / LLM Integrations](../integrations/ai-llm.md) | [Third-Party Integrations](../integrations/third-party-integrations.md)

## Workflow Index

| ID | Workflow | Trigger / endpoint | Actor |
|----|----------|--------------------|-------|
| W1 | Authentication and token issuance | `POST /api/token`, `/api/token/api-key`, `/api/token/play-integrity` | Client |
| W2 | Streaming chat turn (including tool-driven sub-flows) | `GET /api/chat/` | Farmer |
| W3 | Image upload and analysis | `POST /api/image/upload`, then `GET /api/chat/`; `GET /api/image/{id}` | Farmer |
| W4 | Transcribe (speech-to-text) | `POST /api/transcribe/` | Farmer client |
| W5 | TTS (text-to-speech) | `POST /api/tts/` | Farmer client |
| W6 | Telemetry and feedback | `POST /api/telemetry/feedback`, `/events`, `/error` | Client |
| W7 | Suggestions (disabled) | `GET /api/suggest/` (not registered) | n/a |
| W8 | Model routing and failover | Internal, every chat turn | System |
| W9 | HTML file serving | `GET /api/file/{file_hash}` | Farmer client |
| W10 | Health probes | `GET /api/health/`, `/live`, `/ready`, `GET /` | Platform |

All endpoints are prefixed `/api` (`settings.api_prefix`). Only the token endpoints, W9, W10 and the image-serving URL are reachable without a JWT; see [Users and Personas - Access Permissions](./users-and-personas.md#2-access-permissions-verified-in-code).

---

## W1: Authentication and token issuance

Three ways to obtain a JWT (`app/routers/token.py`). Tokens are RS256, signed with the private key at `JWT_PRIVATE_KEY_PATH` (default `private_key.pem`) and validated with the public key by `app/auth/jwt_auth.py`.

### W1.a Guest token

- **Endpoint:** `POST /api/token` (`create_auth_token`), optional JSON body (`mobile`, `name`, `role`, `fingerprint_id`, `metadata`).
- **Steps:** (1) parse metadata (JSON string or object; `surface` defaults to `public_chat`, `fingerprint_id` copied in); (2) build the guest payload (`sub=guest:<fingerprint_id|anon>`, `role` default `public`, `channel=BharatVistaar`, `is_guest_user=true`, `telemetry_context`); (3) sign; (4) return `{token, expires_in}` with expiry `JWT_EXPIRY_MINUTES` (default 15 minutes).
- **Failure:** HTTP 500 if the private key is not loaded.

### W1.b Partner token (API key)

- **Endpoint:** `POST /api/token/api-key` with header `X-API-Key` and body `{client_code, claims?}`.
- **Steps:** require `client_code`; resolve the key from `API_KEY_AUTH_TOKEN_<CLIENT_CODE>` (normalized upper-case, `-`/space to `_`) or `API_KEY_AUTH_TOKEN`; constant-time compare; issue a JWT with `channel=client_code` and expiry `JWT_EXPIRY_MINUTES`.
- **Failure:** 400 (missing client code), 401 (wrong key), 500 (missing key configuration).

### W1.c Play Integrity token

- **Endpoint:** `POST /api/token/play-integrity` with `{integrity_token, nonce, client_code, mobile?}`.
- **Steps:** resolve the package name for the client; obtain a Google access token from the client's service account; call `decodeIntegrityToken`; validate package, nonce, timestamp freshness (120 s default, 10 s future skew), `PLAY_RECOGNIZED`, `MEETS_DEVICE_INTEGRITY`; reject a reused nonce (Redis `integrity_nonce:<nonce>`); issue a JWT (`mobile` or `verified_device`).
- **Failure:** 401 on any failed verdict, 502 when Google is unreachable or rejects, 503 when replay protection (Redis) is unavailable.

### W1.d Token validation on every protected call

`get_current_user` verifies the signature and expiry, maps claims to a user dict (`channel` defaults to `BharatVistaar`) and returns it to the handler. No role is evaluated.

---

## W2: Streaming chat turn

The core workflow. See flowchart F1 in [sequence diagrams](../architecture/sequence-diagrams.md).

- **Trigger:** farmer sends a question.
- **Endpoint:** `GET /api/chat/` (`app/routers/chat.py`) with query parameters `query`, `session_id?`, `qid?`, `source_lang` (default `hi`), `target_lang` (default `hi`), `user_id` (default `anonymous`), `latitude?`, `longitude?`. The response is a streaming body (`text/event-stream`) with header `X-QID`.
- **Steps (`app/services/chat.py::stream_chat_messages`):**
  1. Load session history from Redis (new UUID session if none) and normalize the query (image-id detection, see W3).
  2. Queue a "question" telemetry event (background task).
  3. Resolve the model route (W8).
  4. Open the Langfuse trace (`chain.chat`) with session, user, language, channel and model tags.
  5. Build the user message: last 3 user/assistant pairs, the query and the selected language.
  6. **Moderation** (`agent.moderation`): classify; the result line is appended to the user message.
  7. Trim history to about 64,000 tokens and strip thinking parts.
  8. Run the main agent (`agent.vistaar`) with tools, streaming visible text deltas to the client; 45 s total timeout (`AGRINET_MODEL_TIMEOUT_SECONDS`).
  9. On success: queue the "answer" telemetry event, save updated history to Redis (2 h TTL), refresh the route pin TTL, flush Langfuse.
  10. On error: queue an "error" telemetry event and fail the stream.
- **Client disconnect:** if the client drops during streaming, the agent task keeps running (shielded); a detached finalizer waits up to 180 s, then sends the answer telemetry inline, saves history and refreshes the route TTL. If the run does not finish in time, the turn is dropped and logged.
- **Rules applied:** [Business Rules - moderation, tool selection, response style](./business-rules.md).

### W2 sub-flows (tool-driven, selected by the model per the system prompt)

**Scheme information lookup.**
1. Legacy scheme codes (16, e.g. `kcc`, `pmkisan`, `pmfby`, `shc`, `nbm`) call `get_scheme_info` (Beckn `schemes-agri` search).
2. Vector-indexed schemes from the live catalog (for example MIF, PM-KMY, CDP, Pulses Mission, Cotton Mission, NMEO-OS, Makhana, e-NAM, RWBCIS, PKVY; the exact set depends on the Redis snapshot) call `search_schemes` with a 2-5 word English query. The tool resolves the scheme code locally, queries the BAP and formats `section=Eligibility|Exclusion|...` chunks.
3. MahaVistaar schemes (`ndksp-drip-irrigation`, `ndksp-farm-pond-lining`, `aif`) call `call_maha_vistaar_network`; Amul union schemes call `call_amul_vistaar_network`.
4. Schemes with no dedicated tool fall back to `search_documents`.
5. Eligibility questions are answered as "Who is eligible" plus "Who is not eligible" when exclusion data exists; exclusion-only questions show one section.
6. The answer ends with the translated source label and one follow-up question. For `pmkisan`, the tool appends the 23rd-instalment release section (`helpers/pmkisan_installment_release.py`).

**Scheme status check with OTP.**
- PM-KISAN: `initiate_pm_kisan_status_check(reg_no | phone_number)` sends an OTP to the registered mobile; `check_pm_kisan_status_with_otp(otp, ...)` with a 4-digit OTP.
- PMFBY: `initiate_pmfby_status_check(phone_number)` then `check_pmfby_status_with_otp(otp, phone_number, inquiry_type, year, season)`; 6-digit OTP; `inquiry_type` is `policy_status` or `claim_status`; season `Kharif`, `Rabi` or `Summer`.
- Soil Health Card: `check_shc_status(phone_number, cycle)`; the returned HTML report is cached and exposed as a link to `/api/file/{hash}` (W9).
- SMAM: `check_smam_scheme_status(search_type, search_value)` with `mobile` or `application_no`; Aadhaar is not accepted.
- AIF: `initiate_aif_otp(beneficiary_id)` then `verify_aif_otp(otp, beneficiary_id)` then `check_aif_loan_status(...)` or `check_aif_grievance_status(beneficiary_id)`; verification once per conversation.
- Common: ask for identifiers one at a time; convert native-script digits to ASCII (`to_ascii_digits`); never echo the OTP; "policy status" without a scheme prompts the farmer to choose (offer PMFBY).

**Grievance filing / tracking.**
- If the farmer does not say which scheme, ask once: PMFBY, PM-KISAN or AIF; never mix tools of different schemes.
- PM-KISAN: `pmkisan_grievance_send_otp(reg_no, purpose)` then `pmkisan_submit_grievance` (registration number, 4-digit OTP, grievance type from `assets/grievance_types.json` (10 types), description) or `pmkisan_grievance_status`; the Query ID is shared with the farmer.
- PMFBY: `initiate_pmfby_grievance_otp(phone)` then `check_pmfby_grievance_otp(otp, phone)`, then collect application number, season/year and complaint, then `pmfby_submit_grievance`; tracking uses `pmfby_grievance_status(phone, grievance_support_ticket_no)`.
- AIF: tracking only (`check_aif_grievance_status`).
- Other schemes: search scheme documents for redressal information and say plainly if none is found.

**Weather forecast.** If only a place name is known, `forward_geocode` (Photon); `weather_forecast(latitude, longitude)` calls the network category `WFC`; present today and the multi-day forecast, tie to farming activity, cite the India Meteorological Department.

**Mandi price discovery.** (1) Confirm date intent first (today / latest / specific date or range) - no tool call until confirmed; (2) confirm location to at least district level; (3) `forward_geocode`; (4) `search_commodity` (fuzzy match to the English commodity name); (5) `get_mandi_prices(lat, lon, location_name, commodity_name, price_date DD-MM-YYYY)`; (6) present commodity, market, modal/min/max prices, arrival date, variety, cite "Mandi Prices". If no data for the date, say so without substituting another date.

**Official fertilizer (GFR) recommendation.** Geocode the place; `gfr_get_crop_registries` to resolve crop ids; collect the mobile registered on the Soil Health Card and the cycle year; `gfr_get_recommendations(state_id, crops (max 6), phone_no, cycle, ...)`; cite "GFR Crop Recommendation".

**Seed availability (SATHI).** `get_sathi_crop_groups` (cached 7 days) to choose a group; `list_sathi_crops_in_group` to choose a crop code (never shown to the farmer); ask only for the district; `forward_geocode`; `search_sathi_seed_availability(crop_code, lat, lon)`; list dealers (name, contact or "Contact not listed - visit directly", stock, up to 3 varieties); cite "SATHI". Locations outside Maharashtra get a scope message.

**Crop / livestock / pest advisory and training videos.** `search_terms` (threshold 0.5, language-specific) only for crop, pest and general knowledge queries; `search_documents` for crops, seeds and livestock; `search_pests_diseases` for crop pests/diseases only; eNAM how-to questions use `search_schemes` then `search_video` and add a "Related Videos" list. Marqo results are enriched with glossary translations in brackets.

**Scheme catalog sync.** An external docs-pipeline writes `master-catalog:<tier>:snapshot` to Redis; each chat turn renders the system prompt (English, Maithili, Odia, Punjabi templates) with the current vector-scheme list, and `search_schemes` resolves schemes from the same snapshot. A Redis failure yields an empty list (fails open). The `search_schemes` tool docstring is frozen at process start.

---

## W3: Image upload and analysis

- **Trigger:** farmer uploads a photo of an affected crop.
- **Endpoints:** `POST /api/image/upload` (JPEG/PNG, multipart field `image`) returns `{image_id}`; then `GET /api/chat/` with a query containing the image (a bare UUID, `[IMAGE_ID: <uuid>]`, or a `/api/image/<uuid>` URL). `GET /api/image/{image_id}` serves the stored file (used by the NPSS tool; no JWT). A deprecated one-shot alternative is `POST /api/chat/analyze-image`.
- **Steps:**
  1. Upload validates MIME type (400) and size (413, default limit 10 MB), stores the file under `TEMP_UPLOAD_DIR` and returns the id.
  2. `normalize_chat_query` rewrites the message to `<prompt>\n\n[IMAGE_URL: <BASE_URL>/api/image/<id>]` and flags image analysis.
  3. Moderation runs, then the prompt wrapper (`_wrap_image_analysis_message`) instructs the agent to call `analyze_crop_image` immediately, not to ask for location and not to give treatment advice.
  4. `analyze_crop_image` obtains an NPSS token (cached 25 min), resolves a location (browser coordinates in `FarmerContext` when present, otherwise the Krishi Vigyan Kendra Delhi fallback), downloads the image, validates it, calls NPSS `analyze-image`, marks the image processed and deletes it (or moves it to the GCS mount when `GCS_MOVE_AFTER_PROCESS` is set).
  5. Because the turn is flagged as image analysis, the answer is not streamed live; `post_process_npss_response` strips the model's source line and trailing question, translates and localizes the Pest / Crop / Cause card (Bhashini; untranslated if that fails) and appends a fixed `Source: NPSS` line, then the full text is yielded.
- **Reconciliation notes:** the `[NPSS_LOCATION_REQUIRED]` follow-up path (`app/core/npss_followup.py`) is still wired into `stream_chat_messages`, but the current tool no longer emits that marker (only tests do). The `FarmerContext` for a turn does not receive latitude/longitude, so the Delhi fallback location is used (see [Technical Debt](../knowledge/technical-debt.md)).

> **Note:** `cleanup_expired_images` in `app/core/image_storage.py` is not called by any scheduler in this repository, so images that were uploaded but never analyzed remain on disk until removed by other means.

---

## W4: Transcribe

- **Endpoint:** `POST /api/transcribe/` with `{audio_content (base64), lang_code, service_type ('bhashini'|'whisper'), session_id?, qid?}`.
- **Steps:** validate service type (400 otherwise); Bhashini ASR (retried up to 3 times on timeouts/429/5xx) or Whisper; return `{status, text, lang_code, session_id}`; record an ASR telemetry event in the background regardless of outcome. Errors propagate as HTTP 500.

## W5: TTS

- **Endpoint:** `POST /api/tts/` with `{text, target_lang, session_id?, service_type, qid?}`.
- **Steps:** only `bhashini` is accepted (400 otherwise; `eleven_labs` is in the request schema but rejected); female voice, 8 kHz; returns base64 audio `{status, audio_data, session_id}`; TTS telemetry event recorded in the background.

## W6: Telemetry and feedback

- **Endpoints:** `POST /api/telemetry/feedback` (qid, session_id, feedback_type, text, question/answer text), `POST /api/telemetry/events` (list of UI events), `POST /api/telemetry/error`.
- **Steps:** build the event from JWT identity claims and queue `send_telemetry` as a background task (3 retries). For feedback, read `chat_turn:<qid>` from Redis and, when the session matches, upsert a Langfuse score `user-feedback` on the original trace. Response `{status: "accepted", langfuse_score: "queued"|"skipped"}`.
- **Automatic events:** chat turns queue question, answer and error events (W2); ASR/TTS queue their own (W4, W5).

## W7: Suggestions (disabled)

- **Status:** not active. `agents/suggestions.py`, `app/routers/suggestions.py` and `app/tasks/suggestions.py` exist, but the router and task imports are commented out in `main.py`, `app/routers/__init__.py` and `app/tasks/__init__.py`. `agents/suggestions.py` imports `LLM_AGRINET_MODEL`, which `agents/models.py` does not define, so re-enabling requires a fix first.
- **Intended behaviour (inactive code):** after a chat turn, generate 3-5 follow-up questions, cache them at `suggestions_<session>_<lang>`, and serve them via `GET /api/suggest/`.

## W8: Model routing and failover

- **Steps:** routing disabled -> always route `gpt41`. Enabled -> use the session's Redis pin; for a new session choose by weights (default 70/30) and pin it; for an existing session without a pin, pin to `gpt41` ("state_repair"); if the pinned route is `gemma`, read vLLM concurrency (cached 2 s) and deflect that turn to `gpt41` when at capacity or when metrics are unavailable; on failure before any text is streamed, retry once on the other route and re-pin the session. Details: [AI / LLM - Retries, Timeouts and Failover](../integrations/ai-llm.md#7-retries-timeouts-and-failover).

## W9: HTML file serving

- **Endpoint:** `GET /api/file/{file_hash}` (no JWT). The Soil Health Card tool injects a download header into the report HTML, caches it under `html_file:<hash>` for 10 minutes and gives the farmer a URL built from `API_BASE_URL`. Expired or unknown hashes return 404.

## W10: Health probes

- **Endpoints:** `GET /api/health/live` (always alive), `/api/health/ready` and `/api/health/` (write/read a Redis test key; 503 when unhealthy; `/api/health/` also reports uptime), plus `GET /` (app info). Startup behaviour is described in flowchart F2 in [sequence diagrams](../architecture/sequence-diagrams.md).

---

## Lifecycle notes

There is no database and therefore no persisted entity lifecycle. The only lifecycles are:

| Object | States / expiry | Source |
|--------|-----------------|--------|
| JWT | issued -> expires after `JWT_EXPIRY_MINUTES` (default 15) | `app/routers/token.py` |
| Chat session history | created on first turn -> expires 2 h after last write | `app/utils.py` |
| Session model route pin | created at first turn -> expires after `AGRINET_ROUTE_TTL_SECONDS` (7200) unless refreshed after each turn | `app/services/agrinet_routing.py` |
| Uploaded image | uploaded -> processed (marked once the NPSS call starts) -> deleted/moved after NPSS success | `app/core/image_storage.py`, `agents/tools/npss.py` |
| OTP flows | OTP sent -> verified/consumed in the same conversation; session-scoped transaction id | status/grievance tools |

## Source Files

`app/routers/*.py`, `app/services/chat.py`, `app/services/agrinet_routing.py`, `app/services/npss_response.py`, `app/core/image_storage.py`, `app/core/npss_followup.py`, `agents/tools/*.py`, `assets/prompts/agrinet_en.md`, `helpers/transcription.py`, `helpers/tts.py`, `helpers/telemetry.py`, `main.py`.
