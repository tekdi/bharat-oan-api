# Sequence and Workflow Diagrams

This is the **single canonical file** for request/response sequences and workflow flowcharts of `bharat-oan-api`. Other pages link here instead of repeating the diagrams. Component and system-context views are in [component-diagram.md](component-diagram.md); the narrative of each workflow is in [../business/workflows.md](../business/workflows.md).

Related pages: [API specification](../application/api-specification.md) | [Backend](../application/backend.md) | [Caching](../integrations/caching.md) | [Async processing](../integrations/async-processing.md) | [AI/LLM](../integrations/ai-llm.md) | [Architecture overview](architecture-overview.md)

All diagrams were derived from the source files cited under each heading (status: **Implemented, verified**) unless explicitly labelled otherwise.

## Diagram Index

| ID | Diagram | Type | Primary source |
|----|---------|------|----------------|
| [W1](#w1-authentication-and-token-issuance) | Authentication and token issuance (guest, API key, Play Integrity) and token use | Sequence | `app/routers/token.py`, `app/auth/jwt_auth.py` |
| [W2](#w2-streaming-chat-turn) | Streaming chat turn (including disconnect handling) | Sequence | `app/routers/chat.py`, `app/services/chat.py` |
| [W3](#w3-crop-image-upload-and-analysis) | Crop image upload and NPSS analysis (and the location follow-up) | Sequence + flowchart | `app/routers/image.py`, `app/routers/chat_query_utils.py`, `agents/tools/npss.py` |
| [W4](#w4-voice-transcription-asr) | Voice transcription (ASR) | Sequence | `app/routers/transcribe.py` |
| [W5](#w5-text-to-speech-tts) | Text-to-speech | Sequence | `app/routers/tts.py` |
| [W6](#w6-telemetry-and-feedback) | Telemetry and user feedback | Sequence | `app/routers/telemetry.py`, `app/tasks/telemetry.py` |
| [W7](#w7-conversation-suggestions-disabled) | Conversation suggestions (disabled in the running app) | Sequence | `app/tasks/suggestions.py`, `app/routers/suggestions.py` |
| [W8](#w8-model-routing-and-failover) | Model routing, capacity gate and failover | Flowchart | `app/services/agrinet_routing.py`, `app/services/chat.py` |
| [W9](#w9-cached-html-document-serving) | Cached HTML document serving | Sequence | `app/routers/file.py`, `agents/tools/shc_scheme_status.py` |
| [W10](#w10-health-probes) | Health, liveness and readiness probes | Sequence | `app/routers/health.py` |
| [F1](#f1-chat-request-lifecycle-and-error-paths) | Chat request lifecycle and error paths | Flowchart | `app/services/chat.py` |
| [F2](#f2-service-startup) | Service startup | Flowchart | `main.py` |

> **Note:** The workflow IDs W1-W10 above are the identifiers used by the technical pages of this wiki. The business-facing catalogue in [workflows.md](../business/workflows.md) describes the same flows in prose; if its numbering differs, match by title.

---

## W1 Authentication and token issuance

Source: `app/routers/token.py` (`create_auth_token`, `create_auth_token_with_api_key`, `create_auth_token_with_play_integrity`), `app/auth/jwt_auth.py` (`get_current_user`). Endpoints: `POST /api/token`, `POST /api/token/api-key`, `POST /api/token/play-integrity`.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client app
    participant API as FastAPI token router
    participant R as Redis
    participant G as Google Play Integrity API
    participant P as Protected route

    rect rgb(235, 245, 255)
    Note over C,API: Variant A - guest token (no credential)
    C->>API: POST /api/token (optional mobile, name, role, fingerprint_id, metadata)
    API->>API: build guest payload sub=guest:fingerprint, exp=JWT_EXPIRY_MINUTES
    API->>API: sign with RS256 private key
    API-->>C: 200 token, expires_in
    end

    rect rgb(240, 255, 240)
    Note over C,API: Variant B - API key client
    C->>API: POST /api/token/api-key, header X-API-Key, body client_code, claims
    API->>API: resolve env API_KEY_AUTH_TOKEN_CLIENTCODE
    API->>API: hmac.compare_digest
    alt key mismatch
        API-->>C: 401 Invalid API key
    else key valid
        API-->>C: 200 token with channel=client_code, expires_in
    end
    end

    rect rgb(255, 248, 235)
    Note over C,G: Variant C - Android Play Integrity
    C->>API: POST /api/token/play-integrity (integrity_token, nonce, client_code)
    API->>G: decodeIntegrityToken using service-account access token
    G-->>API: tokenPayloadExternal
    API->>API: validate package, nonce, freshness, app and device verdicts
    API->>R: GET integrity_nonce:nonce
    alt nonce already used
        API-->>C: 401 Nonce already used
    else new nonce
        API->>R: SET integrity_nonce:nonce TTL freshness seconds
        API-->>C: 200 token with channel=client_code
    end
    end

    Note over C,P: Using the token
    C->>P: request with Authorization Bearer token
    P->>P: get_current_user verifies RS256 signature and exp
    alt invalid or expired
        P-->>C: 401 detail Token has expired or Invalid token
    else valid
        P-->>C: handler response
    end
```

Notes: the guest variant requires no credential by design. The nonce check is get-then-set (not atomic). Error codes per variant are listed in [api-specification.md](../application/api-specification.md#4-response-models-and-status-codes).

---

## W2 Streaming chat turn

Source: `app/routers/chat.py:chat_endpoint`, `app/routers/chat_query_utils.py`, `app/services/chat.py:stream_chat_messages`, `app/services/agrinet_routing.py`, `app/utils.py`. Endpoint: `GET /api/chat/`.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant R as chat router
    participant S as chat service
    participant Redis as Redis
    participant Mod as Moderation agent
    participant A as Agrinet agent and tools
    participant T as send_telemetry task
    participant LF as Langfuse

    C->>R: GET /api/chat/?query&session_id&qid&source_lang&target_lang (Bearer JWT)
    R->>R: get_current_user
    R->>Redis: GET session_id__SVA (history)
    Redis-->>R: message history or empty
    R->>R: normalize_chat_query (image id to IMAGE_URL, flag image analysis)
    R-->>C: 200 text/event-stream, headers X-QID
    R->>S: stream_chat_messages (generator starts when body is read)
    S->>T: queue question event (BackgroundTasks)
    S->>Redis: resolve route (see W8)
    S->>LF: open trace, propagate user, session, tags
    S->>Redis: SET chat_turn:qid (trace map, TTL 2h)
    S->>S: build FarmerContext, last 3 message pairs, pending NPSS image check
    S->>Mod: run moderation on assembled message
    Mod-->>S: category and action
    S->>S: trim_history 64000 tokens, strip thinking parts
    S->>A: create_task run_stream_events (timeout AGRINET_MODEL_TIMEOUT_SECONDS)
    loop tool calls and text deltas
        A->>A: call tools (search, weather, schemes, NPSS, ...)
        A-->>S: text delta after final result starts
        S->>S: strip think blocks, put into queue
        S-->>C: yield text chunk
    end
    A-->>S: final result and usage
    S->>S: post_process_npss_response if NPSS used
    opt image analysis turn
        S-->>C: yield single post-processed text
    end
    S->>T: queue answer event (BackgroundTasks)
    S->>Redis: SET session_id__SVA history TTL 2h
    S->>Redis: refresh route TTL
    S->>LF: flush
    S-->>C: end of stream
    T->>T: after response ends POST telemetry with retries

    alt client disconnects mid-stream
        C--xS: connection closed (CancelledError or GeneratorExit)
        S->>S: spawn detached finalizer, keep agent task running via shield
        A-->>S: run finishes within 180 s
        S->>T: send answer telemetry inline
        S->>Redis: save history and refresh route TTL
    end
```

Key behaviours: the response headers are returned before moderation or model work begins; text deltas are forwarded only after the model signals its final result part (the flag resets after every tool result); chunks are raw text (not SSE `data:` frames); the detached finalizer exists because response background tasks do not run after a disconnect. Contract: [api-specification.md](../application/api-specification.md#5-chat-streaming-contract). Error and failover paths: [F1](#f1-chat-request-lifecycle-and-error-paths) and [W8](#w8-model-routing-and-failover).

---

## W3 Crop image upload and analysis

Source: `app/routers/image.py` (`upload_image`, `serve_image`), `app/core/image_storage.py`, `app/routers/chat_query_utils.py`, `app/services/chat.py` (`_wrap_image_analysis_message`), `agents/tools/npss.py` (`analyze_crop_image`), `app/core/npss_followup.py`. Endpoints: `POST /api/image/upload`, `GET /api/chat/`, `GET /api/image/{image_id}`. The deprecated `POST /api/chat/analyze-image` performs the upload and the chat turn in one request.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant IMG as image router
    participant FS as Local disk TEMP_UPLOAD_DIR
    participant CH as chat router and service
    participant AG as Agrinet agent
    participant NT as analyze_crop_image tool
    participant NP as NPSS API
    participant PH as Photon geocoder

    C->>IMG: POST /api/image/upload multipart image (Bearer JWT)
    IMG->>IMG: allow image/jpeg, image/jpg, image/png only
    IMG->>FS: write uuid.ext in 1 MB chunks, limit MAX_IMAGE_UPLOAD_SIZE_MB
    IMG->>FS: write uuid.json metadata processed=false
    IMG-->>C: 200 image_id

    C->>CH: GET /api/chat/?query=image_id (and optional latitude, longitude)
    CH->>CH: extract_image_id, rewrite query with IMAGE_URL BASE_URL/api/image/id
    CH->>CH: is_image_analysis=true, output deferred until post-processing
    CH->>AG: run with wrapped instruction to call analyze_crop_image
    AG->>NT: analyze_crop_image(image_url)
    NT->>NT: get NPSS token from Redis cache or login
    NT->>PH: reverse geocode when deps latitude and longitude exist
    NT->>IMG: GET /api/image/id (no auth, UUID is the secret)
    IMG->>FS: locate file and metadata
    IMG-->>NT: image bytes
    NT->>FS: mark_processed
    NT->>NP: POST image with state, district, sub-district, village ids
    NP-->>NT: pest result
    NT->>FS: cleanup_image delete or move to GCS mount
    NT-->>AG: formatted NPSS result
    AG-->>CH: farmer friendly card text
    CH->>CH: post_process_npss_response translate, localize labels, add Source NPSS line
    CH-->>C: single text chunk then end of stream
```

### Location follow-up and fallback (flowchart)

```mermaid
flowchart TD
    A[analyze_crop_image called] --> B{deps latitude and longitude present?}
    B -- yes --> C[Photon reverse geocode and NPSS id lookup]
    B -- no --> F[Use KVK Delhi fallback location]
    C --> D{state, district, sub-district ids resolved?}
    D -- yes --> E{village id present?}
    D -- no --> F
    E -- yes --> G[Call NPSS with resolved ids]
    E -- no --> H[Use village id 0 sentinel] --> G
    F --> G
    G --> I{NPSS call succeeded?}
    I -- yes --> J[cleanup image, return result]
    I -- no, retryable --> K[ModelRetry keeps image for retry]
    K --> A
```

> **Note:** In `app/services/chat.py` the browser `latitude` and `longitude` are used only inside the instruction text sent to the model. `FarmerContext` is created without those values, and `agents/tools/npss.py` reads `deps.latitude` / `deps.longitude`, so the "location present" branch cannot currently be taken from a chat request. This is recorded as technical debt in [technical-debt.md](../knowledge/technical-debt.md).

Details of the follow-up detection (`find_pending_npss_image_url`) and storage lifecycle: [backend.md](../application/backend.md#65-support-modules) and [database.md](../application/database.md#4-image-storage-local-disk).

---

## W4 Voice transcription (ASR)

Source: `app/routers/transcribe.py:transcribe`, `helpers/transcription.py`, `helpers/telemetry.py:create_asr_event`. Endpoint: `POST /api/transcribe/`.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant R as transcribe router
    participant H as helpers.transcription
    participant B as Bhashini or OpenAI Whisper
    participant BG as BackgroundTasks
    participant T as send_telemetry

    C->>R: POST /api/transcribe/ audio_content base64, lang_code, service_type (Bearer JWT)
    R->>R: validate service_type bhashini or whisper
    R->>H: transcribe_bhashini(audio, lang) or transcribe_whisper(audio)
    H->>B: synchronous HTTP call (retry on timeouts for Bhashini)
    B-->>H: text and language
    H-->>R: transcription
    R->>R: build ASR telemetry event with latency
    R->>BG: add_task send_telemetry
    R-->>C: 200 status success, text, lang_code, session_id
    BG->>T: runs after response
    T-->>T: POST to telemetry API

    alt provider error
        H-->>R: exception
        R-->>C: 500 (exception re-raised, telemetry task not delivered)
    end
```

---

## W5 Text-to-speech (TTS)

Source: `app/routers/tts.py:tts`, `helpers/tts.py:text_to_speech_bhashini`, `helpers/telemetry.py:create_tts_event`. Endpoint: `POST /api/tts/`.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant R as tts router
    participant H as helpers.tts
    participant B as Bhashini TTS
    participant BG as BackgroundTasks

    C->>R: POST /api/tts/ text, target_lang, service_type (Bearer JWT)
    alt service_type is not bhashini
        R-->>C: 400 status error, service not supported
    else bhashini
        R->>H: text_to_speech_bhashini(text, lang, female, 8000 Hz)
        H->>B: synchronous HTTP POST (timeout 30 s connect, 60 s read)
        B-->>H: audio bytes
        H-->>R: audio bytes
        R->>R: base64 encode
        R->>BG: add_task send_telemetry (TTS event, latency)
        R-->>C: 200 status success, audio_data, session_id
    end
```

---

## W6 Telemetry and feedback

Source: `app/routers/telemetry.py`, `app/tasks/telemetry.py`, `helpers/telemetry.py`, `helpers/langfuse_scores.py`, `app/services/chat_turn_map.py`. Endpoints: `POST /api/telemetry/feedback`, `/events`, `/error`; backend-originated chat events inside W2, W4, W5.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant R as telemetry router
    participant Redis as Redis
    participant BG as BackgroundTasks
    participant T as send_telemetry
    participant O as Observability service
    participant LF as Langfuse

    C->>R: POST /api/telemetry/feedback qid, session_id, feedback_type, texts (Bearer JWT)
    R->>R: create_chat_feedback_event using JWT identity
    R->>BG: add_task send_telemetry
    R->>Redis: GET chat_turn:qid
    alt map found and session_id matches
        R->>BG: add_task create_user_feedback_score
        R-->>C: 200 accepted, langfuse_score queued
    else missing, expired or session mismatch
        R-->>C: 200 accepted, langfuse_score skipped
    end
    BG->>T: send_telemetry(events)
    loop up to 3 attempts on timeout or request error
        T->>O: POST telemetry JSON with Authorization header
        O-->>T: status
    end
    BG->>LF: create_score user-feedback:qid (idempotent upsert)

    C->>R: POST /api/telemetry/events (array of UI events)
    R->>BG: add_task send_telemetry (ui interact events)
    R-->>C: 200 accepted, count

    C->>R: POST /api/telemetry/error qid, session_id, error_text
    R->>BG: add_task send_telemetry (chat error event)
    R-->>C: 200 accepted
```

---

## W7 Conversation suggestions (disabled)

Status: **not active in the running application.** `main.py` has the router import and `include_router` commented out, `app/tasks/__init__.py` has the `create_suggestions` import commented out, and the chat service does not schedule the task. The diagram documents the intended design found in `app/tasks/suggestions.py` and `app/routers/suggestions.py`.

```mermaid
sequenceDiagram
    autonumber
    participant S as chat service (no longer schedules)
    participant TS as create_suggestions task
    participant Redis as Redis
    participant SA as suggestions_agent
    participant C as Client
    participant SR as suggestions router (not mounted)

    S-->>TS: schedule create_suggestions(session_id, target_lang)
    TS->>Redis: read session history
    TS->>TS: format last 3 message pairs
    TS->>SA: run prompt asking for 3-5 follow-up questions
    SA-->>TS: list of strings
    TS->>Redis: SET suggestions_session_lang TTL 3600
    C->>SR: GET /api/suggest/?session_id&target_lang
    SR->>Redis: GET suggestions_session_lang
    SR-->>C: JSON list or empty list
```

> **Note:** This flow is retained in code for reference. Reviving it would require fixing the stale references noted in [async-processing.md](../integrations/async-processing.md#4-task-modules-present-but-not-active).

---

## W8 Model routing and failover

Source: `app/services/agrinet_routing.py` (`resolve_agrinet_route`, `_apply_capacity_gate`, `get_gemma_concurrency`), `app/services/chat.py` (`_run_agrinet_with_failover_streaming`), `agents/models.py`. Applies to every chat turn ([W2](#w2-streaming-chat-turn)).

```mermaid
flowchart TD
    S([Chat turn starts]) --> E{AGRINET_ROUTING_ENABLED?}
    E -- no --> D1[Route gpt41, source routing_disabled]
    E -- yes --> K{Route stored in Redis for session?}
    K -- yes --> G1{Stored route is gemma?}
    K -- no --> H{Session has history?}
    H -- yes --> RP[Repair: store gpt41, source state_repair]
    H -- no --> W[Weighted random pick, default 70 gpt41 and 30 gemma]
    W --> ST[Store pick in Redis TTL AGRINET_ROUTE_TTL_SECONDS]
    ST --> G1
    G1 -- no --> USE[Use route as decided]
    G1 -- yes --> M[Read gemma_concurrency from Redis or vLLM metrics]
    M --> CAP{concurrency known and below AGRINET_GEMMA_MAX_CONCURRENCY?}
    CAP -- yes --> USE
    CAP -- no --> DEF[Deflect this turn to gpt41, source capacity_deflect, stored route unchanged]
    D1 --> RUN
    RP --> RUN
    USE --> RUN
    DEF --> RUN
    RUN[Run agent with timeout AGRINET_MODEL_TIMEOUT_SECONDS] --> OK{Run succeeded?}
    OK -- yes --> FIN([Finalize turn])
    OK -- no --> FE{Routing enabled and no chunk streamed yet?}
    FE -- no --> ERR([Raise error, error telemetry queued])
    FE -- yes --> ALT[Retry once on alternate route, source failover]
    ALT --> OK2{Retry succeeded?}
    OK2 -- yes --> SAVE[Store alternate route for session, fallback_used true] --> FIN
    OK2 -- no --> ERR
```

---

## W9 Cached HTML document serving

Source: `agents/tools/shc_scheme_status.py` (`cache_and_update_url`, `FILE_TTL = 600`), `app/routers/file.py`. Endpoint: `GET /api/file/{file_hash}`.

```mermaid
sequenceDiagram
    autonumber
    participant A as Agent tool (Soil Health Card status)
    participant Redis as Redis
    participant C as Client browser
    participant F as file router

    A->>A: receive HTML report from upstream, inject download header
    A->>Redis: SET html_file:hash HTML TTL 600 s
    A-->>C: answer text containing API_BASE_URL/api/file/hash
    C->>F: GET /api/file/hash (no auth)
    F->>Redis: GET html_file:hash
    alt found
        F-->>C: 200 text/html, Cache-Control public max-age 600
    else missing or expired
        F-->>C: 404 File not found or expired
    end
```

---

## W10 Health probes

Source: `app/routers/health.py`. Endpoints: `GET /api/health/`, `/api/health/live`, `/api/health/ready`.

```mermaid
sequenceDiagram
    autonumber
    participant K as Orchestrator or monitor
    participant H as health router
    participant Redis as Redis

    K->>H: GET /api/health/live
    H-->>K: 200 status alive (no dependency check)

    K->>H: GET /api/health/ready or /api/health/
    H->>Redis: SET health_check_test TTL 5 s, then GET
    alt value round-trips
        H-->>K: 200 ready (or app metadata, uptime, cache healthy)
    else error or mismatch
        H-->>K: 503 with cache status and error text
    end
```

---

## F1 Chat request lifecycle and error paths

Workflow flowchart for `stream_chat_messages` showing success, failure and disconnect outcomes (companion to the sequence in [W2](#w2-streaming-chat-turn)).

```mermaid
flowchart TD
    A[Request GET /api/chat/] --> B{JWT valid?}
    B -- no --> B1[401 response]
    B -- yes --> C[Load history from Redis]
    C --> C1{Redis reachable?}
    C1 -- no --> C2[500 server error before streaming]
    C1 -- yes --> D[Return stream headers with X-QID]
    D --> E[Queue question telemetry]
    E --> F[Resolve model route W8]
    F --> G[Moderation agent]
    G --> H[Trim history and start agent task]
    H --> I{Client still connected?}
    I -- no --> J[Detached finalizer waits up to 180 s]
    J --> J1{Agent finished in time?}
    J1 -- yes --> J2[Send telemetry inline and save history]
    J1 -- no --> J3[Drop turn, log warning]
    I -- yes --> K[Stream text chunks]
    K --> L{Agent run ok?}
    L -- no --> M[Queue error telemetry, cancel task, abort stream]
    L -- yes --> N[NPSS post-processing if used]
    N --> O[Queue answer telemetry]
    O --> P[Save history, refresh route TTL, flush Langfuse]
    P --> Q[End of stream]
```

---

## F2 Service startup

Source: `main.py` (`lifespan`), module import side effects in `app/auth/jwt_auth.py`, `app/routers/token.py`, `agents/models.py`, `app/core/image_storage.py`.

```mermaid
flowchart TD
    A[supervisord starts uvicorn with 8 workers] --> B[Import main.py and routers]
    B --> C{JWT public key file readable?}
    C -- no --> X1([Import error, worker exits])
    C -- yes --> D[Load private key if present, else warn]
    D --> E{LLM_PROVIDER valid and its env vars set?}
    E -- no --> X1
    E -- yes --> F[Create TEMP_UPLOAD_DIR]
    F --> G[lifespan: validate multi-provider auth env]
    G --> H{Any per-client key variable empty?}
    H -- yes --> X2([RuntimeError, startup aborted])
    H -- no --> I[validate_agrinet_routing_config]
    I --> J{Routing enabled and config invalid?}
    J -- yes --> X3([ValueError, startup aborted])
    J -- no --> K{QDRANT_URL set?}
    K -- yes --> L[Warm scheme embedder in a thread]
    K -- no --> M[Skip warm-up]
    L --> N([Ready to serve])
    M --> N
```

---

## Source Files

`app/routers/*.py`, `app/auth/jwt_auth.py`, `app/services/chat.py`, `app/services/agrinet_routing.py`, `app/services/chat_turn_map.py`, `app/core/image_storage.py`, `app/tasks/*.py`, `agents/tools/npss.py`, `agents/tools/shc_scheme_status.py`, `helpers/telemetry.py`, `helpers/transcription.py`, `helpers/tts.py`, `helpers/langfuse_scores.py`, `main.py`.
