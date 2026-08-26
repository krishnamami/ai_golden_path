# ADR-0002: Configuration comes from the environment and is validated at startup

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
The same container image must run unchanged on a laptop, in CI, and in three
deployed environments. Configuration mistakes -- a missing variable, a typo in a
deploy manifest -- are among the most common causes of production incidents, and
they are cheapest to catch before traffic arrives.

## Decision
All configuration is read from `APP_*` environment variables into a single typed
`Settings` model, which is:

- **frozen** -- configuration does not change while the process runs;
- **`extra="forbid"`** -- an unrecognised `APP_*` variable fails startup rather
  than being silently ignored;
- **policy-bearing** -- `staging` and `prod` reject console-formatted logs, and
  `prod` rejects `DEBUG` level, at validation time.

## Alternatives considered
- **`os.getenv` at point of use.** No types, no validation, and failures surface
  on the first request that happens to touch that setting.
- **YAML config files per environment.** Another artefact to keep in sync with
  the deploy, and secrets end up in the repository.
- **`extra="ignore"`.** Tolerant of typos, which is exactly the failure we are
  trying to prevent: `APP_LOG_LEVL=DEBUG` would be accepted and do nothing.

## Consequences
A bad configuration crashes at boot, in front of a deploy pipeline, instead of
in front of a user. The cost is that adding a variable means editing the model --
which is the point. Tests construct `Settings()` directly rather than using the
cached `get_settings()`.
