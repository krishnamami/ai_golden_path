# ADR-0006: Multi-stage image, non-root, pinned, read-only at runtime

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
The image is the deployment unit. Every service built from this template
inherits its size, its attack surface, its build time and its shutdown
behaviour, so the defaults here are worth getting right once.

## Decision
- **Two stages.** A builder resolves and installs into `/app/.venv`; the runtime
  stage copies only that venv. uv, compilers and build caches never ship.
- **Dependencies installed before source is copied**, so an ordinary code edit
  reuses the dependency layer instead of re-resolving 39 packages.
- **`uv sync --locked`.** The build fails if `uv.lock` has drifted from
  `pyproject.toml`, rather than quietly resolving something new.
- **Pinned uv version** (`ghcr.io/astral-sh/uv:0.8.17`), not `latest`.
- **Non-root with a fixed uid/gid (1001)**, because Kubernetes `runAsUser`
  takes a number and a named user it cannot resolve breaks the mount.
- **`HEALTHCHECK` hits `/health`, not `/ready`** — see ADR-0004.
- **Exec-form `ENTRYPOINT`**, so uvicorn is PID 1 and receives SIGTERM.
- **`read_only: true` with a tmpfs at `/tmp`** in compose.

## Alternatives considered
- **Single-stage build.** Simpler, and ships a compiler toolchain and package
  cache to production.
- **`latest` for the uv image.** The build is then not reproducible: the image
  shipped on Friday was not produced by the toolchain tested on Monday.
- **Shell-form `ENTRYPOINT`.** Puts `/bin/sh` at PID 1, which does not forward
  SIGTERM. Every deploy then waits out the full kill timeout before the
  container dies, turning a rolling update into a minutes-long stall.
- **A named non-root user without a fixed uid.** Works locally, fails under a
  `runAsUser` policy.
- **Reading the port from `APP_PORT`.** Exec form does not expand variables, and
  the shell form that would is ruled out above. Port remapping belongs to the
  host (`-p 9000:8000`).
- **Alpine base.** Smaller, but musl breaks manylinux wheels and pushes you into
  source builds for anything with C extensions. Not worth it for a template that
  will grow a numeric dependency the moment it does anything AI-shaped.

## Consequences
The runtime image contains a Python runtime, a virtualenv and the source — no
package manager and no compiler. `read_only: true` means a future change that
starts writing to disk fails locally rather than under a hardened production
policy; if a dependency genuinely needs a writable path, add a named tmpfs for
it rather than removing the flag.
