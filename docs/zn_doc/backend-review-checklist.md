# Backend Maintainer Pull Request Review Checklist

> **Purpose**
>
> This document defines the review checklist used by a **backend maintainer** when reviewing pull requests.
>
> It serves as the primary review guide to ensure that every contribution is **correct, maintainable, secure, consistent with the project's architecture, and aligned with established conventions**.
>
> This is a **living document**. As more knowledge about the project is acquired (architecture, coding conventions, recurring patterns, design decisions, etc.), this checklist should be **updated** by replacing generic guidance with project-specific rules while preserving its overall structure.
>
> **Priority Order**
>
> 1. Project-specific conventions and architectural decisions
> 2. This checklist
> 3. General backend engineering best practices

---

# Project Knowledge Base

> Last verified against the repository on 2026-07-04. Re-verify claims below (grep the
> referenced file) before relying on them — this is a fast-moving CE codebase and the
> domain layer is still being built out.

## Project Overview

- **Project Name:** Open TutorAI CE (Community Edition)
- **Purpose:** Standalone educational AI / tutoring platform. Started as an OpenWebUI
  fork but is being re-architected into its own domain model — see Rule #1 below.
- **Tech Stack:** Python 3.11/3.12 backend, TypeScript/Svelte 4 frontend.
- **Framework(s):** FastAPI (backend), SvelteKit + Tailwind CSS 4 (frontend, `ui/`).
- **Database:** SQLAlchemy 2.0, **sync** engine/sessions only (`data/database.py`).
  Default `sqlite:///./var/tutorai.db`; Postgres/MySQL supported via `DATABASE_URL`.
  `peewee`/`peewee-migrate` remain in `pyproject.toml` as OpenWebUI-era holdovers —
  do not build new code on peewee, it should shrink over time, not grow.
- **Architecture:** Hermes-pattern layering: `<boundary>/<domain>/{repository.py,service.py}`
  → `gateway/http/routers/<name>.py` (thin HTTP layer) → `gateway/http/app.py` (router
  registration). Boundaries: `accounts`, `learning`, `ai`, `content`, `governance`, `system`.
  Realtime via `gateway/realtime/socket.py` (Socket.IO), mounted at `/realtime`.
- **API Style:** REST/JSON, versioned under `/api/v1/*`. A few routers are additionally
  mounted at bare paths for UI-bootstrap compatibility (e.g. `auth` at both `/auths/*`
  and `/api/v1/auths/*` — intentional dual-mount, see `gateway/http/app.py`). OpenAPI
  schema is FastAPI-generated and is itself asserted against by
  `tests/test_contract_coverage.py`.
- **Testing Framework:** pytest + `TestClient`, in-memory SQLite per test
  (`tests/conftest.py`). Frontend: Vitest (`ui/npm run test:frontend`).
- **CI/CD Pipeline:** GitHub Actions — `ci-backend.yaml` (Black lint job gates the
  pytest job), `ci-frontend.yaml` (eslint/prettier/i18n-keys/build gate Vitest),
  `build-release.yml` (tag `v*` → GitHub release from CHANGELOG). CI installs
  `requirements-ci.txt` (minimal set), not the full `requirements.txt`.

---

## Coding Conventions

### Naming Conventions

- Python: `snake_case` modules/functions, `PascalCase` classes. Domain classes are
  named `<Domain>Service` / `<Domain>Repository` (e.g. `AccountService`, `UserRepository`).
- Router files are named after the URL prefix/public namespace as a plural noun
  (`chats.py` → `/chats`, `users.py` → `/users`), not after the internal domain folder
  name (e.g. `learning/sessions/service.py` defines `ChatsService`, consumed by
  `gateway/http/routers/chats.py`) — internal domain name and public route name can differ.
- Frontend API clients live at `ui/src/lib/apis/<domain>/index.ts`, one folder per
  backend namespace.

### Folder Structure

- See CLAUDE.md's architecture diagram for the canonical layout.
- **Not every boundary folder is implemented yet** — many are still empty `__init__.py`
  stubs (e.g. `accounts/auth`, `accounts/permissions`, `accounts/roles`,
  `learning/classrooms`, `learning/courses`, `learning/learners`, `learning/teachers`,
  `ai/media`, `ai/memory`, `ai/tools`, `content/resources`). Don't assume a boundary is
  "done" just because the folder exists — check for `repository.py`/`service.py` inside it.
- Domains with a full repository/service pair today: `accounts/users`,
  `content/files`, `governance/self_regulation`, `learning/sessions`, `learning/supports`,
  `ai/model_catalog`, `ai/retrieval/knowledge`.
- **Config-style domains are a deliberate exception to the repository pattern:**
  `system/configs` (`ConfigsService`), `ai/retrieval` (top-level `service.py`), and
  `ai/providers` (`ProviderConfigService`) query the `AppConfig` key/value ORM model
  (`data/models/config.py`) directly from the service — no `repository.py`. Don't
  flag a missing repository for this shape of domain; do flag raw `AppConfig` queries
  showing up in a router instead of a service.

### Controllers (Routers)

- One `APIRouter(prefix=..., tags=[...])` per file in `gateway/http/routers/`.
- Service obtained via `Depends(get_<x>_service)`. **Inconsistency to flag:** most
  factories live centrally in `gateway/http/dependencies.py`, but some routers
  (`chats.py`, `providers.py`) define their own local factory instead — new routers
  should add their factory to `dependencies.py` unless there's a good reason not to.
- Admin/ownership checks are inline (`if not current_user.is_admin: raise HTTPException(403, ...)`),
  sometimes via a local `_require_admin()` helper (see `providers.py`). There is no
  centralized permission decorator/policy layer yet outside `governance/`.
- Request bodies: inline `pydantic.BaseModel` classes defined in the router file, or
  loosely-typed `Dict[str, Any]` for config-style passthrough endpoints. No dedicated
  `schemas.py` convention exists yet.

### Services

- One `<Domain>Service` class, constructor takes `session: Session` and builds its
  own repository (`self.repo = XRepository(session, Model)`).
- Business logic and ownership/authorization checks live here (e.g.
  `FilesService.require_owned`), not in the router.
- Services raise domain exceptions from `common/exceptions.py` — never `HTTPException`
  (that belongs strictly to the router layer).

### Repositories / Data Access

- Inherit `data/repositories/base.py::BaseRepository[T]` for generic CRUD
  (`create`, `get_by_id`, `get_all`, `update`, `delete`).
- `BaseRepository.create`/`update`/`delete` each commit and refresh immediately —
  there is no unit-of-work/explicit transaction boundary spanning multiple repo calls;
  multi-step operations commit incrementally.
- Only `BaseRepository.update` raises `NotFoundError` itself; most domain-specific
  repo methods (`get_by_email`, `get_by_id`, etc.) just return `Optional[T]` and leave
  the "not found → error" decision to the service.

### DTOs / Models

- ORM models live in `data/models/<entity>.py`, registered in `data/models/__init__.py`.
- No consistent Pydantic *response* model convention: routers commonly return
  `model.to_dict()` (an ORM instance method) or raw dicts shaped to match what the
  ported OpenWebUI SvelteKit UI already expects — this is deliberate for UI-contract
  compatibility, not an oversight.

### Validation

- No centralized validation layer. FastAPI/Pydantic validates request bodies only
  where a `BaseModel` is declared; `Dict[str, Any]` bodies get no schema validation.
- Domain-level validation (e.g. upload size limit in `FilesService.upload`) raises
  `ValidationError` from the service layer.

### Error Handling

- `common/exceptions.py` defines the `TutorAIException` hierarchy: `AuthenticationError`,
  `AuthorizationError`, `ValidationError`, `NotFoundError`, `DatabaseError`,
  `FileOperationError` — each carries a `code` string.
- **There is no global FastAPI exception handler for these** (verified: no
  `@app.exception_handler` registered in `gateway/http/app.py`). Each router must
  catch `NotFoundError`/`AuthorizationError`/`ValidationError` explicitly and translate
  to `HTTPException` + status code (see `files.py`, `knowledge.py`, `models.py`,
  `supports.py` for the established pattern). A service that raises one of these
  without the calling router catching it will surface as an unhandled 500 — treat
  this as a blocker in review.
- Some services still raise plain `ValueError` instead of a domain exception (e.g.
  `AccountService.create_user` on duplicate email) — routers catch these separately;
  don't assume every service exclusively uses `common/exceptions.py` types.

### Logging

- `common/logging.py::get_logger(name)` exists as a shared logging helper (configures
  level from `settings.LOG_LEVEL`, a `StreamHandler`, and a
  `%(asctime)s - %(name)s - %(levelname)s - %(message)s` formatter, with
  `propagate = False` to avoid duplicate lines) — **but it is currently unused
  everywhere** (verified: no file imports `common.logging`). Treat it as the intended
  convention for new code rather than dead code to delete.
- What's actually used today is ad hoc `logging.getLogger(__name__)` in a handful of
  `ai/providers/*` modules and `gateway/realtime/socket.py` / `gateway/http/api_routes.py`,
  plus bare `print()` for startup banners in `main.py` and `gateway/http/app.py`'s
  lifespan hook.
- Don't block a PR for matching the ad hoc pattern already in its file, but for new
  modules prefer `common.logging.get_logger(__name__)` over both raw `logging.getLogger`
  and `print()` — and consider raising the inconsistency rather than silently
  perpetuating a fourth pattern.

### Dependency Injection

- FastAPI `Depends()` throughout. `get_db` (`data/database.py`) is the base session
  dependency.
- Auth guard: `get_current_user` (`gateway/http/dependencies.py`) decodes the JWT
  (`settings.JWT_SECRET_KEY` / `JWT_ALGORITHM`) and loads the user via `AccountService`.
- Service factories should be added to `gateway/http/dependencies.py` (see
  "Controllers" note above for the existing exceptions to this).

### Authentication

- JWT bearer tokens (`HTTPBearer` + `PyJWT`), secret sourced from the `SECRET_KEY`
  env var (`settings.JWT_SECRET_KEY`).
- `config/settings.py` fails fast at import time if `SECRET_KEY` is unset and
  `DEBUG` is not `true` — this is a deliberate production safety check; never weaken
  or bypass it outside of dev/test setup (`tests/conftest.py` sets a test key via
  `os.environ.setdefault`).

### Authorization

- Role/ownership checks are manual and inline per-router (`current_user.is_admin`,
  `current_user.id != resource.user_id`). No policy/permission framework exists yet
  beyond the `governance/self_regulation` (HITL) domain.
- Put ownership checks in the service layer when they gate a business rule (pattern:
  `FilesService.require_owned`); simple admin-only gating is fine directly in the router.

### API Response Format

- No standard response envelope (no `{data, error}` wrapper) — endpoints return raw
  JSON dicts/lists or `model.to_dict()`, matching the shape the SvelteKit UI already
  expects from its OpenWebUI ancestry.
- Changing a response shape is a **breaking change for the UI**. Before approving a
  shape change, check the corresponding `ui/src/lib/apis/<domain>/index.ts` consumer.

### Testing Conventions

- `tests/conftest.py` fixtures: `db` (fresh in-memory SQLite engine per test) and
  `client` (FastAPI `TestClient` with `get_db` overridden to the test session).
- Test files: `tests/test_<domain>.py`, plain `def test_<behavior>(client):` functions
  (no test classes).
- **`tests/test_contract_coverage.py` is the project's most important structural
  test.** It scans `ui/src/lib/apis/**/*.ts` for `fetch()` calls and asserts every
  `(METHOD, path)` exists in the OpenAPI schema, and separately asserts no legacy
  OpenWebUI path patterns (`/openai/`, `/ollama/`, `/api/chat`, `/ws/socket`,
  `open_webui`) leak into the backend schema or UI source. Any new router must not
  need a new entry in `_SCANNED_PATH_EXCLUSIONS`; any UI `fetch()` call needs either a
  real backend route or a documented, reasoned exclusion entry.

### Common Utilities & Helpers

- `common/exceptions.py` — shared exception hierarchy.
- `common/logging.py::get_logger()` — shared logger factory (currently unused, but the
  intended convention — see "Logging" above).
- `data/repositories/base.py` — generic CRUD base (`BaseRepository`).
- `config/settings.py` (singleton `settings`, import as `from config import settings`)
  and `config/constants.py`.
- `ai/providers/proxy.py` (`proxy_json`, `proxy_stream`, `resolve_url_key`,
  `resolve_ollama_url`) — shared helpers for proxying to OpenAI-compatible/Ollama
  backends; reuse these instead of hand-rolling new upstream HTTP calls.

### Preferred Design Patterns

- Repository → Service → Router 3-layer pattern (mandatory for new domains, per
  CLAUDE.md).
- Provider proxy pattern for external LLM backends (`ai/providers/proxy.py`,
  `ai/providers/ollama_native.py`): thin pass-through + URL/key resolution, not a
  full SDK wrapper.

---

## Project-Specific Rules

- **Never import from `open_webui` at runtime** (CLAUDE.md Rule #1). Partially
  enforced by `tests/test_contract_coverage.py::test_no_forbidden_paths_in_ui` /
  `test_forbidden_patterns_absent`, which fail CI if legacy OpenWebUI path patterns
  or `open_webui` appear in the UI source or the registered OpenAPI schema.
- **Every UI `fetch()` call needs a matching backend route** — enforced by
  `tests/test_contract_coverage.py::test_required_paths_present`. A new frontend API
  call must ship with either a working router or an entry (with reason) in
  `_SCANNED_PATH_EXCLUSIONS`.
- **Black 24.8.0 formatting is mandatory and CI-blocking** — the `lint` job in
  `ci-backend.yaml` must pass before the `test` job even runs.
- **New domains must follow the repository/service/router layering** from CLAUDE.md,
  register their ORM model in `data/models/__init__.py`, and remove the corresponding
  route from `_SCANNED_PATH_EXCLUSIONS` in `tests/test_contract_coverage.py` once
  implemented (per CLAUDE.md's "Adding a new domain" steps).
- **Routers must explicitly catch domain exceptions** (`NotFoundError`,
  `AuthorizationError`, `ValidationError`) since there is no global exception handler
  — an uncaught one becomes an unhandled 500. Flag this as a blocker, not a suggestion.
- **`SECRET_KEY` must never silently default in production**; `DEBUG=true` is the
  only sanctioned bypass, enforced at import time in `config/settings.py`. Don't
  approve changes that weaken this check.
- **Response shapes mirror the OpenWebUI/SvelteKit UI contract** — don't "clean up" a
  router's response shape without checking/updating the matching
  `ui/src/lib/apis/**/index.ts` client in the same PR.
- **CI installs `requirements-ci.txt`, not `requirements.txt`** — a dependency only
  added to `requirements.txt` won't be available to backend CI; check both files when
  reviewing new third-party dependencies.

This section becomes the highest-priority reference during future reviews. Re-verify
stale claims (grep the referenced file/pattern) before trusting them, since this is a
living document and the codebase moves faster than it does.

---

# Pull Request Review Checklist

## 1. Understand the Change

Before reviewing any code:

- Read the PR title and description.
- Read linked issues or discussions.
- Understand the purpose of the change.
- Identify affected modules.
- Understand why the implementation was chosen.

Do not review implementation details without first understanding the intended behavior.

---

## 2. Scope

Verify that:

- The PR addresses a single concern.
- No unrelated changes are included.
- No accidental formatting-only modifications exist.
- No unnecessary file moves or renames were introduced.
- No unrelated refactoring is mixed into the PR.

Large PRs should be split whenever practical.

---

## 3. Correctness

Verify that the implementation:

- Solves the intended problem.
- Matches the expected behavior.
- Handles common edge cases.
- Preserves existing functionality.
- Does not introduce regressions.

Pay particular attention to:

- Invalid inputs
- Null or undefined values
- Empty collections
- Boundary conditions
- Concurrency
- Race conditions
- Pagination
- Failure scenarios

---

## 4. Architecture

Ensure the implementation follows the project's architecture.

Examples:

- Controllers (routers) remain thin — HTTP concerns only, no ORM queries.
- Business logic belongs in services; services take a `Session` and build their own
  repository, and raise `common/exceptions.py` types, never `HTTPException`.
- Database access stays inside repositories, subclassing `BaseRepository`.
- New domains follow `<boundary>/<domain>/{repository.py,service.py}` and are wired
  through `gateway/http/routers/<name>.py` + `gateway/http/app.py`, per CLAUDE.md's
  "Adding a new domain" steps.
- Service factories are added to `gateway/http/dependencies.py`, not redefined locally
  in the router (existing exceptions: `chats.py`, `providers.py` — don't add more).
- Dependency injection follows `Depends(get_db)` → `Depends(get_<x>_service)`.

Prefer existing project patterns over introducing new ones.

---

## 5. Code Quality

Review for:

- Clear and meaningful naming.
- Readable code.
- Small, focused functions.
- Low complexity.
- Minimal duplication.
- No dead code.
- No commented-out code.
- No debugging statements.

Prefer simplicity whenever possible.

---

## 6. Consistency

Ensure consistency with existing project conventions:

- Naming
- Folder structure
- Formatting
- Logging
- Error handling
- API responses
- Dependency usage
- Existing helper functions

Consistency is generally preferred over personal preference.

---

## 7. Performance

Look for meaningful performance concerns such as:

- N+1 database queries
- Unnecessary database access
- Expensive repeated computations
- Inefficient loops
- Blocking operations
- Missing pagination
- Redundant serialization
- Obvious scalability issues

Avoid premature optimization comments.

---

## 8. Security

Review for:

- Authentication — JWT via `HTTPBearer`/`get_current_user`; no bypass of the
  `SECRET_KEY` production check in `config/settings.py`.
- Authorization — admin/ownership checks present and correct (`current_user.is_admin`,
  `current_user.id == resource.user_id`); no domain gaining silent access because a
  new router forgot the check other routers in its domain have.
- Input validation
- Injection vulnerabilities (raw SQL string interpolation; `ilike`/query filters
  should use SQLAlchemy parameter binding as existing repos do)
- Path traversal — especially in `content/files` (upload/read paths) and Ollama model
  upload/download routes in `gateway/http/routers/providers.py`
- Unsafe file handling — upload size limits enforced (`FilesService.upload`'s
  `MAX_UPLOAD_SIZE_MB` check), safe filename handling
- Secret exposure — no API keys/tokens echoed back in responses or logged
- Sensitive logging
- Insecure defaults — e.g. don't widen `CORS_ALLOW_ORIGIN`/`ALLOWED_HOSTS` defaults
  without cause; don't loosen the `ollama_download` allowed-hosts allowlist in
  `providers.py` without justification

Security issues should receive high priority.

---

## 9. Error Handling

Verify that:

- Domain exceptions (`NotFoundError`, `AuthorizationError`, `ValidationError`, etc.,
  from `common/exceptions.py`) raised by a service are explicitly caught in the
  calling router and translated to an `HTTPException` with the right status code —
  there is **no global exception handler**, so an uncaught one is an unhandled 500.
  Treat a missing catch as a blocker.
- Errors return suitable HTTP status codes.
- Internal implementation details are not exposed.
- Logs contain useful diagnostic information.
- Failures are not silently ignored.

---

## 10. Database

Review:

- Schema changes
- Migrations
- Transactions
- Constraints
- Indexes
- Query efficiency
- Data consistency
- Rollback safety

---

## 11. API Design

When APIs are modified, verify:

- Endpoint naming — public prefix matches the plural-noun router file name; new
  routes live under `/api/v1/*` unless there's a documented bootstrap reason (like
  `auth`'s dual `/auths/*` + `/api/v1/auths/*` mount) not to.
- HTTP methods
- Request validation
- Response consistency — shape matches what `ui/src/lib/apis/<domain>/index.ts`
  expects; no silent envelope/shape changes (see "API Response Format" in the
  Project Knowledge Base).
- Backward compatibility
- Pagination
- Filtering
- Sorting
- New/changed routes are reflected in `tests/test_contract_coverage.py` (either a
  passing scan or a justified `_SCANNED_PATH_EXCLUSIONS` entry), and don't reintroduce
  any of the `FORBIDDEN_PATTERNS` (`/openai/`, `/ollama/`, `/api/chat`, `/ws/socket`,
  `open_webui`) outside their allowed exceptions.

---

## 12. Tests

Verify that:

- New functionality is adequately tested.
- Regression tests are included where appropriate.
- Edge cases are covered.
- Existing tests remain valid.
- Test quality is acceptable.

Do not require tests for trivial documentation or formatting changes.

---

## 13. Documentation

Ensure documentation is updated whenever applicable:

- README
- API documentation
- Environment variables — new/changed env vars should be reflected in CLAUDE.md's
  Environment Variables table
- Configuration
- Deployment instructions (`devops/scripts/*.sh`, `devops/docker/*`)
- Migrations
- Changelog
- i18n — new user-facing frontend strings must have translation keys added (`ui/src/lib/i18n/locales`,
  AR/FR/EN); CI's `npm run i18n:parse` step fails the build if keys are out of sync.

Documentation should be updated only when changes affect users or contributors.

---

## 14. Dependencies

Review newly introduced dependencies.

Determine whether:

- The dependency is necessary.
- An existing project solution already exists.
- The dependency is actively maintained.
- It introduces security concerns.
- It significantly increases complexity.

---

## 15. Git Hygiene

Check for:

- Meaningful commit messages.
- Accidental files.
- Generated artifacts.
- Merge conflicts.
- Temporary files.
- Clean commit history where applicable.

---

## 16. CI/CD

Verify that:

- The project builds successfully.
- Tests pass — backend: `pytest -q` (see `ci-backend.yaml`); frontend: `npm run test:frontend`
  (`ci-frontend.yaml`), which needs a SvelteKit build first.
- Linting passes — Black 24.8.0 for Python (`black . --check --exclude ".venv/|/venv/|ui/"`,
  gates the pytest job); ESLint + Prettier for the frontend (gates frontend build/test jobs).
- Static analysis or type checking passes — frontend `svelte-check` (`npm run lint:types`)
  currently runs with `continue-on-error: true` in `ci-frontend.yaml` (comment there notes
  5026 pre-existing type errors being paid down); don't let new code add to that count
  even though CI won't currently block on it.
- New dependencies are added to the correct file: `pyproject.toml`/`requirements.txt` for
  full backend, and `requirements-ci.txt` too if CI (lint/pytest) needs it.
- Generated files are committed when required (e.g. i18n locale files after `npm run i18n:parse`).

---

## 17. Maintainability

Ask:

- Will future contributors easily understand this code?
- Is the implementation unnecessarily complex?
- Can it be extended without major refactoring?
- Does it introduce unnecessary technical debt?
- Is the abstraction level appropriate?

Maintainability should always be considered alongside correctness.

---

# Severity Levels

## 🚫 Blocker

Must be resolved before merging.

Examples:

- Incorrect functionality
- Security vulnerability
- Regression
- Broken architecture
- Data corruption
- Failing tests
- Breaking API changes

---

## ⚠️ Major

Should normally be addressed before merging.

Examples:

- Missing validation
- Missing tests
- Significant maintainability concerns
- Performance issues
- Architectural inconsistencies

---

## 💡 Minor

Small improvements.

Examples:

- Naming
- Readability
- Documentation
- Small refactoring opportunities

---

## 💬 Suggestion

Optional improvements that may enhance the implementation without being required.

---

# Review Output Format

For each finding, provide:

```text
Severity:
Category:
Location:

Problem:
Describe the issue clearly.

Why it matters:
Explain its impact.

Recommendation:
Describe how it can be improved.

Reference:
Mention an existing project pattern when applicable.
```

---

# Review Principles

- Prioritize correctness over style.
- Follow project conventions before general best practices.
- Focus on maintainability and long-term consistency.
- Avoid speculative or hypothetical issues.
- Do not request unrelated refactoring.
- Review only code impacted by the pull request unless it exposes an existing issue.
- Keep feedback actionable, respectful, and concise.
- Explain the reasoning behind each finding.
- Distinguish clearly between required changes and optional suggestions.

---

# Continuous Improvement

This checklist should evolve alongside the project.

Whenever new architectural decisions, coding conventions, recurring review comments, or best practices emerge, incorporate them into this document so future reviews remain consistent, accurate, and aligned with the project's standards.
