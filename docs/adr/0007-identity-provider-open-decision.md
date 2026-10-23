# 7. Custom identity management or an external OIDC provider

- **Status:** Proposed (open decision)
- **Date:** 2026-10-23
- **Decide by:** before authentication is implemented in `service-identity` and `gateway`

## Context

Users must sign in once and then work in one or more companies. RSO awards
extra points for OAuth2/OIDC-based IAM. Two directions are possible, and we
must not build both.

**Option A: custom identity in `service-identity`.**
Identity stores password hashes (Argon2id), manages sessions and issues signed
tokens (JWT, keys published as JWKS). The gateway verifies them.

- Full control over company membership and invitations, in one place.
- No additional infrastructure.
- We own the security-critical code: password storage, reset, lockout, key rotation.
- Weaker claim to the "OAuth2/OIDC IAM" requirement.

**Option B: external OIDC provider (for example Keycloak).**
Keycloak (or a hosted provider) handles login, passwords, MFA and token
issuance. The gateway validates OIDC tokens. `service-identity` keeps the
CompanyCentral domain (companies, memberships, invitations) and maps the
OIDC subject (`sub`) to users.

- Standard OAuth2/OIDC flows, MFA and account management for free.
- Clearly satisfies the IAM requirement.
- One more stateful component to deploy, configure and back up (Keycloak needs
  its own database).
- User data is split between the provider (credentials, profile) and Identity
  (memberships).

## Decision

**Not decided yet.** Until it is:

- `service-identity` is designed so that "who is this user" is a stable user ID
  plus an external subject reference. Both options can use that.
- The gateway's authentication step sits behind one interface ("validate the
  token and return the user ID"). Membership and tenant validation do not
  depend on the option.
- No password handling and no Keycloak deployment are implemented.

## Consequences

- Whichever option is chosen, the decision is recorded by changing this ADR's
  status to Accepted (or adding a superseding ADR) before implementation starts.
