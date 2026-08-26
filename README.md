# ai-golden-path

The template every AI service in this organisation starts from. Evaluation,
tracing and guardrail hooks are wired in **before** any business logic exists,
so a new service inherits them instead of retrofitting them.

It contains no AI. That is deliberate: a retrieval service, an agent runtime and
a portfolio app all need the same plumbing, and none of them need each other's
domain code.

## Quick start

```bash
uv sync --extra dev
cp .env.example .env
uv run uvicorn app.main:app --reload
```

```bash
curl -i localhost:8000/health
curl -s localhost:8000/ready
```

Or in a container:

```bash
docker compose up --build
curl -i localhost:8000/health
```

`uv run pytest` runs the suite with a 70% coverage gate.
`uv run ruff check src tests` lints.

## What you get

| | |
|---|---|
| **Config** | Typed, frozen, environment-sourced. Unknown `APP_*` vars fail startup. `prod` rejects DEBUG logs and console formatting. |
| **Logging** | One JSON object per line to stdout, with service identity and a correlation id on every record. uvicorn's loggers are folded in. |
| **Correlation ids** | Accepted from `x-correlation-id` or `x-request-id`, generated when absent, echoed on the response, stamped onto every log record. |
| **Probes** | `/health` (liveness, checks nothing) and `/ready` (readiness, checks registered dependencies). |
| **API versioning** | `/v1` router present and empty from day one. |
| **Tests** | pytest with an app factory fixture and an enforced coverage floor. |
| **ADRs** | `docs/adr/` with a template and the five decisions above. |

## Building a service from this

1. Use this template (GitHub → *Use this template*).
2. Add routers under `src/app/api/v1/`.
3. Register a readiness check for each dependency you cannot serve without:

```python
from app.api.health import register_readiness_check

async def check_vector_store() -> None:
    await client.ping()

register_readiness_check("vector_store", check_vector_store)
```

4. Write an ADR for anything you decide that the next person would otherwise
   have to guess at.

## What is deliberately not here

- **Any AI code.** Retrieval, agents, prompts and model clients belong to the
  services, not the template.
- **Evaluation metrics.** The harness interface arrives in session 5; the
  metrics are pattern-specific — groundedness for retrieval, task success for
  agents — so the template ships the interface and no implementations.
- **Guardrail implementations.** The middleware slots arrive in session 4.
  PII redaction and injection detection are per-service policy.

## Roadmap

| Session | Adds |
|---|---|
| 1 ✅ | Config, logging, correlation ids, probes, app factory, tests |
| 2 ✅ | Multi-stage Dockerfile, compose, non-root, read-only runtime |
| 3 | CI pipeline with lint, type check, coverage gate, security scan |
| 4 | OpenTelemetry, error handler, guardrail middleware slots |
| 5 | Evaluation harness interface + fixtures |


