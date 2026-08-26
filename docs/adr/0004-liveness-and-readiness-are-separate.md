# ADR-0004: Liveness and readiness are separate endpoints

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
Orchestrators ask two different questions: *should I restart this process?* and
*should I send it traffic?* A single `/health` endpoint answers both badly.

## Decision
- `GET /health` -- liveness. Returns 200 whenever the process is running. It
  checks **no dependencies**, ever.
- `GET /ready` -- readiness. Runs every registered dependency check and returns
  503 if any fails, with a per-check breakdown.

Dependencies register through `register_readiness_check(name, coro)` at startup.
The template registers none, because it depends on nothing.

## Alternatives considered
- **One `/health` that checks dependencies.** The common default, and it turns a
  slow database into a restart loop: the orchestrator concludes the process is
  dead, kills it, the replacement also cannot reach the database, and the
  service that was merely degraded is now down.
- **Readiness returning 200 with a status body.** Load balancers act on status
  codes, not payloads. It would never be taken out of rotation.

## Consequences
Both endpoints are unversioned, because probes are configured by the platform
rather than by API consumers and must not move between API versions. A failing
check returns its exception *type* only -- exception messages leak connection
strings.
