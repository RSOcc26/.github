# Tenant context

The gateway authenticates the browser's Keycloak token and then replaces it
with its own short-lived JWT, the **tenant context**, for every call it
forwards (ADR-0001, ADR-0003). Services accept only this token; they never
see Keycloak tokens. The machine-readable contract is
[`tenant-context.schema.json`](tenant-context.schema.json) (JSON Schema,
draft-07): `definitions/header` and `definitions/payload`.

## Signing and verification

| Property | Value |
|---|---|
| Algorithm | ES256 (P-256), header `typ: tenant-context+jwt`, `kid` = JWK thumbprint |
| Keys | the gateway's JWKS at `/.well-known/jwks.json` (cluster-internal) |
| Issuer | `TENANT_CONTEXT_ISSUER` (`companycentral-gateway`) |
| Audience | exactly one service: `service-<name>`; a token for one service is rejected by every other |
| Lifetime | `TENANT_CONTEXT_TTL_SECONDS` (60 s); every token has a `jti` |

A service **must** check the signature against the JWKS, `typ`, `alg`, `iss`,
its own `aud` and `exp`, and reject anything else with `401`.

## Two kinds of context

| Kind | When | Identifying claims |
|---|---|---|
| User context | a call on behalf of a signed-in user | `sub` (CompanyCentral user ID; absent only before the first `/me`), `roles`, `idp_iss`, `idp_sub`, `idp_iat`, profile claims; later `tid` |
| Service context | the gateway's own calls (user lookup) | `svc: gateway`, no user claims |

A service tells them apart by `svc`. Routes for users must reject a service
context, and gateway-only routes must require one.

## Rules for services

- Take the user only from `sub` and the tenant only from `tid` (once present);
  never from headers or the request body.
- `roles` are platform roles; company roles will come from Access.
- `rid` is the correlation ID: log it and copy it into events as `correlationId`.
- Unknown claims are a contract change: the schema does not allow them, so the
  gateway and every service's tests must be updated together.

## Keeping copies in sync

Each repository keeps a byte-identical copy of the schema in
`test/contracts/` for its tests (repositories are built separately). Their
tests compare the copy with this one. In CI the shared workflow clones this
repository's `main` next to the repository under test (input
`contract-repositories`) and the comparison must pass; locally it runs when a
`.github` checkout sits next to the repository.

So a change lands in this order: merge the change here first, then update the
copies (and the code) in each repository. Until a repository's copy is
updated, its CI fails, which is the point.
