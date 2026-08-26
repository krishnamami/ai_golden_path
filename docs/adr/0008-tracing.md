# ADR-0008: Vendor-neutral tracing, and two ids rather than one

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
An AI service spends most of its wall-clock time inside calls it does not
control -- retrieval, reranking, tool calls, model inference. Logs alone cannot
show where the time or the money went, because they are points and this is a
tree. And when a user reports a problem they can quote something from their
screen, which is rarely a trace id.

## Decision
OpenTelemetry, exported over OTLP, with **no vendor named anywhere in the code**.
The collector endpoint is configuration.

We carry **two** identifiers and put both on every log line:

| | Purpose |
|---|---|
| `correlation_id` | Ours. Accepted from the caller, echoed in responses, included in every error body, quoted in support tickets. |
| `trace_id` | OpenTelemetry's. Spans the whole call graph and is what the backend indexes. |

Also:

- **Tracing is a no-op when no endpoint is configured.** Instrumentation is
  still installed, so spans exist and trace ids still reach the logs; they are
  simply not exported.
- **`ParentBased(TraceIdRatioBased(ratio))`.** If an upstream already decided to
  sample a trace, that decision is honoured.
- **`/health` and `/ready` are excluded.** Probe traffic is high-volume and
  uninteresting, and at any sampling ratio below 1.0 it would crowd out the
  requests worth keeping.

## Alternatives considered
- **A vendor SDK.** Faster to start, and switching later means touching every
  service. OTLP makes it a config change.
- **Only a trace id.** Nothing a user can quote. Asking someone to read a
  32-character hex string off a screen is not a support process.
- **Only a correlation id.** No call graph, no timing breakdown, and no way to
  follow a request into a downstream service.
- **Sampling per hop.** Produces traces with holes in the middle, which is worse
  than no trace: it looks complete and is not.
- **Tracing that fails hard without a collector.** A template that crashes on a
  laptop gets deleted rather than fixed.

## Consequences
Every log line carries `correlation_id`, `trace_id` and `span_id`. A support
ticket quoting a correlation id finds a log line; that line carries the trace id;
the trace shows the call graph. When services add model and retrieval spans, the
same mechanism gives per-request cost attribution for free, which is what makes
the cost-per-use-case number in the roadmap possible.
