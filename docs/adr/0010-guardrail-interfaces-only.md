# ADR-0010: The template defines guardrail interfaces and ships no guardrails

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
Every service built from this template needs guardrails, and no two need the
same ones. A credit decisioning service and an internal documentation assistant
disagree about what unacceptable input is, what may be logged, and what a
refusal should say.

## Decision
Ship the contract -- `Guardrail`, `GuardrailResult`, `GuardrailChain` -- and no
implementations.

**Where guardrails attach is the substantive decision here:**

- **Request guardrails** run at the HTTP boundary, in middleware. Size limits,
  content-type checks, obvious abuse. Cheap and generic.
- **Model guardrails** run at the *model call* boundary, not the HTTP one.
  Prompt-injection detection, PII redaction, output classification. Services
  import `Guardrail` and apply it around their own model calls.

Filtering an HTTP response body is the wrong abstraction. By the time bytes
reach the response, you have lost which part came from a model, which came from
retrieval, and which the service computed itself -- and those need different
treatment. A single output filter over the whole body either misses things or
blocks legitimate content.

Two smaller choices:

- **`GuardrailResult` supports redaction, not just blocking.** A redacted
  response is still a response, and is usually the better outcome.
- **Every result is returned, not just the blocking one.** Guardrail hit rates
  are an evaluation signal, not only an operational one — which guardrails fire
  and how often is an input to the eval suite in session 5.

## Alternatives considered
- **Shipping default guardrails.** They would be wrong for most services, and
  wrong defaults are worse than none because people assume they are covered.
- **A single generic output filter at the HTTP layer.** Simpler and, per above,
  the wrong boundary.
- **Blocking only, no redaction.** Turns a recoverable situation into a failed
  request.

## Consequences
A service must implement its own guardrails; the template only guarantees they
compose the same way everywhere. **Test guardrails for false positives as
carefully as for false negatives** -- a guardrail that blocks legitimate work
gets switched off, and a switched-off guardrail protects nothing.
