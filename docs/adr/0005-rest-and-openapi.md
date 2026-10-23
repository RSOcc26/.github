# 5. REST/JSON with OpenAPI 3.1 for synchronous calls

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

Services need synchronous calls for questions and commands that need an
answer. The frontend needs the same APIs through the gateway. RSO rewards API
documentation.

## Decision

- Synchronous communication is REST with JSON over HTTP.
- Each service owns an **OpenAPI 3.1** document in `src/api/openapi.yaml`,
  served at `/api-docs`. It is changed before or together with the
  implementation (contract first).
- Paths are versioned (`/v1/...`). Breaking changes get a new version, not an
  in-place edit.
- Errors use `application/problem+json` (RFC 9457).
- Every service exposes `/health/live`, `/health/ready` and `/metrics`.
- gRPC and GraphQL are not used in the MVP.

## Consequences

- One contract format for clients, the gateway and documentation.
- Contract linting can run in CI once each service has its contract.
