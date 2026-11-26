# Users and Personas

Related pages: [Business Overview](./business-overview.md) | [Workflows](./workflows.md) | [Business Rules](./business-rules.md) | [API Specification](../application/api-specification.md) | [Third-Party Integrations](../integrations/third-party-integrations.md) | [Technical Debt](../knowledge/technical-debt.md)

> **Note:** The repository contains no persona or product-requirements document. Personas below are **inferred from code** (token types, prompts, endpoints, tooling) and are labelled accordingly.

---

## 1. Personas (Inferred from code)

### 1.1 Product personas

| Persona | Who | Goals | How they reach the service | Evidence |
|---------|-----|-------|----------------------------|----------|
| **Farmer (end user)** | Indian farmer, often low-literacy, regional-language speaker, possibly voice-first | Get simple, trustworthy answers on crops, pests, weather, mandi prices; check scheme status; raise grievances; find seed dealers | Bharat Vistaar client app/web calling `GET /api/chat/`, `POST /api/transcribe/`, `POST /api/tts/`, `POST /api/image/upload` | Prompt tone ("Farmer-friendly language", short answers, no jargon), ASR/TTS endpoints, 13 language prompts |
| **Guest / anonymous visitor** | Farmer using the public chat without login | Ask questions immediately | `POST /api/token` returns a guest JWT (`is_guest_user=true`, `sub=guest:<fingerprint_id or anon>`) | `_build_guest_auth_payload` in `app/routers/token.py` |
| **Farmer with a registered mobile / identifiers** | Farmer who knows PM-KISAN registration number, PMFBY registered mobile, SHC mobile, AIF beneficiary ID, SMAM application number | Check scheme status and file grievances | Supplies identifiers and OTPs inside the chat; the service forwards them to the network | Status/grievance tools in `agents/tools/` |
| **Verified Android user** | Farmer using a partner Android app | Obtain a token only from a genuine, Play-recognized app/device | `POST /api/token/play-integrity` | `app/routers/token.py` (package name, nonce, freshness, `PLAY_RECOGNIZED`, `MEETS_DEVICE_INTEGRITY`) |

### 1.2 Partner and system personas

| Persona | Role | Evidence |
|---------|------|----------|
| **Partner client application** (e.g. state or union portals) | Backend or app that holds a per-client API key and obtains JWTs for its users, with its own `channel` / `client_code` | `POST /api/token/api-key` with `X-API-Key` and `client_code`; env `API_KEY_AUTH_TOKEN_<CLIENT>` |
| **Vistaar network providers (BPPs)** | Government or partner systems answering Beckn calls (schemes, weather, mandi, GFR, SATHI, AIF, PMFBY, PM-KISAN, SHC, SMAM, MahaVistaar, AmulVistaar) | `agents/tools/*` payloads, `BPP_ID` / `BPP_URI` |
| **Content / data pipeline team** | Publishes the scheme catalog snapshot that the API reads live from Redis | `helpers/master_catalog.py` (docs-pipeline) |

### 1.3 Operational roles

| Role | Typical activities | Evidence |
|------|--------------------|----------|
| **Backend / AI engineer** | Edit prompts in `assets/prompts/`, tools in `agents/tools/`, model env vars, routing weights | Repository structure, `agents/models.py` |
| **Platform / DevOps** | Build image, manage `.env` secrets, Redis, supervisord workers, key files (`jwt_public_key.pem`, `private_key.pem`) | `Dockerfile`, `docker-compose.yml`, `supervisord.conf`, `.github/workflows/build.yml`, `docs/setup/redis.md` |
| **AI quality / observability analyst** | Review Langfuse traces, user-feedback scores, moderation tags, A/B route metadata | `helpers/langfuse_*.py`, `/api/telemetry/feedback` |
| **Data/analytics consumer of telemetry** | Consumes question/answer/error/ASR/TTS events from the observability service | `helpers/telemetry.py`, `app/tasks/telemetry.py` |
| **QA engineer** | Runs script-style moderation and agent evaluation against fixtures | `tests/test_moderation_single_turn.py`, `tests/test_agrinet_single_turn.py`, `tests/fixtures/*.json` |

---

## 2. Access Permissions (verified in code)

### 2.1 Authentication mechanism

- **Bearer JWT, RS256.** `app/auth/jwt_auth.py` loads `JWT_PUBLIC_KEY_PATH` (default `jwt_public_key.pem`) at import time - the process fails to start if the file is missing - and `get_current_user` validates the signature on every protected request via `OAuth2PasswordBearer(tokenUrl="token")`. Audience and issuer are **not** verified (`verify_aud=False`, `verify_iss=False`). Expiry is verified.
- **No environment bypass.** The `DISABLE_JWT` / `DISABLE_JWT_FOR_LOCAL` settings in `app/config.py` are not read by any code; `get_current_user` always validates (the file comment says "no env-based bypass").
- **Tokens without a mobile number are accepted.** The check that would reject them is commented out in `get_current_user` (only a warning is logged).
- **Claims exposed to handlers**: `sub`, `mobile`, `name`, `email`, `role`, `farmer_id`, `unique_id`, `locations`, `channel` (default `BharatVistaar`), `client_code`, `auth_source`, `is_guest_user`, `telemetry_context`, `metadata`.

### 2.2 Token issuance endpoints (`app/routers/token.py`)

| Endpoint | Caller must present | Result |
|----------|---------------------|--------|
| `POST /api/token` | **Nothing** (no authentication) | Guest JWT valid for `JWT_EXPIRY_MINUTES` (default 15). The caller may supply `mobile`, `name`, `role`, `fingerprint_id`, `metadata`; they are written into the token claims. `auth_source=guest_token`, `channel=BharatVistaar` |
| `POST /api/token/api-key` | Header `X-API-Key` and body `client_code`; compared with `API_KEY_AUTH_TOKEN_<CLIENT_CODE>` (or `API_KEY_AUTH_TOKEN`) using `hmac.compare_digest` | JWT with `channel=client_code`; the caller may add arbitrary extra `claims` (merged into the payload before `iat`/`exp` are set) |
| `POST /api/token/play-integrity` | A Google Play Integrity token, nonce and `client_code` | JWT with `mobile` (or `verified_device`), `name`/`role` from `PLAY_INTEGRITY_DEFAULT_NAME` / `PLAY_INTEGRITY_DEFAULT_ROLE` (defaults `android_user` / `public`); nonce single-use (Redis), token freshness `PLAY_INTEGRITY_FRESHNESS_SECONDS` (120 s) |

### 2.3 Endpoint access matrix

| Endpoint | Requires JWT? | Notes |
|----------|---------------|-------|
| `GET /`, `GET /api/health/`, `/live`, `/ready` | No | Health returns Redis status |
| `POST /api/token`, `/api/token/api-key`, `/api/token/play-integrity` | No (own credentials for the latter two) | See above |
| `GET /api/chat/`, `POST /api/chat/analyze-image` | Yes | Any valid token |
| `POST /api/image/upload` | Yes | Any valid token |
| `GET /api/image/{image_id}` | **No** | Documented in code as relying on unguessable UUIDs; the NPSS tool downloads from this URL without a token |
| `POST /api/transcribe/`, `POST /api/tts/` | Yes | Any valid token |
| `POST /api/telemetry/feedback`, `/events`, `/error` | Yes | Any valid token |
| `GET /api/file/{file_hash}` | **No** | Serves cached Soil Health Card HTML by hash (10 min TTL) |
| `GET /api/suggest/` | Would require JWT | Router is not registered (disabled) |

### 2.4 Authorization (roles)

> **Note:** Sign-in with a valid token is required for the chat, speech, image-upload and telemetry endpoints. All authenticated callers currently share the same level of access. The `role` claim (for example `public`) is carried in the token and returned by `get_current_user`, but no endpoint or tool checks it; role-based restrictions, per-farmer data ownership checks and scope-based permissions are not part of this service today.

Practical consequences, all verified in code:

1. Anyone who can reach `POST /api/token` can obtain a valid guest JWT; the caller-chosen `mobile`, `name`, `role` and `metadata` are not verified. Telemetry identity (`resolve_telemetry_identity` in `helpers/telemetry.py`) prefers the token's `mobile` as the user id, so that value is client-asserted for guest tokens.
2. Session ownership is not enforced: `session_id` is a client-supplied string used to load history (`get_session_history` in `app/routers/chat_query_utils.py`); the code does not bind a session to a user id.
3. Feedback is matched to a chat trace by `qid` and a session check only (comment in `app/routers/telemetry.py`: "The turn map stores no user id").
4. Farmer identifiers for scheme status (registration numbers, phone numbers, OTPs) are supplied by the person chatting and validated by the downstream government systems via OTP, not by this service.
5. CORS defaults to `ALLOWED_ORIGINS="*"` with credentials allowed (`app/config.py`, `main.py`).

These points are recorded as debt/risk items in [Technical Debt](../knowledge/technical-debt.md).

---

## 3. Reconciliation: product vision versus implementation

| Product-level expectation (inferred) | Implementation today |
|--------------------------------------|----------------------|
| Farmer-specific services (status, grievances) imply an identified farmer | Identification happens through OTP with the downstream portal, not through this service's login |
| Partner channels (state / union portals) imply per-partner isolation | A `channel` claim is propagated to telemetry and Langfuse tags only; no per-channel permissions or data separation |
| Logged-in versus guest experience | Both use the same endpoints with the same access; `is_guest_user` is carried but unused for authorization |
| `API_DOCUMENTATION.md` says JWT is "disabled for testing" | JWT is always enforced (see [Business Overview](./business-overview.md#42-api_documentationmd-versus-code)) |

## Source Files

`app/auth/jwt_auth.py`, `app/routers/token.py`, `app/routers/chat.py`, `app/routers/image.py`, `app/routers/file.py`, `app/routers/telemetry.py`, `app/routers/health.py`, `app/config.py`, `helpers/telemetry.py`, `app/routers/chat_query_utils.py`, `main.py`.
