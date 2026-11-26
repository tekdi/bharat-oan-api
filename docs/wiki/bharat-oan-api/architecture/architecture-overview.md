# Architecture Overview

High-level architecture of `bharat-oan-api`. Diagrams are not repeated here; this page indexes them. Start with [AI Context](../ai-context.md) if you have not read it.

Related: [Component Diagram](component-diagram.md) | [Sequence & Workflow Diagrams](sequence-diagrams.md) | [Backend](../application/backend.md) | [AI/LLM](../integrations/ai-llm.md) | [Technical Debt](../knowledge/technical-debt.md)

## 1. High-Level Summary

`bharat-oan-api` is a single deployable FastAPI service (`main.py`) that fronts a tool-using LLM agent for farmer advice and scheme services. The request path is thin: JWT auth, a router, one orchestration function (`stream_chat_messages` in `app/services/chat.py`), and two pydantic-ai agents (moderation, then the main agent). The agent reaches farmer-facing government services mainly through a BAP gateway speaking the Beckn protocol, and reaches knowledge search through Marqo and Qdrant. State is limited to Redis (history, route pinning, tokens, cached HTML) and short-lived image files on local disk. No relational database exists.

### Architecture Style (implemented today)

| Aspect | Choice | Evidence |
|--------|--------|----------|
| Deployment unit | Single container, one uvicorn process group (8 workers) under supervisord | `Dockerfile`, `supervisord.conf` |
| Framework | FastAPI, async handlers, `BackgroundTasks` | `main.py`, `app/routers/*.py` |
| Layering | Routers -> services -> agents/tools -> helpers | `app/`, `agents/`, `helpers/` |
| State | Redis only, plus local disk for uploaded images | `app/core/cache.py`, `app/core/image_storage.py` |
| Auth | RS256 JWT; tokens issued by the same service (guest, API key, Play Integrity) | `app/auth/jwt_auth.py`, `app/routers/token.py` |
| LLM integration | pydantic-ai agents over OpenAI-compatible clients (OpenAI, Azure, vLLM) | `agents/models.py` |
| Tool integration | 36 `pydantic_ai.Tool` entries; most call a BAP gateway (Beckn `/search`, `/init`, `/status`) | `agents/tools/__init__.py` |
| Response delivery | Chunked plain-text streaming (`StreamingResponse`) | `app/routers/chat.py` |
| Observability | Langfuse `@observe`, backend-owned telemetry relay, health endpoints | `helpers/langfuse_*.py`, `helpers/telemetry.py`, `app/routers/health.py` |
| Scaling | Stateless workers, Redis-pinned model route per session | `app/services/agrinet_routing.py` |

## 2. Key Decisions

1. **Prompt-driven tool routing.** Which tool to call is decided by the system prompt (`assets/prompts/agrinet_<lang>.md`), not by code. Code validates tool inputs only ([Business Rules](../business/business-rules.md#tool-selection-rules)).
2. **One prompt per language, no translation hop.** The main model answers directly in `target_lang`; translation helpers are used only for NPSS post-processing ([AI/LLM](../integrations/ai-llm.md#1-overview)).
3. **Moderation as a separate agent before the main agent**, with the result passed to the main agent as context. It does not short-circuit the turn ([AI/LLM section 4](../integrations/ai-llm.md#4-guardrails-and-moderation)).
4. **Stateless API, Redis for session memory.** History is stored per `session_id` with a 2 h TTL; routes are pinned per session ([Caching](../integrations/caching.md)).
5. **Optional two-model A/B routing with failover and a capacity gate**, off by default (`AGRINET_ROUTING_ENABLED`) ([AI/LLM section 2.2](../integrations/ai-llm.md#22-optional-ab-routing-between-two-agrinet-models-agentsmodelspy-appservicesagrinet_routingpy)).
6. **Beckn/BAP as the integration backbone.** Scheme, mandi, weather, fertilizer and seed data come through one gateway rather than per-agency SDKs ([Third-Party Integrations](../integrations/third-party-integrations.md)).
7. **Self-issued JWTs including an unauthenticated guest token**, favouring easy client onboarding over strong identity ([Users and Personas](../business/users-and-personas.md), [TD-02](../knowledge/technical-debt.md#td-02--open-guest-token-endpoint-and-no-authorization-model)).
8. **Backend-owned telemetry.** The server builds question and answer events and relays client feedback, instead of relying only on client reporting ([W6](sequence-diagrams.md#w6-telemetry-and-feedback)).

## 3. Reconciliation with Raw Architecture Documents

> **Note:** This repository has no `docs/raw_documents/` folder, so there are no raw architecture documents to reconcile. The only legacy document is `docs/data/archive/CROP ADVISORY SOP.docx`, a crop-advisory standard operating procedure listed under [Legacy Artifacts](../knowledge/legacy-artifacts.md). The hand-written `API_DOCUMENTATION.md` is the other prior document; its differences from the code are tabulated in [API Specification](../application/api-specification.md#reconciliation-with-api_documentationmd).

## 4. Alternative Architecture Approaches

### Software engineering perspectives

| ID | Approach | Status | Summary | Pain addressed | Trade-offs | Fit |
|----|----------|--------|---------|----------------|------------|-----|
| A | Single FastAPI service with Redis (current) | Implemented | Monolith with async handlers and background tasks | Simplicity, one image | Blocking sync calls hurt all sessions on a worker ([TD-04](../knowledge/technical-debt.md#td-04--blocking-synchronous-calls-in-async-handlers)); local-disk images tie NPSS to one replica ([TD-05](../knowledge/technical-debt.md#td-05--uploaded-images-never-cleaned-up-and-stored-on-local-disk)) | Good for current scale |
| B | Async-only IO plus shared object storage | Not implemented | Replace sync `httpx`/`requests` use with async clients; store images in a bucket (`GCS_MOUNT_PATH` is a partial step) | Event-loop stalls, multi-replica safety | Refactor of `helpers/*`, `app/routers/transcribe.py`, `tts.py`, `app/core/image_storage.py` | High value, moderate effort |
| C | Gateway-level auth and rate limiting | Partially considered | `rate_limit_requests_per_minute` and `disable_jwt` exist in `app/config.py` but are unused; move limits to an API gateway or middleware | Open guest token, abuse ([TD-02](../knowledge/technical-debt.md#td-02--open-guest-token-endpoint-and-no-authorization-model)) | Needs product decision on guest access | High value |
| D | Worker queue for telemetry and finalisation | Not implemented (Celery remnants in comments) | Move telemetry and post-turn work off the request process | Lost events on disconnect or failure ([TD-15](../knowledge/technical-debt.md#td-15--telemetry-lost-on-failure-or-client-disconnect)) | New infrastructure | Medium |

### AI / ML engineering perspectives

| ID | Approach | Status | Summary | Pain addressed | Trade-offs | Fit |
|----|----------|--------|---------|----------------|------------|-----|
| I | Prompt-only tool routing and moderation (current) | Implemented | System prompts decide tools and refusals | Fast iteration, no code per rule | Drift between 13 prompt files ([TD-09](../knowledge/technical-debt.md#td-09--prompt-drift-across-languages-and-hard-coded-dates)); refused categories still run the main agent ([TD-06](../knowledge/technical-debt.md#td-06--moderation-is-prompt-only-and-fails-the-whole-turn-on-error)) | Works, needs parity checks |
| J | Shared prompt template with language overlays | Not implemented | One base prompt plus per-language fragments; extend `tests/test_languages.py` parity checks | Prompt drift | Template refactor and re-evaluation | High value |
| K | Hard moderation gate before the main agent | Not implemented | Short-circuit refused categories in code and add a moderation fallback | Cost and safety, moderation outage | Needs policy for fallback answers | High value |
| L | Automated answer evaluation in CI | Partially considered | Live runner scripts in `tests/` plus committed result JSON; turn into a scheduled evaluation with Langfuse scores (`helpers/langfuse_scores.py` exists) | No quality gate | Cost and flakiness of live LLM calls | Medium |

### Decision guide

| Priority | SWE | AI/ML |
|----------|-----|-------|
| Reduce abuse and unauthorised use | C | K |
| Scale beyond one replica | B | none |
| Answer quality across languages | none | J, L |
| Reliability of analytics | D | none |

## 5. Diagram and Reference Index

| Need | Go to |
|------|-------|
| System context and components | [Component Diagram](component-diagram.md) |
| Chat, auth, ASR, TTS, image, telemetry sequences | [Sequence Diagrams](sequence-diagrams.md) |
| Business workflow narrative | [Workflows](../business/workflows.md) |
| Data stores and Redis keys | [Database](../application/database.md) |
| Deployment topology | [Deployment](../operations/deployment.md), [Infrastructure](../operations/infrastructure.md) |
| Config and env vars | [Configuration](configuration.md) |

## 6. Architectural Risks

Detail and remediation are in [Technical Debt](../knowledge/technical-debt.md).

- Access control: any caller can obtain a token ([TD-02](../knowledge/technical-debt.md#td-02--open-guest-token-endpoint-and-no-authorization-model)).
- Event-loop blocking from sync calls ([TD-04](../knowledge/technical-debt.md#td-04--blocking-synchronous-calls-in-async-handlers)).
- Redis is a hard dependency of the chat path; an outage fails chat ([TD-16](../knowledge/technical-debt.md#td-16--redis-outage-fails-chat-and-nonce-check-is-not-atomic)).
- Unpinned dependencies and no lock file make rebuilds non-reproducible ([TD-21](../knowledge/technical-debt.md#td-21--unpinned-dependencies-and-no-lock-file)).

## Source Files

`main.py`, `app/config.py`, `app/services/chat.py`, `app/services/agrinet_routing.py`, `agents/models.py`, `agents/agrinet.py`, `agents/tools/__init__.py`, `app/core/cache.py`, `Dockerfile`, `supervisord.conf`, `requirements.txt`.
