# 2. Identity provider: custom or external OIDC

- **Status:** Proposed (open decision)
- **Date:** 2026-10-23
- **Decide by:** before authentication is implemented in `service-identity` and `gateway`

## Context

Users sign in once and then work in one or more companies. The RSO course
awards extra points for OAuth2/OIDC-based IAM. Exactly one authentication
approach will be built.

## Options

**A. Custom identity in `service-identity`.** Identity stores password hashes
(Argon2id), manages sessions and issues signed tokens (JWT, keys as JWKS). The
gateway verifies them.

- Full control over membership and invitations in one place; no extra infrastructure.
- We own security-critical code: password storage, reset, lockout, key rotation.
- Weaker claim to the OAuth2/OIDC IAM criterion.

**B. External OIDC provider (for example Keycloak).** The provider handles
login, passwords, MFA and tokens; the gateway validates OIDC tokens.
`service-identity` keeps companies, memberships and invitations and maps the
OIDC subject (`sub`) to users.

- Standard flows, MFA and account management; clearly meets the IAM criterion.
- One more stateful component (Keycloak needs its own database).
- User data is split between the provider and Identity.

## Decision

Not decided. Until it is:

- Users are identified by a stable user ID plus an external subject reference,
  which works with both options.
- The gateway's authentication step sits behind one interface ("validate the
  request and return the user ID"); membership and tenant checks do not depend
  on the option.
- No password handling and no Keycloak deployment are implemented.

## Consequences

The chosen option is recorded by setting this record to *Accepted* before
authentication is implemented.
