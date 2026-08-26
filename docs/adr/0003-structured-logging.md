# ADR-0003: JSON logs to stdout, with correlation ids stamped at record creation

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
Logs need to be machine-parseable in deployed environments and readable on a
laptop, and a single request must be reconstructable from an aggregator.

## Decision
One JSON object per line to stdout (twelve-factor XI). A `ContextVar` holds the
correlation id, and a **`logging.setLogRecordFactory` hook stamps it onto every
record as the record is created**. uvicorn's own loggers have their handlers
cleared so they propagate to root, and its access log is silenced in favour of
ours, which carries timing and the correlation id.

## Alternatives considered
- **Reading the ContextVar in the formatter.** This was the first
  implementation and it is subtly wrong. A formatter runs when a *handler*
  processes the record; under a `QueueHandler` -- the standard way to keep
  logging off the request path -- that happens on another thread after the
  request has completed, and every line records `-`. We shipped this bug, and a
  test that only checked the response header did not catch it.
- **A `logging.Filter` on the handler.** Correct for our own handler, but filters
  are per-handler, so anything else attached to root (pytest's `caplog`, an APM
  agent) would miss the field.
- **structlog.** Excellent, and one more dependency in a template whose whole
  job is to be understood by whoever copies it.
- **Writing to a file.** Requires rotation, mounts, and different config per
  environment. The platform already solves this for stdout.

## Consequences
Every log line carries service, environment, version and correlation id, and
`extra=` fields are promoted to top-level JSON keys. Formatters must read
`record.correlation_id`, never the ContextVar. Request bodies and headers are
never logged here -- redaction is a guardrail concern (session 4), so that it
can be tested in isolation.
