# Caching

Redis is the only shared cache in this service. It is accessed through an `aiocache` instance (`app/core/cache.py`) for all application keys, plus a separate synchronous `redis.Redis` client for a read-only master-catalog snapshot (`helpers/master_catalog.py`). The same Redis also holds conversation state (chat history, model route), so it acts as both cache and lightweight session store.

Related pages: [Database and data stores](../application/database.md) | [Backend](../application/backend.md) | [Async processing](async-processing.md) | [Third-party integrations](third-party-integrations.md) | [AI/LLM](ai-llm.md) | [Configuration](../architecture/configuration.md) | [Monitoring](../operations/monitoring.md) | [Technical debt](../knowledge/technical-debt.md)

---

## 1. Cache Configuration (Implemented, verified)

`app/core/cache.py`:

```python
cache = Cache(
    Cache.REDIS,
    endpoint=settings.redis_host, port=settings.redis_port, db=settings.redis_db,
    serializer=JsonSerializer(),
    ttl=settings.default_cache_ttl,          # 24 h default per-entry TTL
    timeout=settings.redis_socket_timeout,
    pool_max_size=settings.redis_max_connections,
    key_builder=lambda key, namespace: f"{prefix}{namespace}:{key}" if namespace else f"{prefix}{key}",
)
```

| Setting (`app/config.py`) | Env var | Default | Effect |
|---------------------------|---------|---------|--------|
| `redis_host` | `REDIS_HOST` | `localhost` | Redis endpoint |
| `redis_port` | `REDIS_PORT` | `6379` | |
| `redis_db` | `REDIS_DB` | `0` | |
| `redis_key_prefix` | `REDIS_KEY_PREFIX` | `sva-cache-` | Prefix on every key written through `cache` |
| `redis_socket_timeout` | `REDIS_SOCKET_TIMEOUT` | `10` | Passed as aiocache `timeout` |
| `redis_max_connections` | `REDIS_MAX_CONNECTIONS` | `100` | Passed as aiocache `pool_max_size` |
| `default_cache_ttl` | - (constant) | `86400` s (24 h) | Default TTL of the cache instance when `ttl` is not given |
| `redis_socket_connect_timeout` | `REDIS_SOCKET_CONNECT_TIMEOUT` | `10` | Used only by the master-catalog `redis.Redis` client |
| `redis_retry_on_timeout` | `REDIS_RETRY_ON_TIMEOUT` | `true` | Declared in `Settings`; not passed to the aiocache instance |
| `suggestions_cache_ttl` | - (constant) | `1800` s | Declared in `Settings`; not referenced by code (the disabled suggestions task uses its own constant) |
| `master_catalog_redis_key` | `MASTER_CATALOG_REDIS_KEY` | empty | Overrides the master-catalog snapshot key |

Serialization: `JsonSerializer`, so only JSON-serializable values can be cached. Message history is converted with `pydantic_core.to_jsonable_python` and restored with `ModelMessagesTypeAdapter.validate_python` (`app/utils.py`).

Access patterns:

- Wrapper helpers `get_cache(key)` / `set_cache(key, value, ttl=DEFAULT_CACHE_TTL)` in `app/utils.py` (`DEFAULT_CACHE_TTL = 2 h`).
- Direct `cache.get / cache.set / cache.delete` calls in `app/routers/token.py`, `app/routers/file.py`, `app/routers/health.py`, `agents/tools/npss.py`, `agents/tools/sathi_seed.py`, `agents/tools/shc_scheme_status.py`.
- A connection is created lazily by aiocache; `app/core/cache.py` only logs the configuration at import (it does not ping Redis). The first real check is `/api/health/ready` or the first cache use.

> **Note:** There is no in-process (LRU) cache of API data in front of Redis. In-memory structures that exist are object caches rather than data caches: `lru_cache` for model/client construction (`agents/models.py`), the embedder and Qdrant client dicts (`helpers/scheme_qdrant_search.py`), Play Integrity credentials (`app/routers/token.py`), and the image upload registry (`app/core/image_storage.py`).

---

## 2. What Is Cached

| Purpose | Key (before prefix) | Value | Owner module | Consumer |
|---------|---------------------|-------|--------------|----------|
| Conversation history | `<session_id>__SVA` | pydantic-ai message list | `app/utils.py` | `app/routers/chat_query_utils.py:get_session_history`, `app/services/chat.py` |
| Moderation history (helpers exist) | `<session_id>__SVA_MODERATION` | message list | `app/utils.py` | no caller of the get/update functions in the repository |
| Session model route (sticky A/B) | `<session_id>_AGRINET_ROUTE` | `"gpt41"` / `"gemma"` | `app/services/agrinet_routing.py` | chat service |
| Gemma load metric | `gemma_concurrency` | int | `app/services/agrinet_routing.py` | capacity gate |
| qid to Langfuse trace map | `chat_turn:<qid>` | dict | `app/services/chat_turn_map.py` | `POST /api/telemetry/feedback` |
| Play Integrity nonce replay guard | `integrity_nonce:<nonce>` | `true` | `app/routers/token.py` | same |
| Generated HTML documents | `html_file:<hash>` | HTML string | `agents/tools/shc_scheme_status.py` | `GET /api/file/{file_hash}` |
| NPSS API bearer token | `npss:bearer_token` | string | `agents/tools/npss.py` | same |
| SATHI crop groups | namespace `sathi`, key `crop_groups` | list | `agents/tools/sathi_seed.py` | same |
| Health probe | `health_check_test` | `"test"` | `app/routers/health.py` | same |
| Master catalog snapshot (read-only, unprefixed) | `master-catalog:<tier>:snapshot` | JSON dict | external pipeline | `helpers/master_catalog.py` |

The cache-hit/miss flow for chat history is part of the chat sequence: [W2 - Streaming chat turn](../architecture/sequence-diagrams.md#w2-streaming-chat-turn); route stickiness: [W8](../architecture/sequence-diagrams.md#w8-model-routing-and-failover); HTML serving: [W9](../architecture/sequence-diagrams.md#w9-cached-html-document-serving).

---

## 3. TTL

| Key family | TTL | Where defined | Refreshed on use? |
|------------|-----|---------------|-------------------|
| Chat history `<session_id>__SVA` | 7200 s (2 h) | `DEFAULT_CACHE_TTL` in `app/utils.py` | Yes: re-written with a full TTL at the end of every completed turn (`update_message_history`) |
| Moderation history | 7200 s | `app/utils.py` | on write only (unused) |
| Session route `<session_id>_AGRINET_ROUTE` | `AGRINET_ROUTE_TTL_SECONDS`, default 7200 s | `app/config.py`, `app/services/agrinet_routing.py:_route_ttl_seconds` | Yes: `refresh_session_agrinet_route_ttl` at the end of each turn; also rewritten on failover |
| `gemma_concurrency` | `AGRINET_GEMMA_METRICS_CACHE_TTL`, default 2 s | `app/config.py` | No (short-lived on purpose) |
| `chat_turn:<qid>` | 7200 s (`DEFAULT_CACHE_TTL` default argument) | `app/services/chat_turn_map.py` | No. Feedback arriving after 2 h is logged as a Langfuse "miss" and the score is skipped |
| `integrity_nonce:<nonce>` | `PLAY_INTEGRITY_FRESHNESS_SECONDS`, default 120 s | `app/routers/token.py` | No |
| `html_file:<hash>` | 600 s (`FILE_TTL = 10 * 60`) | `agents/tools/shc_scheme_status.py` | No. The route also sends `Cache-Control: public, max-age=600` to browsers |
| `npss:bearer_token` | 1500 s (25 min) | `NPSS_TOKEN_TTL_SECONDS` in `agents/tools/npss.py` | No |
| `sathi:crop_groups` | 604800 s (7 days) | `agents/tools/sathi_seed.py` | No |
| `health_check_test` | 5 s | `app/routers/health.py` | No |
| Anything stored without an explicit TTL | 86400 s (24 h) | `settings.default_cache_ttl` | No |

Related timeouts that interact with the cache: JWT lifetime `JWT_EXPIRY_MINUTES` (default 15 min), image TTL `IMAGE_TTL_MINUTES` (60 min, disk not Redis), model-attempt timeout `AGRINET_MODEL_TIMEOUT_SECONDS` (45 s).

> **Note:** Redis memory limits and eviction policy (for example `maxmemory-policy`) are configured on the Redis server and are not part of this repository.

---

## 4. Invalidation Strategy

The service relies on **time-based expiry**; explicit invalidation is rare.

| Mechanism | Where | Behaviour |
|-----------|-------|-----------|
| TTL expiry (primary) | All keys listed above | Entries disappear automatically; there is no cache-aside refresh job or background warmer |
| Sliding TTL | Chat history, session route | Re-written each turn, so an active session keeps its state alive; an idle session loses history 2 h after its last completed turn |
| Explicit delete on auth failure | `agents/tools/npss.py` (`cache.delete(NPSS_TOKEN_CACHE_KEY)` when the NPSS API rejects the token, around line 266) | Forces a new token fetch on the next call |
| Overwrite on state change | `set_session_agrinet_route` after a failover; `update_message_history` each turn; the qid map is rewritten when failover changes the route (`_finalize_chat_turn` calls `_record_chat_turn` when `fallback_used`) | New value replaces the old one |
| Invalid-value self-heal | `get_stored_agrinet_route` ignores values other than `gpt41` / `gemma` (logs a warning and behaves as if absent) | A corrupted route key is replaced on the next resolution (`state_repair` or a new weighted pick) |
| Replay marker | `integrity_nonce:<nonce>` is intentionally never deleted; it expires after the freshness window | Prevents reuse of a Play Integrity verdict |
| Pull-based external invalidation | `master-catalog:*:snapshot` is written by an external pipeline; this service re-reads it on demand and logs when `version` changes (`_log_if_new_sync`) | No pub/sub; freshness equals the next read |
| Manual | Delete keys in Redis directly (prefix `sva-cache-`) | Only way to force-reset a session or route |

Not implemented: bulk invalidation APIs, cache versioning in keys, tag-based invalidation, or any admin endpoint to clear a session. Session history is never deleted when a conversation ends.

### Failure and consistency behaviour

| Scenario | Behaviour |
|----------|-----------|
| Redis down during chat | `get_session_history` (called by `GET /api/chat/` before streaming) is not wrapped in error handling, so the request fails with a server error; `get_gemma_concurrency` and the master-catalog reader are the exceptions and fail open |
| Redis down during health check | `/api/health/` and `/api/health/ready` return 503 |
| Redis down during Play Integrity | Nonce check returns 503 `Replay protection is unavailable.` (fails closed) |
| Concurrent turns on the same session | Last writer wins for history (no locking); the nonce check is a non-atomic get-then-set |
| Key collisions between sessions | None expected: ids are UUIDs, except when a client supplies its own `session_id`; any client may read or overwrite state for a session id it knows (no per-user ownership check) |

### Recommended (not yet implemented)

- Use `SET NX` (aiocache `add`) for the nonce check to make replay protection atomic.
- Add a session-scoped lock or version check for history writes if concurrent turns per session are expected.
- Wrap the history read in a fallback so a Redis blip degrades to an empty history instead of a failed request.
- Introduce a key-version suffix to allow safe invalidation when the stored message schema changes after a pydantic-ai upgrade.

---

## Source Files

`app/core/cache.py`, `app/config.py`, `app/utils.py`, `app/services/agrinet_routing.py`, `app/services/chat_turn_map.py`, `app/services/chat.py`, `app/routers/token.py`, `app/routers/file.py`, `app/routers/health.py`, `agents/tools/npss.py`, `agents/tools/sathi_seed.py`, `agents/tools/shc_scheme_status.py`, `helpers/master_catalog.py`.
