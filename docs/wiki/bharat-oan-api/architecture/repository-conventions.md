# Repository Conventions

How work is organised in this repository: branches, commits, file naming, documentation style, and step-by-step recipes for the three most common extensions (language, tool, router). Branch and commit observations come from read-only `git log` and `git branch -a` output (367 commits on `main` at documentation time).

Related: [Coding Patterns](coding-patterns.md) | [Testing](testing.md) | [Repository Map](../repository-map.md) | [Deployment](../operations/deployment.md) | [Configuration](configuration.md)

## Branching (observed)

| Aspect | Observation |
|--------|-------------|
| Default branch | `main` |
| Long-lived branches | `main`, `bh-dev` (both trigger CI in `.github/workflows/build.yml`), plus `bh-main`, `prod/bh-oan`, `main-bkp`, `bh-dev-bkp` visible on the remotes |
| Feature branches | `feat/<topic>` (for example `feat/adding_or_pa_mai_main`), `fix/<topic>`, `chore/<topic>`, `release/<topic>`, `prod/<topic>`, `codex/<topic>` |
| Ticket-named branches | `OD-<number>-<kebab-description>` (for example `OD-2127-npss-api-main`, `OD-2943-AIF-AI-layer-Integration`) |
| Remotes | `origin` and `upstream` expose the same branch set; PRs come from both the `OpenAgriNet` organisation and contributor forks (merge commits reference `OpenAgriNet/...` and personal forks) |
| Suffix convention | A `-main` or `-dev` suffix on a branch name indicates a copy of the change targeted at `main` or `bh-dev` (for example `OD-2112-Pmkisan-release-message` and `OD-2112-Pmkisan-release-message-main`) |
| Tags | Only two tags exist (`advisory-personalisation-dev-1`, `advisory-personalisation-dev-2`); releases are not tagged |

> **Note:** A written branching policy, protected-branch rules, and release process are not part of this repository. The pattern above is inferred from history. GitHub Environments named after the target branch (`main`, `bh-dev`) are referenced by the build workflow; their configuration lives in GitHub settings.

## Commits and pull requests (observed)

- Work lands through GitHub pull requests: roughly 81 of 367 commits are `Merge pull request #<n> from <owner>/<branch>` merge commits (latest: #247, #235, #231).
- Commit subjects are short and imperative, sentence case, no required prefix: `Upgrade API Docker base image to Python 3.13`, `Add Odia, Punjabi and Maithili (or/pa/mai) language support`.
- Three styles coexist: plain imperative; ticket-prefixed (`OD-2127 fix NPSS response finalization`); and conventional-commit style (`fix: make translation term mapping optional`, `fix(maha-vistaar): target MahaVistaar BPP directly ...`). About 68 subjects use `feat/fix/chore/docs/refactor/test` prefixes.
- Longer commits carry a bullet-list body explaining what changed and why (see `e44d44a`, `6a415a9`).
- Trailers such as `Co-Authored-By:` / `Co-authored-by:` are present for AI-assisted commits.
- Top contributors by commit count: `smrit18`, `anjalirao-b`, `shashank-kenpath`, `Aditya Chhabra`, `Smrithi Shaji`, `kanakkshk-deloitte`.

> **Note:** No pull-request template, `CODEOWNERS`, or commit-lint configuration is part of this repository.

## Merging between environments

Environment branches are kept in sync by cherry-picks and "reconcile" branches (`feat/reconcile-bh-main-into-bh-dev`, `chore/cherry-pick-bh-main-workflow-20261023`, `release/bh-dev-to-bh-main`). Commit messages such as "Apply the runtime image change from 12764e21 onto main" (`2f9f19e`) show the practice of porting a change to `main` rather than merging branches wholesale. CI tags images by branch (see [Deployment](../operations/deployment.md)).

## File and folder naming

| Item | Convention | Example |
|------|-----------|---------|
| Python modules | `snake_case.py` | `app/services/agrinet_routing.py` |
| Routers | one file per resource, filename equals URL prefix | `app/routers/telemetry.py` (`/telemetry`) |
| Tool modules | integration name, with `_scheme_status` / `_grievance` suffix where applicable | `agents/tools/pmfby_grievance.py` |
| Prompts | `assets/prompts/agrinet_<iso-639-code>.md`, `moderation_system.md`, `suggestions_system.md` | `agrinet_mai.md` |
| Tests | `tests/test_<area>.py`; manual runners `tests/check_*.py` or `tests/test_*_single_turn.py` | `tests/test_languages.py` |
| Docs | `docs/setup/<topic>.md` for host setup; `docs/wiki/<service>/` for the wiki | `docs/setup/redis.md` |
| Backups | timestamped folder names (`assets/prompts_backup_20260922_164719/`) | see [Legacy Artifacts](../knowledge/legacy-artifacts.md) |

## Documentation conventions

- The wiki lives in `docs/wiki/bharat-oan-api/` with the layout `business/`, `architecture/`, `application/`, `integrations/`, `operations/`, `knowledge/` plus top-level `README.md`, `wiki-index.md`, `ai-context.md`, `repository-map.md`. It is generated and maintained with the `llm-wiki` skill (`.cursor/skills/llm-wiki/SKILL.md`).
- Paths in docs are repo-relative; links between wiki pages are relative markdown links.
- Claims are labelled **Implemented (verified)** or **Recommended (not yet implemented)**; recommendations are never presented as current fact.
- Call out undocumented areas with plain-language **Note** callouts using the blockquote form `> **Note:** ...` (what is true today, what is not part of the service or wiki, who is responsible). Avoid alarming phrasing such as "not found" or "gap" in reader-facing pages.
- One canonical Mermaid diagram per flow; sequence and workflow diagrams live in `architecture/sequence-diagrams.md`, deployment topology in `operations/deployment.md` and `operations/infrastructure.md`; other pages link instead of copying.
- Debt, bugs and limitations are tracked only in [knowledge/technical-debt.md](../knowledge/technical-debt.md).
- The repo's `README.md` is a short Docker command cheat-sheet and `API_DOCUMENTATION.md` is hand-written; the wiki, [API Specification](../application/api-specification.md) and `/docs` (FastAPI OpenAPI UI) are the more complete references.

## How to add a language

Derived from commit `e44d44a` ("Add Odia, Punjabi and Maithili") and `tests/test_languages.py`.

1. Add `assets/prompts/agrinet_<code>.md` (translate from `agrinet_en.md`, keep every Jinja token such as `{{ today_date }}` unchanged; `test_translated_prompts_keep_english_template_tokens` checks this for `or`, `pa`, `mai`). The agent loads it as `agrinet_{lang_code}` in `agents/agrinet.py`.
2. Update the supported-language list text inside every existing `agrinet_*.md` prompt.
3. In `agents/tools/terms.py`: add the code to `SUPPORTED_LANGS`, add a `Language` enum member, and add a `TermPair` field (use `alias` and a trailing underscore if the code is a Python keyword, as done for `as_` and `or_`; extend the keyword branch of `TermPair.get_term`).
4. Add terms for the new language to every entry in `assets/glossary_terms.json`.
5. In `app/services/npss_response.py`: add the code to `LABELS`, `SOURCE_LINES`, `LANGUAGE_SCRIPT_RANGES`, and the native source prefix to `SOURCE_PREFIXES`.
6. Confirm Bhashini supports the code for transcription, TTS and translation (`helpers/transcription.py`, `helpers/tts.py`, `helpers/translation.py`); the repo notes a sibling service uses `od` for Odia while this one uses `or`.
7. Run `tests/test_languages.py` (it fails if a prompt language is not wired into `SUPPORTED_LANGS`, `LABELS`, `SOURCE_LINES`, or `LANGUAGE_SCRIPT_RANGES`).

## How to add a tool

1. Create `agents/tools/<name>.py`. Use `@observe(name="tool:<function_name>", as_type="tool")` from `langfuse`; write a clear docstring because pydantic-ai exposes it to the model; take `ctx: RunContext[FarmerContext]` when you need the session, language, or coordinates.
2. Call external services with `httpx` and `app.config.get_default_httpx_timeout()` / `DEFAULT_HTTP_TIMEOUT`; read endpoint settings from environment variables; raise `ModelRetry` for retryable problems and return plain, user-safe strings for expected failures. Beckn-based tools read `BAP_ENDPOINT`, `BAP_ID`, `BAP_URI`, `BPP_ID`, `BPP_URI`.
3. Update Langfuse metadata with `helpers/langfuse_tracing.lf_update_current_observation` where other tools do.
4. Import the function in `agents/tools/__init__.py` and append `Tool(<fn>, takes_ctx=<True|False>, strict=False)` to `TOOLS`.
5. Document the tool's trigger rules in the agrinet prompts (all `assets/prompts/agrinet_*.md` languages) so the model knows when to call it.
6. Add the new environment variables to `.env.example` and [Configuration](configuration.md); add tests (see [Testing](testing.md)) and a leakage check keyword if the tool name or its arguments should never appear in answers (`tests/check_streaming_leakage.py` lists tool names).

## How to add a router

1. Create `app/routers/<name>.py` with `router = APIRouter(prefix="/<name>", tags=["<name>"])`.
2. Protect endpoints with `current_user: dict = Depends(get_current_user)` from `app/auth/jwt_auth.py` unless they are intentionally public (as the health and token routers are).
3. Define request/response models in `app/models/requests.py` / `responses.py`.
4. Register it in `main.py`: add it to `from app.routers import ...` and call `app.include_router(<name>.router, prefix=settings.api_prefix)`. (`app/routers/__init__.py` lists only some routers and is not required for registration; `file` and `image` are imported only by `main.py`.)
5. Add a `TestClient` test with `app.dependency_overrides[get_current_user]` as in `tests/test_chat_stream_contract.py` and `tests/test_backend_owned_telemetry.py`.
6. Update [API Specification](../application/api-specification.md).

## Source Files

`.github/workflows/build.yml`, `agents/tools/__init__.py`, `agents/tools/terms.py`, `app/services/npss_response.py`, `tests/test_languages.py`, `main.py`, `.cursor/skills/llm-wiki/SKILL.md`, git history (`git log`, read-only).
