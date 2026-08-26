# ADR-0005: Applications are built by a factory, not at import time

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
A module-level `app = FastAPI()` reads configuration during import, which makes
importing the module a side-effecting operation and makes tests share one
global instance.

## Decision
`create_app(settings: Settings | None = None) -> FastAPI`. Tests build an app
with whatever `Settings` they need. A module-level `app = create_app()` is kept
for `uvicorn app.main:app`.

## Alternatives considered
- **Module-level app.** Importing the module fails if an environment variable is
  missing -- including in a linter or an IDE.
- **Dependency-injection container.** More machinery than a service of this size
  needs, and one more thing for a reader of the template to learn.

## Consequences
Tests are isolated by construction and can exercise production-only behaviour
(such as docs being disabled) without environment variables. Anything that must
happen once per process goes in the `lifespan` handler, not at module scope.
