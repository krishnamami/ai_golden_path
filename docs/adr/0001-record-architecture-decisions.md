# ADR-0001: Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
Services built from this template will be maintained by people who were not in
the room. Without a record, every past decision looks arbitrary and gets either
cargo-culted or reversed by accident.

## Decision
Every significant decision is recorded as a numbered ADR in `docs/adr/`, using
`TEMPLATE.md`. ADRs are immutable once accepted: a changed decision gets a new
ADR that supersedes the old one, and the old one stays in the repository.

## Alternatives considered
- **A wiki.** Drifts from the code, and nobody reads it during a code review.
- **Comments in code.** Good for *how*, useless for *what we rejected and why*.
- **Nothing.** The default, and the reason most teams re-litigate the same
  argument annually.

## Consequences
Onboarding gets faster and design reviews get shorter. The cost is a few minutes
per decision, and the discipline to write one when it is inconvenient.
