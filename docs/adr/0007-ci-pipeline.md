# ADR-0007: CI is a gate, not a report

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
A pipeline that reports problems and lets the merge through is documentation
with a spinner. The point of CI is that some things become impossible, not
merely visible.

## Decision
Five jobs, aggregated behind a single `ci` check that branch protection points
at:

| Job | Fails the build on |
|---|---|
| `quality` | lint, formatting drift, or a mypy strict error |
| `test` | a failing test, or coverage below the floor in `pyproject.toml` |
| `security` | a known CVE in a shipped dependency, or a secret in git history |
| `build` | the image failing to build, or the built container failing a live smoke test |
| `ci` | any of the above |

Supporting choices:

- **The coverage floor lives in `pyproject.toml`, not in the workflow.** A
  threshold defined in CI drifts from the one developers see locally, and the
  first anyone hears about it is a red build.
- **`security` is its own job.** A CVE disclosed overnight then shows up as a
  security failure rather than looking like a lint error in a mixed job.
- **pip-audit runs against the exported lockfile**, not the installed
  environment: it is exactly what ships in the container, it excludes dev
  tooling that never reaches production, and it avoids pip-audit trying to
  resolve this project on PyPI.
- **gitleaks runs from its image**, not `gitleaks-action`, which requires a paid
  licence when the repository belongs to an organisation.
- **The build job runs a smoke test** -- boots the image, waits for it, and
  asserts `/health` and `/ready`. An image that builds is not an image that runs.
- **The image is built but never pushed.** Publishing is a release concern, and
  a CI job holding registry credentials on every pull request is a far larger
  blast radius than one without them.
- **`permissions: contents: read` at the top level**, with jobs escalating only
  if they need to.
- **`concurrency` with `cancel-in-progress`**, so a new push frees the runner
  its predecessor was holding.

## Alternatives considered
- **One job running everything.** Simpler, and gives one undifferentiated red
  cross. Five jobs tell you *what* broke from the check name alone, and they run
  in parallel.
- **Making all five required in branch protection.** Works, until someone adds a
  sixth job and forgets the rule, which then silently stops being enforced. One
  aggregate check that `needs` the rest cannot drift.
- **Coverage reporting without a floor.** Coverage that is measured but not
  enforced only ever goes down.
- **Actions pinned to commit SHAs.** The hardened choice, and correct for a
  regulated production repository: a tag can be moved, a SHA cannot. Not adopted
  here because it makes the file substantially harder to read for a template
  whose job is to be understood by whoever copies it, and because Dependabot
  keeps the tags current. **A service handling regulated data should escalate to
  SHA pinning**; that is a per-service decision, recorded in that service's own
  ADR.
- **Pushing the image from CI.** Deferred to a release workflow with its own
  environment and credentials.

## Consequences
Nothing merges that is unformatted, untyped, untested, below the coverage floor,
carrying a known CVE, or unable to boot. The cost is roughly three minutes per
pull request and the discipline not to lower the floor when it is inconvenient.
GitHub Actions cache keeps the container build near a minute after the first run.
