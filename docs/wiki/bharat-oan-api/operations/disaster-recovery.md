# Disaster Recovery

Recovery considerations for `bharat-oan-api`. The service itself is largely stateless, so recovery centres on redeploying the container, restoring configuration and key files, and re-establishing its dependencies (Redis, Qdrant, Marqo, LLM endpoints, Beckn network).

Related: [Infrastructure](infrastructure.md) | [Deployment](deployment.md) | [Monitoring](monitoring.md) | [Caching](../integrations/caching.md) | [Configuration](../architecture/configuration.md) | [Troubleshooting](troubleshooting.md)

> **Note:** A formal disaster recovery plan (recovery time and recovery point objectives, backup schedules, failover sites, and restore drills) is not documented in this wiki and is not part of this repository. Backup and recovery of hosts, Redis, Qdrant, Marqo, and secrets are handled by the hosting environment and operations team. The sections below describe what the code and setup documents show about what must be restored.

## What state exists (Implemented, verified)

| State | Where | Lifetime | Loss impact |
|-------|-------|----------|-------------|
| Chat history and moderation history | Redis keys `<session>__SVA`, `..._SVA_MODERATION` (`app/utils.py`) | 2 hours (`DEFAULT_CACHE_TTL`) | Conversations lose context; users can start a new turn |
| Agrinet route decision per session | Redis (`AGRINET_ROUTE` suffix, `app/services/agrinet_routing.py`) | `AGRINET_ROUTE_TTL_SECONDS` (7200) | Route is re-chosen on the next turn |
| Chat turn map (qid to Langfuse trace) | Redis `chat_turn:<qid>` (`app/services/chat_turn_map.py`) | 2 hours | Late feedback is accepted but not attached to a Langfuse trace |
| Cached HTML files (SHC), SATHI and other tool caches, Play Integrity nonces | Redis (`html_file:*`, `integrity_nonce:*`, `sathi` namespace) | Minutes to hours | Links expire; nonces cannot be replay-checked while Redis is down (503) |
| Master catalog snapshot (`master-catalog:<tier>:snapshot`) | Redis, **written by the external docs-pipeline**, read by `helpers/master_catalog.py` | Until the pipeline rewrites it | Scheme routing degrades (fail-open) until the pipeline republishes |
| Uploaded crop images | `TEMP_UPLOAD_DIR` (default `<repo>/.oan-uploads`), optional `GCS_MOUNT_PATH` | `IMAGE_TTL_MINUTES` (60) | Files and per-image JSON metadata live in `TEMP_UPLOAD_DIR`, and each process reloads metadata lazily into its in-memory registry (`app/core/image_storage.py`). If the directory is lost, users re-upload |
| Vector indexes | Qdrant and Marqo (external) | Persistent, owned by the ingestion pipeline | Search tools return nothing until restored |
| Prompts, glossary, commodity codes | `assets/` in git | Persistent | Restored from git |
| Secrets and key files | Host `.env`, `jwt_public_key.pem`, `private_key.pem`, service-account JSON | Persistent | Service cannot start or issue tokens; must be restored from your secrets store |

There is no relational database, object store, or message queue owned by this service (see [Database](../application/database.md) and [Async Processing](../integrations/async-processing.md)).

## Redis persistence as documented (`docs/setup/redis.md`)

- `appendonly no`, with RDB snapshots (`save 900 1`, `save 300 10000`), data directory mounted from host `/etc/redis`, `--restart always`.
- `maxmemory 8gb` with `allkeys-lru` eviction: under memory pressure any key, including history, may be evicted. The cache is treated as disposable by the application (all keys carry TTLs).
- The document shows an ad-hoc backup: `redis-cli SAVE` then `docker cp redis-stack:/data/dump.rdb /backup/redis/dump.rdb`. No schedule, retention policy, or offsite copy is described.
- Because history lives at most two hours, restoring an old RDB file has limited value for chat data. It matters mainly for the master catalog snapshot and for keeping warm caches.

## Recovery scenarios

| Scenario | Effect | Recovery (derived from code and docs) |
|----------|--------|---------------------------------------|
| API container crash | Supervisord restarts uvicorn (`autorestart=true`, `startretries=3`); Docker `restart: unless-stopped` restarts the container | Check logs; see [Troubleshooting](troubleshooting.md) |
| Redis down or wiped | `/api/health/ready` returns 503; chat history unavailable; token nonce checks return 503; master catalog fails open | Start Redis per `docs/setup/redis.md` on `svanetwork`; restore `dump.rdb` if available; trigger the docs-pipeline to republish the master catalog |
| Host lost | Everything on the host is gone | Provision host, install Docker, recreate `svanetwork` and Redis, restore `.env` and key files from your secrets store, pull the image (or rebuild with `docker compose up --build`) |
| Bad release | Faulty behaviour after deploy | Redeploy a previous image tag or commit (see rollback in [Deployment](deployment.md)) |
| LLM endpoint outage | Chat fails or fails over | With `AGRINET_ROUTING_ENABLED`, a failed route is retried on the alternate route unless streaming has begun (`app/services/chat.py`); otherwise restore the provider |
| Qdrant/Marqo outage | Scheme/document search tools fail or degrade | Restore from the ingestion pipeline's source data (owned outside this repo) |
| Image registry unavailable | Cannot pull images | Rebuild locally from git with the `Dockerfile` (requires PyPI, PyTorch index, HuggingFace access) |

## Source of truth

- Code, prompts, glossary: this git repository.
- Built images: GCP Artifact Registry (published by `.github/workflows/build.yml` on pushes to `main` and `bh-dev`; commit-SHA tags are available for pinning).
- Configuration and keys: outside git by design (`.env`, `*.pem`, `*credentials.json` are gitignored).

## Recommended (not yet implemented)

- Document and automate Redis backups (scheduled `BGSAVE` or enabling AOF if the master catalog snapshot must survive restarts), with offsite copies and a periodic restore test.
- Keep the master catalog snapshot reproducible by documenting the docs-pipeline command that republishes it.
- Store `.env` contents and key files in a secrets manager rather than on a single host; document how to re-provision them.
- Pin images by digest or commit SHA in deployments, and record the current SHA on the host to speed up rollback.
- Define RTO/RPO targets and add a short runbook for the host-loss scenario above.
- Add a `HEALTHCHECK` (see [Monitoring](monitoring.md)) so orchestrators restart unhealthy containers.

## Source Files

`docs/setup/redis.md`, `docker-compose.yml`, `supervisord.conf`, `app/utils.py`, `app/services/agrinet_routing.py`, `app/services/chat_turn_map.py`, `helpers/master_catalog.py`, `app/core/image_storage.py`, `app/routers/token.py`, `.github/workflows/build.yml`, `.gitignore`.
