# 8. Tenant context is validated by the gateway, never trusted from clients

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

A user may belong to several companies and switches between them in the Hub.
The browser therefore tells us which company it is working in. Anything the
browser sends can be forged.

## Decision

1. The client may *request* a tenant (company switcher → `X-Tenant-Id` header
   or route parameter). It is a request, not a fact.
2. The gateway authenticates the user, then asks Identity (cached for a short
   time and invalidated by membership events) whether the user is an **active
   member** of that tenant. If not, the request fails with `403`.
3. The gateway **removes** any client-supplied internal identity or tenant
   headers and forwards a **signed, short-lived tenant context** (user ID,
   tenant ID, roles, correlation ID) to the service.
4. Services verify the signature, audience and expiry, and take the tenant only
   from that context. They never read a raw tenant header.
5. Service-to-service calls forward the same context. Event consumers use the
   `tenantId` in the event envelope, which only trusted producers can publish.

## Consequences

- Isolation does not depend on the frontend behaving correctly.
- Services stay simple: one middleware validates the context.
- The signing key for the tenant context is a Secret shared between the gateway
  and the services' verification config (or a JWKS endpoint on the gateway).
