# Wiki Index

Navigational hub for the `bharat-oan-api` wiki. The executive entry point with a service summary is [README.md](README.md).

## Document Control

| Attribute | Value |
|-----------|-------|
| Service | `bharat-oan-api` |
| Generated | 2026-11-26 |
| Method | Repository analysis using the `llm-wiki` skill (`.cursor/skills/llm-wiki/SKILL.md`) |
| Source of truth | Repository source code, configuration, Docker and CI assets, tests |
| Status labels | **Implemented (verified)**, **Recommended (not yet implemented)** |
| Gaps | Called out with plain-language **Note** sections where details are not documented in this wiki |
| Maintenance | Update the relevant page when behaviour changes; re-run the skill for a full refresh |

## Table of Contents

# Executive Documentation
* [AI Context](ai-context.md)
* [Architecture Overview](architecture/architecture-overview.md)
* [Business Overview](business/business-overview.md)

# Business
* [Users](business/users-and-personas.md)
* [Workflows](business/workflows.md)
* [Business Rules](business/business-rules.md)

# Architecture
* [Components](architecture/component-diagram.md) (includes system context)
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

# Repository
* [Repository Map](repository-map.md)

## Reading Paths

| Goal | Read |
|------|------|
| Understand the service in 10 minutes | [AI Context](ai-context.md), [README](README.md), [Architecture Overview](architecture/architecture-overview.md) |
| Add or change an API | [API Specification](application/api-specification.md), [Backend](application/backend.md), [Coding Patterns](architecture/coding-patterns.md) |
| Change agent behaviour or prompts | [AI/LLM](integrations/ai-llm.md), [Business Rules](business/business-rules.md) |
| Run or deploy | [Configuration](architecture/configuration.md), [Deployment](operations/deployment.md), [Troubleshooting](operations/troubleshooting.md) |
| Plan clean-up work | [Technical Debt](knowledge/technical-debt.md), [Legacy Artifacts](knowledge/legacy-artifacts.md) |

## Source / Raw Documents

> **Note:** The repository has no `docs/raw_documents/` folder. Other prior documents are listed below; code is the source of truth wherever they differ.

| Document | Location | Notes |
|----------|----------|-------|
| Project README | `README.md` | Docker commands only |
| API documentation | `API_DOCUMENTATION.md` | Partly outdated; see [API Specification](application/api-specification.md#reconciliation-with-api_documentationmd) |
| Redis and Portainer setup | `docs/setup/redis.md`, `docs/setup/portainer.md` | Portainer page contains a clear-text credential, see [TD-01](knowledge/technical-debt.md#td-01--administrator-credential-and-host-details-committed-in-docs) |
| Crop advisory SOP | `docs/data/archive/CROP ADVISORY SOP.docx` | Marathi, not referenced by code; appears to contain personal phone numbers, review before sharing. See [Legacy Artifacts](knowledge/legacy-artifacts.md) |
