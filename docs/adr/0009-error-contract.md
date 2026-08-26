# ADR-0009: RFC 9457 problem+json, and no exception text in production

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
Errors are an API surface. Left to itself, a service returns FastAPI's
`{"detail": ...}` for some failures, a validation array for others, and an HTML
stack trace for the rest -- three shapes, so every client writes three handlers.

Separately, exception text is a security problem. `RuntimeError: could not
connect to postgres://svc:hunter2@db-prod-3.internal:5432` is a credential, a
hostname and a topology hint, rendered helpfully in the browser.

## Decision
Every error returns **RFC 9457 Problem Details** as `application/problem+json`,
always including the `correlation_id`.

- **Deliberate `HTTPException`s** keep their detail: our code chose that message.
- **Validation errors** return full field-level detail: the caller sent the data,
  so naming the bad field tells them nothing they did not already know, and it is
  the difference between a usable API and a frustrating one.
- **Unhandled exceptions** are logged in full with a stack trace, and return a
  generic title with the correlation id in production. Outside production the
  exception text is returned, because a developer staring at a laptop is not a
  threat model.
- **`GuardrailBlockedError` returns 400, not 500**, and names the guardrail but
  not the reason. The request was understood and deliberately refused, so 4xx is
  correct; the guardrail name lets a caller distinguish "too long" from "policy
  violation" without a support ticket, while a detailed rejection message would
  be a free oracle for probing the filter.

## Alternatives considered
- **FastAPI's default shapes.** Three shapes, no correlation id, and stack traces
  in the response by default.
- **A bespoke error envelope.** Works, and RFC 9457 is already a standard with
  library support on the client side.
- **Returning exception text everywhere.** Convenient during an incident, and it
  is exactly how connection strings end up in a screenshot in a chat channel.
- **Returning the guardrail's reason.** Helpful to a legitimate caller, and an
  interactive tutorial in evading the guardrail for everyone else.

## Consequences
One error shape, machine-countable by `type`, always carrying the id that finds
the trace. Adding an error type means adding a handler, not a new shape. The
production-leakage test is deliberately explicit -- it asserts that a fake
password does **not** appear in the response body, so a future refactor that
reintroduces the leak fails a test rather than an audit.
