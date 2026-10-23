# 10. Node.js LTS backends and one React frontend

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

Two developers maintain seven deployable units. One language across the
backend reduces context switching and lets us share conventions and small
libraries.

## Decision

- Backends (services and gateway) use **Node.js LTS**, the major version pinned
  in each repository's `.nvmrc` (currently 24).
- The frontend is **one React application** (`web`) with the Hub shell and one
  UI module per business application, built to static files. No runtime
  microfrontends or module federation in the MVP.
- Applications are described by a small manifest (ID, name, version, entry
  route, required permissions, dependencies) defined in `web/docs/app-manifest.md`.

## Consequences

- One CI path (`npm ci`, `lint`, `test`, `build`) for all repositories.
- A heavy business computation later could still become a separate service in
  another language, behind the same REST and event contracts.
