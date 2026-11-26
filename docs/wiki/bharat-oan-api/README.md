# bharat-oan-api Wiki

**Bharatvistaar AI API** is a FastAPI backend that answers Indian farmers' questions in 13 languages. Each chat turn runs a moderation agent and then a tool-using LLM agent (pydantic-ai) that reaches government-scheme, mandi, weather, fertilizer, seed, search and crop-image services, and streams the answer. It also provides speech-to-text, text-to-speech, image upload, telemetry relay and JWT token issuance. State lives in Redis; there is no relational database.

This wiki serves two readers: new engineers (onboarding) and AI agents (long-term repository memory). Everything was derived from the code in this repository and is labelled **Implemented**, **Recommended**, or covered by a **Note** where a topic is outside the repository.

**Fastest path:** [AI Context](ai-context.md) -> this page -> [Architecture Overview](architecture/architecture-overview.md). For the full navigation hub see [wiki-index.md](wiki-index.md).

# Executive Documentation
* [AI Context](ai-context.md)
* [Architecture Overview](architecture/architecture-overview.md)
* [Business Overview](business/business-overview.md)
* [Repository Map](repository-map.md)

# Business
* [Users](business/users-and-personas.md)
* [Workflows](business/workflows.md)
* [Business Rules](business/business-rules.md)

# Architecture
* [Components](architecture/component-diagram.md) (system context and internal components)
* [Sequence & Workflow Diagrams](architecture/sequence-diagrams.md)
* [Configuration](architecture/configuration.md)
* [Testing](architecture/testing.md)
* [Coding Standards](architecture/coding-patterns.md)
* [Repository Conventions](architecture/repository-conventions.md)

# Application
* [Frontend](application/frontend.md)
* [Backend](application/backend.md)
* [APIs](application/api-specification.md)
* [Database](application/database.md)

# Integrations
* [AI/LLM](integrations/ai-llm.md)
* [Third Party Services](integrations/third-party-integrations.md)
* [Cache](integrations/caching.md)
* [Async Processing](integrations/async-processing.md)

# Operations
* [Infrastructure](operations/infrastructure.md)
* [Deployment](operations/deployment.md)
* [Monitoring](operations/monitoring.md)
* [Troubleshooting](operations/troubleshooting.md)
* [Disaster Recovery](operations/disaster-recovery.md)

# Knowledge Base
* [Glossary](knowledge/glossary.md)
* [Technical Debt](knowledge/technical-debt.md)
* [References](knowledge/references.md)
* [Legacy Artifacts](knowledge/legacy-artifacts.md)

## Quick Facts

| Item | Value |
|------|-------|
| Language / runtime | Python 3.13, FastAPI, uvicorn |
| Entry point | `main.py` |
| Main flow | `GET /api/chat/` -> `app/services/chat.py` -> `agents/agrinet.py` |
| Data stores | Redis, Qdrant, Marqo, local disk (images) |
| Run | `docker compose up --build --force-recreate --detach` |
| Health | `GET /api/health/ready` |
