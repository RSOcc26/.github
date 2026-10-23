# 1. Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

CompanyCentral is built by two people across many repositories. Decisions made
in chat or in pull request comments get lost, and the RSO assessment asks for
technical documentation that explains the design.

## Decision

We record significant architectural decisions as numbered Markdown files:
cross-cutting ones in `RSOcc26/.github/docs/adr/`, repository-specific ones in
that repository's `docs/adr/`. A decision is significant when it is hard to
reverse, affects more than one service, or a reviewer would ask "why?".

## Consequences

- Every significant pull request either links an existing ADR or adds one.
- Changing an accepted decision means a new ADR that supersedes the old one.
